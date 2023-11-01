# 6.s081_lab3


# Chapter 3

## 3.1页式硬件

该树的根是一个4096字节的页表页，其中包含512个PTE，每个PTE中包含该树下一级页表页的物理地址（PPN）。这些页中的每一个PTE都包含该树最后一级的512个PTE。分别使用virtual address 的L2,L1,L0各9位（2^9=512）去确定页表页的哪一个PTE。最终的Physical address是由最后一级的PPN和Virtual address的最开始的12位offset组成。

![image-20230323203632265](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312128021.png)

每个PTE包含10位的标志位。这些标志位告诉分页硬件允许如何使用关联的虚拟地址。`PTE_V`指示PTE是否存在：如果它没有被设置，对页面的引用会导致异常（即不允许）。`PTE_R`控制是否允许指令读取到页面。`PTE_W`控制是否允许指令写入到页面。`PTE_X`控制CPU是否可以将页面内容解释为指令并执行它们。`PTE_U`控制用户模式下的指令是否被允许访问页面；如果没有设置`PTE_U`，PTE只能在管理模式下使用。图3.2显示了它是如何工作的。标志和所有其他与页面硬件相关的结构在（***kernel/riscv.h\***）中定义。

为了告诉硬件使用页表，内核必须将根页表页的物理地址写入到`satp`寄存器中（`satp`的作用是存放根页表页在物理内存中的地址）。每个CPU都有自己的`satp`，一个CPU将使用自己的`satp`指向的页表转换后续指令生成的所有地址。每个CPU都有自己的`satp`，因此不同的CPU就可以运行不同的进程，每个进程都有自己的页表描述的私有地址空间。

![image-20230323204347684](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312129907.png)

## 3.2内核地址空间

Xv6为每个进程维护一个页表，用以描述每个进程的用户地址空间，外加一个单独描述内核地址空间的页表。内核配置其地址空间的布局，以允许自己以可预测的虚拟地址访问物理内存和各种硬件资源。图3.3显示了这种布局如何将内核虚拟地址映射到物理地址。文件(***kernel/memlayout.h\***) 声明了xv6内核内存布局的常量。

![image-20230323204941190](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312129979.png)

QEMU模拟了一台计算机，它包括从物理地址`0x80000000`开始并至少到`0x86400000`结束的RAM（物理内存），xv6称结束地址为`PHYSTOP`。QEMU模拟还包括I/O设备，如磁盘接口。QEMU将设备接口作为内存映射控制寄存器暴露给软件，这些寄存器位于物理地址空间`0x80000000`以下。内核可以通过读取/写入这些特殊的物理地址与设备交互；这种读取和写入与设备硬件而不是RAM通信。第4章解释了xv6如何与设备进行交互。

内核使用“直接映射”获取**内存**和**内存映射设备寄存器**；也就是说，将资源映射到等于物理地址的虚拟地址。例如，内核本身在虚拟地址空间和物理内存中都位于`KERNBASE=0x80000000`。直接映射简化了读取或写入物理内存的内核代码。例如，当`fork`为子进程分配用户内存时，分配器返回该内存的物理地址；`fork`在将父进程的用户内存复制到子进程时直接将该地址用作虚拟地址。

有几个内核虚拟地址不是直接映射：

- 蹦床页面(trampoline page)。它映射在虚拟地址空间的顶部；用户页表具有相同的映射。第4章讨论了蹦床页面的作用，但我们在这里看到了一个有趣的页表用例；一个物理页面（持有蹦床代码）在内核的虚拟地址空间中映射了两次：一次在虚拟地址空间的顶部，一次直接映射。
- 内核栈页面。每个进程都有自己的内核栈，它将映射到偏高一些的地址，这样xv6在它之下就可以留下一个未映射的保护页(guard page)。保护页的PTE是无效的（也就是说`PTE_V`没有设置），所以如果内核溢出内核栈就会引发一个异常，内核触发`panic`。如果没有保护页，栈溢出将会覆盖其他内核内存，引发错误操作。恐慌崩溃（panic crash）是更可取的方案。*（注：Guard page不会浪费物理内存，它只是占据了虚拟地址空间的一段靠后的地址，但并不映射到物理地址空间。）*

