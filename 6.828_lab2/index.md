# 6.828_lab2


# lab2

Virtual memory map

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
 */
```

Physical Memory

```
We will now dive into a bit more detail about how a PC starts up. A PC's physical address space is hard-wired to have the following general layout:
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |   64KB
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |    128KB
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

## Part1 Physical Page Management

### exercise1

补全在kern/pmap.c下的几个函数。

```c
boot_alloc()
mem_init() (only up to the call to check_page_free_list(1))
page_init()
page_alloc()
page_free()
```

首先我们要知道一点，在代码中，所有的变量的地址都是虚拟地址，JOS中虚拟地址到物理地址的转换很简单:

```c
虚拟地址 = 物理地址 + KERNBASE(0xF0000000)
```



在JOS中，一开始的物理内存布局如下图所示

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305051309959.webp)

由lab1我们可以得知，内核代码从0x100000处开始，也就是物理地址1MB的地方，第一条指令从0x10000c处开始，随后内核主要做如下几件事情

1. 开启分页(entry.S 62行)
2. 设置栈指针(entry.S 77行)
3. 调用i386_init(entry.S 80行)

I386_init()会调用mem_init()。

mem_init会先去调用i386_detect_memory();去找到这个机器有多少内存，也就是确定npages和npages_basemem。其中npages表示一共有多少物理内存页，而npages_basemem则表示物理内存中的Low Memory一共有多少物理内存页。我们知道，在4G的物理内存中，0x000A000(640KB)~0x00100000(1MB)存在一个IO洞，这个IO洞用于VGA和BIOS等，这在分配空闲页面时是绝对不能使用的

```c
// Find out how much memory the machine has (npages & npages_basemem).
	i386_detect_memory();
```

随后mem_init()调用boot_alloc分配一个PASIZE大小的页用于存放页表，将页表的起始地址存放在kern_pgdir中，本实验中地址是0xf0117000~0xf0118000，并通过memset初始化该页面

```c
kern_pgdir = (pde_t *)boot_alloc(PGSIZE);
memset(kern_pgdir, 0, PGSIZE);
```

随后给页表添加映射，从UVPT虚拟地址映射到kern_pgdir的物理地址，在UVPT处形成一个虚拟页表	

```c
	//递归地将PD作为一个页表插入自身，以形成
	// 在虚拟地址UVPT处形成一个虚拟页表。
kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
```

之后为PageInfo分配空间，用于管理物理页面。从0xf0118000开始即是通过boot_alloc()函数来分配npages个PageInfo结构体来管理物理页面。

```c
pages = (struct PageInfo *)boot_alloc(npages * sizeof(struct PageInfo));
memset(pages, 0, sizeof(struct PageInfo) * npages);
```

之后对数据结构初始化，详细见page_init()

```c
page_init();
```

#### boot_alloc()

boot_alloc()中维护了一个静态指针nextfree，初始值是在/kern/kernel.ld中定义的符号，指向bss段末尾。

```c
if (!nextfree)
	{
		extern char end[];
		nextfree = ROUNDUP((char *)end, PGSIZE);
		cprintf("kern code end ROUNDUP 4k: %x\n", nextfree);
	}
```

![image-20230505132452401](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305051324448.png)

可以看到0xf0117000是end（4K对齐）的末尾，随后的4K页面用于存储页表即0xf0117000~0xf0118000。

boot_alloc()函数主要做的工作就是当申请n字节大小空间的内存时，将当前nextfree保存在result当做函数返回值，然后将其向后移动ROUNDUP(n, PGSIZE)，此时[result, nextfree)的空间就分配出来了。因为是按页管理内存，所以分配的内存大小需要页对齐。特别地，如果n是0，那么直接返回nextfree。

```c
if (0 == n)
		return nextfree;

	n = ROUNDUP(n, PGSIZE);
	if ((uint32_t)nextfree + n > KERNBASE + npages * PGSIZE)
		panic("boot_alloc: out of memory\n");
	result = nextfree;
	nextfree += n;
	return result;
```

#### mem_init() 只需要完成到调用check_page_free_list(1)之前

在内核代码中每个物理页都由一个PageInfo的数据结构来标识，一共有npages个物理页。所有的PageInfo组成一个pages数组。所以在mem_init需要先对pages结构进行物理内存分配。 **之后所出现的物理页其实是指PageInfo，代码中对物理页的操作其实都是操作PageInfo这个结构**

