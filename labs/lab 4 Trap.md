# Lab：traps

准备：需要阅读第四章内容，以及trampline.S和trap.c代码

# 任务1：RISC-V assembly

对于代码user/call.c文件，使用make fs.img进行编译，产生一个可读的汇编文件user/call.asm

阅读代码中call.asm中的g、f、和main函数。指令手册在参考文件中。回答以下问题。

# 任务2：Backtrace

![image-20230709152912264](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-1.png)

在kernel/printf.c中实现backtrace函数，并在sys_sleep函数中调用这个函数。然后运行bttest调用sys_sleep。你的输出应该类似于下面。在bttest瑞出qemu之后，在你的终端上，地址可能会稍微不同，但是如果你运行以下代码并且复制粘贴上面的地址，你会看到类似于这样的信息。

编译器会在每一个stack frame中放置一个fram指针，指向调用者的frame指针的地址。你的backtrace应该使用这些frame指针去遍历栈并且打印每一个stack frame保存的返回地址。

## 对于函数调用栈的理解

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-2.png" alt="img" style="zoom:67%;" />

## 设计步骤

![image-20230709162625386](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-3.png)

根据HINT1：首先在defs.h中添加backtrace的函数原型

```c++
// printf.c
void            printf(char*, ...);
void            panic(char*) __attribute__((noreturn));
void            printfinit(void);
void            backtrace(void);
```

根据HINT2：GCC编译器将当前正在执行的函数的frame pointer保存在寄存器s0中，在kernel/riscv.h中增加如下函数：

```c++
static inline uint64
r_fp() {
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

并且在backtrace函数中调用这个函数读取当前的frame pointer。这个函数使用in_line assembly读取s0寄存器的内容。

在3-1的图中表示了stack frame的格式。注意，返回地址在一个stack frame的fp指针的固定偏移处（-8），并且stack frame保存的前一个函数调用栈的fp指针地址也在固定偏移处（-16）

xv6为每一个栈申请一页内存，这一页在页对齐的地址。你可以通过使用**PGROUNDDOWN(fp)**和**PGROUNDUP(fp)**计算栈的顶部和底部地址。使用这些数字有助于终止循环。

根据HINT3和HINT4编写backtrace函数：==fp寄存器的内容是uint64位整数，应当看作一个指向某个内存单元的64位地址。这个地址减去8个字节就是返回地址的内存单元起点，因此(uint64 *)(fp - 8)指向返回地址的内存单元，解引用可以得到内存单元的地址。

```c++
// print backtrace of current function call
void 
backtrace(void) {
  // Print "backtrace"
  printf("%s\n", "backtrace:");

  // get current fp
  uint64 fp = r_fp();
  uint64 page_top = PGROUNDUP(fp);

  while (fp < page_top) {
    printf("%p\n", *((uint64 *)(fp - 8)));
    fp = *((uint64 *)(fp - 16));
  }

}
```

将函数添加进sys_sleep函数：

```c++
  ...
  if(argint(0, &n) < 0)
    return -1;

  backtrace();
  
  acquire(&tickslock);
  ...
