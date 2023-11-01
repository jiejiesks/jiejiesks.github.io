# 6.s081_lab4


# Chapter 4

## 4.1RISC-V陷入机制

每个RISC-V CPU都有一组控制寄存器，内核通过向这些寄存器写入内容来告诉CPU如何处理陷阱，内核可以读取这些寄存器来明确已经发生的陷阱。RISC-V文档包含了完整的内容。***riscv.h\***(***kernel/riscv.h\***:1)包含在xv6中使用到的内容的定义。以下是最重要的一些寄存器概述：

- `stvec`：内核在这里写入其陷阱处理程序的地址；RISC-V跳转到这里处理陷阱。
- `sepc`：当发生陷阱时，RISC-V会在这里保存程序计数器`pc`（因为`pc`会被`stvec`覆盖）。`sret`（从陷阱返回）指令会将`sepc`复制到`pc`。内核可以写入`sepc`来控制`sret`的去向。
- `scause`： RISC-V在这里放置一个描述陷阱原因的数字。
- `sscratch`：内核在这里放置了一个值，这个值在陷阱处理程序一开始就会派上用场。
- `sstatus`：其中的**SIE**位控制设备中断是否启用。如果内核清空**SIE**，RISC-V将推迟设备中断，直到内核重新设置**SIE**。**SPP**位指示陷阱是来自用户模式还是管理模式，并控制`sret`返回的模式。

上述寄存器都用于在管理模式下处理陷阱，在用户模式下不能读取或写入。在机器模式下处理陷阱有一组等效的控制寄存器，xv6仅在计时器中断的特殊情况下使用它们。

多核芯片上的每个CPU都有自己的这些寄存器集，并且在任何给定时间都可能有多个CPU在处理陷阱。

当需要强制执行陷阱时，RISC-V硬件对所有陷阱类型（计时器中断除外）执行以下操作：

1. 如果陷阱是设备中断，并且状态**SIE**位被清空，则不执行以下任何操作。
2. 清除**SIE**以禁用中断。
3. 将`pc`复制到`sepc`。
4. 将当前模式（用户或管理）保存在状态的**SPP**位中。
5. 设置`scause`以反映产生陷阱的原因。
6. 将模式设置为管理模式。
7. 将`stvec`复制到`pc`。
8. 在新的`pc`上开始执行。

请注意，CPU不会切换到内核页表，不会切换到内核栈，也不会保存除`pc`之外的任何寄存器。内核软件必须执行这些任务。CPU在陷阱期间执行尽可能少量工作的一个原因是为软件提供灵活性；例如，一些操作系统在某些情况下不需要页表切换，这可以提高性能。

你可能想知道CPU硬件的陷阱处理顺序是否可以进一步简化。例如，假设CPU不切换程序计数器。那么陷阱可以在仍然运行用户指令的情况下切换到管理模式。但因此这些用户指令可以打破用户/内核的隔离机制，例如通过修改`satp`寄存器(保存页表根地址的寄存器)来指向允许访问所有物理内存的页表。因此，CPU使用专门的寄存器切换到内核指定的指令地址，即`stvec`，是很重要的。

## 4.2从用户空间陷入

如果用户程序发出系统调用（`ecall`指令），或者做了一些非法的事情，或者设备中断，那么在用户空间中执行时就可能会产生陷阱。来自用户空间的陷阱的高级路径是`uservec` (***kernel/trampoline.S\***:16)，然后是`usertrap` (***kernel/trap.c\***:37)；返回时，先是`usertrapret` (***kernel/trap.c\***:90)，然后是`userret` (***kernel/trampoline.S\***:16)。

来自用户代码的陷阱比来自内核的陷阱更具挑战性，因为`satp`指向不映射内核的用户页表，栈指针可能包含无效甚至恶意的值。

由于RISC-V硬件在陷阱期间不会切换页表，所以用户页表必须包括`uservec`（**stvec**指向的陷阱向量指令）的映射。`uservec`必须切换`satp`以指向内核页表；为了在切换后继续执行指令，`uservec`必须在内核页表中与用户页表中映射相同的地址。