分配npages个PageInfo结构体用于管理物理页面，pages的起始地址即上述中的0xf0118000

```c
pages = (struct PageInfo *)boot_alloc(npages * sizeof(struct PageInfo));
memset(pages, 0, sizeof(struct PageInfo) * npages);
```

#### page_init()

分配完内存后自然就要对数据结构进行初始化，即将物理内存中的每一页都与pageInfo关联，其中分为可用页和不可用页。物理内存页到pages数组下标的映射关系为: ==物理地址/PGSIZE(4k)==。根据提示可知，一共有两大块空闲物理内存块。[1, npages_basemem)。第二块就是从内核代码往后，这个地址可以用boot_alloc(0)取到，即分配了pages内存之后的地址。但这个地址在代码中是虚拟地址，所以需要将其转换成物理地址，可以用PADDR()宏来转换。所以第二块范围就是[PADDR(boot_alloc(0)/PGSIZE)，npages)。找出这些空闲页后需要用page_free_list链表串起来。方便后续内存分配。

```c
size_t i;
	for (i = 1; i < npages_basemem; i++)
	{
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}
	// boot_alloc(0) returns the first free virtual address after the kernel
	for (i = PADDR(boot_alloc(0)) / PGSIZE; i < npages; i++)
	{
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}
```

![image-20230505135449738](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305051354774.png)

由此图也可以看出，pages即PageInfo数组的首地址是0xf0118000，其管理的虚拟地址是0xf0000000，其对应的物理地址是0x00000000，由此也说明pages数组的下标和物理地址的映射关系。

#### page_alloc()

空闲物理页的分配。在空闲物理页链表中取出一个物理页即可。返回的是PageInfo*，这个怎么与物理内存中的物理页对应呢？ - 注意: 两个指针相减，结果并不是两个指针数值上的差，而是把这个差除以指针指向类型的大小的结果，即个数

可以用page2pa(PageInfo*)的宏，因为pages数组是连续的物理内存，所以直接将PageInfo* pp 的地址减去pages就可以知道在数组中的下标是多少。在乘以4K就可以得到物理地址了: (pp-pages) << PGSHIFT。PGSHIFT = 1<<12 = 4096 = 4k。

```c
struct PageInfo *
page_alloc(int alloc_flags)
{
	// Fill this function in
	if (page_free_list == NULL)
	{
		return NULL;
	}
	struct PageInfo *page = page_free_list;
	page_free_list = page->pp_link;
	page->pp_link = NULL;
	if (alloc_flags & ALLOC_ZERO)
	{
		/*
		page还是虚拟地址，用page减去pages数组的起始地址，得到的是pages数组的下标
		然后在乘上PGSIZE，得到的是其管理的物理地址.
		最后通过KADDR也就是加上KERNBASE得到对应的管理物理地址的内核虚拟地址
		*/
		memset(page2kva(page), 0, PGSIZE);
		cprintf("page: %x page2kva(page): %x page2pa(page): %x pages: %x page2kva(pages): %x PGSIZE: %d\n", page, page2kva(page), page2pa(page), pages,page2kva(pages), PGSIZE);
	}

	return page;
}
```

由上图也可只，page的虚拟地址是0xf0119fe8，其管理的虚拟地址是0xf03fd000，对应的物理地址是0x3fd000

#### page_free()

这个比较简单。页面释放，将物理页重新插入到page_free_list中。前提是要保证该页面没有被引用，并且也不在空闲链表中。

```c
void page_free(struct PageInfo *pp)
{
	// Fill this function in
	// Hint: You may want to panic if pp->pp_ref is nonzero or
	// pp->pp_link is not NULL.
	if (pp->pp_ref != 0 || pp->pp_link != NULL)
	{
		panic("pp->pp_ref is nonzero or pp->pp_link is not NULL");
	}
	pp->pp_link = page_free_list;
	page_free_list = pp;
}

```

## Part2 Virtual Address

### exercise2

阅读[Intel 80386 Reference Manual](https://link.zhihu.com/?target=https%3A//pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)的第5第6章。

在x86结构下，使用的是分段分页机制，虚拟地址转换为物理地址需要中间还需要经历线性地址(分段的过程)。参考lab1的实模式和保护模式

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305051911759.webp)



下图是具体的地址结构转换过程。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305051911896.webp)

