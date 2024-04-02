# Lab: system calls

![image-20230607164400375](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-1.png)

## 书本内容阅读

第二章：

操作系统的关键服务：多路复用（进程数多于CPU数，每个进程都有机会执行）、隔离、交互

抽象物理资源：操作系统将对于资源的操作抽象成为一种服务，应用程序更加方便的使用这种服务，而不是直接与硬件交互。

对比：类似于咖啡店，每个进程类似于一个客户，咖啡店里有多种硬件资源，比如咖啡机、盘子等，如果让客户们自己操作这些硬件资源，很容易出现彼此并不配合的情况。但是如果将制作咖啡的任务交给一个服务员，由服务员提供客户们想要的服务，就会使得资源得到合理的运用。

用户模式、管理者模式、机器模式

微内核与统一内核。

进程的描述：

每个进程拥有一块地址空间，好像自己在占用整个机器。使用页表（硬件实现）将虚拟地址（就是高级语言会变成低级语言形成的地址，比如一个变量可以翻译成一个地址）转换成硬件地址（硬件内存地址）。每个进程有自己的页表即一个逻辑上的地址空间。

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-2.png" alt="image-20230608171508889" style="zoom:67%;" />

xv6内核将进程的信息保存在一个结构体中（类似于一个PCB），proc.h中定义了这个结构体。进程最重要的信息有页表、内核栈、运行状态。

```c++
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
};
```

这里接下来的形容是每一个进程都拥有一个线程，用于执行进程的指令。（这里的描述就类似于内核级线程的实现，而不是用户级线程的概念。）进程切换时挂起一个线程激活另一个进程的线程。线程的状态（包括局部变量、函数调用返回地址）都保存在线程的栈中。

一个进程有了两个栈：当进程执行用户指令时，使用用户栈；当进程进入内核时，内核代码在进程的内核栈上运行。这样即使一个进程在内核中运行，用户栈仍保存数据。

进程通过执行ecall进入系统调用。这个指令改变CPU硬件状态，并将程序计数器（CPU的PC寄存器）指向内核定义的入口点。入口点的代码跳转到内核栈并执行系统调用的内核指令。系统调用完成之后，内核转换到用户栈并调用sret执行返回用户空间，降低硬件特权级别，恢复执行系统调用前的用户指令。

xv6启动过程：

第四章部分

4.3：以exec系统调用为例，进行系统调用时将参数放在a0，a1寄存器，将系统调用的标签放在寄存器a7。然和使用ecall指令进入管理者模式，执行一些操作之后执行syscall，这个函数在syscall.c中，读取寄存器a7的内容，看看是否合法（与保存的系统调用函数指针数组进行匹配）。系统调用返回时，syscall会记录返回值在寄存器a0中。

4.4：系统调用的参数管理

用户代码调用系统调用时通过函数包装，参数实际上被放置在RISC-V函数调用约定的寄存器中。

The kernel trap code saves user registers to the current process’s trap frame，这样内核代码就能够找到这些参数。

函数argint、argaddr、argfd从trap frame中取出第n个系统调用的参数，作为integer、指针和文件描述符。它们都调用argraw从保存的用户寄存器中取出合适的。

有些函数调用传递指针作为参数，内核使用这些指针访问读写用户内存。由这些指针引发两个挑战：1.传递的指针可能是非法的或者想要访问内核代码；2.xv6内核页表映射与用户页表映射不一致，因此内核不能使用用户页表映射将指针所指的地址映射为物理地址。

内核实现了一些函数，这些函数能够安全地从用户提供的地址发送和接收数据。比如fetchaddr，文件系统调用例如exec使用它从用户空间提取string 文件名参数。

fetchaddr调用copyinstr函数完成工作。

copyinstr从用户页表的虚拟地址srcva最多复制max个字节到dst。它使用walkaddr遍历软件中页表去确定虚拟地址srcva对应的物理地址pa0。由于内核将所有的物理RAM地址映射到相同的内核虚拟地址，copyinstr直接将字节从pa0赋值到dst。walkaddr检查用户提供的虚拟地址是否属于用户地址空间的一部分。

copuout函数将内核数据赋值到用户提供地址空间。



## 学习添加系统调用步骤

**上述内容给出了最开始进行一个系统调用exec的过程，不过是用汇编写的。**
**下面需要了解以下如何调用一个系统调用的过程。**

