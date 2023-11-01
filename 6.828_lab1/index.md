# 6.828_lab1


# lab1

## part1 PC Bootstrap

实验分为三个部分：

- 熟悉汇编语言、QEMU x86模拟器、PC上电启动过程
- 检查我们的6.828内核的boot loader程序，它位于`lab`的`boot`目录下。
- 深入研究6.828内核本身的初始模板，位于`kernel`目录下。

使用qemu编译



```c
$ make qemu-nox-gdb
***
*** Now run 'make gdb'.
***
qemu-system-i386 -nographic -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::26002 -D qemu.log  -S
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 
```

使用help和kerninfo两个命令

```c
K> help
help - Display this list of commands
kerninfo - Display information about the kernel
K> kerninfo
Special kernel symbols:
  _start                  0010000c (phys)
  entry  f010000c (virt)  0010000c (phys)
  etext  f0101acd (virt)  00101acd (phys)
  edata  f0113060 (virt)  00113060 (phys)
  end    f01136a0 (virt)  001136a0 (phys)
Kernel executable memory footprint: 78KB
K> 
```

### The PC’s Physical Address Space

接下来会介绍PC的启动。一个PC的物理地址空间可以分成以下组成。

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



为了兼容性的考虑。PC在一开始是16位的。但是地址线却有20位。也就是能够寻址1MB的地址空间。其中640KB为低端内存。
这段内容非常重要，所以需要好好注意。Lab2内存分配的时候会用到这个。
除去低端的640KB。那么1MB还留下 `1024KB - 640KB = 384KB`。这`384KB`的范围就是
`0x000A0000 ~ 0x000FFFFF`。

其中BIOS占掉了顶端的`64KB`的内存。尽管后来内存从1MB前进到了16MB，后来又进展到了4GB。但是PC的内存布局还是没有改变。主要是为了兼容性考虑。因此，32位的CPU在这里还是会有个洞。`0x000A0000 〜 0x00100000`。

原本低端内存可以连续的1MB，变成了两段

- 0~640KB，
- 然后1MB〜更高的内存。

即

- “conventional memory” (the first 640KB)
- “extended memory” 1MB以上

最新的x86架构可以支持4GB以上的物理内存了。所以RAM也可以扩展到0xFFFFFFFF以上的地址。在这种情况下BIOS需要设置第二个洞。也就是在32位地址的顶端。但是JOS目前来说，只是支持256MB的物理内存。所以这里设计时只考虑到了具有32位地址的地址空间的情况。

### The ROM BIOS

接下来的操作里面，你会用到QEMU的debug功能来深入了解IA-32计算机的启动流程。
需要做以下事情：
打开两个termainal

一个窗口运行`make qemu-nox-gdb`
另外一个窗口运行`make gdb`

```
# make gdb
GNU gdb (GDB) 6.8-debian
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
+ target remote localhost:26000
The target architecture is assumed to be i8086
[f000:fff0] 0xffff0:    ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb)
```

这里能够通过gdb一下子就连接上来，这是因为提供了一个`.gdbinit`文件，能够自动地attach到想要调试的程序上来。当然前提是已经把这个debug的程序运行起来的情况。

PC中BIOS大小为64k, 物理地址范围0x000f0000-0x000fffff
PC 开机首先0xfffff0处执行 `jmp [0xf000,0xe05b]` 指令。在gdb中使用`si(Step Instruction)`进行跟踪。第一条要执行的指令就是`ljmp`。

```
[f000:fff0] 0xffff0:    ljmp   $0xf000,$0xe05b
```

```c
(gdb) si
[f000:e05b]    0xfe05b:    cmpw   $0xffc8,%cs:(%esi)   # 比较大小，改变PSW
0x0000e05b in ?? ()
(gdb) si
[f000:e062]    0xfe062:    jne    0xd241d416           # 不相等则跳转
0x0000e062 in ?? ()
(gdb) si
[f000:e066]    0xfe066:    xor    %edx,%edx            # 清零edx
0x0000e066 in ?? ()
(gdb) si
[f000:e068]    0xfe068:    mov    %edx,%ss
0x0000e068 in ?? ()
(gdb) si
[f000:e06a]    0xfe06a:    mov    $0x7000,%sp
0x0000e06a in ?? ()
```

从这个要执行的指令可以看出来。

IBM PC开始执行的物理位置是`0x000ffff0`。这个是位于1MB里面的很高的地址。也就是ROM BIOS最顶上`64KB`的顶部。

```
[f000:fff0]可以看出来，此时CS = 0xf000 and IP = 0xfff0.
```

如果执行完这条指令之后 `CS = 0xf000 and IP = 0xe05b`.
为什么QEMU一开始执行的时候是这样的？这是因为8088的芯片就是这样的。
在IBM最原始的PC就是这么使用的。总之一句话，当PC上电之后。CS:IP两个寄存器就强制被设置为这个值。`CS = 0xf000 and IP = 0xfff0`。

注意16位的寻址模式

```
: physical address = 16 * segment + offset. 
16 * 0xf000 + 0xfff0   # in hex multiplication by 16 is
   = 0xf0000 + 0xfff0     # easy--just append a 0.
   = 0xffff0
```

1MB尾巴上的地址就是`0xffff0 + 16bytes`。除了放个ljmp之外，你也不要指望16bytes能做啥了。

当 BIOS 运行时，它会建立一个中断描述符表并初始化各种设备，例如 VGA 显示器。这就是您在 QEMU 窗口中看到的 “ `Starting SeaBIOS ”消息的来源。`

初始化 PCI 总线和 BIOS 知道的所有重要设备后，它会搜索可引导设备，如软盘、硬盘或 CD-ROM。最终，当它找到可引导磁盘时，BIOS从磁盘 读取*引导加载程序并将控制权转移给它。*

## Part2 The BootLoader

软盘和磁盘一般都是被切分为512 byte区域，也就是扇区。一个扇区是一个块设备的最小传输单位。每次读写都是必须是扇区的整数倍。
如果这个软盘或者磁盘是可启动的。那么第一个扇区被叫做可启动扇区。这里也存放的就是可启动的代码。当BIOS找到这个扇区的时候，就把这512 byte读到0x7c00至0x7cff。然后用一个跳转指令

```
ljmp 0x07c0:0000
```

对于CD-ROM的支持，需要看 “El Torito” Bootable CD-ROM Format Specification.
对于6.828来说，由于完全是使用硬盘来启动的。所以在硬盘的开始必须是boot loader，并且这个boot loader必须是512 bytes大小。
这个boot loader主要是由两个文件构成

```
boot/boot.S
boot/main.c
```

这里需要好好地读一下这个文件。然后知道这两个文件做了些啥。
boot loader把模式切到了32位保护模式。因为只有在这种模式下软件才可以访问1MB+以上的内存空间。保护模式在`1.2.7` and `1.2.8` of `PC Assembly Language` 进行了介绍。 `Intel architecture manuals`也对这个有详细介绍。

在16位模式下只需要考虑段地址。
其次，需要注意的是`boot loader`读了kernel。从硬盘到内存。在操作的时候走的是PC的寄存器操作。如果想要了解更多，可以读一下`"IDE hard drive controller"` in `the 6.828 reference page`.的这一部分。