## 3.3创建一个地址空间

大多数用于操作地址空间和页表的xv6代码都写在 ***vm.c\*** ([kernel/vm.c:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L1)) 中。其核心数据结构是`pagetable_t`，它实际上是指向RISC-V根页表页的指针；一个`pagetable_t`可以是内核页表，也可以是一个进程页表。最核心的函数是`walk`和`mappages`，前者为虚拟地址找到PTE，后者为新映射装载PTE。名称以`kvm`开头的函数操作内核页表；以`uvm`开头的函数操作用户页表；其他函数用于二者。`copyout`和`copyin`复制数据到用户虚拟地址或从用户虚拟地址复制数据，这些虚拟地址作为系统调用参数提供; 由于它们需要显式地翻译这些地址，以便找到相应的物理内存，故将它们写在***vm.c\***中。

在启动的初期，main会调用kvminit，kvminit首先会分配一个物理内存页用来装根页表，后续调用kvmmap去安装内核需要的内存，这些转换包括 内核的指令和数据，到PHYSTOP的物理内存，以及实际上是设备的内存范围。实际上是设备。（即上图中的CLINT PLIC UARTO VIRTIO KERNBASE PHYSTOP TRAMPOLINE）

`kvmmap`(***kernel/vm.c\***:127)调用`mappages`(***kernel/vm.c\***:138)，`mappages`将范围虚拟地址到同等范围物理地址的映射装载到一个页表中。它以页面大小为间隔，为范围内的每个虚拟地址单独执行此操作。对于要映射的每个虚拟地址，`mappages`调用`walk`来查找该地址的PTE地址。然后，它初始化PTE以保存相关的物理页号、所需权限（`PTE_W`、`PTE_X`和/或`PTE_R`）以及用于标记PTE有效的`PTE_V`(***kernel/vm.c\***:153)。先通过walk找到PTE，然后通过mappages装载PTE

在查找PTE中的虚拟地址（参见图3.2）时，`walk`(***kernel/vm.c\***:72)模仿RISC-V分页硬件。`walk`一次从3级页表中获取9个比特位。它使用上一级的9位虚拟地址来查找下一级页表或最终页面的PTE (***kernel/vm.c\***:78)。如果PTE无效，则所需的页面还没有分配；如果设置了`alloc`参数，`walk`就会分配一个新的页表页面，并将其物理地址放在PTE中。它返回树中最低一级的PTE地址(***kernel/vm.c\***:88)。

上面的代码依赖于直接映射到内核虚拟地址空间中的物理内存。例如，当`walk`降低页表的级别时，它从PTE (***kernel/vm.c\***:80)中提取下一级页表的（物理）地址，然后使用该地址作为虚拟地址来获取下一级的PTE (***kernel/vm.c\***:78)。

```
 for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
```

![image-20230323213441931](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312129892.png)

![image-20230323213453571](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312129922.png)

由于虚拟地址和物理地址直接映射，在这个函数中把va先当做虚拟地址获取第一个9位来获取根页表的PTE，获取到二级页表的物理地址后（每页4096B，所以低12位为0，获取pte之后由于pte的低10位是标志位，pte>>10<<12便是物理地址），将这个地址转换为虚拟地址即是二级页表的虚拟地址，然后便可以在通过va>>12+level*9获取到PTE进而找到最后一级页表

## 3.4物理内存分配

内核必须在运行时为页表、用户内存、内核栈和管道缓冲区分配和释放物理内存。xv6使用内核末尾到`PHYSTOP`之间的物理内存进行运行时分配。它一次分配和释放整个4096字节的页面。它使用链表的数据结构将空闲页面记录下来。分配时需要从链表中删除页面；释放时需要将释放的页面添加到链表中。

## 3.5代码：物理内存分配

分配器(allocator)位于***kalloc.c\***(***kernel/kalloc.c\***:1)中。分配器的数据结构是可供分配的物理内存页的空闲列表。每个空闲页的列表元素是一个`struct run`(***kernel/kalloc.c\***:17)。分配器从哪里获得内存来填充该数据结构呢？它将每个空闲页的`run`结构存储在空闲页本身，因为在那里没有存储其他东西。