```

## 测试结果

![image-20230709211244847](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-4.png)

![image-20230709211702039](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-5.png)

# 任务3：Alarm

![image-20230709211903419](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-6.png)

在这个练习中，你将会位xv6增加一个特性：如果一个进程使用CPU时间的话，周期性提醒它。这对于想要限制占用CPU时间的计算绑定进程，或者对于想要计算但也想要采取一些周期性操作的进程来说可能很有用。更一般地说，您将实现用户级中断/故障处理程序的原始形式；例如，您可以使用类似的方法来处理应用程序中的页面错误。如果您的解决方案通过了alarmtest和usertests，那么它就是正确的。

![image-20230709212606930](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-7.png)

你应该增加一个新的sigalarm(interval, handler)系统调用。如果一个应用程序调用sigalarm(n, fn)，那么这个进程每占用n个CPU时间，内核应该调用应用函数fn。当fn返回时，应用程序应该从它离开的地方恢复。在xv6中，tick是一个相当任意的时间单位，由硬件计时器生成中断的频率决定。如果应用程序调用sigalarm（0，0），内核应该停止生成周期性的警报调用。

你将会在你的xv6库中找到user/alarmtest.c文件，将它添加进Makefile文件，这个文件只有当你添加sigalarm和sigreturn系统调用之后才会正确编译。

alarmtest在test0中调用sigalarm(2, periodic)，请求内核每2次ticks调用一次periodic函数，然后旋转一段时间。您可以在user/armtest.asm中看到alarmtest的汇编代码，这可能对调试很方便。当alarmtest生成这样的输出并且用户测试也正确运行时，您的解决方案是正确的：

![image-20230709213432480](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-8.png)

![image-20230709213456285](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-9.png)

当你完成后，你的解决方案将只有几行代码，但可能很难把它做好。我们将使用原始存储库中的alarmtest.c版本来测试您的代码。您可以修改alarmtest.c来帮助您进行调试，但要确保原始的alarmtest表明所有测试都通过了。

## test0：invoke handler

![image-20230709213934620](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-10.png)

根据HINT1：修改Makefile文件使得alarmtest.c文件能够作为应用程序编译

```
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_alarmtest
```

根据HINT2：在user/user.h中添加函数声明：

```
...
int uptime(void);
int sigalarm(int ticks, void (*handler)());
int sigreturn(void);

// ulib.c
int stat(const char*, struct stat*);
...
```

根据HINT3：修改user/usys.pl、kernel/syscall.h、kernel/syscall.c文件，添加sigalarm和sigreturn系统调用

user/usys.pl:

```c++
...
entry("uptime");
entry("sigalarm");
entry("sigreturn");
```

kernel/syscall.h

```c++
...
#define SYS_close  21
#define SYS_sigalarm 22
#define SYS_sigreturn 23
```

在kernel/sysproc.c文件中添加sigalarm和sigretrun函数的定义：

```
uint64
sys_sigalarm(void) {
  return 0;
}

uint64
sys_sigreturn(void) {
  return 0;
}
```

修改kernel/syscall.c文件，增加对于两个新系统调用的方法



根据HINT4：目前的sigreturn函数只需要返回0

```
uint64
sys_sigreturn(void) {
  return 0;
}
```

根据HINT5：sigalarm函数应该在proc结构体的新字段中保存alarm interval和指向处理函数的指针。

首先在proc结构体中增加了两个新字段分别表示这两个值：

```c++
  ...
  char name[16];               // Process name (debugging)
  int alarm_interval;          // Interval between alarm
  void (*handler)(void);       // handler function
  ...
```

然后在sigalarm中保存这两个值：

```c++
uint64
sys_sigalarm(void) {
  // get parameters
  int ticks;
  void (*handler)(void) = 0;
  if (argint(0, &ticks) < 0)
    return -1;
  if (argaddr(1, (uint64 *)&handler) < 0)
    return -1;
  
  // get proc
  struct proc *p = myproc();
  p->alarm_interval = ticks;
  p->handler = handler;

  return 0;
}
```

在汇编语言中可以看到确实是通过a0和a1寄存器传递两个参数：
![image-20230710105955419](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-11.png)

根据HINT6：需要跟踪自从上次调用handler函数之后经过了几次ticks（或者距离下次handler调用还剩几次ticks）；需要在proc结构体中增加一个字段记录。你可以在allocproc函数中初始化proc结构体。

在proc结构体中增加一个int类型变量记录距离上次handler函数调用经过了几次ticks：

```c++
  ...
  char name[16];               // Process name (debugging)
  int alarm_interval;          // Interval between alarm
  void (*handler)(void);       // handler function
  int nticks;                  // ticks after last call to handler
  ...
```

然后在allocproc函数中初始化nticks为0：

```c++
  ...
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  p->nticks = 0;
  return p;
  ...