当你理解了`boot loader`的源码之后。接下来可以看一下`obj/boot/boot.asm`。b *0x7c00`就可以把断点设置在`0x7c00`。

```
b *0x7c00 # 设置断点在0x7c00
si 表示单步执行
si 2 表示接着执行两条指令
c 表示不再单步执行。直接开始运行了
```

查看内存中的指令，有时候可能需要查看内存操作的结果。这个时候需要用

```
x/i 调试命令
x/Ni 基中N是指令的数目; 会把指定内存里面的指令翻译成汇编。
```

### 实模式和保护模式

实模式和保护模式都是CPU的工作模式，而CPU的工作模式是指CPU的寻址方式、寄存器大小等用来反应CPU在该环境下如何工作的概念。

1.实模式工作原理

实模式出现于早期8088CPU时期。当时由于CPU的性能有限，一共只有20位地址线（所以地址空间只有1MB），以及8个16位的通用寄存器，以及4个16位的段寄存器。所以为了能够通过这些16位的寄存器去构成20位的主存地址，必须采取一种特殊的方式。当某个指令想要访问某个内存地址时，它通常需要用下面的这种格式来表示：

　　(段基址：段偏移量)

　 其中第一个字段是段基址，它的值是由**段寄存器**提供的(一般来说，段寄存器有6种，分别为cs，ds，ss，es，fs，gs，这几种段寄存器都有自己的特殊意义，这里不做介绍)。

　 第二字段是段内偏移量，代表你要访问的这个内存地址距离这个段基址的偏移。它的值就是由通用寄存器来提供的，所以也是16位。那么两个16位的值如何组合成一个20位的地址呢？CPU采用的方式是把段寄存器所提供的段基址先向左移4位。这样就变成了一个20位的值，然后再与段偏移量相加。

即：

　　物理地址 = 段基址<<4 + 段内偏移

　　所以假设段寄存器中的值是0xff00，段偏移量为0x0110。则这个地址对应的真实物理地址是 0xff00<<4 + 0x0110 = 0xff110。

由上面的介绍可见，实模式的"实"更多地体现在其地址是真实的物理地址。

2.保护模式工作原理

随着CPU的发展，CPU的地址线的个数也从原来的20根变为现在的32根，所以可以访问的内存空间也从1MB变为现在4GB，寄存器的位数也变为32位。所以实模式下的内存地址计算方式就已经不再适合了。所以就引入了现在的保护模式，实现更大空间的，更灵活也**更安全**的内存访问。

在保护模式下，CPU的32条地址线全部有效，可寻址高达4G字节的物理地址空间; 但是我们的内存寻址方式还是得兼容老办法(这也是没办法的，有时候是为了方便，有时候是一种无奈)，即(段基址：段偏移量)的表示方式。当然此时CPU中的通用寄存器都要换成32位寄存器(除了段寄存器，原因后面再说)来保证寄存器能访问所有的4GB空间。

我们的偏移值和实模式下是一样的，就是变成了32位而已，而段值仍旧是存放在原来16位的段寄存器中，**但是这些段寄存器存放的却不再是段基址了**，毕竟之前说过实模式下寻址方式不安全，我们在保护模式下需要加一些限制，而这些限制可不是一个寄存器能够容纳的，于是我们把这些关于内存段的限制信息放在一个叫做**全局描述符表(GDT)**的结构里。全局描述符表中含有一个个表项，每一个表项称为**段描述符。**而段寄存器在保护模式下存放的便是相当于一个数组索引的东西，通过这个索引，可以找到对应的表项。段描述符存放了段基址、段界限、内存段类型属性(比如是数据段还是代码段,注意**一个段描述符只能用来定义一个内存段**)等许多属性,具体信息见下图：

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312111104.jpeg)

其中，段界限表示段边界的扩张最值，即最大扩展多少或最小扩展多少，用20位来表示，它的单位可以是字节，也可以是4KB，这是由G位决定的(G为1时表示单位为4KB)。

实际段界限边界值=(描述符中的段界限+1)*（段界限的单位大小(即字节或4KB))-1，如果偏移地址超过了段界限，CPU会抛出异常。

全局描述符表位于内存中，需要用专门的寄存器指向它后， CPU 才知道它在哪里。这个专门的寄存器便是**GDTR**(一个48位的寄存器),专门用来存储 GDT 的内存地址及大小。

### Boot.S&main.c

设置一个断点在地址0x7c00处，这是boot sector被加载的位置。然后让程序继续运行直到这个断点。跟踪/boot/boot.S文件的每一条指令，同时使用boot.S文件和系统为你反汇编出来的文件obj/boot/boot.asm。你也可以使用GDB的x/i指令来获取去任意一个机器指令的反汇编指令，把源文件boot.S文件和boot.asm文件以及在GDB反汇编出来的指令进行比较。

  追踪到bootmain函数中，而且还要具体追踪到readsect()子函数里面。找出和readsect()c语言程序的每一条语句所对应的汇编指令，回到bootmain()，然后找出把内核文件从磁盘读取到内存的那个for循环所对应的汇编语句。找出当循环结束后会执行哪条语句，在那里设置断点，继续运行到断点，然后运行完所有的剩下的语句。

　答：

　　下面我们将分别分析一下这道练习中所涉及到的两个重要文件，它们一起组成了boot loader。分别是**/boot/boot.S**和**/boot/main.c**文件。其中前者是一个汇编文件，后者是一个C语言文件。当BIOS运行完成之后，CPU的控制权就会转移到boot.S文件上。所以我们首先看一下boot.S文件。

　　**/boot/boot.S：**

```
1 .globl start
2 start:
3   .code16                # Assemble for 16-bit mode
4   cli                    # Disable interrupts
```

　　这几条指令就是boot.S最开始的几句，其中cli是boot.S，也是boot loader的第一条指令。这条指令用于把所有的中断都关闭。因为在BIOS运行期间有可能打开了中断。此时CPU工作在实模式下。

```
5  cld                         # String operations increment
```

　　这条指令用于指定之后发生的串处理操作的指针移动方向。在这里现在对它大致了解就够了。

```
6  # Set up the important data segment registers (DS, ES, SS).
7  xorw    %ax,%ax             # Segment number zero
8  movw    %ax,%ds             # -> Data Segment
9  movw    %ax,%es             # -> Extra Segment
10 movw    %ax,%ss             # -> Stack Segment
```

　　这几条命令主要是在把三个段寄存器，ds，es，ss全部清零，因为经历了BIOS，操作系统不能保证这三个寄存器中存放的是什么数。所以这也是为后面进入保护模式做准备。

```
11  # Enable A20:
12  #   For backwards compatibility with the earliest PCs, physical
13  #   address line 20 is tied low, so that addresses higher than
14  #   1MB wrap around to zero by default.  This code undoes this.
15 seta20.1:
16  inb     $0x64,%al               # Wait for not busy
17  testb   $0x2,%al
18  jnz     seta20.1

19  movb    $0xd1,%al               # 0xd1 -> port 0x64
20  outb    %al,$0x64

21 seta20.2:
22  inb     $0x64,%al               # Wait for not busy
23  testb   $0x2,%al
24  jnz     seta20.2

25  movb    $0xdf,%al               # 0xdf -> port 0x60
26  outb    %al,$0x60
```

　　这部分指令就是在准备把CPU的工作模式从实模式转换为保护模式。我们可以看到其中的指令包括inb，outb这样的IO端口命令。所以这些指令都是在对外部设备进行操作。根据下面的链接：

　　 http://bochs.sourceforge.net/techspec/PORTS.LST

　　我们可以查看到，0x64端口属于键盘控制器804x，名称是控制器读取状态寄存器。下面是它各个位的含义。

　　![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312111273.jpeg)

　　所以16~18号指令是在不断的检测bit1。bit1的值代表输入缓冲区是否满了，也就是说CPU传送给控制器的数据，控制器是否已经取走了，如果CPU想向控制器传送新的数据的话，必须先保证这一位为0。所以这三条指令会一直等待这一位变为0，才能继续向后运行。

　　当0x64端口准备好读入数据后，现在就可以写入数据了，所以19~20这两条指令是把0xd1这条数据写入到0x64端口中。当向0x64端口写入数据时，则代表向键盘控制器804x发送指令。这个指令将会被送给0x60端口。

　　![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305082224333.jpg)

　　通过图中可见，D1指令代表下一次写入0x60端口的数据将被写入给804x控制器的输出端口。可以理解为下一个写入0x60端口的数据是一个控制指令。

　　然后21~24号指令又开始再次等待，等待刚刚写入的指令D1，是否已经被读取了。

　　如果指令被读取了，25~26号指令会向控制器输入新的指令，0xdf。通过查询我们看到0xDF指令的含义如下

　　![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305082224257.jpg)

　　这个指令的含义可以从图中看到，使能A20线，==代表可以进入保护模式==了。

```
27   # Switch from real to protected mode, using a bootstrap GDT
28   # and segment translation that makes virtual addresses 
29   # identical to their physical addresses, so that the 
30   # effective memory map does not change during the switch.
31   lgdt    gdtdesc
32   movl    %cr0, %eax
33   orl     $CR0_PE_ON, %eax
34   movl    %eax, %cr0
```

　　首先31号指令 lgdt gdtdesc，是把gdtdesc这个标识符的值送入全局映射描述符表寄存器GDTR中。这个GDT表是处理器工作于保护模式下一个非常重要的表。具体可以参照我们的Appendix 1关于实模式和保护模式的介绍。至于这条指令的功能就是把关于GDT表的一些重要信息存放到CPU的GDTR寄存器中，其中包括GDT表的内存起始地址，以及GDT表的长度。这个寄存器由48位组成，其中低16位表示该表长度，高32位表该表在内存中的起始地址。所以gdtdesc是一个标识符，标识着一个内存地址。从这个内存地址开始之后的6个字节中存放着GDT表的长度和起始地址。我们可以在这个文件的末尾看到gdtdesc，如下：

```
 1 # Bootstrap GDT
 2 .p2align 2                               # force 4 byte alignment
 3 gdt:
 4   SEG_NULL                               # null seg
 5   SEG(STA_X|STA_R, 0x0, 0xffffffff)      # code seg
 6   SEG(STA_W, 0x0, 0xffffffff)            # data seg
 7 
 8 gdtdesc:
 9   .word   0x17                           # sizeof(gdt) - 1
