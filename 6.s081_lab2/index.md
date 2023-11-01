# 6.s081_lab2


## chapter 2

### 2.2用户态、核心态、系统调用

RISC-V分为三个模式machine mode,supervisor mode and user mode

在机器模式下执行的指令具有完全权限；CPU以机器模式启动。机器模式主要用于配置计算机。Xv6在机器模式下执行几行，然后切换到管理模式。

### 2.3内核组织

整个os都驻留在内核中，所有系统调用的实现都是以supervisor mode运行，这个被称为amonolithic内核（宏内核）

为了降低内核出错的风险，操作系统设计者可以最大限度地减少在监督模式下运行的操作系统代码的数量，并在用户模式下执行大部分操作系统。这个内核组织被称为amicrokernel（微内核）

### 2.5进程概述

Risc-v的指针是64位的，硬件仅使用低39位；而xv6仅使用这39个比特中的38个。因此，最大地址为238−1=0x3fffffffff

xv6内核为每个进程维护许多状态片段，并将它们聚集到一个`proc`(***kernel/proc.h\***:86)结构体中。一个进程最重要的内核状态片段是它的页表、内核栈区和运行状态。我们将使用符号`p->xxx`来引用`proc`结构体的元素；例如，`p->pagetable`是一个指向该进程页表的指针。

线程的大部分状态（局部变量、函数调用返回地址）都存储在线程的堆栈中。每个进程有两个堆栈：一个用户堆栈和一个内核堆栈（p->kstack）。当进程执行用户指令时，只有它的用户堆栈在使用，而它的内核堆栈是空的。当进程进入内核（用于系统调用或中断）时，内核代码在进程的内核堆栈上执行；当进程在内核中时，它的用户堆栈仍然包含保存的数据，但没有被实际使用。进程的线程在主动使用其用户堆栈和内核堆栈之间交替。内核堆栈是独立的（并受用户代码保护），因此即使进程破坏了其用户堆栈，内核也可以执行。

### 2.6xv6启动和运行第一个线程的概述

为了使xv6更加具体，我们将概述内核如何启动和运行第一个进程。接下来的章节将更详细地描述本概述中显示的机制。

当RISC-V计算机上电时，它会初始化自己并运行一个存储在只读内存中的引导加载程序。引导加载程序将xv6内核加载到内存中。然后，在机器模式下，中央处理器从`_entry` (***kernel/entry.S\***:6)开始运行xv6。Xv6启动时页式硬件（paging hardware）处于禁用模式：也就是说虚拟地址将直接映射到物理地址。

加载程序将xv6内核加载到物理地址为`0x80000000`的内存中。它将内核放在`0x80000000`而不是`0x0`的原因是地址范围`0x0:0x80000000`包含I/O设备。

`_entry`的指令设置了一个栈区，这样xv6就可以运行C代码。Xv6在***start. c (kernel/start.c:11)\***文件中为初始栈***stack0\***声明了空间。由于RISC-V上的栈是向下扩展的，所以`_entry`的代码将栈顶地址`stack0+4096`加载到栈顶指针寄存器`sp`中。现在内核有了栈区，`_entry`便调用C代码`start`(***kernel/start.c\***:21)。

函数`start`执行一些仅在机器模式下允许的配置，然后切换到管理模式。RISC-V提供指令`mret`以进入管理模式，该指令最常用于将管理模式切换到机器模式的调用中返回。而`start`并非从这样的调用返回，而是执行以下操作：它在寄存器`mstatus`中将先前的运行模式改为管理模式，它通过将`main`函数的地址写入寄存器`mepc`将返回地址设为`main`，它通过向页表寄存器`satp`写入0来在管理模式下禁用虚拟地址转换，并将所有的中断和异常委托给管理模式。

在进入管理模式之前，`start`还要执行另一项任务：对时钟芯片进行编程以产生计时器中断。清理完这些“家务”后，`start`通过调用`mret`“返回”到管理模式。这将导致程序计数器（PC）的值更改为`main`(***kernel/main.c\***:11)函数地址。

 TIPS

**注：**`mret`执行返回，返回到先前状态，由于`start`函数将前模式改为了管理模式且返回地址改为了`main`,因此`mret`将返回到`main`函数，并以管理模式运行