```

根据HINT7：每次发生tick，硬件时钟会产生一次中断，由usertrap（kernel/trap.c）函数处理。

首先观察usertrap如何让处理tick中断：
在判断产生trap的原因时会先判断是否为系统调用，如果不是系统调用引起的trap，那么就会which_dev = devintr()来回去硬件中断原因，如果不等于0，说明确实是一个硬件中断。后续处理中当which_dev等于2时是一个timer interrupt。因此可以利用which_dev为2表示发生了一次tick。



根据HINT8：只有当计时器发生中断时（timer interrupt），你才需要操作一个进程的alarm ticks。你需要这样的东西：

```
if(which_dev == 2) ...
```

与上一个提示说明一致，即当发生一次计时器中断时，需要操作进程的alarm ticks。



根据HINT9：仅当进程有未完成的计时器时才调用报警功能。请注意，用户的报警功能的地址可能为0（例如，在user/armtest.asm中，periodical位于地址0）。

说明处理逻辑应当是每次发生计时器中断，p->nticks（表示距离上次调用handler函数经过了几次ticks，初始为0）需要加1，当这个值等于p->alarm_interval时，就需要调用一次handler函数。



根据HINT10：您需要修改usertrap（），以便在进程的alarm interval到期时，用户进程执行处理程序函数。当RISC-V上的trap返回到用户空间时，是什么决定了用户空间代码恢复执行的指令地址？

这个问题的关键是，如何从trap调用handler函数，并且调用之后还能正确返回。当一个trap返回到用户空间时，恢复指令执行是从sepc控制寄存器中将下一条指令复制到pc寄存器，这样就可以正确恢复到之前用户程序执行的地方。

```c++
  ...
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  if (which_dev == 2) {
    p->nticks++;

    if (p->nticks == p->alarm_interval) {
      p->nticks = 0;
      
      // 将handler函数地址保存为返回之后下一条应该执行的执行地址
      p->trapframe->epc = (uint64)(p->handler);
    }
  }

  usertrapret();
  ...
```

测试结果：
![image-20230710155529344](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-12.png)

可以通过test0，但是后面会一直重复打印“...alarm”。

## test1/test2(): resume interrupted code

上面的代码有可能发生的情况是：alarmtest在打印“alarm！”后在test0或test1中崩溃，或者alarmtest（最终）打印“test1 failed”，或者alarm test在未打印“test1passed”的情况下退出。

要解决此问题，必须确保在完成alarm handler程序时，控制返回到用户程序最初被计时器中断的指令。您必须确保寄存器内容恢复到中断时的值，这样用户程序才能在alarm后不受干扰地继续运行。

最后，您应该在每次报警计数器熄灭后“重新武装”它，以便周期性地调用处理程序。

作为一个起点，我们已经为您做出了一个设计决定：用户alarm handler程序需要在完成后调用sigreturn系统调用。以alarmtest.c中的periodic为例。这意味着您可以将代码添加到usertrap和sys_sigreturn中，这两个代码协同工作，使用户进程在处理完警报后能够正常恢复。

根据StartPoint，查看在periodic函数最后调用了sigretrun系统调用。

根据HINT1：您的解决方案将要求您保存和恢复寄存器——您需要保存和恢复哪些寄存器才能正确恢复中断的代码？（提示：会有很多）。

==思考上一个小节修改思路的问题：==目前的程序逻辑是，当一个用户程序发生timer interrupt时，检查距离上次调用handler函数的ticks是否达到了定义的alarm interval，如果达到了，那么久在usertrap中修改p**->**trapframe**->**epc = (uint64)(p**->**handler)。这个代码的意思就是程序恢复之后，将从handler函数的地址开始执行。程序在执行完handler函数之后能够正常返回被中断的程序的原因是：以periodic函数为例：
![image-20230710162023253](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-13.png)

即这个函数在最开始保存了当前的stack frame的信息，也就是说创建了一个新的stack frame，然后保存ra寄存器（返回地址），保存s0寄存器（指向前一个函数stack frame的固定位置）。所以这个函数在执行完毕之后，执行下面代码：
![image-20230710162444231](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-14.png)
载入前一个函数stack frame的信息，于是就返回前一个函数调用，也就是最开始被timer interrupt的用户程序。

但是，**这个过程并没有保存和恢复其它寄存器的内容，所以返回之后程序不能够从被中断的地方按照原来的逻辑继续执行。**（因为有可能在handler函数中会使用到一些寄存器，从而改变其内容）。

因此，需要在调用handler函数之前，保存寄存器内容。然后在系统调用sigreturn中，恢复进程的这些寄存器内容。于是需要修改proc结构体，将寄存器内容保存在其中。
那么能够利用proc原本的trapframe结构体保存这些寄存器内容呢？不能，因为如果在handler函数中发生了trap事件，那么也会使用到这个结构体。所以需要单独的一块结构体保存寄存器内容。

这里采取比较偷懒的方法，直接不去仔细考虑用到哪些寄存器，而是复制一个新的tramframe。

首先在proc结构体中声明这个字段：

```c++
  struct trapframe *alarm_trapframe;