`main`函数调用`kinit`(***kernel/kalloc.c\***:27)来初始化分配器。`kinit`初始化空闲列表以保存从内核结束到`PHYSTOP`之间的每一页。xv6应该通过解析硬件提供的配置信息来确定有多少物理内存可用。然而，xv6假设机器有128兆字节的RAM。`kinit`调用`freerange`将内存添加到空闲列表中，在`freerange`中每页都会调用`kfree`。PTE只能引用在4096字节边界上对齐的物理地址（是4096的倍数），所以`freerange`使用`PGROUNDUP`来确保它只释放对齐的物理地址。分配器开始时没有内存；这些对`kfree`的调用给了它一些管理空间。

分配器有时将地址视为整数，以便对其执行算术运算（例如，在`freerange`中遍历所有页面），有时将地址用作读写内存的指针（例如，操纵存储在每个页面中的`run`结构）；这种地址的双重用途是分配器代码充满C类型转换的主要原因。另一个原因是释放和分配从本质上改变了内存的类型。

函数`kfree` (***kernel/kalloc.c\***:47)首先将内存中的每一个字节设置为1。这将导致使用释放后的内存的代码（使用“悬空引用”）读取到垃圾信息而不是旧的有效内容，从而希望这样的代码更快崩溃。然后`kfree`将页面前置（头插法）到空闲列表中：它将`pa`转换为一个指向`struct run`的指针`r`，在`r->next`中记录空闲列表的旧开始，并将空闲列表设置为等于`r`。

`kalloc`删除并返回空闲列表中的第一个元素。

## 3.6进程地址空间

每个进程都有一个单独的页表，当xv6在进程之间切换时，也会更改页表。如图2.3所示，一个进程的用户内存从虚拟地址零开始，可以增长到MAXVA (***kernel/riscv.h\***:348)，原则上允许一个进程内存寻址空间为256G。

![image-20230324132116284](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312129717.png)

当进程向xv6请求更多的用户内存时，xv6首先使用`kalloc`来分配物理页面。然后，它将PTE添加到进程的页表中，指向新的物理页面。Xv6在这些PTE中设置`PTE_W`、`PTE_X`、`PTE_R`、`PTE_U`和`PTE_V`标志。大多数进程不使用整个用户地址空间；xv6在未使用的PTE中留空`PTE_V`。

我们在这里看到了一些使用页表的很好的例子。首先，不同进程的页表将用户地址转换为物理内存的不同页面，这样每个进程都拥有私有内存。第二，每个进程看到的自己的内存空间都是以0地址起始的连续虚拟地址，而进程的物理内存可以是非连续的。第三，内核在用户地址空间的顶部映射一个带有蹦床（trampoline）代码的页面，这样在所有地址空间都可以看到一个单独的物理内存页面。

图3.4更详细地显示了xv6中执行态进程的用户内存布局。栈是单独一个页面，显示的是由`exec`创建后的初始内容。包含命令行参数的字符串以及指向它们的指针数组位于栈的最顶部。再往下是允许程序在`main`处开始启动的值（即`main`的地址、`argc`、`argv`），这些值产生的效果就像刚刚调用了`main(argc, argv)`一样。

![image-20230324132648102](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312129219.png)

为了检测用户栈是否溢出了所分配栈内存，xv6在栈正下方放置了一个无效的保护页（guard page）。如果用户栈溢出并且进程试图使用栈下方的地址，那么由于映射无效（`PTE_V`为0）硬件将生成一个页面故障异常。当用户栈溢出时，实际的操作系统可能会自动为其分配更多内存。

![image-20230324132823706](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312129123.png)

## 3.7 sbrk

`sbrk`是一个用于进程**减少或增长其内存**的系统调用。这个系统调用由函数`growproc`实现(***kernel/proc.c\***:239)。`growproc`根据`n`是正的还是负的调用`uvmalloc`或`uvmdealloc`。`uvmalloc`(***kernel/vm.c\***:229)用`kalloc`分配物理内存，并用`mappages`将PTE添加到用户页表中。`uvmdealloc`调用`uvmunmap`(***kernel/vm.c\***:174)，`uvmunmap`使用`walk`来查找对应的PTE，并使用`kfree`来释放PTE引用的物理内存。