10   .long   gdt                            # address gdt
```

　　其中第3行的gdt是一个标识符，标识从这里开始就是GDT表了。可见这个GDT表中包括三个表项(4,5,6行)，分别代表三个段，null seg，code seg，data seg。由于xv6其实并没有使用分段机制，也就是说数据和代码都是写在一起的，所以数据段和代码段的起始地址都是0x0，大小都是0xffffffff=4GB。

　　在第4~6行是调用SEG()子程序来构造GDT表项的。这个子函数定义在mmu.h中，形式如下：　　

```
　#define SEG(type,base,lim)                    \
                    .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);    \
                    .byte (((base) >> 16) & 0xff), (0x90 | (type)),        \
                    (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```

 　可见函数需要3个参数，一是type即这个段的访问权限，二是base，这个段的起始地址，三是lim，即这个段的大小界限。gdt表中的每一个表项的结构如图所示：

　　![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305082224651.jpg)

　　 每个表项一共8字节，其中limit_low就是limit的低16位。base_low就是base的低16位，依次类推，所以我们就可以理解SEG函数为什么要那么写（其实还是有很多不理解的。。）。

　　 然后在gdtdesc处就要存放这个GDT表的信息了，其中0x17是这个表的大小-1 = 0x17 = 23，至于为什么不直接存表的大小24，根据查询是官方规定的。紧接着就是这个表的起始地址gdt。

```
27   # Switch from real to protected mode, using a bootstrap GDT
28   # and segment translation that makes virtual addresses 
29   # identical to their physical addresses, so that the 
30   # effective memory map does not change during the switch.
31   lgdt    gdtdesc
32   movl    %cr0, %eax
33   orl     $CR0_PE_ON, %eax
34   movl    %eax, %cr0
```

　　再回到刚才那里，当加载完GDT表的信息到GDTR寄存器之后。紧跟着3个操作，32~34指令。 这几步操作明显是在修改CR0寄存器的内容。CR0寄存器还有CR1~CR3寄存器都是80x86的控制寄存器。其中$CR0_PE的值定义于"mmu.h"文件中，为0x00000001。可见上面的操作是把CR0寄存器的bit0置1，CR0寄存器的bit0是保护模式启动位，把这一位值1代表==保护模式启动==。

```
35  ljmp    $PROT_MODE_CSEG, $protcseg
```

　　这只是一个简单的跳转指令，这条指令的目的在于把当前的运行模式切换成32位地址模式

```
protcseg:
  # Set up the protected-mode data segment registers
36  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
37  movw    %ax, %ds                # -> DS: Data Segment
38  movw    %ax, %es                # -> ES: Extra Segment
39  movw    %ax, %fs                # -> FS
40  movw    %ax, %gs                # -> GS
41  movw    %ax, %ss                # -> SS: Stack Segment
```

　　  修改这些寄存器的值。这些寄存器都是段寄存器。大家可以戳这个链接看一下具体介绍 http://www.eecg.toronto.edu/~amza/[www.mindsec.com/files/x86regs.html](http://www.mindsec.com/files/x86regs.html)

　　 这里的23~29步之所以这么做是按照规定来的，https://en.wikibooks.org/wiki/X86_Assembly/Global_Descriptor_Table链接中指出，如果刚刚加载完GDTR寄存器我们必须要重新加载所有的段寄存器的值，而其中CS段寄存器必须通过长跳转指令，即23号指令来进行加载。所以这些步骤是在第19步完成后必须要做的。这样才能是GDTR的值生效。

 

```
# Set up the stack pointer and call into C.
42  movl    $start, %esp
43  call bootmain
```

　　接下来的指令就是要设置当前的esp寄存器的值，然后准备正式跳转到main.c文件中的bootmain函数处。我们接下来分析一下这个函数的每一条指令：

```
// read 1st page off disk
1 readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);
```

　 这里面调用了一个函数readseg，这个函数在bootmain之后被定义了：

  void readseg(uchar *pa, uint count, uint offset);**
**

　 它的功能从注释上来理解应该是，把距离内核起始地址offset个偏移量存储单元作为起始，将它和它之后的count字节的数据读出送入以pa为起始地址的内存物理地址处。

　 所以这条指令是把内核的第一个页(4MB = 4096 = SECTSIZE*8 = 512*8)的内容读取的内存地址ELFHDR(0x10000)处。其实完成这些后相当于把操作系统映像文件的elf头部读取出来放入内存中。

　 读取完这个内核的elf头部信息后，需要对这个elf头部信息进行验证，并且也需要通过它获取一些重要信息。所以有必要了解下elf头部。

------

　 注： http://wiki.osdev.org/ELF

  elf文件：elf是一种文件格式，主要被用来把程序存放到磁盘上。是在程序被编译和链接后被创建出来的。一个elf文件包括多个段。对于一个可执行程序，通常包含存放代码的文本段(text section)，存放全局变量的data段，存放字符串常量的rodata段。elf文件的头部就是用来描述这个elf文件如何在存储器中存储。

  需要注意的是，你的文件是可链接文件还是可执行文件，会有不同的elf头部格式。

 

```
2 if (ELFHDR->e_magic != ELF_MAGIC)
3        goto bad;
```

　 elf头部信息的magic字段是整个头部信息的开端。并且如果这个文件是格式是ELF格式的话，文件的elf->magic域应该是=ELF_MAGIC的，所以这条语句就是判断这个输入文件是否是合法的elf可执行文件。

```
4 ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
```

　 我们知道头部中一定包含Program Header Table。这个表格存放着程序中所有段的信息。通过这个表我们才能找到要执行的代码段，数据段等等。所以我们要先获得这个表。

　 这条指令就可以完成这一点，首先elf是表头起址，而phoff字段代表Program Header Table距离表头的偏移量。所以ph可以被指定为Program Header Table表头。

　

```
5 eph = ph + ELFHDR->e_phnum;
```

　 由于phnum中存放的是Program Header Table表中表项的个数，即段的个数。所以这步操作是吧eph指向该表末尾。

```c
6 for (; ph < eph; ph++)
    // p_pa is the load address of this segment (as well
    // as the physical address)
7    readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```

　 这个for循环就是在加载所有的段到内存中。ph->paddr根据参考文献中的说法指的是这个段在内存中的物理地址。ph->off字段指的是这一段的开头相对于这个elf文件的开头的偏移量。ph->filesz字段指的是这个段在elf文件中的大小。ph->memsz则指的是这个段被实际装入内存后的大小。通常来说memsz一定大于等于filesz，因为段在文件中时许多未定义的变量并没有分配空间给它们。

　  所以这个循环就是在把操作系统内核的各个段从外存读入内存中。

 

```
8 ((void (*)(void)) (ELFHDR->e_entry))();
```

　 e_entry字段指向的是这个文件的执行入口地址。所以这里相当于开始运行这个文件。也就是内核文件。 自此就把控制权从boot loader转交给了操作系统的内核。

### exercise3

```
Q：处理器什么时候开始执行 32 位代码？究竟是什么导致从 16 位模式切换到 32 位模式？

A：movl    %eax, %cr0
```

如果指令被读取了，25~26号指令会向控制器输入新的指令，0xdf。通过查询我们看到0xDF指令的含义如下

　　![img](../../../../study/6.828/lab1.assets/809277-20160108222031012-642740036.jpg)

这个指令的含义可以从图中看到，使能A20线，==代表可以进入保护模式==了。

```
movb    $0xdf,%al               # 0xdf -> port 0x60
outb    %al,$0x60
```

31指令当加载完GDT表的信息到GDTR寄存器之后。紧跟着3个操作，32~34指令。 这几步操作明显是在修改CR0寄存器的内容。CR0寄存器还有CR1~CR3寄存器都是80x86的控制寄存器。其中$CR0_PE的值定义于"mmu.h"文件中，为0x00000001。可见上面的操作是把CR0寄存器的bit0置1，CR0寄存器的bit0是保护模式启动位，把这一位值1代表==保护模式启动==。

```
27   # Switch from real to protected mode, using a bootstrap GDT
28   # and segment translation that makes virtual addresses 
29   # identical to their physical addresses, so that the 
30   # effective memory map does not change during the switch.
31   lgdt    gdtdesc
32   movl    %cr0, %eax
33   orl     $CR0_PE_ON, %eax
34   movl    %eax, %cr0
```



```
Q：引导加载程序执行的最后一条指令 是什么，它刚刚加载的内核的第一条指令是什么？

A：((void (*)(void)) (ELFHDR->e_entry))();
自此bootloader就把控制权转交给了OS。查看boot.asm文件找到bootloader，	
((void (*)(void)) (ELFHDR->e_entry))();
    7d81:	ff 15 18 00 01 00    	call   *0x10018
随后通过在gdb中输入b *0x7d81在0x7d81处打上断点，随后c再si，查看加载内核的第一条指令。
0x10000c:    movw   $0x1234,0x472
```



```
Q：内核的第一条指令在哪里？

A：0x10000c:    movw   $0x1234,0x472   第一条指令在0x10000c处。
也可以通过objdump -f obj/kern/kernel查看
obj/kern/kernel:     file format elf32-i386
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c
```



```
Q：引导加载程序如何决定它必须读取多少个扇区才能从磁盘中获取整个内核？它在哪里找到这些信息？

A：详细的将涉及到ELF格式，从ELF头部中获取到这部分信息
```

```
4 ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
```

　 我们知道头部中一定包含Program Header Table。这个表格存放着程序中所有段的信息。通过这个表我们才能找到要执行的代码段，数据段等等。所以我们要先获得这个表。

　 这条指令就可以完成这一点，首先elf是表头起址，而phoff字段代表Program Header Table距离表头的偏移量。所以ph可以被指定为Program Header Table表头。

```
5 eph = ph + ELFHDR->e_phnum;
```

　 由于phnum中存放的是Program Header Table表中表项的个数，即段的个数。所以这步操作是吧eph指向该表末尾。

```
6 for (; ph < eph; ph++)
    // p_pa is the load address of this segment (as well
    // as the physical address)
7    readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```

　 这个for循环就是在加载所有的段到内存中。ph->paddr根据参考文献中的说法指的是这个段在内存中的物理地址。ph->off字段指的是这一段的开头相对于这个elf文件的开头的偏移量。ph->filesz字段指的是这个段在elf文件中的大小。ph->memsz则指的是这个段被实际装入内存后的大小。通常来说memsz一定大于等于filesz，因为段在文件中时许多未定义的变量并没有分配空间给它们。

　  所以这个循环就是在把操作系统内核的各个段从外存读入内存中。

### Loading the kernel

### exercise4

ELF程序文是由一个固定长度的ELF头开始的。紧接着的是一个动态可变的程序头list。每个程序头，指明了每个程序段需要被加载的位置，长度，相对于整个程序的偏移量。

程序头list

```c
struct Proghdr {
	uint32_t p_type;
	uint32_t p_offset;
	uint32_t p_va;
	uint32_t p_pa;
	uint32_t p_filesz;
	uint32_t p_memsz;
	uint32_t p_flags;
	uint32_t p_align;
};