```

然后需要在allocproc中为它分配内存，参考trapframe的分配操作：

```c++
  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  if ((p->alarm_trapframe = (struct trapframe *)kalloc()) == 0) {
    release(&p->lock);
    return 0;
  }
```

然后在freeproc中为它释放内存，同样参考trapframe的释放操作：

```c++
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if (p->alarm_trapframe)
    kfree(p->alarm_trapframe);
  p->alarm_trapframe = 0;
```

接着修改usertrap函数，在达到间隔之后，将trapframe中的内容复制到alarm_trapframe中，此时的trapframe中保存的就是被中断用户程序的寄存器内容。

```c++
  if (which_dev == 2) {
    p->nticks++;

    if (p->nticks == p->alarm_interval) {
      p->nticks = 0;

      memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));

      // printf("usertrap(): should call handler.\n");
      // 将handler函数地址保存为返回之后下一条应该执行的执行地址
      p->trapframe->epc = (uint64)(p->handler);
    }
  }
```

最后在系统调用sigreturn中，恢复寄存器内容：

```c++
uint64
sys_sigreturn(void) {
  struct proc *p = myproc();
  memmove(p->trapframe, p->alarm_trapframe, sizeof(struct trapframe));

  return 0;
}
```

根据HINT2：当计时器关闭时，让usertrap在struct proc中保存足够的状态，以便sigreturn能够正确地返回到中断的用户代码。



根据HINT3：防止对handler程序的重入调用——如果处理程序还没有返回，内核就不应该再次调用它。test2对此进行了测试。

这是要求在handler函数调用期间不能再次调用它。因此需要增加一个标志位，只有当handler函数完成并返回之后，才能再次调用handler函数，也就是在sigreturn系统调用中释放这个限制，当进入handler程序之前加上这个限制（也就是在usertrap函数中）

首先在proc结构体中增加一个字段表示能够调用handler程序：

```c++
  int flag;    // 1 means handler can be called, 0 means not
```

然后在allocproc函数中初始化这个值为1：

```c++
  p->nticks = 0;
  p->flag = 1;
```

然后在freeproc函数中重置这个值：

```c++
  p->nticks = 0;
  p->flag = 1;
```

修改usertrap逻辑，只有当flag为1时，才能调用handler程序，并且把这个值设为0：

```c++
  if (which_dev == 2) {
    p->nticks++;

    if (p->nticks == p->alarm_interval) {
      p->nticks = 0;
      if (p->flag == 1) {
        // printf("usertrap(): should call handler.\n");
        // 将handler函数地址保存为返回之后下一条应该执行的执行地址
        memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
        p->trapframe->epc = (uint64)(p->handler);
        p->flag = 0;
      }
  
    } // end of if nticks
  } // end of if which_dev
```

修改函数调用sigreturn逻辑，只有当flag为0时才能恢复寄存器内容：

```c++
uint64
sys_sigreturn(void) {
  struct proc *p = myproc();
  if (p->flag == 0) {
    memmove(p->trapframe, p->alarm_trapframe, sizeof(struct trapframe));
    p->flag = 1;
  }

  return 0;
}
```

## 测试结果

![image-20230710165750587](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-15.png)

测试usertests：

![image-20230710165955898](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-16.png)

![image-20230710170608246](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab4-17.png)

# 总结

首先第一个任务是关于理解一个函数调用的汇编形式，一般是由函数体前一部分（往往需要创建stack frame的过程）、函数体、函数体后一部分（出栈stack frame）组成。

接着第二个任务就是加深对于函数调用栈的理解，即每一个函数调用栈里面主要内容是什么，最重要的两点是返回地址和指向前一个stack frame的指针，位置是固定的。然后就是xv6的栈是从高地址向低地址生长。

最后一个任务是对于trap的理解。即trap是如何保存寄存器内容，以及如何恢复到被中断程序继续执行的。

