1.首先在用户空间的user.h中定义所有的系统调用函数的原型：使得用户程序能够调用系统调用。
2.在usys.pl文件（perl语言程序）中，定义系统调用的入口函数entry，生成系统调用的存根。
3.在entry函数中，其操作为：

```c++
sub entry {
    my $name = shift;
    print ".global $name\n";
    print "${name}:\n";
    print " li a7, SYS_${name}\n";
    print " ecall\n";
    print " ret\n";
}
```

貌似是把系统调用对应的整数SYS_name放进a7寄存器，然后调用ecall指令转入内核代码。
4.转入内核代码之后，进行一系列操作之后，调用前面所说的syscall函数，右syacall函数读取a7寄存器的整数，选择这个整数对应的kernel/syscall.h中系统调用的地址。
5.syscall函数调用执行系统调用函数并记录返回值。

**综上所述，如果想要编写一个系统调用，并且把它添加进系统逻辑中，分为以下步骤：**
1.编写这个系统调用函数的代码；
2.在kernel/syscall.h和syscall.c中的系统调用函数指针数组中添加这个函数地址；
3.在user/user.h中声明系统调用函数原型，在user/usys.pl中添加这个系统调用函数的入口函数。

## tracing

![image-20230612164357698](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-3.png)

![image-20230612164531467](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-4.png)

trace函数接收一个参数，这个参数是一个整数，表示1左移”系统调用对应的整数“位数，比如sys_read函数对应的整数是5，1<<5，得到传入的参数应该为32，追踪sys_read调用。

### **阅读user级别的trace函数**

```c
int
main(int argc, char *argv[])
{
  int i;
  char *nargv[MAXARG];

  if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
    fprintf(2, "Usage: %s mask command\n", argv[0]);
    exit(1);
  }

  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "%s: trace failed\n", argv[0]);
    exit(1);
  }
  
  for(i = 2; i < argc && i < MAXARG; i++){
    nargv[i-2] = argv[i];
  }
  exec(nargv[0], nargv);
  exit(0);
}
```

```c++
  if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
    fprintf(2, "Usage: %s mask command\n", argv[0]);
    exit(1);
  }
```

这段代码要求：trace 整数 命令
至少三个参数，并且第二个参数必须是整数开头





```
  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "%s: trace failed\n", argv[0]);
    exit(1);
  }
```

这段代码调用trace(整数)，说明trace系统调用的原型是int trace(int)。即trace系统调用返回值不能是负数，负数表示异常。



```
  for(i = 2; i < argc && i < MAXARG; i++){
    nargv[i-2] = argv[i];
  }
  exec(nargv[0], nargv);
```

这段代码把整数后面的参数全部复制到一个新的字符串数组中，然后执行后面的命令。





 **You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask.** 

如果这个系统调用在mask整数中，那就每次进行这个系统调用时就打印一行信息。

因此trace的目的是传递一个整数给syscall？设置这个整数，然后使得syscall每次执行系统调用时能够判断是否输出信息？



### **系统调用设计**

![image-20230613152325781](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-5.png)

1.函数的位置在kernel/sysproc.c文件中，主要工作是设置在进程的PCB中添加一个变量记录mask值；

2.fork调用需要复制mask值到子进程中；

3.syscall函数需要打印输出所有被mask标记的系统调用。我的想法是，检查每个进程的这个mask值，判断要进行的系统调用对应的整数位（比如需要跟踪的read对应数字5，那么mask从右到左第5位值为1）是否为1，如果为1就打印这个系统调用的信息。

1.在PCB中添加一个mask值，表示需要跟踪的系统调用，问题是，如何初始化。
在PCB中分为两类，一类使用时需要上锁，一类使用时不需要上锁。这个mask值，暂时认为不是临界资源，不需要上锁。
使用一个int类型表示，32位整数足够表示目前的系统调用个数。

在kernel/sysproc.c中编写sys_trace函数：（==这里可能存在的问题是，修改proc中的mask值并没有上锁，也没有检查是否能够修改成功==）

还有一个问题，系统调用不能够有参数，那么如何把参数传递呢？
上面虽然举例说明exec是从寄存器a0和a1中获得参数（利用argstr和argaddr函数），但是用户在执行exec时是如何把参数放进a0和a1中的呢？
**或许是RISC-V默认约定，函数调用参数从a0开始存放？**