xv6使用包含`uservec`的蹦床页面（trampoline page）来满足这些约束。xv6将蹦床页面映射到内核页表和每个用户页表中相同的虚拟地址。这个虚拟地址是`TRAMPOLINE`（如图2.3和图3.3所示）。蹦床内容在***trampoline.S\***中设置，并且（当执行用户代码时）`stvec`设置为`uservec` (***kernel/trampoline.S\***:16)。

当`uservec`启动时，所有32个寄存器都包含被中断代码所拥有的值。但是`uservec`需要能够修改一些寄存器，以便设置`satp`并生成保存寄存器的地址。RISC-V以`sscratch`寄存器的形式提供了帮助。`uservec`开始时的`csrrw`指令交换了`a0`和`sscratch`的内容。现在用户代码的`a0`被保存了；`uservec`有一个寄存器（`a0`）可以使用；`a0`包含内核以前放在`sscratch`中的值。

`uservec`的下一个任务是保存用户寄存器。在进入用户空间之前，内核先前将`sscratch`设置为指向一个每个进程的trapframe，该帧（除此之外）具有保存所有用户寄存器的空间(***kernel/proc.h\***:44)。因为`satp`仍然指向用户页表，所以`uservec`需要将trapframe映射到用户地址空间中。每当创建一个进程时，xv6就为该进程的trapframe分配一个页面，并安排它始终映射在用户虚拟地址`TRAPFRAME`，该地址就在`TRAMPOLINE`下面。尽管使用物理地址，该进程的`p->trapframe`仍指向trapframe，这样内核就可以通过内核页表使用它。

因此在交换`a0`和`sscratch`之后，`a0`持有指向当前进程trapframe的指针。`uservec`现在保存那里的所有用户寄存器，包括从`sscratch`读取的用户的`a0`。

陷阱帧包含指向当前进程内核栈的指针、当前CPU的`hartid`、`usertrap`的地址和内核页表的地址。`uservec`取得这些值，将`satp`切换到内核页表，并调用`usertrap`。

`usertrap`的任务是确定陷阱的原因，处理并返回(***kernel/trap.c\***:37)。如上所述，它首先改变`stvec`，这样内核中的陷阱将由`kernelvec`处理。它保存了`sepc`（保存的用户程序计数器），再次保存是因为`usertrap`中可能有一个进程切换，可能导致`sepc`被覆盖。如果陷阱来自系统调用，`syscall`会处理它；如果是设备中断，`devintr`会处理；否则它是一个异常，内核会杀死错误进程。系统调用路径在保存的用户程序计数器`pc`上加4，因为在系统调用的情况下，RISC-V会留下指向`ecall`指令的程序指针（返回后需要执行`ecall`之后的下一条指令）。在退出的过程中，`usertrap`检查进程是已经被杀死还是应该让出CPU（如果这个陷阱是计时器中断）。

返回用户空间的第一步是调用`usertrapret` (***kernel/trap.c\***:90)。该函数设置RISC-V控制寄存器，为将来来自用户空间的陷阱做准备。这涉及到将`stvec`更改为指向`uservec`，准备`uservec`所依赖的陷阱帧字段，并将`sepc`设置为之前保存的用户程序计数器。最后，`usertrapret`在用户和内核页表中都映射的蹦床页面上调用`userret`；原因是`userret`中的汇编代码会切换页表。

`usertrapret`对`userret`的调用将指针传递到`a0`中的进程用户页表和`a1`中的`TRAPFRAME` (***kernel/trampoline.S\***:88)。`userret`将`satp`切换到进程的用户页表。回想一下，用户页表同时映射蹦床页面和`TRAPFRAME`，但没有从内核映射其他内容。同样，蹦床页面映射在用户和内核页表中的同一个虚拟地址上的事实允许用户在更改`satp`后继续执行。`userret`复制陷阱帧保存的用户`a0`到`sscratch`，为以后与`TRAPFRAME`的交换做准备。从此刻开始，`userret`可以使用的唯一数据是寄存器内容和陷阱帧的内容。下一个`userret`从陷阱帧中恢复保存的用户寄存器，做`a0`与`sscratch`的最后一次交换来恢复用户`a0`并为下一个陷阱保存`TRAPFRAME`，并使用`sret`返回用户空间。