在`main`(***kernel/main.c\***:11)初始化几个设备和子系统后，便通过调用`userinit` (***kernel/proc.c\***:212)创建第一个进程，第一个进程执行一个用RISC-V程序集写的小型程序：***initcode. S\*** (***user/initcode.S:\***1)，它通过调用`exec`系统调用重新进入内核。正如我们在第1章中看到的，`exec`用一个新程序（本例中为 `/init`）替换当前进程的内存和寄存器。一旦内核完成`exec`，它就返回`/init`进程中的用户空间。如果需要，`init`(***user/init.c\***:15)将创建一个新的控制台设备文件，然后以文件描述符0、1和2打开它。然后它在控制台上启动一个shell。系统就这样启动了。



## lab2

### system call tracing

trace系统调用有一个参数，这个参数是一个整数“掩码”（mask），它的比特位指定要跟踪的系统调用。例如，要跟踪fork系统调用，程序调用`trace(1 << SYS_fork)`，其中`SYS_fork`是kernel/syscall.h中的系统调用编号，这个掩码是01也即2。32是read的系统调用，因为1<<5为32，100000。1111……1共31个1，即2147483647，这个掩码代表追踪所有的系统调用。

我们提供了一个用户级程序版本的`trace`，它运行另一个启用了跟踪的程序（参见user/trace.c）。完成后，您应该看到如下输出：

```bash
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
$
$ trace 2147483647 grep hello README
4: syscall trace -> 0
4: syscall exec -> 3
4: syscall open -> 3
4: syscall read -> 1023
4: syscall read -> 966
4: syscall read -> 70
4: syscall read -> 0
4: syscall close -> 0
$
$ grep hello README
$
$ trace 2 usertests forkforkfork
usertests starting
test forkforkfork: 407: syscall fork -> 408
408: syscall fork -> 409
409: syscall fork -> 410
410: syscall fork -> 411
409: syscall fork -> 412
410: syscall fork -> 413
409: syscall fork -> 414
411: syscall fork -> 415
...
$
```

在上面的第一个例子中，`trace`调用`grep`，仅跟踪了`read`系统调用。`32`是`1<。在第二个示例中，`trace`在运行`grep`时跟踪所有系统调用；`2147483647`将所有31个低位置为1。在第三个示例中，程序没有被跟踪，因此没有打印跟踪输出。在第四个示例中，在`usertests`中测试的`forkforkfork`中所有子孙进程的`fork`系统调用都被追踪。如果程序的行为如上所示，则解决方案是正确的（尽管进程ID可能不同）。

#### 解决步骤

- 在Makefile的**UPROGS**中添加`$U/_trace`
- 在user/user.h中添加原型

```c
int trace(int);
```

添加存根到user/usys.pl

```c
entry("trace");
```

添加系统调用号到kernel/syscall.h

```c
#define SYS_trace  22
```

在syscall.c中添加trace相关的定义

```c
extern uint64 sys_trace(void);
[SYS_trace]   sys_trace,
```

现在make qemu可以通过编译，但是内核中还没有实现系统调用，执行测试trace 32 grep hello README将失败

- 在kernel/sysproc.c中添加一个`sys_trace()`函数，它通过将参数保存到`proc`结构体（请参见kernel/proc.h）里的一个新变量中来实现新的系统调用。从用户空间检索系统调用参数的函数在kernel/syscall.c中。

  在proc结构体中添加成员变量mask

```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  int mask;                    // mask for system call
};

```

通过argint获取到第一个参数，将其赋值给proc的mask成员变量

```c
// realize trace
// save params into struct proc
uint64
sys_trace(void)
{
  int mask;
  if (argint(0, &mask) < 0)
  {
    return -1;
  }
  myproc()->mask = mask;
  return 0;

}
```

- 修改`fork()`（请参阅**kernel/proc.c**）将跟踪掩码从父进程复制到子进程。在fork函数中将parent process的mask赋值给child process

```c
safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  np->state = RUNNABLE;

  np->mask = p->mask;

  release(&np->lock);

  return pid;