在JOS中，虚拟地址=线性地址，为什么呢？因为在boot/boot.S中把所有的段地址都设置成了0 到0xffffffff，即段基址都等于0，相当于0+offset，所以就没有分段的效果了。这样我们就可以专注于实现分页机制了。

在lab1中已经安装了一个简易的页目录和页表，将虚拟地址[0, 4MB)映射到物理地址[0, 4MB)，[0xF0000000, 0xF0000000+4MB)映射到[0, 4MB）。具体实现在kern/entry.S中，临时的页目录线性地址为entry_pgdir，定义在kern/entrypgdir.c中。

### exercise4

补全kern/pmap.c下的这些函数，实现页表管理。

```c
pgdir_walk()
        boot_map_region()
        page_lookup()
        page_remove()
        page_insert()
```

在补全这些函数之前，需要先明白一个图的含义。JOS采用的是二级页表机制，主要由五个元素组成，页目录表-页目录项(PDE, page diretory entry)，页表-页表项(PTE, page table entry)，物理页。PDE和PTE存储的都是地址。 其中一个页目录项对应一个页表，一个页表项对应一个物理页。页目录表的地址存储在CR3寄存器中。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305081056014.webp)

#### pgdir_walk()

根据(页目录表，虚拟地址，创建标志)找到该虚拟地址所对应的物理页的虚拟地址。 通过PDX获得va的页目录项在页目录表中的偏移取得PDE，如果该PDE所指向的PT是空的话且create == 1，那就创建一个页目录表，即申请一页的物理内存。并设置为用户可读可写。然后再根据PTX获得va在页表项在页表中的偏移获取PTE，返回此PTE的地址。 **PTE_ADDR(\*pde)**的作用是去掉后面的权限位。

> 需要注意的是通过一级页表项获取二级页表的起始地址时，页表项中存储的是物理地址，但是CPU需要接收的是虚拟地址，因此需要通过KADDR函数进行转换。而在6.s081中之所以不需要通过KADDR的转换是因为，在xv6中虚拟地址和物理地址是直接映射的关系，即虚拟地址也是物理地址，因此直接使用即可，但是要知道在取索引时CPU使用的是虚拟地址而不是物理地址。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305081102255.jpeg)



```c
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
	// Fill this function in
	// PDX和PTX都是取低10位，都与3FF相与，一个是左移22位一个是左移12位
	uint32_t pde_index = PDX(va);
	uint32_t pte_index = PTX(va);
	pde_t *pde = &pgdir[pde_index]; // 根据索引获得pde页目录项的地址
	if (*pde & PTE_P)				// 如果pde存在
	{
		//*pde是物理地址，但是内核需要读写的是虚拟地址，所以需要用KADDR转换一下
		pte_t *pte = (pte_t *)KADDR(PTE_ADDR(*pde)); // 获得pte的地址
		return &pte[pte_index];						 // 返回对应索引的pte的地址
	}
	else if (create) // 如果pde不存在，且create为真
	{
		struct PageInfo *page = page_alloc(ALLOC_ZERO); // 分配一个物理页
		if (page == NULL)
		{
			return NULL;
		}
		page->pp_ref++;								  // 引用计数加1
		*pde = page2pa(page) | PTE_P | PTE_W | PTE_U; // page2pa用于获取page管理的物理页面的地址，设置pde的值
		pte_t *pte = (pte_t *)KADDR(PTE_ADDR(*pde));  // 获得pte页表首地址
		return &pte[pte_index];						  // 返回pte索引对应页表项的地址，但是此时*pte即页表项中还未填充内容，需要后续的函数填充
	}
	return NULL;
}
```

#### boot_map_region()

之前的pgdir_walk是取到页表项，但页表项还未真正的映射到物理页上，此函数将从va开始的大小为size的地址按页从物理地址pa开始映射。相当于对页表项赋值。

```c
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{

	// 计算有多少页，并向上取整
	size_t pages_num = size / PGSIZE;
	if (size % PGSIZE != 0)
		pages_num++;
	// 分别对每页调整
	for (int i = 0; i < pages_num; i++)
	{
		pte_t *pte = pgdir_walk(pgdir, (void *)va, 1); // 获取va对应的页表地址
		if (pte == NULL)
		{
			panic("boot_map_region(): out of memory\n");
		}
		// 修改va对应的页表PTE的值
		*pte = pa | perm | PTE_P;
		pa += PGSIZE;c
		va += PGSIZE;
	}
}
```