## 4.3 代码：调用系统调用

第2章以***initcode.S\***调用`exec`系统调用（***user/initcode.S\***:11）结束。让我们看看用户调用是如何在内核中实现`exec`系统调用的。

用户代码将`exec`需要的参数放在寄存器`a0`和`a1`中，并将系统调用号放在`a7`中。系统调用号与`syscalls`数组中的条目相匹配，`syscalls`数组是一个函数指针表（***kernel/syscall.c\***:108）。`ecall`指令陷入(trap)到内核中，执行`uservec`、`usertrap`和`syscall`，和我们之前看到的一样。

`syscall`（***kernel/syscall.c\***:133）从陷阱帧（trapframe）中保存的`a7`中检索系统调用号（`p->trapframe->a7`），并用它索引到`syscalls`中，对于第一次系统调用，`a7`中的内容是`SYS_exec`（***kernel/syscall. h\***:8），导致了对系统调用接口函数`sys_exec`的调用。

当系统调用接口函数返回时，`syscall`将其返回值记录在`p->trapframe->a0`中。这将导致原始用户空间对`exec()`的调用返回该值，因为RISC-V上的C调用约定将返回值放在`a0`中。系统调用通常返回负数表示错误，返回零或正数表示成功。如果系统调用号无效，`syscall`打印错误并返回-1。

## 4.4 系统调用参数

内核中的系统调用接口需要找到用户代码传递的参数。因为用户代码调用了系统调用封装函数，所以参数最初被放置在RISC-V C调用所约定的地方：寄存器。内核陷阱代码将用户寄存器保存到当前进程的陷阱框架中，内核代码可以在那里找到它们。函数`artint`、`artaddr`和`artfd`从陷阱框架中检索第n个**系统调用参数**并以整数、指针或文件描述符的形式保存。他们都调用`argraw`来检索相应的保存的用户寄存器（***kernel/syscall.c\***:35）。

有些系统调用传递指针作为参数，内核必须使用这些指针来读取或写入用户内存。例如：`exec`系统调用传递给内核一个指向用户空间中字符串参数的指针数组。这些指针带来了两个挑战。首先，用户程序可能有缺陷或恶意，可能会传递给内核一个无效的指针，或者一个旨在欺骗内核访问内核内存而不是用户内存的指针。其次，xv6内核页表映射与用户页表映射不同，因此内核不能使用普通指令从用户提供的地址加载或存储。

内核实现了安全地将数据传输到用户提供的地址和从用户提供的地址传输数据的功能。`fetchstr`是一个例子（***kernel/syscall.c\***:25）。文件系统调用，如`exec`，使用`fetchstr`从用户空间检索字符串文件名参数。`fetchstr`调用`copyinstr`来完成这项困难的工作。

`copyinstr`（***kernel/vm.c\***:406）从用户页表页表中的虚拟地址`srcva`复制`max`字节到`dst`。它使用`walkaddr`（它又调用`walk`）在软件中遍历页表，以确定`srcva`的物理地址`pa0`。由于内核将所有物理RAM地址映射到同一个内核虚拟地址，`copyinstr`可以直接将字符串字节从`pa0`复制到`dst`。`walkaddr`（***kernel/vm.c\***:95）检查用户提供的虚拟地址是否为进程用户地址空间的一部分，因此程序不能欺骗内核读取其他内存。一个类似的函数`copyout`，将数据从内核复制到用户提供的地址。

## 4.5 从内核空间陷入

xv6根据执行的是用户代码还是内核代码，对CPU陷阱寄存器的配置有所不同。当在CPU上执行内核时，内核将`stvec`指向`kernelvec`(***kernel/kernelvec.S\***:10)的汇编代码。由于xv6已经在内核中，`kernelvec`可以依赖于设置为内核页表的`satp`，以及指向有效内核栈的栈指针。`kernelvec`保存所有寄存器，以便被中断的代码最终可以不受干扰地恢复。