XV6使用进程的页表，不仅是告诉硬件如何映射用户虚拟地址，也是明晰哪一个物理页面已经被分配给该进程的唯一记录。这就是为什么释放用户内存（在`uvmunmap`中）需要检查用户页表的原因。

在heap中获取内存

## 3.8 exec

Stack是单独一个页面，显示的是由`exec`创建后的初始内容





lab3

allocproc分配进程，而在procinit中所有的内核栈都在其中设置，把这个功能迁移到allocproc中，为proc中新增的字段kernel_pagetable赋值

# lab3

## Print a page table

定义一个名为`vmprint()`的函数。它应当接收一个`pagetable_t`作为参数，并以下面描述的格式打印该页表。在`exec.c`中的`return argc`之前插入`if(p->pid==1) vmprint(p->pagetable)`，以打印第一个进程的页表。如果你通过了`pte printout`测试的`make grade`，你将获得此作业的满分。

- 首先现在exec.c的return argc之前插入if(p->pid==1) vmprint(p->pagetable)

- 然后观察kernel/vm.c中的freewalk方法

```
// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}

```

首先会遍历第一级页表，当遇到有效的页表并且不是最后一级，就会递归。RWX均为0表示不是最后一层，因为最后一层页表中的页表项至少有一个为1

根据freewalk函数便可去仿照写出vmprint函数

```c
/**
 * @param pagetable 所要打印的页表
 * @param level 页表的层级
 */
void
_vmprint(pagetable_t pagetable, int level){
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    // PTE_V is a flag for whether the page table is valid
    if(pte & PTE_V){
      for (int j = 0; j < level; j++){
        if (j) printf(" ");
        printf("..");
      }
      uint64 child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // this PTE points to a lower-level page table.
        _vmprint((pagetable_t)child, level + 1);
      }
    }
  }
}

/**
 * @brief vmprint 打印页表
 * @param pagetable 所要打印的页表
 */
void
vmprint(pagetable_t pagetable){
  printf("page table %p\n", pagetable);
  _vmprint(pagetable, 1);
}
```

在`_vmprint`中，首先判断是否有效，并根据level打印出对应的..表示第几层，如果其还有下层页表，那么就递归调用`_vmprint`

- 最后在kernel/defs.h加上新增函数的声明

```
void            vmprint(pagetable_t);
```

## A kernel page table per process

Xv6有一个单独的用于在内核中执行程序时的内核页表。内核页表直接映射（恒等映射）到物理地址，也就是说内核虚拟地址`x`映射到物理地址仍然是`x`。Xv6还为每个进程的用户地址空间提供了一个单独的页表，只包含该进程用户内存的映射，从虚拟地址0开始。因为内核页表不包含这些映射，所以用户地址在内核中无效。因此，当内核需要使用在系统调用中传递的用户指针（例如，传递给`write()`的缓冲区指针）时，内核必须首先将指针转换为物理地址。本节和下一节的目标是允许内核直接解引用用户指针。

> ```
> 你的第一项工作是修改内核来让每一个进程在内核中执行时使用它自己的内核页表的副本。修改struct proc来为每一个进程维护一个内核页表，修改调度程序使得切换进程时也切换内核页表。对于这个步骤，每个进程的内核页表都应当与现有的的全局内核页表完全一致。如果你的usertests程序正确运行了，那么你就通过了这个实验。
> ```

- 首先在kernel/proc.h的proc结构体中添加进程的内核页表成员变量

```c
  pagetable_t kernelpt;      // Kernel page table
```

- 需要初始化进程的内核页表，在vm.c中添加一个新的函数proc_kpt_init，用于在allocproc中，另外需要一个辅助函数uvmmap与kvmmap类似，kvmmap用于对内核的内核页表进行映射，而uvmmap用于对进程的内核页表进行映射。

```c
// Just follow the kvmmap on vm.c
void
uvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(pagetable, va, sz, pa, perm) != 0)
    panic("uvmmap");
}

// Create a kernel page table for the process
pagetable_t
proc_kpt_init(){
  pagetable_t kernelpt = uvmcreate();
  if (kernelpt == 0) return 0;
  uvmmap(kernelpt, UART0, UART0, PGSIZE, PTE_R | PTE_W);
  uvmmap(kernelpt, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
  uvmmap(kernelpt, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
  uvmmap(kernelpt, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
  uvmmap(kernelpt, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
  uvmmap(kernelpt, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  uvmmap(kernelpt, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
  return kernelpt;
}

```