```

**注意：相对的是整个程序头的偏移量，而不是相对于磁盘头的偏移量**

由于在编译完成之后，才会把kernel写入磁盘的某个扇区。在编译的时候是无法知道会被写入到哪个扇区的。所以编译的时候只能说把相对的位置写入到ELF里面。
这些各种程序头比较常见的有以下几个：

```
.text: The program's executable instructions.
.rodata: Read-only data, such as ASCII string constants produced by the C compiler. (We will not bother setting up the hardware to prohibit writing, however.)
.data: The data section holds the program's initialized data, such as global variables declared with initializers like int x = 5;.
```

当链接器在计算内存部局的时候，也会保留足够的空间给各种未初始化的全局变量（初始化为0）。一般而言这个空间被称之为.bss段。
由于全部都是被设置为0.所以也就没有必要记录这些内容在ELF里面。所以ELF文件里面只需要记住.bss在内存里面的起始位置，以及大小。然后加载方需要确保这些.bss在内存里面正确的设置。以及初始化为0。
可以通过如下命令来查看`obj/kern/kernel`里面的`names, sizes`, 以及各种链接地址。

```
$ objdump -h obj/kern/kernel
obj/kern/kernel:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001917  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       00000714  f0101920  00101920  00002920  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00003889  f0102034  00102034  00003034  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000018af  f01058bd  001058bd  000068bd  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .bss          00000644  f0112300  00112300  00013300  2**5
                  ALLOC
  6 .comment      0000002b  00000000  00000000  00013300  2**0
                  CONTENTS, READONLY
```

真正要了解这个文件，需要查看链接设定文件：

```
kern/kernel.ld
```

通过这个文件可以知道程序被加载的虚拟地址(VMA)，物理地址(LMA)分别是如何指定的。也可以通过File off查看相对文件的偏移量。这个File off偏移量是如何指定的？这个非常有意思。刚好在boot/main.c里面就是一开始就读了了8个扇区，也就是`0x1000 bytes`。

#### ELF文件格式简图

kernel在编译和链接的时候，其虚拟地址与物理地址是不一样的。在加载的时候，也就相应地需要设置好页表。

但是考虑一下`boot loader`。在加载的时候，肯定是没有什么页表等着给你用的。`BIOS`可不会给你设置页表。所以

```
$ objdump -h obj/boot/boot.out
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  # 注意这里的VMA与LMA是完全一致的。这是由于被加载进BIOS的时候，没有页表与段表可用。
  # size = 380 bytes.
  # 这里面就包含了boot loader所需要的所有信息。
  0 .text         0000017c  00007c00  00007c00  00000074  2**2
                  CONTENTS, ALLOC, LOAD, CODE

  # size = 176这个段实际上是没有什么用的？
  1 .eh_frame     000000b0  00007d7c  00007d7c  000001f0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  # 接下来的三个段，里面都是用于DEBUG的。所以真实用的时候，不会被用到。
  2 .stab         000007b0  00000000  00000000  000002a0  2**2
                  CONTENTS, READONLY, DEBUGGING
  3 .stabstr      00000846  00000000  00000000  00000a50  2**0
                  CONTENTS, READONLY, DEBUGGING
  4 .comment      0000002b  00000000  00000000  00001296  2**0
                  CONTENTS, READONLY

$ objdump -x obj/boot/boot.out
obj/boot/boot.out:     file format elf32-i386
obj/boot/boot.out
architecture: i386, flags 0x00000012:
EXEC_P, HAS_SYMS
start address 0x00007c00

Program Header:
    LOAD off    0x00000074 vaddr 0x00007c00 paddr 0x00007c00 align 2**2
         filesz 0x0000022c memsz 0x0000022c flags rwx
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000017c  00007c00  00007c00  00000074  2**2
                  CONTENTS, ALLOC, LOAD, CODE
  1 .eh_frame     000000b0  00007d7c  00007d7c  000001f0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         000007b0  00000000  00000000  000002a0  2**2
                  CONTENTS, READONLY, DEBUGGING
  3 .stabstr      00000846  00000000  00000000  00000a50  2**0
                  CONTENTS, READONLY, DEBUGGING
  4 .comment      0000002b  00000000  00000000  00001296  2**0
                  CONTENTS, READONLY
SYMBOL TABLE:
00007c00 l    d  .text    00000000 .text
00007d7c l    d  .eh_frame    00000000 .eh_frame
00000000 l    d  .stab    00000000 .stab
00000000 l    d  .stabstr    00000000 .stabstr
00000000 l    d  .comment    00000000 .comment
00000000 l    df *ABS*    00000000 obj/boot/boot.o
00000008 l       *ABS*    00000000 PROT_MODE_CSEG
00000010 l       *ABS*    00000000 PROT_MODE_DSEG
00000001 l       *ABS*    00000000 CR0_PE_ON
00007c0a l       .text    00000000 seta20.1
00007c14 l       .text    00000000 seta20.2
00007c64 l       .text    00000000 gdtdesc
00007c32 l       .text    00000000 protcseg
00007c4a l       .text    00000000 spin
00007c4c l       .text    00000000 gdt
00000000 l    df *ABS*    00000000 main.c
00000000 l    df *ABS*    00000000
00007c6a g     F .text    00000012 waitdisk
00007d0a g     F .text    00000072 bootmain
00007cd1 g     F .text    00000039 readseg
00007e2c g       .eh_frame    00000000 __bss_start
00007c7c g     F .text    00000055 readsect
00007e2c g       .eh_frame    00000000 _edata
00007e2c g       .eh_frame    00000000 _end
00007c00 g       .text    00000000 start
```

为什么说只有text段是被使用到的呢？

```
boot/Makefrag

$(OBJDIR)/boot/boot: $(BOOT_OBJS)
    @echo + ld boot/boot
    $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o $@.out $^ 
    $(V)$(OBJDUMP) -S $@.out >$@.asm
    $(V)$(OBJCOPY) -S -O binary -j .text $@.out $@ # 380 bytes
    $(V)perl boot/sign.pl $(OBJDIR)/boot/boot
```

boot loader自己是没有利用ELF格式的。需要用boot sector固定的格式。不过加载的时候，却是采用了ELF格式来加载内核。
如果要详细看一下kernel各个段的加载情况，可以通过如下命令：
需要注意program header与sections的区别。program header是给加载程序方用的。
section是给写程序的人以及与编译器看的。

```
$ objdump -x obj/kern/kernel
obj/kern/kernel:     file format elf32-i386
obj/kern/kernel
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c

Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x0000716c memsz 0x0000716c flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000a300 memsz 0x0000a944 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001917  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       00000714  f0101920  00101920  00002920  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00003889  f0102034  00102034  00003034  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000018af  f01058bd  001058bd  000068bd  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .bss          00000644  f0112300  00112300  00013300  2**5
                  ALLOC
  6 .comment      0000002b  00000000  00000000  00013300  2**0
                  CONTENTS, READONLY