`kernelvec`将寄存器保存在被中断的内核线程的栈上，这是有意义的，因为寄存器值属于该线程。如果陷阱导致切换到不同的线程，那这一点就显得尤为重要——在这种情况下，陷阱将实际返回到新线程的栈上，将被中断线程保存的寄存器安全地保存在其栈上。

`Kernelvec`在保存寄存器后跳转到`kerneltrap`(***kernel/trap.c\***:134)。`kerneltrap`为两种类型的陷阱做好了准备：设备中断和异常。它调用`devintr`(***kernel/trap.c\***:177)来检查和处理前者。如果陷阱不是设备中断，则必定是一个异常，内核中的异常将是一个致命的错误；内核调用`panic`并停止执行。

如果由于计时器中断而调用了`kerneltrap`，并且一个进程的内核线程正在运行（而不是调度程序线程），`kerneltrap`会调用`yield`，给其他线程一个运行的机会。在某个时刻，其中一个线程会让步，让我们的线程和它的`kerneltrap`再次恢复。第7章解释了`yield`中发生的事情。

当`kerneltrap`的工作完成后，它需要返回到任何被陷阱中断的代码。因为一个`yield`可能已经破坏了保存的`sepc`和在`sstatus`中保存的前一个状态模式，因此`kerneltrap`在启动时保存它们。它现在恢复这些控制寄存器并返回到`kernelvec`(***kernel/kernelvec.S\***:48)。`kernelvec`从栈中弹出保存的寄存器并执行`sret`，将`sepc`复制到`pc`并恢复中断的内核代码。

值得思考的是，如果内核陷阱由于计时器中断而调用`yield`，陷阱返回是如何发生的。

当CPU从用户空间进入内核时，xv6将CPU的`stvec`设置为`kernelvec`；您可以在`usertrap`(***kernel/trap.c\***:29)中看到这一点。内核执行时有一个时间窗口，但`stvec`设置为`uservec`，在该窗口中禁用设备中断至关重要。幸运的是，RISC-V总是在开始设置陷阱时禁用中断，xv6在设置`stvec`之前不会再次启用中断

## 4.6 页面错误异常

Xv6对异常的响应相当无趣: 如果用户空间中发生异常，内核将终止故障进程。如果内核中发生异常，则内核会崩溃。真正的操作系统通常以更有趣的方式做出反应。

例如，许多内核使用页面错误来实现写时拷贝版本的`fork`——*copy on write (COW) fork*。要解释*COW fork*，请回忆第3章内容：xv6的`fork`通过调用`uvmcopy`(***kernel/vm.c\***:309) 为子级分配物理内存，并将父级的内存复制到其中，使子级具有与父级相同的内存内容。如果父子进程可以共享父级的物理内存，则效率会更高。然而武断地实现这种方法是行不通的，因为它会导致父级和子级通过对共享栈和堆的写入来中断彼此的执行。

由页面错误驱动的*COW fork*可以使父级和子级安全地共享物理内存。当CPU无法将虚拟地址转换为物理地址时，CPU会生成页面错误异常。Risc-v有三种不同的页面错误: 加载页面错误 (当加载指令无法转换其虚拟地址时)，存储页面错误 (当存储指令无法转换其虚拟地址时) 和指令页面错误 (当指令的地址无法转换时)。`scause`寄存器中的值指示页面错误的类型，`stval`寄存器包含无法翻译的地址。

COW fork中的基本计划是让父子最初共享所有物理页面，但将它们映射为只读。因此，当子级或父级执行存储指令时，risc-v CPU引发页面错误异常。为了响应此异常，内核复制了包含错误地址的页面。它在子级的地址空间中映射一个权限为读/写的副本，在父级的地址空间中映射另一个权限为读/写的副本。更新页表后，内核会在导致故障的指令处恢复故障进程的执行。由于内核已经更新了相关的PTE以允许写入，所以错误指令现在将正确执行。

