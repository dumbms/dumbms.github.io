# 6.828 Lab 2: Memory Management


[实验地址 Lab 2: Memory Management ](https://pdos.csail.mit.edu/6.828/2018/labs/lab2/)



## 0 Introduction

在这个lab中，将实现操作系统的内存管理部分：

**第一个部分**是实现一个内核的物理内存分配器，这样内核就可以分配内存，然后释放内存。该分配器将以4096字节(称为页)为单位进行操作。实现这个物理内存分配器需要维护一个数据结构，以记录哪些物理页是空闲的，哪些页是分配的，以及共享每个分配页的进程数量。并且需要编写分配和释放内存页的相关代码。

内存管理的**第二个部分**是虚拟内存，它将内核和用户使用的虚拟地址映射到物理内存中的地址。当指令使用到内存时，x86硬件的内存管理单元(MMU)通过查询一组页表来执行映射。

---



## 1 Physical Page Management

操作系统必须`keep track of`物理 RAM 的哪些部分是空闲的，哪些部分当前正在使用。JOS 通过页面粒度管理 PC 的物理内存，这样它就可以使用 MMU 来映射和保护分配的每块内存。

<div align=center>{{< image src="assets/JOS.png" caption="JOS初始化内存分布" >}}</div>

#### Exercise 1

在`kern/pmap.c`, 需要实现以下几个函数

```c
boot_alloc()
mem_init() (only up to the call to check_page_free_list(1))
page_init()
page_alloc()
page_free()
```

##### boot_alloc

该函数实现内核初始化的过程中，物理内存的分配。

`end`变量是定义在`kernel.ld`文件里面的，指向了内核地址的末位。

**注意**这里要到的地址是虚拟地址，不是物理地址。

```c
static void *
boot_alloc(uint32_t n)
{
	static char *nextfree;	// virtual address of next byte of free memory
	char *result;

	// Initialize nextfree if this is the first time.
	// 'end' is a magic symbol automatically generated by the linker,
	// which points to the end of the kernel's bss segment:
	// the first virtual address that the linker did *not* assign
	// to any kernel code or global variables.
	if (!nextfree) {
		extern char end[];
		nextfree = ROUNDUP((char *) end, PGSIZE);
	}

	// Allocate a chunk large enough to hold 'n' bytes, then update
	// nextfree.  Make sure nextfree is kept aligned
	// to a multiple of PGSIZE.
	//
// LAB 2: Your code here.
	if (n == 0) {
		return nextfree;
	} 
	result = nextfree;
	nextfree = ROUNDUP(nextfree + n, PGSIZE);	

	if ((uint32_t)(nextfree + n - KERNBASE) > npages * PGSIZE) {
		panic("out of memory !");
	}
	return result;
}
```

之前做这个exerciese的时候有个疑惑🤔：boot_alloc 代码实现是分配了内核end 结束后的虚拟地址，但要求分配的确是物理地址。

答案是因为，在进入内核的时候已经将CR3设置为手写页表的地址可参考LAB1 的总结，这个页表中存放着[0, 4MB] 和 [KERNBASE, KERNBASE+4MB) 到物理地址的 [0, 4MB) 的映射，这样初始化内核分配物理内存的时候，就可以直接使用虚拟地址来分配，mmu硬件在背后会帮我们自动转为物理地址，cool！也正因为有了mmu硬件这种功能，c语言操作的地址均为虚拟地址。

---

##### mem_init

<div align=center>{{< image src="assets/物理地址分布.jpg" caption="JOS初始化物理内存分布" >}}</div>

在物理内存地址，分配了一个 pages 数组，该数组的元素为struct PageInfo，抽象出pages数组来描述物理内存的使用情况，PageInfo 中有两个变量，一个是指向下一个空闲页面的指针，一个是表示该物理地址页面被不同的虚拟地址映射的数量(写时复制)。

Is it NOT the physical page itself, but there is a one-to-one correspondence between physical pages and struct PageInfo's.

```c
struct PageInfo {
	// Next page on the free list.
	struct PageInfo *pp_link;

	// pp_ref is the count of pointers (usually in page table entries)
	// to this page, for pages allocated using page_alloc.
	// Pages allocated at boot time using pmap.c's
	// boot_alloc do not have valid reference count fields.

	uint16_t pp_ref;
};
```

npages 是在 i386_detect_memory 中 检测出真实物理页的数量。

```c
void
mem_init(void)
{	
	...
	// Permissions: kernel R, user R
	kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;

	//////////////////////////////////////////////////////////////////////
	// Allocate an array of npages 'struct PageInfo's and store it in 'pages'.
	// The kernel uses this array to keep track of physical pages: for
	// each physical page, there is a corresponding struct PageInfo in this
	// array.  'npages' is the number of physical pages in memory.  Use memset
	// to initialize all fields of each struct PageInfo to 0.
// Your code goes here:
	pages = (struct PageInfo *) boot_alloc(npages*sizeof(struct PageInfo)); 
// 这里memset 使用的pages是虚拟地址，猜猜看为什么可以使用虚拟地址来初始化内存😀
	memset(pages, 0, npages*sizeof(struct PageInfo));
	...
}
```

---

##### page_init

该函数来通过pages 的每个PageInfo与真实的物理地址制造联系，从而来追踪真实物理页的使用情况。

在pmap.h有两个重要的宏PADDR、KADDR需要了解以下，简单来说

PADDR：将高位虚拟地址转为低位的物理地址。

KADDR：将低位物理地址转为高位的虚拟地址。

```c
/* This macro takes a kernel virtual address -- an address that points above
 * KERNBASE, where the machine's maximum 256MB of physical memory is mapped --
 * and returns the corresponding physical address.  It panics if you pass it a
 * non-kernel virtual address.
 */
#define PADDR(kva) _paddr(__FILE__, __LINE__, kva)

static inline physaddr_t
_paddr(const char *file, int line, void *kva)
{
	if ((uint32_t)kva < KERNBASE)
		_panic(file, line, "PADDR called with invalid kva %08lx", kva);
	return (physaddr_t)kva - KERNBASE;
}

/* This macro takes a physical address and returns the corresponding kernel
 * virtual address.  It panics if you pass an invalid physical address. */
#define KADDR(pa) _kaddr(__FILE__, __LINE__, pa)

static inline void*
_kaddr(const char *file, int line, physaddr_t pa)
{
	if (PGNUM(pa) >= npages)
		_panic(file, line, "KADDR called with invalid pa %08lx", pa);
	return (void *)(pa + KERNBASE);
}
```

根据给定的提示很容易模仿写出相应的代码。

```c
void
page_init(void)
{
	...
	size_t i;
	for (i = 0; i < npages; i++) {
		if (i == 0) {
			pages[i].pp_ref = 1;
			pages[i].pp_link = NULL;
		}
		else if (i >= 1 && i < npages_basemem) {
			pages[i].pp_ref = 0;
			pages[i].pp_link = page_free_list;
			page_free_list = &pages[i];
		}
		else if (i >= (IOPHYSMEM / PGSIZE) && i < (EXTPHYSMEM / PGSIZE)) {
			pages[i].pp_ref = 1;
			pages[i].pp_link = NULL;
		}
// 将[EXTPHYSMEM, 内核当前最后一页的地址）标记为已被引用，防止触碰内核代码。
		else if (i >= (EXTPHYSMEM / PGSIZE) && i < (PADDR(boot_alloc(0)) / PGSIZE)) {
			pages[i].pp_ref = 1;
		}
// 这里要拿到内核第一个未使用的地址，使用boot_alloc(0)很巧妙的拿到。 
		else if (i >= (PADDR(boot_alloc(0)) / PGSIZE) && i < npages){
			pages[i].pp_ref = 0;
			pages[i].pp_link = page_free_list;
			page_free_list = &pages[i];
		}	
	}
}
```

---

##### page_alloc

这个是真实的最底层的物理分配器。

通过查看page_free_list 物理页空闲页表来看是否有空闲也，有的话就从链表中取出一个就行。通过 alloc_flags 来判断是否需要初始化内存。

```c
// Allocates a physical page.  If (alloc_flags & ALLOC_ZERO), fills the entire
// returned physical page with '\0' bytes.  Does NOT increment the reference
// count of the page - the caller must do these if necessary (either explicitly
// or via page_insert).
//
// Be sure to set the pp_link field of the allocated page to NULL so
// page_free can check for double-free bugs.
//
// Returns NULL if out of free memory.
//
// Hint: use page2kva and memset
struct PageInfo *
page_alloc(int alloc_flags)
{
// Fill this function in
	struct PageInfo *p;
	if (!page_free_list) 
		return NULL;
	p = page_free_list;
	page_free_list = page_free_list->pp_link;
	p->pp_link = NULL;

	if (alloc_flags & ALLOC_ZERO) {
// page2kva将 pages数组中的虚拟地址p转为 物理页 + KERNBASE 对应的虚拟地址，
// 然后直接通过虚拟地址初始化这块物理页面的值，mmu硬件在这个过程中做了自动翻译。
		memset(page2kva(p), 0, PGSIZE);
	}
	return p;
}
```

##### page_free

该函数做的是释放页面。

pp_ref 不为0说明该页面正在被某虚拟地址映射，pp_link不为NULL，说明该页面已不再空闲链表中，也就是说正在被使用。

```c
// Return a page to the free list.
// (This function should only be called when pp->pp_ref reaches 0.)
void
page_free(struct PageInfo *pp)
{
	// Fill this function in
	// Hint: You may want to panic if pp->pp_ref is nonzero or
	// pp->pp_link is not NULL.
	if (pp->pp_ref || pp->pp_link != NULL) {
		panic("This page has other mappings in use!");
	}
	pp->pp_link = page_free_list;
	page_free_list = pp;
}
```

可以通过两个test。

```c
check_page_free_list() succeeded!
check_page_alloc() succeeded!
kernel panic at kern/pmap.c:723: assertion failed: page_insert(kern_pgdir, pp1, 0x0, PTE_W) < 0
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 
```

写了那么多make grade 发现只得了20分。。



## 2 Virtual Memory

#### Exercise 2

阅读手册[80386 Programmer's Reference Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)第五章、第六章，了解页面翻译和基于页面保护得内容。

#### Exercise 3

熟悉qemu操作。

#### Exercise 4

##### pgdir_walk

[80386 Programmer's Reference Manual -- Section 5.2](https://pdos.csail.mit.edu/6.828/2018/readings/i386/s05_02.htm)

该函数为 mmu 硬件的提供服务，只有页表中存放有虚拟地址到物理地址的翻译，mmu才能工作。按照提示大概可以得到以下思路：

1. 该函数的作用是将给定的虚拟地址和指向页目录pgdir的指针，返回一个指向页表项（PTE page table entry）的指针。PTE 与 PDE 表示是一样的，只是在顶级目录与二级页表区分开。

2. JOS 是二级页表，参考下图。CR3指针指向页目录项，然后将给定的虚拟地址va分成三个部分，取出DIR这个部分当成索引，在页目录中找到对应页目录项，其实就是一个PTE。如何取出DIR呢，mmu.h中提供了相关的宏。

   <div align=center>{{< image src="https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-9.gif" caption="JOS Page Translation" width="50%">}}</div>

   ```c
   // A linear address 'la' has a three-part structure as follows:
   //
   // +--------10------+-------10-------+---------12----------+
   // | Page Directory |   Page Table   | Offset within Page  |
   // |      Index     |      Index     |                     |
   // +----------------+----------------+---------------------+
   //  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
   //  \---------- PGNUM(la) ----------/
   //
   // The PDX, PTX, PGOFF, and PGNUM macros decompose linear addresses as shown.
   // To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
   // use PGADDR(PDX(la), PTX(la), PGOFF(la)).
   
   // page number field of address
   #define PGNUM(la)	(((uintptr_t) (la)) >> PTXSHIFT)
   
   // page directory index 
   // 做过csapp第一章的datalab后，对这种二进制表示的运算不会感到很陌生。 
   // 这里做的就是将给定的va右移22位，得到高10位，并& 0x3FF,即将高22位取0，低10位保留1。
   // 得到纯净的高10位。
   #define PDX(la)		((((uintptr_t) (la)) >> PDXSHIFT) & 0x3FF)
   
   // page table index
   #define PTX(la)		((((uintptr_t) (la)) >> PTXSHIFT) & 0x3FF)
   
   // offset in page
   #define PGOFF(la)	(((uintptr_t) (la)) & 0xFFF)
   
   ...
   #define PTXSHIFT	12		// offset of PTX in a linear address
   #define PDXSHIFT	22		// offset of PDX in a linear address
   ```

   

3. 下图是页表项的具体细节，一共32位，低12位为标识符。从页目录中找到对应的PTE后，通过PTE中的最低位（PTE_P）判断该页目录项是否有效，如果有效，取出高20位，将其转为虚拟地址指针，返回即得到所需答案。

   注意：PTE中高23位存放的是真实的物理地址，我们需要使用的是虚拟地址。

   <div align=center>{{< image src="https://pdos.csail.mit.edu/6.828/2018/readings/i386/fig5-10.gif" caption="Format of a Page Table Entry" width="50%">}}</div>

4. 如果2中在页目录表中找到的PTE的最低位P为0，即表示该PTE无效，这时候就需要新分配一个页面，使用之前写过的函数page_alloc分配一个页面，该函数返回的是Struct PageInfo，在pmap.h中有个函数page2pa可以将pageinfo转为物理地址，因为上面讲了PTE中存放的是真实的物理地址（为什么是这么做的呢，可以想想mmu翻译的逻辑），将物理地址放到2中找到的PTE高23位中，并给它一些标志位权限。

5. 有了上面思路，就可以写出相应代码：

```c
// Given 'pgdir', a pointer to a page directory, pgdir_walk returns
// a pointer to the page table entry (PTE) for linear address 'va'.
// This requires walking the two-level page table structure.
//
// The relevant page table page might not exist yet.
// If this is true, and create == false, then pgdir_walk returns NULL.
// Otherwise, pgdir_walk allocates a new page table page with page_alloc.
//    - If the allocation fails, pgdir_walk returns NULL.
//    - Otherwise, the new page's reference count is incremented,
//	the page is cleared,
//	and pgdir_walk returns a pointer into the new page table page.
//
// Hint 1: you can turn a PageInfo * into the physical address of the
// page it refers to with page2pa() from kern/pmap.h.
//
// Hint 2: the x86 MMU checks permission bits in both the page directory
// and the page table, so it's safe to leave permissions in the page
// directory more permissive than strictly necessary.
//
// Hint 3: look at inc/mmu.h for useful macros that manipulate page
// table and page directory entries.
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
	// Fill this function in
	pde_t *pde;
	pte_t *pte;
	struct PageInfo * p;
	pde = &pgdir[PDX(va)];
	if (!(*pde & PTE_P)) {
		if (!create || (p = page_alloc(1)) ==  NULL) {
				return NULL;
		}
		p->pp_ref++;
		pte = (pte_t *)page2kva(p);
		*pde = page2pa(p) | PTE_P |PTE_U |PTE_W;
	} else {
		pte = KADDR(PTE_ADDR(*pde));
	}
	return &pte[PTX(va)];
}
```

pgdir_walk 并没有写入映射，只是根据虚拟地址va来找到对应的PTE。

##### boot_map_region

该函数是在页表中填映射的部分，从pgdir_walk中获得虚拟地址va对应的PTE，然后把物理地址填到这里面，并给perm权限就行。

```c
// Map [va, va+size) of virtual address space to physical [pa, pa+size)
// in the page table rooted at pgdir.  Size is a multiple of PGSIZE, and
// va and pa are both page-aligned.
// Use permission bits perm|PTE_P for the entries.
//
// This function is only intended to set up the ``static'' mappings
// above UTOP. As such, it should *not* change the pp_ref field on the
// mapped pages.
//
// Hint: the TA solution uses pgdir_walk
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	// Fill this function in
	pte_t * pte;
// 虽然提示说va 和 pa 都是页对齐的，但还是再对齐一下保险，怕后面自己用的时候往手动对齐。
	uintptr_t _va = ROUNDDOWN(va, PGSIZE);
	physaddr_t _pa = ROUNDDOWN(pa, PGSIZE);
	int n = size / PGSIZE;
	for (int i = 0; i < n; ++i) {
		if ((pte = pgdir_walk(pgdir, (void *)_va, 1)) == NULL) {
			return;
		}
		*pte = PTE_ADDR(_pa) | perm | PTE_P;
		_va += PGSIZE;
		_pa += PGSIZE;
	}
}
```

##### page_lookup

该函数的作用是，给定虚拟地址va，转换成对应的pageinfo，hint 给了一个pa2page 的辅助函数，然后把找到的PTE存进给定的pte_store里面。这个函数还是比较好实现的。

```c
// Return the page mapped at virtual address 'va'.
// If pte_store is not zero, then we store in it the address
// of the pte for this page.  This is used by page_remove and
// can be used to verify page permissions for syscall arguments,
// but should not be used by most callers.
//
// Return NULL if there is no page mapped at va.
//
// Hint: the TA solution uses pgdir_walk and pa2page.
//
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
	// Fill this function in
	pte_t *pte;
	if ((pte = pgdir_walk(pgdir, va, 0)) == NULL) {
		return NULL;
	}
// 做的lab 5 找了半天bug，下面这行。🤯
	if (!(*pte & PTE_P))
		return NULL;
	physaddr_t ph = PTE_ADDR(*pte);
	if (pte_store) {
		// 双指针解引用存指针
		*pte_store = pte;
	}
	return pa2page(ph);
}
```

这里有个在lab 5卡了很久的bug，pgdir_walk返回一个指向二级页表项PTE的指针，如果PTE的pte_p标志位为0，就说明这个PTE页表项里面存放的映射是无效的，就不能取其中的值存到pte_store给上层函数使用，一个无效的物理页可能会导致未知的错误，当然在上层函数也能提前先判断va对应的PTE是否有效，但上层函数使用的时候并没有想到判断。。。导致心态爆炸🤯🤯，卡在lab 5 shell 卡了快一周。

收获是对页表的工作原理的理解更加深入，简而言之，进程的虚拟地址空间的实现是靠页表，页表实现虚拟化的方式，就是在其页表项中填入物理地址和对应的位权限项，然后mmu硬件会通过CR3寄存器对应的页目录地址，不同的操作系统页表分级也有不同，JOS实现的是二级页表，xv6实现的是三级页表，原理都是一样的。物理页表之间的隔离也实现了进程之间的真实物理地址隔离性。

简单看看[MMU原理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/354142930)

##### page_remove

这个函数就更简单了，按照注释它的作用是取消映射并且将pageinfo的引用减少，上面讲了虚拟地址映射实际上是将物理页地址存在PTE页表项里，取消映射就直接将页表项至0就好了。

```c
// Unmaps the physical page at virtual address 'va'.
// If there is no physical page at that address, silently does nothing.
//
// Details:
//   - The ref count on the physical page should decrement.
//   - The physical page should be freed if the refcount reaches 0.
//   - The pg table entry corresponding to 'va' should be set to 0.
//     (if such a PTE exists)
//   - The TLB must be invalidated if you remove an entry from
//     the page table.
//
// Hint: The TA solution is implemented using page_lookup,
// 	tlb_invalidate, and page_decref.
//
void
page_remove(pde_t *pgdir, void *va)
{
	// Fill this function in
	struct PageInfo* p;
	pte_t *pte;
	if ((p = page_lookup(pgdir, va, &pte)) == NULL) {
		return;
	}
	page_decref(p);
	if (pte) {
		*pte = 0;
	}
	tlb_invalidate(pgdir, va);
}
```

##### page_insert

该函数的作用跟remove相反，将va映射到对应的pa上，增加pageinfo结构的引用，可以很简单地实现，但这个有个Corner-case hint的地方，讲的是如果同一个虚拟地址映射到同一个物理地址该怎么办：

1. 首先，判断va对应的PTE的pte_p位是否有效，有效的话说明已经映射过了；
2. 为了保持已经写好的代码的结构性，只需要将映射过的内容删了，再映射一遍就行了。

```c
// Map the physical page 'pp' at virtual address 'va'.
// The permissions (the low 12 bits) of the page table entry
// should be set to 'perm|PTE_P'.
//
// Requirements
//   - If there is already a page mapped at 'va', it should be page_remove()d.
//   - If necessary, on demand, a page table should be allocated and inserted
//     into 'pgdir'.
//   - pp->pp_ref should be incremented if the insertion succeeds.
//   - The TLB must be invalidated if a page was formerly present at 'va'.
//
// Corner-case hint: Make sure to consider what happens when the same
// pp is re-inserted at the same virtual address in the same pgdir.
// However, try not to distinguish this case in your code, as this
// frequently leads to subtle bugs; there's an elegant way to handle
// everything in one code path.
//
// RETURNS:
//   0 on success
//   -E_NO_MEM, if page table couldn't be allocated
//
// Hint: The TA solution is implemented using pgdir_walk, page_remove,
// and page2pa.
//
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
	// Fill this function in
	pte_t* pte;
	if ((pte = pgdir_walk(pgdir, va, 1)) == NULL) {
		return -E_NO_MEM; 	
	}
	pp->pp_ref++;
// 重复映射的话，只需要将之前的解除映射，然后再映射一遍
	if (*pte & PTE_P) {
		page_remove(pgdir, va);
	}
	*pte = page2pa(pp) | perm  | PTE_P; 
	return 0;
}
```

OK，目前得分40分。

```
  Physical page allocator: OK 
  Page management: OK 
  ...
  Score: 40/70
```

## 3 Kernel Address Space

JOS将内存分为了两部分，一部分是用户空间，运行在虚拟的低地址，另一部分是内核空间，运行在了高虚拟地址空间，分割线是ULIM，为了保护内核的代码和数据，需要在页中增加权限限制，使得用户空间的代码只能访问用户空间的内存，这样用户的代码产生的bug不会伤害到内核空间

在[UTOP, ULIM)的范围内，用户代码和内核代码都是可以访问的，但是都不可以写。这部分空间的设立主要是为了能让用户读取一些内核中的数据结构。

#### Exercise 5.

##### mem_init

这个就很简单了，意思就是让我们做映射，函数上面已经实现了，参考memlayout.h中的图

```
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 *
 * (*) Note: The kernel ensures that "Invalid Memory" is *never* mapped.
 *     "Empty Memory" is normally unmapped, but user programs may map pages
 *     there if desired.  JOS user programs map pages temporarily at UTEMP.
```

内核默认有读权限，PTE_W 表示内核有写权限。

用户要用PTE_U权限位表示有读权限，| PTE_W 表示有写权限。

```C
void
mem_init(void)
{
	//////////////////////////////////////////////////////////////////////
	// Map 'pages' read-only by the user at linear address UPAGES
	// Permissions:
	//    - the new image at UPAGES -- kernel R, user R
	//      (ie. perm = PTE_U | PTE_P)
	//    - pages itself -- kernel RW, user NONE
	// Your code goes here:
   	boot_map_region(kern_pgdir, 
                    UPAGES, 
                    PTSIZE,
                    PADDR(pages), 
                    (PTE_U));
               
	//////////////////////////////////////////////////////////////////////
	// Use the physical memory that 'bootstack' refers to as the kernel
	// stack.  The kernel stack grows down from virtual address KSTACKTOP.
	// We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
	// to be the kernel stack, but break this into two pieces:
	//     * [KSTACKTOP-KSTKSIZE, KSTACKTOP) -- backed by physical memory
	//     * [KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE) -- not backed; so if
	//       the kernel overflows its stack, it will fault rather than
	//       overwrite memory.  Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	// Your code goes here:
	boot_map_region(kern_pgdir, KSTACKTOP - KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
	
	//////////////////////////////////////////////////////////////////////
	// Map all of physical memory at KERNBASE.
	// Ie.  the VA range [KERNBASE, 2^32) should map to
	//      the PA range [0, 2^32 - KERNBASE)
	// We might not have 2^32 - KERNBASE bytes of physical memory, but
	// we just set up the mapping anyway.
	// Permissions: kernel RW, user NONE
	// Your code goes here:
	boot_map_region(kern_pgdir, KERNBASE, 0xffffffff- KERNBASE, 0, PTE_W);
}
```

OK，目前即可得满分：

<div align=center>{{< image src="assets/score.png" caption="score" >}}</div>

## 4 Challenge

目前没做。😅😅

可参考 [Pims的博客](https://phimos.github.io/2020/03/30/6828-lab2/)，这哥们写了两个Challenge。

----

{{<admonition tip "参考资料"  >}}

[xv6 中文文档 — MIT 6.828 (xv6-chinese.readthedocs.io)](https://xv6-chinese.readthedocs.io/zh/latest/index.html)

[JOS的物理页表分配实现](http://lzz5235.github.io/2014/03/04/jos.html)

{{< /admonition >}}