```c++
uint64
sys_trace(void)
{
  struct proc *p = myproc();
  argint(0, &p->mask);	// 从a0寄存器读取一个32位整数

  return 0;
}
```



2.修改fork函数：

```c++
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  int i, pid;
  struct proc *np;	// 新的进程的proc
  struct proc *p = myproc();	// 当前进程的proc

  // Allocate process.
  if((np = allocproc()) == 0){	// 创建进程
    return -1;
  }

  // Copy user memory from parent to child.
  // uvmcopy函数应该是复制user virtual memory函数，将当前进程的页表复制到新的进程中
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);	// 释放进程
    release(&np->lock);	// 释放锁
    return -1;
  }
  np->sz = p->sz;	// 复制进程的sz，表示size of process memory

  np->parent = p;	// 新进程的父进程为当前进程

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);	// 复制寄存器内容

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;	// 当前进程的a0寄存器设为0，保存的是系统调用返回值

  // increment reference counts on open file descriptors.
  // 复制打开文件列表
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);	// 复制当前目录 current directory

  safestrcpy(np->name, p->name, sizeof(p->name));	// 复制进程名字

  // copy trace mask
  np->mask = p->mask;
  pid = np->pid;	

  np->state = RUNNABLE;	// 新进程的状态为可执行

  release(&np->lock);	// 释放锁

  return pid;	// 返回当前进程的pid
}
```

加深了对于fork和trapframe的理解：
当一个进程调用fork时，进入系统调用，当前进程的所有CPU寄存器内容保存在trapframe中(包括CPU下面的程序计数器寄存器PC指向下一条指令)
子进程与父进程完全一样，除了trapframe中的a0寄存器，保存的是fork调用的返回值。子进程的a0被显式设置为0，父进程的是子进程的pid。

因此想要复制父进程的mask直接复制就可以。

3.修改syscall函数：（需要一个装有系统调用名字的数组）

```c++
void
syscall(void)
{
  int num;
  struct proc *p = myproc();	// 当前进程PCB

  num = p->trapframe->a7;	// 系统调用对应的整数值
  // 检查整数值是否合法
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num](); // 进行系统调用并且记录返回值到a0寄存器
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

获取PCB中的mask值，在执行系统调用之后，检查从右向左第$(系统调用对应的整数)位是否为1，如果为1就需要打印信息。

4.需要在kernel/syscall.h中为SYS_trace添加一个对应的整数；
在kernel/syscall.c中将声明kernel/sysproc.c文件中的外部函数sys_trace，并保存函数指针数组中；

在user/user.h中声明系统调用原型；
在user/usys.pl中声明系统调用入口存根；

**测试结果：**

![image-20230613172841456](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-6.png)

![image-20230613172905034](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-7.png)

![image-20230613173012074](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-8.png)

![image-20230613173111276](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-9.png)

...

![image-20230613173137839](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-10.png)

![image-20230615162207796](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-12.png)

## sysinfo

![image-20230613214054307](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-11.png)

需要实现一个系统调用sysinfo，收集正在运行的系统的信息。这个系统调用接收一个参数：指向结构体sysinfo（kernel/sysinfo.h）的指针。内核需要补充结构体的内容：
1.freemem表示空白内存的字节数；2.nproc表示状态为UNUSED的进程数目

需要了解的知识：
1.需要把结构体sysinfo的内容复制回用户空间；参考sys_fstat()函数和filestat()函数的实现中使用copyout的例子。
2.为了收集空白内容的数量，需要在kernel/kalloc.c文件中增加一个函数；
3.为了收集进程的数量，需要在kernel/proc.c文件中增加一个函数。



### **学习如何从内核空间将数据传送回用户空间**

kernel/sysfile.c文件的sys_fstat函数，两个参数，一个是文件描述符，另一个是结构体信息int **fstat**(int fd, **struct** stat*);

```c++
uint64
sys_fstat(void)
{
  struct file *f;	// 文件描述符是一个结构体？
  uint64 st; // user pointer to struct stat
  // 从a0寄存器读取第一个参数，利用的是argfd函数
  // 从a1寄存器读取第二个参数，利用的是argaddr函数
  if(argfd(0, 0, &f) < 0 || argaddr(1, &st) < 0)
    return -1;
  return filestat(f, st);	// 调用filestat函数
}
```

kernel/file.c文件中的filestat函数：

```c++
// Get metadata about file f.
// addr is a user virtual address, pointing to a struct stat.
// f是文件描述符对应的结构体
// addr是一个用户空间的地址，指向一个结构体
int
filestat(struct file *f, uint64 addr)
{
  struct proc *p = myproc();	// 当前进程的proc
  struct stat st;	// 创建一个stat结构体
  
  if(f->type == FD_INODE || f->type == FD_DEVICE){
    ilock(f->ip);
    stati(f->ip, &st);	// 将文件索引结点信息复制到结构体中
    iunlock(f->ip);
    // 将结构体st内容使用copyout函数复制到地址用户地址addr中
    if(copyout(p->pagetable, addr, (char *)&st, sizeof(st)) < 0)
      return -1;
    return 0;
  }
  return -1;
}
```

关于kernel/vm.c中的copyout函数（不用了解细节，因为里面有很多内核预定义的内容）

```c++
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