SYMBOL TABLE:
f0100000 l    d  .text    00000000 .text
f0101920 l    d  .rodata    00000000 .rodata
f0102034 l    d  .stab    00000000 .stab
f01058bd l    d  .stabstr    00000000 .stabstr
f0108000 l    d  .data    00000000 .data
f0112300 l    d  .bss    00000000 .bss
00000000 l    d  .comment    00000000 .comment
00000000 l    df *ABS*    00000000 obj/kern/entry.o
f010002f l       .text    00000000 relocated
f010003e l       .text    00000000 spin
00000000 l    df *ABS*    00000000 entrypgdir.c
00000000 l    df *ABS*    00000000 init.c
00000000 l    df *ABS*    00000000 console.c
f01001a0 l     F .text    0000001c serial_proc_data
f01001bc l     F .text    00000044 cons_intr
f0112320 l     O .bss    00000208 cons
f0100200 l     F .text    00000117 kbd_proc_data
f0112300 l     O .bss    00000004 shift.1330
f0101b00 l     O .rodata    00000100 shiftcode
f0101a00 l     O .rodata    00000100 togglecode
f01019e0 l     O .rodata    00000010 charcode
f0100317 l     F .text    000001e0 cons_putc
f0112528 l     O .bss    00000002 crt_pos
f011252c l     O .bss    00000004 crt_buf
f0112530 l     O .bss    00000004 addr_6845
f0112534 l     O .bss    00000001 serial_exists
f0112200 l     O .data    00000100 normalmap
f0112100 l     O .data    00000100 shiftmap
f0112000 l     O .data    00000100 ctlmap
00000000 l    df *ABS*    00000000 monitor.c
f0101de4 l     O .rodata    00000018 commands
00000000 l    df *ABS*    00000000 printf.c
f01008eb l     F .text    00000013 putch
00000000 l    df *ABS*    00000000 kdebug.c
f010094b l     F .text    000000dd stab_binsearch
00000000 l    df *ABS*    00000000 printfmt.c
f0100c10 l     F .text    000000ef printnum
f0100cff l     F .text    0000001d sprintputch
f0102008 l     O .rodata    0000001c error_string
00000000 l    df *ABS*    00000000 readline.c
f0112540 l     O .bss    00000400 buf
00000000 l    df *ABS*    00000000 string.c
00000000 l    df *ABS*    00000000
f010000c g       .text    00000000 entry
f0101337 g     F .text    00000020 strcpy
f0100513 g     F .text    00000012 kbd_intr
f010079f g     F .text    0000000a mon_backtrace
f01000f8 g     F .text    0000005f _panic
f010009d g     F .text    0000005b i386_init
f01014d4 g     F .text    00000068 memmove
f0101208 g     F .text    00000028 snprintf
f0100d44 g     F .text    0000046c vprintfmt
f0100525 g     F .text    0000004a cons_getc
f0100931 g     F .text    0000001a cprintf
f010153c g     F .text    00000021 memcpy
f0101230 g     F .text    000000ca readline
f0111000 g     O .data    00001000 entry_pgtable
f0100040 g     F .text    0000005d test_backtrace
f01011b0 g     F .text    00000058 vsnprintf
f0112300 g       .data    00000000 edata
f010056f g     F .text    000000f2 cons_init
f01058bc g       .stab    00000000 __STAB_END__
f01058bd g       .stabstr    00000000 __STABSTR_BEGIN__
f01017c0 g     F .text    00000157 .hidden __umoddi3
f01004f7 g     F .text    0000001c serial_intr
f0101690 g     F .text    00000124 .hidden __udivdi3
f0100682 g     F .text    0000000a iscons
f01015b3 g     F .text    000000d3 strtol
f0101318 g     F .text    0000001f strnlen
f0101357 g     F .text    0000002b strcat
f0112940 g     O .bss    00000004 panicstr
f0112944 g       .bss    00000000 end
f0100157 g     F .text    00000045 _warn
f010146b g     F .text    0000001c strfind
f0101917 g       .text    00000000 etext
0010000c g       .text    00000000 _start
f01013af g     F .text    0000003d strlcpy
f0101412 g     F .text    00000038 strncmp
f0101382 g     F .text    0000002d strncpy
f010155d g     F .text    00000039 memcmp
f0100661 g     F .text    00000010 cputchar
f0101487 g     F .text    0000004d memset
f0100671 g     F .text    00000011 getchar
f0100d1c g     F .text    00000028 printfmt
f010716b g       .stabstr    00000000 __STABSTR_END__
f01013ec g     F .text    00000026 strcmp
f0100a28 g     F .text    000001d9 debuginfo_eip
f01008fe g     F .text    00000033 vcprintf
f0110000 g       .data    00000000 bootstacktop
f0110000 g     O .data    00001000 entry_pgdir
f0108000 g       .data    00000000 bootstack
f0102034 g       .stab    00000000 __STAB_BEGIN__
f0101300 g     F .text    00000018 strlen
f010144a g     F .text    00000021 strchr
f01006d5 g     F .text    000000ca mon_kerninfo
f01007a9 g     F .text    00000142 monitor
f0101596 g     F .text    0000001d memfind
f0100690 g     F .text    00000045 mon_help
```

因此，这里可以看出来。program headers 就是写在ELF头里面给加载程序用的。但是这里主要关注：

- vaddr 加载后程序内部引用的各种虚拟地址基地址
- paddr 加载到的物理基地址
- memsz 这个程序段的大小
- filesz 这个程序段所在位置起始位置。注意，这里是相对于文件头而言。 ！！

这么四个变量。使用-x 输出太长了。虽然里面有些信息lab2会用到。不过对于lab1而言。只需要如下命令就可以理解了。

```
$ objdump -p obj/kern/kernel

obj/kern/kernel:     file format elf32-i386

Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x0000716c memsz 0x0000716c flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000a300 memsz 0x0000a944 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx
```

### exercise5

将bootloader的起始地址修改为-Ttext 0x7c01，在make qemu，随后在该地址打上断点。发生了陷入

```
Breakpoint 1 at 0x7c01
(gdb) c
Continuing.

Program received signal SIGTRAP, Trace/breakpoint trap.
[   0:7c30] => 0x7c30:  ljmp   $0x8,$0x7c36
```

```
EAX=00000011 EBX=00000000 ECX=00000000 EDX=00000080
ESI=00000000 EDI=00000000 EBP=00000000 ESP=00006f20
EIP=00007c30 EFL=00000006 [-----P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
CS =0000 00000000 0000ffff 00009b00 DPL=0 CS16 [-RA]
SS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
DS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
FS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
GS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
LDT=0000 00000000 0000ffff 00008200 DPL=0 LDT
TR =0000 00000000 0000ffff 00008b00 DPL=0 TSS32-busy
GDT=     0000007c 00005000
IDT=     00000000 000003ff
CR0=00000011 CR2=00000000 CR3=00000000 CR4=00000000
DR0=00000000 DR1=00000000 DR2=00000000 DR3=00000000 
DR6=ffff0ff0 DR7=00000400
EFER=0000000000000000
Triple fault.  Halting for inspection via QEMU monitor
```

### exercies6

```
Q:在 BIOS 进入引导加载程序时检查 0x00100000 处的 8 个内存字，然后在引导加载程序进入内核时再次检查。他们为什么不同？第二个断点是什么？

A:
(gdb) b *0x7c00
Breakpoint 1 at 0x7c00
(gdb) b *0x7d81
Breakpoint 2 at 0x7d81
(gdb) c
Continuing.
[   0:7c00] => 0x7c00:  cli    

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) x/8x 0x100000
0x100000:       0x00000000      0x00000000      0x00000000      0x00000000
0x100010:       0x00000000      0x00000000      0x00000000      0x00000000
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x7d81:      call   *0x10018

Breakpoint 2, 0x00007d81 in ?? ()
(gdb) x/8x 0x100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
0x100010:       0x34000004      0x2000b812      0x220f0011      0xc0200fd8

不运行QEMU也可以知道原因，在BIOS进入引导程序的时候，还没有讲内核程序加载到内存中，因此0X10000之后的8个字肯定为0。而在BootLoader进入内核时，此时已经在内核程序加载在0X10000中。
```

在entry.S中	

```
.text

# The Multiboot header
.align 4
.long MULTIBOOT_HEADER_MAGIC
.long MULTIBOOT_HEADER_FLAGS
.long CHECKSUM

# '_start' specifies the ELF entry point.  Since we haven't set up
# virtual memory when the bootloader enters this code, we need the
# bootloader to jump to the *physical* address of the entry point.
.globl		_start
_start = RELOC(entry)

.globl entry
entry:
	movw	$0x1234,0x472			# warm boot
```

前三个字正好对应0X10000中的前三个字

```
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
```

## Part3 The Kernel

### exercise7

```
Ｑ：用Qemu和GDB跳到JOS的内核里面。并且暂停在movl %eax, %cr0这条指令这里。验证内存两个地址：0x00100000 and at 0xf0100000。接下来用s指令一条一条地执行。然后再验证一下这个两个内存地址的内容。确保你理解整个发生的过程。
```

首先需要明白：程序地址与寻址地址

```
1. 程序代码地址
2. 支持的寻址地址
```

如果支持的寻址地址不支持汇编里面的地址（比如页表没有建立起来）。比如：

```
mov %eax, *$0xf0100000
```

这个时候必须要知道0xf0100000真正的物理地址是什么。程序代码里直接成相应的物理地址。

```
#define    RELOC(x) ((x) - KERNBASE)
```

示例1： kernel的入口地址

```
# '_start' specifies the ELF entry point.  Since we haven't set up
# virtual memory when the bootloader enters this code, we need the
# bootloader to jump to the *physical* address of the entry point.
.globl        _start
_start = RELOC(entry)

.globl entry
entry:
```

kernel在编译的时候是并不知道会被加载到哪里的。通过链接的时候kern/kernel.ld链接脚本可以指令被加载到的物理地址。但是程序的入口地址仍然需要告知ELF。
ELF执行的格式是`_start`是入口。如果不加任何处理，那么`_start`就是一个虚拟地址。这个值会反应在ELF header->e_entry值上面。(看boot/main.c)里面的跳转到内核的代码：

```
// call the entry point from the ELF header
 // note: does not return!
 ((void (*)(void)) (ELFHDR->e_entry))();
```

这里面`e_entry`就指向`_start`值。由于从`boot loader`跳转到的内核的时候，还在物理地址与虚拟地址完全重合的情况。并且也没有开启分页。所以这个时候必须在kern/entry.S里面

```
.globl        _start
_start = RELOC(entry)
```

把`_start`地址改造成物理地址。这会儿，

```
root@debug:~/6.828/lab# objdump -f obj/kern/kernel

obj/kern/kernel:     file format elf32-i386
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c
```

这个时候`start address`就是一个物理地址。

```
f010000c <entry>:
f010000c:	66 c7 05 72 04 00 00 	movw   $0x1234,0x472
```

这也就是为什么进入kernel的第一条指令在地址0x10000c（0xf01000c是虚拟地址）

**原问题的正解**

首先对`entry.S`代码加以注释。

```c
	# Load the physical address of entry_pgdir into cr3.  entry_pgdir
	# is defined in entrypgdir.c.
	movl	$(RELOC(entry_pgdir)), %eax
f0100015:	b8 00 20 11 00       	mov    $0x112000,%eax
	movl	%eax, %cr3
	将0x112000也就是页目录的物理地址放入cr3寄存器中
f010001a:	0f 22 d8             	mov    %eax,%cr3
	# Turn on paging.
	movl	%cr0, %eax
f010001d:	0f 20 c0             	mov    %cr0,%eax
	orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
f0100020:	0d 01 00 01 80       	or     $0x80010001,%eax
	movl	%eax, %cr0