然后在kernel/proc.c里面的`allocproc`调用。

```c
...
// An empty user page table.
p->pagetable = proc_pagetable(p);
if(p->pagetable == 0){
  freeproc(p);
  release(&p->lock);
  return 0;
}

// Init the kernal page table
p->kernelpt = proc_kpt_init();
if(p->kernelpt == 0){
  freeproc(p);
  release(&p->lock);
  return 0;
}
...
```

- 为了确保每一个进程的内核页表都关于该进程的内核栈有一个映射。我们需要将`procinit`方法中相关的代码迁移到`allocproc`方法中。很明显就是下面这段代码，将其剪切到上述内核页表初始化的代码后。

```c
// Allocate a page for the process's kernel stack.
// Map it high in memory, followed by an invalid
// guard page.
char *pa = kalloc();
if(pa == 0)
  panic("kalloc");
uint64 va = KSTACK((int) (p - proc));
uvmmap(p->kernelpt, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
p->kstack = va;
```

- 修改scheduler()来讲进程的内核页表加载进SATP寄存器中。

```c
// Switch h/w page table register to the kernel's page table,
// and enable paging.
void
kvminithart()
{
  w_satp(MAKE_SATP(kernel_pagetable));
  sfence_vma();
}
```

`kvminithart`是用于原先的内核页表，我们将进程的内核页表传进去就可以。在*vm.c*里面添加一个新方法`proc_inithart`。

```c
// Store kernel page table to SATP register
void
proc_inithart(pagetable_t kpt){
  w_satp(MAKE_SATP(kpt));
  sfence_vma();
}
```

然后在`scheduler()`内调用即可，但在结束的时候，需要切换回原先的`kernel_pagetable`。直接调用调用上面的`kvminithart()`就能把Xv6的内核页表加载回去。

```c
...
p->state = RUNNING;
c->proc = p;

// Store the kernal page table into the SATP
proc_inithart(p->kernelpt);

swtch(&c->context, &p->context);

// Come back to the global kernel page table
kvminithart();
...
```

- 在freeproc中释放进程的内核页表，首先需要释放页表内的内核栈，调用uvmunmap可以解除虚拟地址到物理地址的映射并将物理内存释放

```c
// free the kernel stack in the RAM
uvmunmap(p->kernelpt, p->kstack, 1, 1);
p->kstack = 0;

```

然后释放进程的内核页表，先在*kernel/proc.c*里面添加一个方法`proc_freekernelpt`。如下，历遍整个内核页表，然后将所有有效的页表项清空为零。如果这个页表项不在最后一层的页表上，需要继续进行递归，每次遍历完页表之后就会将其物理空间释放。

```c
void
proc_freekernelpt(pagetable_t kernelpt)
{
  // similar to the freewalk method
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = kernelpt[i];
    if(pte & PTE_V){
      kernelpt[i] = 0;
      if ((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        uint64 child = PTE2PA(pte);
        proc_freekernelpt((pagetable_t)child);
      }
    }
  }
  kfree((void*)kernelpt);
}
```

- 修改`vm.c`中的`kvmpa`，将原先的`kernel_pagetable`改成`myproc()->kernelpt`，使用进程的内核页表。

  ```c
  #include "spinlock.h" 
  #include "proc.h"
  
  uint64
  kvmpa(uint64 va)
  {
    uint64 off = va % PGSIZE;
    pte_t *pte;
    uint64 pa;
  
    pte = walk(myproc()->kernelpt, va, 0); // 修改这里
    if(pte == 0)
      panic("kvmpa");
    if((*pte & PTE_V) == 0)
      panic("kvmpa");
    pa = PTE2PA(*pte);
    return pa+off;
  }
  ```

- 最后讲定义的函数加入kernel/defs.h

```c
void            proc_freekernelpt(pagetable_t );
void            uvmmap(pagetable_t, uint64, uint64, uint64, int);
pagetable_t     proc_kpt_init(void); // 用于内核页表的初始化
void            proc_inithart(pagetable_t); // 将进程的内核页表保存到SATP寄存器
```