只需指导使用copyout函数，传递参数为当前进程proc的页表、用户空间地址、需要复制的数据的地址、需要复制的字节数，就可以将内核中某些数据传递到用户空间进程。

### **如何维护内核中的sysinfo结构体信息**

**1.维护剩余内存（字节）信息，阅读kernel/kalloc.c文件**：
预定义物理地址空间，一页是4096bytes

```c++
// Physical memory allocator, for user processes,
// kernel stacks, page-table pages,
// and pipe buffers. Allocates whole 4096-byte pages.

#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "spinlock.h"
#include "riscv.h"
#include "defs.h"

void freerange(void *pa_start, void *pa_end);

extern char end[]; // first address after kernel.
                   // defined by kernel.ld.

struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;	// 这里定义了一个freelist，根据后面的kalloc函数，每次申请一页都是从freelist上拿下一页
} kmem;	// kmem可以理解为 kernel memory，维护内核的内存数量

// 内核初始化
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}

// 释放从start~end的内存
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}

// Free the page of physical memory pointed at by v,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist; // 头插法
  kmem.freelist = r;
  release(&kmem.lock);
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
// 申请一个4096字节的物理内存的一页
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);	// 上锁
  r = kmem.freelist;	// 从kmem维护的freelist取下一页
  if(r)		// 如果这一页不是空指针
    kmem.freelist = r->next;	// 将列表后移
  release(&kmem.lock);	// 解锁

  if(r)	// 将一页填上垃圾数据5
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

```

综上，如果想要记录内核中空闲内存数量，从kmem维护的freelist统计非空指针数（即表示可使用页数），再利用页数*4096就是空白字节数。

```c++
// collect the amount of free memory
uint64
countFreeMem(void) {
  struct run *r;
  int count = 0;

  acquire(&kmem.lock);
  r = kmem.freelist;
  while (r) {
    count++;
    r = r->next;
  }

  release(&kmem.lock);

  return count * PGSIZE;
}
```

**2.维护当前进程数目信息**

阅读kernel/proc.c文件
内核维护了一个proc表，里面最多有NPROC=64个进程。proc表会进行初始化。
~~内核也维护了一个next_pid，这个值表示如果有一个新进程就分配给它当前pid值并加1，初始化为1。~~

```c++
int
allocpid() {
  int pid;
  
  acquire(&pid_lock);
  pid = nextpid;
  nextpid = nextpid + 1;
  release(&pid_lock);

  return pid;
}
```

~~使用这个变量必须申请互斥锁pid_lock~~
~~因此可以根据next_pid的值判断当前有多少个进程。~~

```c++
// count the amount of processes
int
countProc(void) {
  int current;

  acquire(&pid_lock);
  current = nextpid;
  release(&pid_lock);

  return current - 1;
}
```

**修正：**~~结构体中的进程数指的是进程状态为UNUSED的进程~~，结构体中的进程数表示进程状态**不为UNUSED**的进程个数，因此可以遍历proc表，统计里面状态为UNUSED的进程个数，

（其实在分配一个进程的时候就是将一个进程表中的proc分配给一个进程使用。这个时候也会分配进程号，~~其实也可以使用next_pid表示~~，进程号表示已经分配的进程个数，而UNUSED是表示为未设置的进程？可以试试最大进程数即proc表的个数减去当前进程号试试）