COW策略对`fork`很有效，因为通常子进程会在`fork`之后立即调用`exec`，用新的地址空间替换其地址空间。在这种常见情况下，子级只会触发很少的页面错误，内核可以避免拷贝父进程内存完整的副本。此外，*COW fork*是透明的: 无需对应用程序进行任何修改即可使其受益。

除*COW fork*以外，页表和页面错误的结合还开发出了广泛有趣的可能性。另一个广泛使用的特性叫做惰性分配——*lazy allocation。*它包括两部分内容：首先，当应用程序调用`sbrk`时，内核增加地址空间，但在页表中将新地址标记为无效。其次，对于包含于其中的地址的页面错误，内核分配物理内存并将其映射到页表中。由于应用程序通常要求比他们需要的更多的内存，惰性分配可以称得上一次胜利: 内核仅在应用程序实际使用它时才分配内存。像COW fork一样，内核可以对应用程序透明地实现此功能。

利用页面故障的另一个广泛使用的功能是从磁盘分页。如果应用程序需要比可用物理RAM更多的内存，内核可以换出一些页面: 将它们写入存储设备 (如磁盘)，并将它们的PTE标记为无效。如果应用程序读取或写入被换出的页面，则CPU将触发页面错误。然后内核可以检查故障地址。如果该地址属于磁盘上的页面，则内核分配物理内存页面，将该页面从磁盘读取到该内存，将PTE更新为有效并引用该内存，然后恢复应用程序。为了给页面腾出空间，内核可能需要换出另一个页面。此功能不需要对应用程序进行更改，并且如果应用程序具有引用的地址 (即，它们在任何给定时间仅使用其内存的子集)，则该功能可以很好地工作。

结合分页和页面错误异常的其他功能包括自动扩展栈空间和内存映射文件。

# lab4

## RISC-V assembly

理解一点RISC-V汇编是很重要的，你应该在6.004中接触过。xv6仓库中有一个文件***user/call.c\***。执行`make fs.img`编译它，并在***user/call.asm\***中生成可读的汇编版本。