#### page_lookup()

返回页表项所对应的物理页的虚拟地址，并把页表项存储在pte_store中。**pte_store二级指针相当于传入指针的引用。

```c
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
    // Fill this function in
    pte_t* pte = pgdir_walk(pgdir, va, 0);
    if (pte == NULL || !(*pte & PTE_P)) {
        return NULL;
    }
    if (pte_store) {
        *pte_store = pte;
    }
    return (struct PageInfo*)pa2page(PTE_ADDR(*pte));
}
```

#### page_remove()

清空页表项对应的物理页,并把物理页引用减减。

```c
void
page_remove(pde_t *pgdir, void *va)
{
    // Fill this function in
    pte_t* pte_store;
    struct PageInfo* pp = page_lookup(pgdir, va, &pte_store);
    if(pp == NULL || !(*pte_store & PTE_P)) 
        return;
    page_decref(pp);
    *pte_store = 0;
    tlb_invalidate(pgdir, va);
}
```

#### page_insert()

给页表项赋值一个物理页。

```c
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
    // Fill this function in

    pte_t* pte = pgdir_walk(pgdir, va, 1);
    if (pte == NULL) {
        return -E_NO_MEM;
    }
    pp->pp_ref++;
    if (*pte & PTE_P) {
        page_remove(pgdir, va);
    }
    *pte = page2pa(pp) | perm | PTE_P;
    // cprintf("page_insert: %x\n", *pte);
    return 0;
}
```

## Part 3: Kernel Address Space

### exercise5

JOS的内核空间为[UTOP, KERNBASE)，一共为256MB。 填充完整mem_init()，将虚拟内核地址空间映射到物理地址上。 1. [UPAGES, UPAGES+PTSIZE)这段空间是pages数组的空间，将其映射到PADDR(pages)上。 2. [KSTACKTOP-KSTKSIZE, KSTACKTOP)是内核栈的空间，将其映射到PADDR(bootstack) 3. [KERNBASE, 2^32-1)，其中32位系统无法计算 2^32，但 2^32-1 == -KERNBASE。这段地址从物理地址0开始映射。

```c
boot_map_region(kern_pgdir, UPAGES, PTSIZE, PADDR(pages), PTE_U);
PTE_U);
boot_map_region(kern_pgdir, KSTACKTOP-KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
boot_map_region(kern_pgdir, KERNBASE, -KERNBASE, 0, PTE_W);
```

![image-20230508111122975](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305081111012.png)

也就是我们需要将图中三块高亮区域进行映射，其中KSTACKTOP-KSTKSIZE是8*PGSIZE。

![image-20230508111306308](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305081113352.png)

之后我们使用JOS的qemu提供查看虚拟地址被映射的区域可以看到：

0xefff8000\~0x100000000即是KSTACKTOP-KSTKSIZE\~4G的虚拟地址被映射。

0xef000000~0xef400000即是UPAGES开始的虚拟地址被映射，

0xef400000\~0xef800000是虚拟页表被映射，所以中间两块0xef7bc000\~0xef80000即是虚拟页表被映射，暂时实验还未提及，后续应该会提到，应该和mem_init中这行代码有关。

```c
kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;
```

![image-20230508215542680](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305082155715.png)

![image-20230508215752898](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305082157942.png)

2. 到目前为止页目录表中已经包含多少有效页目录项？他们都映射到哪里？

3BC号页目录项，指向的是pages数组

3BD号页目录项，指向的是kern_pgdir本身，即虚拟地址UVPT~ULIM指向了kern_pgdir的物理地址，即页目录本身（一级页表指向本身，即二级页表仍然是用的一级页表）

![image-20230508220636035](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305082206062.png)

由此图可以知道kern_dir的虚拟地址是0xf011b000,物理地址是0x11b000。再看之前的页表，pde[3bd]指向的是kern_pgdir的物理地址，虽然我们不知道pde[3bd]里面的内容，但是我们可以从二级页表中看出pte[3bd]指向的物理地址是0x0011b000，也就是kern_pgdir的物理地址。

也就是说在PED[3bd]下的pte[]均代表的是二级页表的物理地址，我们可以看出二级页表最终映射到0x3ff000并未超过4M。

3BF号页目录项，指向的是bootstack，正好是8*PGSIZE

剩下的对应物理地址[0M-256M]