```c++
// count the amount of processes
uint64
countProc(void) {
  int current_num = 0;

  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);

    if(p->state == UNUSED) {
      current_num++;
    }

    release(&p->lock);
  }

  return current_num;
}
```

测试UNUSED是否表示空闲进程表个数：

结果不对。==那么UNUSED状态表示什么呢？==表示不为挂起状态的进程个数，而next_pid智能表示已经分配了多少个进程数，不能表示位挂起的进程数目。

### 编写sys_info调用

在这个函数调用中：
1.从寄存器中读取用户参数地址addr；
2.申请一个info结构体的内存，然后调用上面的两个函数统计信息，保存到内核结构体中：
3.接着调用copyout函数将内核结构体内容复制到用户传进来的结构体地址中；

如何在内核中申请一块空间，可以直接使用malloc申请一个结构体内存吗？
==仔细阅读内核代码！==其实内核调用时都没有为一个结构体申请空间。（傻了傻了，一个结构体变量声明就会有空间，为什么要使用malloc，又不是指针...）

在kernel/defs.h中的对应位置添加countFreeMem函数和countProc函数

在sysproc.c文件添加头文件声明sysinfo.h，并编写函数

```c++
uint64
sys_sysinfo(void) {
  uint64 addr; // 用户内存地址
  struct sysinfo sy;  // 内核信息结构体

  // 尝试从寄存器中读取用户参数地址addr
  if (argaddr(0, &addr) < 0)
    return -1;
  
  // 统计当前可用内存（字节）
  sy.freemem = countFreeMem();

  // 统计当前进程数
  sy.nproc = countProc();

  // 将内核结构体复制到用户空间
  // 获取当前进程proc
  struct proc *p = myproc();
  if (copyout(p->pagetable, addr, (char *)&sy, sizeof(sy)) < 0)
    return -1;
  
  return 0;
}
```

**配置工作：**

1.在kernel/syscall.h中定义SYS_info为23；

2.在kernel/syscall.c的函数指针数组添加sys_info函数；

3.在user/user.h中添加sysinfo结构体和sysinfo函数原型；

4.在user/usys.pl文件中添加sysinfo入口存根；

在makefile文件中添加测试文件。

**测试结果：**

![image-20230615164736891](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-13.png)

![image-20230615164832964](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-14.png)

# 总结

测试全部代码：

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab2-15.png" alt="image-20230615165232062" style="zoom:80%;" />

耗时15个小时左右。

**关于trace的收获：**
了解一个系统调用的具体流程：
1.用户进程发出系统调用请求（这些系统调用接口都会公开），用户区会保存一个用户调用存根（这里是user/usys.pl文件给出系统调用接口的入口存根）；
2.入口存根会将函数调用作为参数传递给内核（这里是通过一个整数代表对应的系统调用保存在trapframe中的a0寄存器中。接着调用ecall指令使得CPU状态发生改变，并且进入一些预处理操作，是关于trapframe的一些操作（这里是关于进程的描述的内容）；
3.进入syscall调用函数，这个函数会根据传入的代表系统调用的整数去调用函数数组中的系统调用，并记录返回值，保存到a0寄存器中；（用户程序总是通过函数形式调用系统调用，因此需要了解用户区如何与内核区交换参数，一般是将用户将参数保存在寄存器中，然后发生trap时将寄存器内容会保存在trapframe中，内核可以读trapframe中的内存，因此就完成参数传递）；

**关于sysinfo的收获：**

系统调用部分的知识与上面一样，不同的是：
1.了解内核中的内存分配机制：在这个内核中，内核空白内存以页的形式挂在一个空闲列表中，每次分配一页的时候就可以从列表中取下一页，因此可以统计剩余页数；
2.了解内核中进程情况：内核中使用一个proc的数组表示进程的PCB，即proc是一个结构体，描述一个进程的信息；内核初始化时，会分配64个proc，因此可以统计状态不为UNUSED的进程个数。
3.了解如何在内核区和用户去传递数据：内核提供了一些从用户到内核和从内核到用户区的读取和写入函数。已经了解的比如argaddr，就是从trapframe的寄存器中读取参数，copyout就是向用户去某个地址写入一定的字节。

**本次实验主要是学习进程的描述、系统调用的具体过程、系统调用传递参数。**



























