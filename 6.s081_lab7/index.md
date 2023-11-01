# 6.s081_lab7




# Chapter 7

## 7.1 多路复用

Xv6通过在两种情况下将每个CPU从一个进程切换到另一个进程来实现多路复用（Multiplexing）。第一：当进程等待设备或管道I/O完成，或等待子进程退出，或在`sleep`系统调用中等待时，xv6使用睡眠（sleep）和唤醒（wakeup）机制切换。第二：xv6周期性地强制切换以处理长时间计算而不睡眠的进程。这种多路复用产生了每个进程都有自己的CPU的错觉，就像xv6使用内存分配器和硬件页表来产生每个进程都有自己内存的错觉一样。

实现多路复用带来了一些挑战。首先，如何从一个进程切换到另一个进程？尽管上下文切换的思想很简单，但它的实现是xv6中最不透明的代码之一。第二，如何以对用户进程透明的方式强制切换？Xv6使用标准技术，通过定时器中断驱动上下文切换。第三，许多CPU可能同时在进程之间切换，使用一个用锁方案来避免争用是很有必要的。第四，进程退出时必须释放进程的内存以及其他资源，但它不能自己完成所有这一切，因为（例如）它不能在仍然使用自己内核栈的情况下释放它。第五，多核机器的每个核心必须记住它正在执行哪个进程，以便系统调用正确影响对应进程的内核状态。最后，`sleep`允许一个进程放弃CPU，`wakeup`允许另一个进程唤醒第一个进程。需要小心避免导致唤醒通知丢失的竞争。Xv6试图尽可能简单地解决这些问题，但结果代码很复杂。

## 7.2 代码：上下文切换

![img](../../../../6.s081_notes/lab7.assets/p1.png)

图7.1概述了从一个用户进程（旧进程）切换到另一个用户进程（新进程）所涉及的步骤：一个到旧进程内核线程的用户-内核转换（系统调用或中断），一个到当前CPU调度程序线程的上下文切换，一个到新进程内核线程的上下文切换，以及一个返回到用户级进程的陷阱。调度程序在旧进程的内核栈上执行是不安全的：其他一些核心可能会唤醒进程并运行它，而在两个不同的核心上使用同一个栈将是一场灾难，因此xv6调度程序在每个CPU上都有一个专用线程（保存寄存器和栈）。在本节中，我们将研究在内核线程和调度程序线程之间切换的机制。

从一个线程切换到另一个线程需要保存旧线程的CPU寄存器，并恢复新线程先前保存的寄存器；栈指针和程序计数器被保存和恢复的事实意味着CPU将切换栈和执行中的代码。

函数`swtch`为内核线程切换执行保存和恢复操作。`swtch`对线程没有直接的了解；它只是保存和恢复寄存器集，称为上下文（contexts）。当某个进程要放弃CPU时，该进程的内核线程调用`swtch`来保存自己的上下文并返回到调度程序的上下文。每个上下文都包含在一个`struct context`（***kernel/proc.h\***:2）中，这个结构体本身包含在一个进程的`struct proc`或一个CPU的`struct cpu`中。`Swtch`接受两个参数：`struct context *old`和`struct context *new`。它将当前寄存器保存在`old`中，从`new`中加载寄存器，然后返回。

让我们跟随一个进程通过`swtch`进入调度程序。我们在第4章中看到，中断结束时的一种可能性是`usertrap`调用了`yield`。依次地：`Yield`调用`sched`，`sched`调用`swtch`将当前上下文保存在`p->context`中，并切换到先前保存在`cpu->scheduler`（***kernel/proc.c\***:517）中的调度程序上下文。

> 注：当前版本的XV6中调度程序上下文是`cpu->context`

`Swtch`（***kernel/swtch.S\***:3）只保存被调用方保存的寄存器（callee-saved registers）；调用方保存的寄存器（caller-saved registers）通过调用C代码保存在栈上（如果需要）。`Swtch`知道`struct context`中每个寄存器字段的偏移量。它不保存程序计数器。但`swtch`保存`ra`寄存器，该寄存器保存调用`swtch`的返回地址。现在，`swtch`从新进程的上下文中恢复寄存器，该上下文保存前一个`swtch`保存的寄存器值。当`swtch`返回时，它返回到由`ra`寄存器指定的指令，即新线程以前调用`swtch`的指令。另外，它在新线程的栈上返回。

注：关于callee-saved registers和caller-saved registers请回看视频课程LEC5以及文档《Calling Convention》



 Note

这里不太容易理解，这里举个课程视频中的例子：

**以`cc`切换到`ls`为例，且`ls`此前运行过**

1. XV6将`cc`程序的内核线程的内核寄存器保存在一个`context`对象中

2. 因为要切换到`ls`程序的内核线程，那么`ls` 程序现在的状态必然是`RUNABLE` ，表明`ls`程序之前运行了一半。这同时也意味着：

   a. `ls`程序的用户空间状态已经保存在了对应的trapframe中

   b. `ls`程序的内核线程对应的内核寄存器已经保存在对应的`context`对象中

   所以接下来，XV6会恢复`ls`程序的内核线程的`context`对象，也就是恢复内核线程的寄存器。

3. 之后`ls`会继续在它的内核线程栈上，完成它的中断处理程序

4. 恢复`ls`程序的trapframe中的用户进程状态，返回到用户空间的`ls`程序中

5. 最后恢复执行`ls`

