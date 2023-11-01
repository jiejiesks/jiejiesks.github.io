# 6.828_lab3


For GCC 7 or later, after switching to `lab3` branch an error like `kernel panic at kern/pmap.c:147: PADDR called with invalid kva 00000000` will occur. This is a bug caused by the linker script, modify `kern/kernel.ld` as follow will fix it.

```
--- a/kern/kernel.ld
+++ b/kern/kernel.ld
@@ -50,6 +50,7 @@ SECTIONS
        .bss : {
                PROVIDE(edata = .);
                *(.bss)
+               *(COMMON)
                PROVIDE(end = .);
                BYTE(0)
        }
```



切换进程但是其内核页表不进行切换并非直接不切换，而是因为每个进程都对内核页表进行了赋值，并将UVPT这个虚拟地址映射到页表的物理地址上



## 中断描述符

一共有三种必须将控制由用户程序转移到内核的情形：系统调用、异常和中断。系统调用发生在用户程序请求一项操作系统的服务时。异常发生在用户程序想要执行某种非法的操作时，如除以零或者访问不存在的页表项。中断发生在某个外部设备需要操作系统的留意时，比如时钟芯片会定时产生一个中断提醒操作系统可以将硬件资源切换给下一个进程使用一会儿了。在大多数处理器上，这三种情形都是由同一种硬件机制来处理的。对于x86，系统调用和异常本质上也是生成一个中断，因此操作系统只需要提供一套针对中断的处理策略就可以了。

异常和中断都是“受保护的控制传输”，这会导致处理器从用户模式切换到内核模式 (CPL=0)，而不会给用户模式代码任何干扰内核或其他环境运行的机会。在 Intel 的术语中，*中断*是一种受保护的控制传输，它由通常在处理器外部的异步事件引起，例如外部设备 I/O 活动的通知。*相反，异常*是由当前运行的代码同步引起的受保护的控制传输，例如由于被零除或无效的内存访问。

为了确保这些受保护的控制传输真正受到*保护*，处理器的中断/异常机制被设计成当中断或异常发生时当前正在运行的代码 *无法随意选择进入内核的位置或方式*。相反，处理器确保只有在仔细控制的条件下才能进入内核。在 x86 上，有两种机制共同提供这种保护：

1. **中断描述符表IDT**。处理器确保中断和异常只能导致内核在由内核本身确定的几个特定的、定义明确的入口点进入 ，而不是在中断或异常发生时运行的代码。

   x86 允许多达 256 个不同的中断或异常入口点进入内核，每个都有不同的*中断向量*。向量是介于 0 和 255 之间的数字。中断的向量由中断源决定：不同的设备、错误条件和对内核的应用程序请求会生成具有不同向量的中断。CPU 使用向量作为处理器*中断描述符表*(IDT) 的索引，内核在内核专用内存中设置该表，与 GDT 非常相似。处理器从该表中的适当==条目==（EIP和CS）加载：

   - `加载到指令指针 ( EIP` ) 寄存器 的值，指向指定用于处理该类型异常的内核代码。
   - `要加载到代码段 ( CS` ) 寄存器 中的值，它在位 0-1 中包含异常处理程序运行的特权级别。（在 JOS 中，所有异常都在内核模式下处理，权限级别为 0。）

2. **任务状态段TSS。** 处理器需要一个地方来保存中断或异常发生之前的旧处理器状态，例如处理器调用异常处理程序之前的`EIP`和`CS`的原始值，以便异常处理程序稍后可以恢复旧状态并恢复被中断的状态从它停止的地方开始的代码。但是这个旧处理器状态的保存区域必须反过来保护不受非特权用户模式代码的影响（保存的旧状态不可以被用户模式的代码所访问或者影响，因此需要切换到内核态来保存）；否则错误或恶意的用户代码可能会危及内核。

   因此，当 x86 处理器发生中断或陷阱导致特权级别从用户模式更改为内核模式时，它也会切换到内核内存中的堆栈。*称为任务状态段*(TSS)的结构指定了该堆栈所在的段选择器和地址。具体地，JOS只会用到TSS中的esp0和ss0字段，JOS 不使用任何其他 TSS 字段。

   处理器在读取IDT中的中断描述符之前，会先从TSS中读取将要切换的栈的ss0和esp0，并切换到新栈；接着往新栈压入原来进程的ss、esp、eflags、cs、eip、err code（如有的话）等信息；然后，处理器访问IDT，取出中断处理程序的cs、eip并跳转到中断处理程序。这些都是由处理器自动完成的工作，接下来便是操作系统（中断处理程序）的任务了。

   尽管 TSS 很大并且可能有多种用途，但 JOS 仅使用它来定义处理器在从用户模式转移到内核模式时应该切换到的内核堆栈。

   

所有中断处理程序的入口都位于trapentry.S中，我们要通过其中定义的两个宏来设置需要用到的中断处理程序。JOS为多个入口设计了同一个处理函数trap，然后在trap中将处理任务分发给具体的中断处理函数。TRAPHANDLER和TRAPHANDLER_NOEC两个宏分别用于处理器会自动压入err code与不会自动压入的情形。我们可以直接对照xv6知道哪些中断号会压入错误代码而哪些不会。xv6中是通过一段perl脚本生成256个中断号入口的，JOS中暂时用不到这么多中断号，我们通过定义好的两个宏可以逐个生成需要用到的入口程序，其余的中断号会在后面的实验过程中创建。具体如下：

```text
TRAPHANDLER_NOEC(t_divide, T_DIVIDE)
TRAPHANDLER_NOEC(t_debug, T_DEBUG)
TRAPHANDLER_NOEC(t_nmi, T_NMI)
TRAPHANDLER_NOEC(t_brkpt, T_BRKPT)
TRAPHANDLER_NOEC(t_oflow, T_OFLOW)
TRAPHANDLER_NOEC(t_bound, T_BOUND)
TRAPHANDLER_NOEC(t_illop, T_ILLOP)
TRAPHANDLER_NOEC(t_device, T_DEVICE)
TRAPHANDLER(t_dblflt, T_DBLFLT)
TRAPHANDLER(t_tss, T_TSS)
TRAPHANDLER(t_segnp, T_SEGNP)
TRAPHANDLER(t_stack, T_STACK)
TRAPHANDLER(t_gpflt, T_GPFLT)
TRAPHANDLER(t_pgflt, T_PGFLT)
TRAPHANDLER_NOEC(t_fperr, T_FPERR)
TRAPHANDLER(t_align, T_ALIGN)
TRAPHANDLER_NOEC(t_mchk, T_MCHK)
TRAPHANDLER_NOEC(t_simderr, T_SIMDERR)
```

![image-20230522125607731](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305221256760.png)

这些入口程序都会跳转到紧随其后的_alltraps过程，该过程的任务其实就是继续保存用户进程其余的现场信息。而且JOS让它压入新栈的信息与之前处理器自动压入的信息（ss、esp、eflags、cs、eip、err code）最终成为一个Trapframe结构，并将结构体的地址传给trap函数。这份状态信息将会在之后用于恢复现场。也就是在这里，Trapframe成功地完成了原本属于TSS的任务。\_allotraps的实现比较简单，和xv6不同，JOS的trap函数是不会返回的，所以在调用完trap之后没必要再执行任何操作。

```text
_alltraps:
    # push values to make the stack look like a struct Trapframe
    pushl %ds
    pushl %es
    pushal
    # load GD_KD into %ds and %es
    movw $GD_KD, %ax
    movw %ax, %ds
    movw %ax, %es
    # pushl %esp to pass a pointer to the Trapframe as an argument to trap()
    pushl %esp
    call trap
```