f0100025:	0f 22 c0             	mov    %eax,%cr0
```

所以原问题中在开启分页前去看`0xf0100000`地址时。肯定为0。因为在当前地址空间里面，这部分虚拟地址是没有内容的。页表也还没有。只能是假装去访问物理地址。
分页后去查看地址时。`0xf0100000`与`0x00100000`内容就完全一样了。这是因为把`[0, 4MB)`映射到了`[0xf0000000, 0xf0000000 + 4MB)`的地方了。
开启分页后的跳转

```c
       # Now paging is enabled, but we're still running at a low EIP
        # (why is this okay?).  Jump up above KERNBASE before entering
        # C code.
        mov     $relocated, %eax
        jmp     *%eax
relocated:
```

当开启分页之后，立马会进行相应的跳转。这里主要是因为后面会开始执行C语言的函数了。必须设置好相应的CS:IP, esp, ebp, ss等寄存器。如果还是在物理地址空间运行。但是C语言是以为自己在虚拟地址空间运行的。

- 1. CPU跑在物理地址空间上，而不是虚拟地址空间上。（尽管CS:IP会被翻译到真正的地址。）
- 1. C语言认为是自己是跑在虚拟地址空间。

通过jmp，可以使得两者正常化。CPU在取指，寻址的时候，就会在有页映射的地址空间里面了。环境设置好，就可以开始跳转到C语言里面了。

![image-20230425151557478](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312111296.png)

可以发现，两者内容完全一致，虚拟地址`0xf0100000`已经被映射到`0x00100000`处了,为什么会出现这种变化？
在修改cr0之前修改了cr3寄存器。将地址`0x112000`写入了页目录寄存器，页目录表应该就是存放在地址`0x112000`处。其他操作应该是由`entry_pgdir`的// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)，完成了映射。使得再读取`0xf0100000`地址时，自动映射到了`0~4M`的某个位置（暂时不清楚）。

> CR3是页目录基址寄存器，保存页目录表的物理地址，页目录表总是放在以4K字节为单位的存储器边界上，因此，它的地址的低12位总为0，不起作用，即使写上内容，也不会被理会。

**2. What is the first instruction after the new mapping is established that would fail to work properly if the mapping weren’t in place? Comment out the `movl %eax, %cr0` in `kern/entry.S`, trace into it, and see if you were right.**

```
Q:*建立新映射后，*如果映射不存在，将无法正常工作的 第一条指令是什么？注释掉`kern/entry.S``movl %eax, %cr0`中的 内容，跟踪它，看看你是否正确。
```

主要意思是说，如果把movl %eax, %cr0删除掉会发生什么样的情况。
删除掉之后，只要后面有涉及到寻址的地方，就会立马出错。假设把这一行注释掉。

```
    # movl    %eax, %cr0

    # Now paging is enabled, but we're still running at a low EIP
    # (why is this okay?).  Jump up above KERNBASE before entering
    # C code.
    mov    $relocated, %eax
    jmp    *%eax
relocated:

    # Clear the frame pointer register (EBP)
    # so that once we get into debugging C code,
    # stack backtraces will be terminated properly.
    movl    $0x0,%ebp            # nuke frame pointer

    # Set the stack pointer
    movl    $(bootstacktop),%esp
```

那么在`movl $(bootstacktop), %esp`这里就立马出错了。

因为把`$bootstacktop`当成物理地址了。但是实际上，哪有那么大的物理地址空间。所以肯定会报错了。(万一真给了qemu那么大的物理地址空间，那边物理地址也没有内容，跳到C语言之后就会出错。)

### exercise8

仿照case ‘u’，其中putch是将字符打印在屏幕上

```
case 'o':
			// Replace this with your code.
			putch('0', putdat);//print 0 in front of octal
            num = getuint(&ap, lflag);
            base = 8;
            goto number;
```

![image-20230425160250233](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312111126.png)

```
Q：解释printf.c和 console.c之间的接口。具体 console.c导出的是什么功能？printf.c如何使用这个函数 ？

A：
```

这里分析过程如下：
printf.c里面的调用链如下：

```
cprintf -> vcprintf -> vprintfmt -> putch -> cputchar
```

然后cputchar的声明是在

```
./inc/stdio.h:11:void    cputchar(int c);
```



这个函数的定义是在console.c

```
void cputchar(int c)
{
        cons_putc(c);
}
// output a character to the console
static void
cons_putc(int c)
{
        serial_putc(c);
        lpt_putc(c);
        cga_putc(c);
}
```

接下主要就是看`cga_putc`。也就是显示到屏幕上的函数。首先看一下`cga_init`。这个函数的功能就是选定特定的屏幕。比如`vga, cga`等。

```
static void cga_init(void)
{
        volatile uint16_t *cp;
        uint16_t was;
        unsigned pos;

        cp = (uint16_t*) (KERNBASE + CGA_BUF);
        was = *cp;
        *cp = (uint16_t) 0xA55A;
        if (*cp != 0xA55A) {
                cp = (uint16_t*) (KERNBASE + MONO_BUF);
                addr_6845 = MONO_BASE;
        } else {
                *cp = was;
                addr_6845 = CGA_BASE;
        }

        /* Extract cursor location */
        outb(addr_6845, 14);
        pos = inb(addr_6845 + 1) << 8;
        outb(addr_6845, 15);
        pos |= inb(addr_6845 + 1);

        crt_buf = (uint16_t*) cp;
        crt_pos = pos;
}
```

一般而言，显示操作的时候，启动的时候，都是使用提`CGA`。也就是

```
./kern/console.h:14:#define CGA_BUF        0xB8000
```

初始化的时候，需要设定光标的位置。设置完成之后。就可以利用`cga_putc`来CGA屏幕上显示字符了。这里可以看出来，除了各个字符的设定之外。还随时移动的光标。

```c
static void cga_putc(int c)
{
        // if no attribute given, then use black on white
        if (!(c & ~0xFF))
                c |= 0x0700;

        switch (c & 0xff) {
        case '\b':
                if (crt_pos > 0) {
                        crt_pos--;
                        crt_buf[crt_pos] = (c & ~0xff) | ' ';
                }
                break;
        case '\n':
                crt_pos += CRT_COLS;
                /* fallthru */
        case '\r':
                crt_pos -= (crt_pos % CRT_COLS);
                break;
        case '\t':
                cons_putc(' ');
                cons_putc(' ');
                cons_putc(' ');
                cons_putc(' ');
                cons_putc(' ');
                break;
        default:
                crt_buf[crt_pos++] = c;         /* write the character */
                break;
        }

        // What is the purpose of this?
        if (crt_pos >= CRT_SIZE) {
                int i;

                memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
                for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
                        crt_buf[i] = 0x0700 | ' ';
                crt_pos -= CRT_COLS;
        }

        /* move that little blinky thing */
        outb(addr_6845, 14);
        outb(addr_6845 + 1, crt_pos >> 8);
        outb(addr_6845, 15);
        outb(addr_6845 + 1, crt_pos);
}
```

```c
Q：从console.c解释以下内容：
1 如果 (crt_pos >= CRT_SIZE) { 
2 int i; 
3 memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t)); 
4 for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++) 
5 crt_buf[i] = 0x0700 | ' '; 
6 crt_pos -= CRT_COLS; 
7 }


A：
#define CRT_ROWS    25 行
#define CRT_COLS    80 列
#define CRT_SIZE    (CRT_ROWS * CRT_COLS)  25*80
// 一页写满，滚动一行。
if (crt_pos >= CRT_SIZE) {
    int i;
    // 把从第1~n行的内容复制到0~(n-1)行，第n行未变化
    // 通过这一行代码完成了整个屏幕向上移动一行的操作。
  	// 即将[1,24]*80移到[0,23]*80
    memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
    // 把最后一行清空
    for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
        crt_buf[i] = 0x0700 | ' ';
    // 清空了最后一行，同步crt_pos
    crt_pos -= CRT_COLS;
}
CRT_SIZE - CRT_COLS  24*80
void *memmove(void *str1, const void *str2, size_t n)
参数
str1 -- 指向用于存储复制内容的目标数组，类型强制转换为 void* 指针。
str2 -- 指向要复制的数据源，类型强制转换为 void* 指针。
n -- 要被复制的字节数。
```



```
Q：对于以下问题，您可能希望查阅第 2 讲的注释。这些注释涵盖了 GCC 在 x86 上的调用约定。
逐步跟踪以下代码的执行情况：

整数 x = 1, y = 3, z = 4; 
cprintf("x %d, y %x, z %d\n", x, y, z);
在调用 cprintf()时，fmt指向什么？ap指向什么？
列出（按执行顺序）对 cons_putc、va_arg和 的每次调用vcprintf。对于cons_putc，也列出其参数。对于 va_arg，列出ap调用前后指向的内容。对于vcprintf列出其两个参数的值。

A：
```

```c
static void
putch(int ch, int *cnt)
{
        cputchar(ch);
        *cnt++;
}

int
vcprintf(const char *fmt, va_list ap)
{
        int cnt = 0;

        vprintfmt((void*)putch, &cnt, fmt, ap);
        return cnt;
}

int
cprintf(const char *fmt, ...)
{
        va_list ap;
        int cnt;

        va_start(ap, fmt);
        cnt = vcprintf(fmt, ap);
        va_end(ap);

        return cnt;
}
```

比如压入了5个char。也是需要用到2个long。在32位机器上。

```
// 指针定义为char *可以指向任意一个内存地址。
typedef char *va_list;