在我们的示例中，`sched`调用`swtch`切换到`cpu->scheduler`，即每个CPU的调度程序上下文。调度程序上下文之前通过`scheduler`对`swtch`（***kernel/proc.c\***:475）的调用进行了保存。当我们追踪`swtch`到返回时，他返回到`scheduler`而不是`sched`，并且它的栈指针指向当前CPU的调用程序栈（scheduler stack）

# lab7

## Uthread: switching between threads (moderate)

![img](../../../../6.s081_notes/lab7.assets/p1.png)

在xv6 book的chapter7中讲的是用户态进程切换到另一个用户态进程，通过在内核中的调度程序去进行切换。在这个实验中我们需要去写一个用户态的调度程序（其中切换寄存器需要在内核中运行）去模拟内核中的调度程序去实现用户态线程之间的切换，所以我们需要自己定义属于thread的context去保存寄存器的值。

- 定义用户态的上下文结构体tcontext

  ```c
  // 用户线程的上下文结构体
  struct tcontext {
    uint64 ra;
    uint64 sp;
  
    // callee-saved
    uint64 s0;
    uint64 s1;
    uint64 s2;
    uint64 s3;
    uint64 s4;
    uint64 s5;
    uint64 s6;
    uint64 s7;
    uint64 s8;
    uint64 s9;
    uint64 s10;
    uint64 s11;
  };
  
  ```

- 修改thread结构体，添加context字段

  ```c
  struct thread {
    char            stack[STACK_SIZE];  /* the thread's stack */
    int             state;              /* FREE, RUNNING, RUNNABLE */
    struct tcontext context;            /* 用户进程上下文 */
  };
  ```

- 模仿kernel/swtch.S，在kernel/uthread_switch.S中写入以下代码

  ```c
  .text
  
  /*
  * save the old thread's registers,
  * restore the new thread's registers.
  */
  
  .globl thread_switch
  thread_switch:
      /* YOUR CODE HERE */
      sd ra, 0(a0)
      sd sp, 8(a0)
      sd s0, 16(a0)
      sd s1, 24(a0)
      sd s2, 32(a0)
      sd s3, 40(a0)
      sd s4, 48(a0)
      sd s5, 56(a0)
      sd s6, 64(a0)
      sd s7, 72(a0)
      sd s8, 80(a0)
      sd s9, 88(a0)
      sd s10, 96(a0)
      sd s11, 104(a0)
  
      ld ra, 0(a1)
      ld sp, 8(a1)
      ld s0, 16(a1)
      ld s1, 24(a1)
      ld s2, 32(a1)
      ld s3, 40(a1)
      ld s4, 48(a1)
      ld s5, 56(a1)
      ld s6, 64(a1)
      ld s7, 72(a1)
      ld s8, 80(a1)
      ld s9, 88(a1)
      ld s10, 96(a1)
      ld s11, 104(a1)
      ret    /* return to ra */
  ```

- 修改`thread_scheduler`，添加线程切换语句

  ```c
  ...
  if (current_thread != next_thread) {         /* switch threads?  */
    ...
    /* YOUR CODE HERE */
    thread_switch((uint64)&t->context, (uint64)&current_thread->context);
  } else
    next_thread = 0;
  
  ```

- 在`thread_create`中对`thread`结构体做一些初始化设定，主要是`ra`返回地址和`sp`栈指针，其他的都不重要，将回调函数放在thread的返回地址上来让第一次调用thread_scheduler时可以调用该线程的回调函数

  ```c
  // YOUR CODE HERE
  t->context.ra = (uint64)func;                   // 设定函数返回地址
  t->context.sp = (uint64)t->stack + STACK_SIZE;  // 设定栈指针
  ```

## Using threads

来看一下程序的运行过程：设定了五个散列桶，根据键除以5的余数决定插入到哪一个散列桶中，插入方法是头插法，下面是图示

不支持在 Docs 外粘贴 block

这个实验比较简单，首先是问为什么为造成数据丢失：

> 假设现在有两个线程T1和T2，两个线程都走到put函数，且假设两个线程中key%NBUCKET相等，即要插入同一个散列桶中。两个线程同时调用insert(key, value, &table[i], table[i])，insert是通过头插法实现的。如果先insert的线程还未返回另一个线程就开始insert，那么前面的数据会被覆盖

因此只需要对插入操作上锁即可

- 为每个散列桶定义一个锁，将五个锁放在一个数组中，并进行初始化

  ```c
  pthread_mutex_t lock[NBUCKET] = { PTHREAD_MUTEX_INITIALIZER }; // 每个散列桶一把锁
  ```

- (2). 在`put`函数中对`insert`上锁

  ```c
  if(e){
      // update the existing key.
      e->value = value;
  } else {
      pthread_mutex_lock(&lock[i]);
      // the new is new.
      insert(key, value, &table[i], table[i]);
      pthread_mutex_unlock(&lock[i]);
  }
  ```

# Barrier

- 保证在所有线程到达之前barrier之前不会有线程先退出barrier，否则会导致断言函数abort

  ```c
  static void 
  barrier()
  {
    // 申请持有锁
    pthread_mutex_lock(&bstate.barrier_mutex);
  
    bstate.nthread++;
    if(bstate.nthread == nthread) {
      // 所有线程已到达
      bstate.round++;
      bstate.nthread = 0;
      pthread_cond_broadcast(&bstate.barrier_cond);
    } else {
      // 等待其他线程
      // 调用pthread_cond_wait时，mutex必须已经持有
      pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    }
    // 释放锁
    pthread_mutex_unlock(&bstate.barrier_mutex);
  }
  ```

  