## copyin/copyinstr

第三个实验是讲用户空间的映射添加到内核页表，目的就是用户空间传递的地址内核可以直接解引用，如果不这么做那么用户态传递的指针，内核需要转换为物理地址之后才能够进行memmove，修改之后memmove直接可以用两个地址即可。

该方案依赖于用户虚拟地址范围不与内核用于其自己的指令和数据的虚拟地址范围重叠。xv6 为用户地址空间使用从零开始的虚拟地址，幸运的是内核的内存从更高的地址开始。但是，该方案确实将用户进程的最大大小限制为小于内核的最低虚拟地址。内核启动后，该地址在 xv6 中为`0xC000000`，即 PLIC 寄存器的地址；

此图是内核空间布局图

![image-20230328210348290](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312130873.png)

下图是用户空间布局图，其虚拟地址不可超过PLIC

![image-20230328210424802](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312130334.png)

- 首先添加复制函数。需要注意的是，在内核模式下，无法访问设置了`PTE_U`的页面，所以我们要将其移除。

  ```c
  void
  u2kvmcopy(pagetable_t pagetable, pagetable_t kernelpt, uint64 oldsz, uint64 newsz){
    pte_t *pte_from, *pte_to;
    oldsz = PGROUNDUP(oldsz);
    for (uint64 i = oldsz; i < newsz; i += PGSIZE){
      if((pte_from = walk(pagetable, i, 0)) == 0)
        panic("u2kvmcopy: src pte does not exist");
      if((pte_to = walk(kernelpt, i, 1)) == 0)
        panic("u2kvmcopy: pte walk failed");
      uint64 pa = PTE2PA(*pte_from);
      uint flags = (PTE_FLAGS(*pte_from)) & (~PTE_U);
      *pte_to = PA2PTE(pa) | flags;
    }
  }
  ```

- 然后在内核更改进程的用户映射的每一处 （`fork()`, `exec()`, 和`sbrk()`），都复制一份到进程的内核页表。

  - `exec()`：

  ```c
  int
  exec(char *path, char **argv){
    ...
    sp = sz;
    stackbase = sp - PGSIZE;
  
    // 添加复制逻辑
    u2kvmcopy(pagetable, p->kernelpt, 0, sz);
  
    // Push argument strings, prepare rest of stack in ustack.
    for(argc = 0; argv[argc]; argc++) {
    ...
  }
  ```

  - `fork()`:

  ```c
  int
  fork(void){
    ...
    // Copy user memory from parent to child.
    if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
      freeproc(np);
      release(&np->lock);
      return -1;
    }
    np->sz = p->sz;
    ...
    // 复制到新进程的内核页表
    u2kvmcopy(np->pagetable, np->kernelpt, 0, np->sz);
    ...
  }
  ```

  - `sbrk()`， 在*kernel/sysproc.c*里面找到`sys_sbrk(void)`，可以知道只有`growproc`是负责将用户内存增加或缩小 n 个字节。以防止用户进程增长到超过`PLIC`的地址，我们需要给它加个限制。

  ```c
  int
  growproc(int n)
  {
    uint sz;
    struct proc *p = myproc();
  
    sz = p->sz;
    if(n > 0){
      // 加上PLIC限制
      if (PGROUNDUP(sz + n) >= PLIC){
        return -1;
      }
      if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
        return -1;
      }
      // 复制一份到内核页表
      u2kvmcopy(p->pagetable, p->kernelpt, sz - n, sz);
    } else if(n < 0){
      sz = uvmdealloc(p->pagetable, sz, sz + n);
    }
    p->sz = sz;
    return 0;
  }
  ```

- 更改userinit和copyin、copyinstr

```c
p->sz = PGSIZE;
u2kvmcopy(p->pagetable, p->kernelpt, 0, p->sz);
```

```c
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}

// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

- 最后将copyin、copyinstr添加至kernel/defs.h

```c
// vmcopyin.c
int             copyin_new(pagetable_t, char *, uint64, uint64);
int             copyinstr_new(pagetable_t, char *, uint64, uint64);
```