// 类型大小，注意这里是与CPU位数对齐 ＝ sizeof(long)的作用。
#define    __va_size(type) \
    (((sizeof(type) + sizeof(long) - 1) / sizeof(long)) * sizeof(long))

// 这里个宏并不是取得参数的起始地址。而是说参数将从什么地址开始放。
#define    va_start(ap, last) \
    ((ap) = (va_list)&(last) + __va_size(last))

// va_arg就是用来取参数的起始地址的。然后返回type类型。
// 从整个表达式的意义来说没有什么好用的。
// 其实等价于(*(type*)ap)
// 但是实际上使ap指针移动一个参数大小。
#define    va_arg(ap, type) \
    (*(type *)((ap) += __va_size(type), (ap) - __va_size(type)))

// 空指令，没有什么用
#define    va_end(ap)    ((void)0)
```

所以这里回到原来的代码：

```
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```

`fmt`就是指向那个`const char *`的字符串。当调用的时候，栈中的结构是如下：

```
+-----------------+
|                 |
|     Z           |
|                 |
|                 |
+-----------------+
|                 |
|     Y           |
|                 |
|                 |
+-----------------+
|                 |
|     X           |
|                 |
|                 |
+-----------------+
|                 |
|     fmt         |
|                 |
|                 |
+-----------------+ <-----------+&fmt
va_start(fmt, ap) 作用如下

#define    va_start(ap, last) \
    ((ap) = (va_list)&(last) + __va_size(last))

展开就是
ap = (char *)(&fmt) + align_long(fmt);

+-----------------+
|                 |
|     Z           |
|                 |
|                 |
+-----------------+
|                 |
|     Y           |
|                 |
|                 |
+-----------------+
|                 |
|     X           |
|                 |
|                 |
+-----------------+ <--------------+ap
|                 |
|     fmt         |
|                 |
|                 |
+-----------------+
```

接下来简单一点，看一下调用到`%c`输出的时候，代码是怎么走的

```
void
vprintfmt(void (*putch)(int, void*), void *putdat, const char *fmt, va_list ap)
{

    while (1) {
        // 如果只是一般的字符串，直接输出。
        while ((ch = *(unsigned char *) fmt++) != '%') {
            if (ch == '\0')
                return;
            putch(ch, putdat);
        }

        // 如果发现是%c
    reswitch:
        // 先把%号跳掉，取出'c'
        switch (ch = *(unsigned char *) fmt++) {
        // .. 
        case 'c':
            putch(va_arg(ap, int), putdat);
            break;

        }
    }
}
```

这个时候通过`%c`就知道应该从栈中取出一个参数char类型。

```
va_arg(ap, int) 展开后就是

#define    va_arg(ap, type) \
    (*(type *)((ap) += __va_size(type), (ap) - __va_size(type)))

// putat用来统计输出的字符的个数。在这里可以不用去管
char temp = *(char*)ap;
putch(temp, putdat); // 输出到console上。

ap += align_long(char);

执行完成之后。
+-----------------+
|                 |
|     Z           |
|                 |
|                 |
+-----------------+
|                 |
|     Y           |
|                 |
|                 |
+-----------------+ <------+ap
|                 |
|     X           |   这个x会被%d提出来进行输出。
|                 |
|                 |
+-----------------+
|                 |
|     fmt         |
|                 |
|                 |
+-----------------+
```

从这里也可以总结出来。ap的作用实际上就是利用fmt里面的`%`依次把后面的类型提出来。
然后去栈中找到参数。一个一个输出。

> 从这个练习可以看出来，正是因为C函数调用实参的入栈顺序是从右到左的，才使得调用参数个数可变的函数成为可能(且不用显式地指出参数的个数)。但是必须有一个方式来告诉实际调用时传入的参数到底是几个，这个是在格式化字符串中指出的。如果这个格式化字符串指出的参数个数和实际传入的个数不一致，比如说传入的参数比格式化字符串指出的要少，就可能会使用到栈上错误的内存作为传入的参数，编译器必须检查出这样的错误。

```
Q：运行以下代码。
    无符号整数 i = 0x00646c72；
    cprintf("H%x Wo%s", 57616, &i);
输出是什么？解释如何按照上一个练习的逐步方式得出这个输出。 这是一个将字节映射到字符的 ASCII 表。
输出取决于 x86 是小端字节序这一事实。如果 x86 是 big-endian 你会设置什么i来产生相同的输出？您需要更改 57616为不同的值吗？

这是对小端和大端的描述 以及 更异想天开的描述。

A：
57616 = 0xE110。
i = 0x00646c72
那么如果把i占用的4byte转换成为char[4]数组。结果就是：
char str[4] = {0x72, 0x6c, 0x64, 0x00}; // = {'r', 'l', 'd', 0}
所以输出就是
Hell0 World
```



```
Q：在下面的代码中，将在 之后打印什么 'y='？（注意：答案不是具体值。）为什么会出现这种情况？
    cprintf("x=%dy=%d", 3);

A：在a0-a7中存放参数，因此取决于寄存器a2保存的值
```



```
Q：假设 GCC 更改了它的调用约定，以便它按声明顺序将参数压入堆栈，以便最后一个参数被压入最后。您将如何更改cprintf它的接口，以便仍然可以向它传递可变数量的参数？

A：其实也还是可以拿到参数的。只不过需要把宏的加减法改一下就可以了。把这里的加法改成减法，减法改成加法。

// 指针定义为char *可以指向任意一个内存地址。
typedef char *va_list;
// 类型大小，注意这里是与CPU位数对齐 ＝ sizeof(long)的作用。
#define    __va_size(type) \
    (((sizeof(type) + sizeof(long) - 1) / sizeof(long)) * sizeof(long))
// 这里个宏并不是取得参数的起始地址。而是说参数将从什么地址开始放。
#define    va_start(ap, last) \
    ((ap) = (va_list)&(last) + __va_size(last))
// va_arg就是用来取参数的起始地址的。然后返回type类型。
// 从整个表达式的意义来说没有什么好用的。
// 其实等价于(*(type*)ap)
// 但是实际上使ap指针移动一个参数大小。
#define    va_arg(ap, type) \
    (*(type *)((ap) += __va_size(type), (ap) - __va_size(type)))
// 空指令，没有什么用
#define    va_end(ap)    ((void)0)

```

### The Stack

### exercise9

```
Q:确定内核初始化堆栈的位置，以及堆栈在内存中的确切位置。内核如何为其堆栈保留空间？堆栈指针初始化指向这个保留区域的哪个“端”？
```

\#结论

1. entry.S 77行初始化栈
2. 栈的位置是0xf0108000-0xf0110000
3. 设置栈的方法是在kernel的数据段预留32KB空间(entry.S 92行)
4. 栈顶的初始化位置是0xf0110000

\#分析 bootloader最后一条语句进入内核，进入内核后的几件事情顺序如下：

1. 开启分页(entry.S 62行)
2. 设置栈指针(entry.S 77行)
3. 调用i386_init(entry.S 80行)

设置栈指针的代码如下：

```
    # Set the stack pointer
    movl    $(bootstacktop),%esp
```

可以从kern/kernel文件中找出符号bootstacktop的位置：

```
zzzz@ubuntu:~/workspace/github/xv6-note/Lab1/2014-jos-Lab1$ objdump -D obj/kern/kernel | grep -3 bootstacktop
f0108000 <bootstack>:
        ...

f0110000 <bootstacktop>:
f0110000:       01 10                   add    %edx,(%eax)
f0110002:       11 00                   adc    %eax,(%eax)
        ...
```

因为没有设置CS，因此CS还是指向之前在bootloader阶段设置的数据段描述符，该描述符指定的基地址为0x0，因此%esp的值就是栈顶的位置。因此栈顶的位置就是0xf0110000。 堆栈的大小由下面的指令设置(entry.S 92行):

```
.data
###################################################################
# boot stack
###################################################################
    .p2align    PGSHIFT     # force page alignment
    .globl      bootstack
bootstack:
    .space      KSTKSIZE
    .globl      bootstacktop   
bootstacktop:
```

可以看出，栈的设置方法是在数据段中预留出一些空间来用作栈空间。memlayout.h 97行定义的栈的大小:

```
#define PGSIZE      4096        // bytes mapped by a page
...
#define KSTKSIZE    (8*PGSIZE)          // size of a kernel stack
```

因此栈大小为32KB，栈的位置为0xf0108000-0xf0110000

\#gdb验证 调用call i386_init函数的位置为0xf0100039，在该位置设置断点，查看寄存器内容：

```
(gdb) b *0xf0100039
Breakpoint 1 at 0xf0100039: file kern/entry.S, line 80.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100039 <relocated+10>:   call   0xf010009d <i386_init>

Breakpoint 1, relocated () at kern/entry.S:80
80              call    i386_init
(gdb) info r
eax            0xf010002f       -267386833
ecx            0x0      0
edx            0x9d     157
ebx            0x10094  65684
esp            0xf0110000       0xf0110000 <entry_pgdir>
```

可以看到，调用函数i386_init之前，栈的位置确实是在0xff010000(%esp)。查看栈的内容如下：

```
(gdb) x /8xw 0xf010fff0
0xf010fff0:     0x00000000      0x00000000      0x00000000      0x00000000
0xf0110000 <entry_pgdir>:       0x00111021      0x00000000      0x00000000      0x0000000
```

第一个压入栈的数据应该是call i386_init的返回地址，即这条指令的下一条指令的地址，stepi单步之后再查看：

```
(gdb) x /8xw 0xf010fff0
0xf010fff0:     0x00000000      0x00000000      0x00000000      0xf010003e
0xf0110000 <entry_pgdir>:       0x00111021      0x00000000      0x00000000      0x00000000
(gdb) x /2i 0xf0100039
   0xf0100039 <relocated+10>:   call   0xf010009d <i386_init>
   0xf010003e <spin>:   jmp    0xf010003e <spin>