阅读***call.asm\***中函数`g`、`f`和`main`的代码。RISC-V的使用手册在[参考页](https://pdos.csail.mit.edu/6.828/2020/reference.html)上。以下是您应该回答的一些问题（将答案存储在***answers-traps.txt\***文件中）：

1. 哪些寄存器保存函数的参数？例如，在`main`对`printf`的调用中，哪个寄存器保存13？
2. `main`的汇编代码中对函数`f`的调用在哪里？对`g`的调用在哪里(提示：编译器可能会将函数内联）
3. `printf`函数位于哪个地址？
4. 在`main`中`printf`的`jalr`之后的寄存器`ra`中有什么值？
5. 运行以下代码。

```c
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```

程序的输出是什么？这是将字节映射到字符的[ASCII码表](http://web.cs.mun.ca/~michael/c/ascii-table.html)。

输出取决于RISC-V小端存储的事实。如果RISC-V是大端存储，为了得到相同的输出，你会把`i`设置成什么？是否需要将`57616`更改为其他值？

[这里有一个小端和大端存储的描述](http://www.webopedia.com/TERM/b/big_endian.html)和一个[更异想天开的描述](http://www.networksorcery.com/enp/ien/ien137.txt)。

1. 在下面的代码中，“`y=`”之后将打印什么(注：答案不是一个特定的值）？为什么会发生这种情况？

```c
printf("x=%d y=%d", 3);
```



- 在a0-a7中存放参数，13存放在a2中

- 在C代码中，main调用f，f调用g。而在生成的汇编中，main函数进行了内联优化处理。从代码`li a1,12`可以看出，main直接计算出了结果并储存

- 在`0x616`

- auipc`(Add Upper Immediate to PC)：`auipc rd imm`，将高位立即数加到PC上，从下面的指令格式可以看出，该指令将20位的立即数左移12位之后（右侧补0）加上PC的值，将结果保存到dest位置，图中为`rd`寄存器

![image-20230401144844984](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312131911.png)

​	下面来看`jalr` (jump and link register)：`jalr rd, offset(rs1)`跳转并链接寄存器。jalr指令会将当前PC+4	保存在rd中，然后跳转到指定的偏移地址`offset(rs1)`。

![image-20230401144821862](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312131631.png)

```
	30:	00000097          	auipc	ra,0x0
  34:	5e6080e7          	jalr	1510(ra) # 616 <printf>
```

第一列代表pc的值，第二列代表指令，最后一列是详细指令。

第一行pc值为0x30，指令是`00000097H=00...0 0000 1001 0111B`，对比指令格式，可见imm=0，dest=00001，opcode=0010111，对比汇编指令可知，auipc的操作码是0010111，ra寄存器代码是00001。这行代码将0x0左移12位（还是0x0）加到PC（当前为0x30）上并存入ra中，即ra中保存的是0x30。

第2行pc值为0x34，指令是`5e6080e7H=0101 1110 0110 0000 1000 0000 1110 0111B`，offset=0101 1110 0110，rs1=00001，rd=00001。rs1和rd都为寄存器ra。因此现在pc的值为x[ra]+offset即0x30+0x5e6=0x616,即printf的地址。并将PC+4=0x34+4=0x38保存在ra中

- ​	57616=0xE110，0x00646c72小端存储为72-6c-64-00，对照ASCII码表

72:r 6c:l 64:d 00:充当字符串结尾标识

因此输出为：HE110 World

若为大端存储，i应改为0x726c6400，不需改变57616

- 取决于寄存器a2保存的值

##  Backtrace



- 在kernel/defs.h中添加backtrace的原型。
- GCC编译器将当前正在执行的函数的帧指针保存在`s0`寄存器，将下面的函数添加到***kernel/riscv.h\***

```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

 并在`backtrace`中调用此函数来读取当前的帧指针。这个函数使用[内联汇编](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html)来读取`s0`

- 最后完成backtrace函数

  - 首先先来认识下帧栈（stack frame）

  ![image-20230403211008017](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312131503.png)

  每一次函数的调用,都会在调用栈(call stack)上维护一个独立的栈帧(stack frame).每个独立的栈帧一般包括:

  - 函数的返回地址和参数
  - 临时变量:  包括函数的非静态局部变量以及编译器自动生成的其他临时变量
  - 函数调用的上下文
    栈是从高地址向低地址延伸,一个函数的栈帧用ebp 和 esp 这两个寄存器来划定范围.ebp 指向当前的栈帧的底部,esp 始终指向栈帧的顶部;
    ebp 寄存器又被称为帧指针(Frame Pointer);
    esp 寄存器又被称为栈指针(Stack Pointer);
  - 在xv6中，返回地址位于frame pointer（fp）固定偏移-8的位置（fp-8,fp）。并且保存的前一个fp指针在固定偏移-16的位置（fp-16,fp-8）

  - xv6在内核中以页面对齐的地址为每个栈分配一个页面，所以可以通过PGROUNDUP/DOWN函数来判断帧栈页面是否有效。
  - 如果有效，那么分别获取返回地址(fp-8)和前一个fp指针的地址(fp-16)

- 最终可以打印函数返回地址

```
/**
 * @brief backtrace 回溯函数调用的返回地址
 */
void
backtrace(void) {
  printf("backtrace:\n");
  // 读取当前帧指针
  uint64 fp = r_fp();
  while (PGROUNDUP(fp) - PGROUNDDOWN(fp) == PGSIZE) {
    // 返回地址保存在-8偏移的位置
    uint64 ret_addr = *(uint64*)(fp - 8);
    printf("%p\n", ret_addr);
    // 前一个帧指针保存在-16偏移的位置
    fp = *(uint64*)(fp - 16);
  }
}

```

## Alarm

### test0

程序计数器的过程是这样的：

1. `ecall`指令中将PC保存到SEPC
2. 在`usertrap`中将SEPC保存到`p->trapframe->epc`
3. `p->trapframe->epc`加4指向下一条指令
4. 执行系统调用
5. 在`usertrapret`中将SEPC改写为`p->trapframe->epc`中的值
6. 在`sret`中将PC设置为SEPC的值

可见执行系统调用后返回到用户空间继续执行的指令地址是由`p->trapframe->epc`决定的，因此在`usertrap`中主要就是完成它的设置工作。

**(1)**. 在`struct proc`中增加字段，同时记得在`allocproc`中将它们初始化为0，并在`freeproc`中也设为0

```c
int alarm_interval;          // 报警间隔
void (*alarm_handler)();     // 报警处理函数
int ticks_count;             // 两次报警间的滴答计数
```

**(2)**. 在`sys_sigalarm`中读取参数

```c
uint64
sys_sigalarm(void) {
  if(argint(0, &myproc()->alarm_interval) < 0 ||
    argaddr(1, (uint64*)&myproc()->alarm_handler) < 0)
    return -1;

  return 0;
}
```

**(3)**. 修改usertrap()，将函数指针赋值给trapframe->epc，然后将ticks_count改为0

```c
// give up the CPU if this is a timer interrupt.
if(which_dev == 2) {
    if(++p->ticks_count == p->alarm_interval) {
        // 更改陷阱帧中保留的程序计数器
        p->trapframe->epc = (uint64)p->alarm_handler;
        p->ticks_count = 0;
    }
    yield();
}
```

### Test1&test2

考虑一下没有alarm时运行的大致过程

1. 进入内核空间，保存用户寄存器到进程陷阱帧
2. 陷阱处理过程
3. 恢复用户寄存器，返回用户空间

而当添加了alarm后，变成了以下过程

1. 进入内核空间，保存用户寄存器到进程陷阱帧
2. 陷阱处理过程
3. 恢复用户寄存器，返回用户空间，但此时返回的并不是进入陷阱时的程序地址，而是处理函数`handler`的地址（因为修改了trapframe的epc寄存器），而`handler`可能会改变用户寄存器（比如trapframe的epc寄存器）

因此我们要在`usertrap`中再次保存用户寄存器，当`handler`调用`sigreturn`时将其恢复，并且要防止在`handler`执行过程中重复调用，过程如下

**(1)**. 再在`struct proc`中新增两个字段

```c
int is_alarming;                    // 是否正在执行告警处理函数
struct trapframe* alarm_trapframe;  // 告警陷阱帧
```

**(2)**. 在allocproc和freeproc中设定好相关分配，回收内存的代码

```c
/**
 * allocproc.c
 */
// 初始化告警字段
if((p->alarm_trapframe = (struct trapframe*)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
}
p->is_alarming = 0;
p->alarm_interval = 0;
p->alarm_handler = 0;
p->ticks_count = 0;

/**
 * freeproc.c
 */
if(p->alarm_trapframe)
    kfree((void*)p->alarm_trapframe);
p->alarm_trapframe = 0;
p->is_alarming = 0;
p->alarm_interval = 0;
p->alarm_handler = 0;
p->ticks_count = 0;
```

**(3)**. 更改usertrap函数，保存进程陷阱帧`p->trapframe`到`p->alarm_trapframe`

```c
// give up the CPU if this is a timer interrupt.
if(which_dev == 2) {
  if(p->alarm_interval != 0 && ++p->ticks_count == p->alarm_interval && p->is_alarming == 0) {
    // 保存寄存器内容
    memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
    // 更改陷阱帧中保留的程序计数器，注意一定要在保存寄存器内容后再设置epc
    p->trapframe->epc = (uint64)p->alarm_handler;
    p->ticks_count = 0;
    p->is_alarming = 1;
  }
  yield();
}
```

**(4)**. 更改`sys_sigreturn`，恢复陷阱帧

```c
uint64
sys_sigreturn(void) {
  memmove(myproc()->trapframe, myproc()->alarm_trapframe, sizeof(struct trapframe));
  myproc()->is_alarming = 0;
  return 0;
}
```