```

- 修改**kernel/syscall.c**中的`syscall()`函数以打印跟踪输出。您将需要添加一个系统调用名称数组以建立索引。当调用trace的时候才会有mask属性，因此将掩码和系统调用号相与，为1则按要求输出即可。`syscall`（***kernel/syscall.c\***:133）从陷阱帧（trapframe）中保存的`a7`中检索系统调用号（`p->trapframe->a7`），并用它索引到`syscalls`中，对于第一次系统调用，`a7`中的内容是`SYS_exec`（***kernel/syscall. h\***:8），导致了对系统调用接口函数`sys_exec`的调用。

  当系统调用接口函数返回时，`syscall`将其返回值记录在`p->trapframe->a0`中。这将导致原始用户空间对`exec()`的调用返回该值，因为RISC-V上的C调用约定将返回值放在`a0`中。系统调用通常返回负数表示错误，返回零或正数表示成功。如果系统调用号无效，`syscall`打印错误并返回-1

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();//return current process

  num = p->trapframe->a7;//system call number
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();//return number,if err return fushu
    if(1<<num & p->mask){
      printf("%d: syscall %s -> %d\n",p->pid,sysname[num],p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;//if syscall number is unknown return -1
  }
}

```

### Sysinfo

将添加一个系统调用`sysinfo`，它收集有关正在运行的系统的信息。系统调用采用一个参数：一个指向`struct sysinfo`的指针（参见**kernel/sysinfo.h**）。内核应该填写这个结构的字段：`freemem`字段应该设置为空闲内存的字节数，`nproc`字段应该设置为`state`字段不为`UNUSED`的进程数。我们提供了一个测试程序`sysinfotest`；如果输出“**sysinfotest: OK**”则通过。

#### 解决步骤

- 在**Makefile**的**UPROGS**中添加`$U/_sysinfotest`

- 在user/user.h中添加原型

```c
int sysinfo(int);
```

添加存根到user/usys.pl

```c
entry("sysinfo");
```

添加系统调用号到kernel/syscall.h

```c
#define SYS_sysinfo  23
```

在syscall.c中添加trace相关的定义

```c
extern uint64 sys_sysinfo(void);
[SYS_sysinfo]   sys_sysinfo,
```

现在make qemu可以通过编译，但是内核中还没有实现系统调用，执行测试trace 32 grep hello README将失败

- `sysinfo`需要将一个`struct sysinfo`复制回用户空间；请参阅`sys_fstat()`(**kernel/sysfile.c**)和`filestat()`(**kernel/file.c**)以获取如何使用`copyout()`执行此操作的示例。
  - 先定义一个指向用户态struct sysinfo的指针
  - 我们在用户态时会传递一个地址，通过argaddr获取该地址存入sysinfo指针中
  - 定义一个内核态的sysinfo结构体（加头文件），接下来我我们需要通过两个函数为sysinfo的两个成员变量赋值。
  - 最后通过copyout函数将内核态的sysinfo结构体复制到用户态sysinfo指针指向的地址

```c
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
// int
// copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
// {
uint64
sys_sysinfo(void)
{
  uint64 sysinfo; // user pointer to struct sysinfo
  // get the address of sysinfo in user space
  if (argaddr(0, &sysinfo) < 0)
    return -1;
  struct proc *p = myproc();
  //struct sysinfo in kernel
  struct sysinfo sys_info;
  //kernel need to fill the struct sysinfo

  //copy the struct sysinfo in kernel to the pointer to sysinfo in user space
  if (copyout(p->pagetable, sysinfo, (char *)&sys_info, sizeof(sys_info)) < 0)
    return -1;
  return 0;
}
```

- 要获取空闲内存量，请在**kernel/kalloc.c**中添加一个函数。定义一个指针p指向记录空闲内存的链表的表头，每次遍历到一个num就+1，最后返回num*PAGESIZE即可

```c
uint64
free_memory(void)
{
  struct run *p = kmem.freelist;
  uint64 num = 0;
  while (p)
  {
    num++;
    p = p->next;
  }
  return num * PGSIZE;
}
```

- 要获取进程数，请在**kernel/proc.c**中添加一个函数。遍历proc[NPROC]，如果p的状态为UNUSED就为n加一，最后的n即为空闲的process数量。

```c
uint64 free_proc(void)
{
  uint64 n = 0;
  struct proc *p;
  for (p = proc; p < &proc[NPROC]; p++)
  {
    acquire(&p->lock);
    if (p->state != UNUSED)
      n++;
    release(&p->lock);
  }
  return n;
}
```

- 最后将刚才的sysinfo系统调用补充完整，同时需要在kernel/defs.h添加上刚才所写的两个函数，才可使用。

```c
 //kernel need to fill the struct sysinfo
  sys_info.freemem=free_memory();
  sys_info.nproc=free_proc();
```