```

再次查看栈中的内容验证了之前的猜测。

### exercise10

在test_backtrace(5)出打上断点即0xf01000f0，随后查看寄存器$esp为0xf010ffe0

```
*// Test the stack backtrace function (lab 1 only)*

​    test_backtrace(5)*;*

f01000f0:   c7 04 24 05 00 00 00    movl   $0x5,(%esp)

f01000f7:   e8 44 ff ff ff          call   f0100040 <test_backtrace>

f01000fc:   83 c4 10                add    $0x10,%esp
```

![image-20230425211123200](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312111264.png)

```
		mon_backtrace(0, 0, 0);
f0100097:	83 ec 04             	sub    $0x4,%esp
f010009a:	6a 00                	push   $0x0
f010009c:	6a 00                	push   $0x0
f010009e:	6a 00                	push   $0x0
f01000a0:	e8 0c 08 00 00       	call   f01008b1 <mon_backtrace>
```

然后再mon_backtrace函数上打上断点，查看esp寄存器的值。

![image-20230425211431063](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312111220.png)

反汇编代码如下：

```
test_backtrace(int x)
{
        cprintf("entering test_backtrace %d\n", x);
        if (x > 0)
                test_backtrace(x-1);
        else
                mon_backtrace(0, 0, 0);
        cprintf("leaving test_backtrace %d\n", x);
}

f0100040:       55                      push   %ebp                             ;压入调用函数的%ebp
f0100041:       89 e5                   mov    %esp,%ebp                        ;将当前%esp存到%ebp中，作为栈帧
f0100043:       53                      push   %ebx                             ;保存%ebx当前值，防止寄存器状态被破坏
f0100044:       83 ec 14                sub    $0x14,%esp                       ;开辟20字节栈空间用于本函数内使用
f0100047:       8b 5d 08                mov    0x8(%ebp),%ebx                   ;取出调用函数传入的第一个参数
f010004a:       89 5c 24 04             mov    %ebx,0x4(%esp)                   ;压入cprintf的最后一个参数，x的值
f010004e:       c7 04 24 e0 19 10 f0    movl   $0xf01019e0,(%esp)               ;压入cprintf的倒数第二个参数，指向格式化字符串"entering test_backtrace %d\n"
f0100055:       e8 27 09 00 00          call   f0100981 <cprintf>               ;调用cprintf函数，打印entering test_backtrace (x)
f010005a:       85 db                   test   %ebx,%ebx                        ;测试是否小于0
f010005c:       7e 0d                   jle    f010006b <test_backtrace+0x2b>   ;如果小于0，则结束递归，跳转到0xf010006b处执行
f010005e:       8d 43 ff                lea    -0x1(%ebx),%eax                  ;如果不小于0，则将x的值减1，复制到栈上
f0100061:       89 04 24                mov    %eax,(%esp)                      ;接上一行
f0100064:       e8 d7 ff ff ff          call   f0100040 <test_backtrace>        ;递归调用test_backtrace
f0100069:       eb 1c                   jmp    f0100087 <test_backtrace+0x47>   ;跳转到f0100087执行
f010006b:       c7 44 24 08 00 00 00    movl   $0x0,0x8(%esp)                   ;如果x小于等于0，则跳到这里执行，压入mon_backtrace的最后一个参数
f0100072:       00 
f0100073:       c7 44 24 04 00 00 00    movl   $0x0,0x4(%esp)                   ;压入mon_backtrace的倒数第二个参数
f010007a:       00 
f010007b:       c7 04 24 00 00 00 00    movl   $0x0,(%esp)                      ;压入mon_backtrace的倒数第三个参数
f0100082:       e8 68 07 00 00          call   f01007ef <mon_backtrace>         ;调用mon_backtrace，这是这个练习需要实现的函数
f0100087:       89 5c 24 04             mov    %ebx,0x4(%esp)                   ;压入cprintf的最后一个参数，x的值
f010008b:       c7 04 24 fc 19 10 f0    movl   $0xf01019fc,(%esp)               ;压入cprintf的倒数第二个参数，指向格式化字符串"leaving test_backtrace %d\n"
f0100092:       e8 ea 08 00 00          call   f0100981 <cprintf>               ;调用cprintf函数，打印leaving test_backtrace (x)
f0100097:       83 c4 14                add    $0x14,%esp                       ;回收开辟的栈空间
f010009a:       5b                      pop    %ebx                             ;恢复寄存器%ebx的值
f010009b:       5d                      pop    %ebp                             ;恢复寄存器%ebp的值
f010009c:       c3                      ret                                     ;函数返回
```

一个栈帧(stack frame)的大小计算如下：

1. 在执行call test_backtrace时有一个副作用就是压入这条指令下一条指令的地址，压入4字节返回地址
2. push %ebp，将上一个栈帧的地址压入，增加4字节
3. push %ebx，保存ebx寄存器的值，增加4字节
4. sub $0x14, %esp，开辟20字节的栈空间，后面的函数调用传参直接操作这个栈空间中的数，而不是用pu sh的方式压入栈中

加起来一共是32字节，也就是8个int。因此上面打印出来的栈内容，每两行表示一个栈帧，看v起来还算清晰。

\#第一次调用分析 以第一调用栈为例分析，32个字节代码的含义如下图所示：

```
0xf010ffc0:     0x00000004      0x00000005      0x00000000      0xf010004e
0xf010ffd0:     0xf0111308      0x00010094      0xf010fff8      0xf01000fc
             +--------------------------------------------------------------+
             |    next x    |     this x     |  don't know   |  don't know  |
             +--------------+----------------+---------------+--------------+
             |  don't know  |    last ebx    |  last ebp     | return addr  |
             +------ -------------------------------------------------------+
```

中间的两字节不知道是干嘛用的(靠近this x的那一个在调用mon_backtrace时会用到)，按照理论分析，一个完整的调用栈最少需要的字节数等于4+4+4+4*3=24字节，即返回地址，上一个函数的ebp，保存的ebx，函数内没有分配局部变量，需要再加12个字节用来调用mon_backtrace时传参数。

有一个说法是，因为x86的栈大小必须是16的整数倍，所以才分配了32个字节的栈大小。

### exercise11

![image-20230426103040997](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202310312111744.png)

```
--- a/Lab1/2014-jos-Lab1/kern/monitor.c
+++ b/Lab1/2014-jos-Lab1/kern/monitor.c
@@ -58,8 +58,17 @@ mon_kerninfo(int argc, char **argv, struct Trapframe *tf)
 int
 mon_backtrace(int argc, char **argv, struct Trapframe *tf)
 {
-    // Your code here.
-    return 0;
+    uint32_t ebp, *p;
+
+    ebp = read_ebp();
+    while (ebp != 0)
+    {
+        p = (uint32_t *) ebp;
+        cprintf("ebp %x eip %x args %08x %08x %08x %08x %08x\n", ebp, p[1], p[2], p[3], p[4], p[5], p[6]);
+        ebp = p[0];
+    }
+    
+    return 0;
 }
```

先把ebp寄存器中存的地址存入ebp中并打印出来，然后把返回地址即ebp+4的地址打印出来，随后是args[1-5]。最后将ebp存的地址所指向的内容即上一个调用者的ebp地址赋值给ebp寄存器

### exercise12

- 仿照debuginfo_eip中的其他操作写出给eip_line赋值

```
stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
	if (lline <= rline)
		info->eip_line = stabs[lline].n_desc;
	else
		cprintf("lline > rline\n");
```

- 随后修改monitor.h中的Command结构体

```
static struct Command commands[] = {
	{"help", "Display this list of commands", mon_help},
	{"kerninfo", "Display information about the kernel", mon_kerninfo},
	{"backtrace", "Display information about the stack", mon_backtrace},
};
```

- 最后在mon_backtrace函数中打印出函数名，函数所在的行等信息

```
int mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	// Your code here.
	uint32_t ebp = read_ebp();
	uint32_t eip = 0;
	struct Eipdebuginfo info;
#define TO_INT(x) *((uint32_t *)(x))
	while (ebp)
	{
		eip = TO_INT((ebp + 4));
		// ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
		cprintf("ebp %08x  eip %08x  args %08x %08x %08x %08x %08x\n",
				ebp,		 /*ebp*/
				eip,	 /*eip*/
				TO_INT((ebp + 8)),	 /*arg1*/
				TO_INT((ebp + 12)),	 /*arg2*/
				TO_INT((ebp + 16)),	 /*arg3*/
				TO_INT((ebp + 20)),	 /*arg4*/
				TO_INT((ebp + 24))); /*arg5*/
		if(!debuginfo_eip(eip, &info))
		{
			cprintf("%s:%d: %.*s+%d\n", info.eip_file, info.eip_line, info.eip_fn_namelen, info.eip_fn_name, eip - info.eip_fn_addr);
		}
		else
		{
			cprintf("debuginfo_epi error\n");
		}
		ebp = TO_INT(ebp);
	}
	return 0;
}
```


