# lab 3 page tables

![image-20230620161304971](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-1.png)

# 阅读第三章内容

每个进程有一个页表，用于描述每个进程拥有的空间。将进程的逻辑地址映射到物理地址。

## 页表硬件

将虚拟地址映射到物理内存地址。

xv6是39位地址实现，即一个64位虚拟地址的低39位被使用。

![image-20230620161734788](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-2.png)

39位的高27位作为页表表项的索引，页表表项称为PTE（page table entry），即最多有2^27张页表。

每一个页表项拥有44位物理页号PPN（physical page number），以及10位标志位。

因此一个64位虚拟地址只有低39位有效，并且39位中的高27位表示页号，低12位标识页内地址，即一页是2^12，即4096字节。

xv6的页表地址转换如下图：

![image-20230620162402355](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-3.png)

根页表是一个4096的页，里面装有512个表项（2^9），占用27位的最高9位；这里面的每个表项装有下一级页表的物理地址号。

如果某一级的页表中的页表项的物理地址块号并不存在，那么硬件就会报错（page-fault exception）。

这种三级页表的好处允许页表在大范围的虚拟地址没有映射的情况下省略这些整个页表页面。

页表表项中的flag能够说明对于这个物理地址内容的权限，V表示是否存在，R只读等等。



为了能够使得硬件使用页表，内核必须把**根页表**装进寄存器satp中，每个CPU都有一个satp，因此每个CPU都能够运行自己的进程。

## **内核地址空间**

xv6除了为每一个进程都维护一个页表描述用户地址空间外，还有一个单独的**页表描述内核地址空间**。

内核需要仔细配置地址空间布局，使得能够内核自己能够访问物理内存和各种在可预测虚拟地址的硬件资源。

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-4.png" alt="image-20230623153803125" style="zoom:80%;" />

QEMU模式计算机的RAM的地址从KERNBASE到PHYSTOP。低于KERNBAE的是一些IO设备和磁盘的接口，用户可以读写这些地址完成与IO设备和磁盘的交互。

内核与RAM的映射是直接映射，即虚拟地址就是物理地址。

也有一些部分不是直接映射：

- trampoline page:在虚拟地址的顶部，用户地址页表也有这种页表。这部分代码在内核虚拟地址空间中映射两次，第一次映射到虚拟地址空间顶部，第二次直接映射
- 内核栈的页：每一个进程都有自己的内核栈，映射在内核地址空间的顶部，并且紧跟着一个未映射的页，这样就保证如果发生栈的溢出，那么就会由于未映射而发生错误。

另外需要注意的就是内核不同区域的的权限不同，比如kernel text和trampoline区域允许读和执行，因此保护了内核。

## **代码：创建地址空间**

操作地址空间和页表的代码在vm.c中。核心数据结构是pagetable_t，一个指向根页表页的指针。核心函数是walk（通过虚拟地址得到PTE）和mappages（将虚拟地址映射到物理地址）

在启动过程中，main函数调用kvminit函数，这个函数

```c++
void
kvminit()
{
  kernel_pagetable = (pagetable_t) kalloc();
  memset(kernel_pagetable, 0, PGSIZE);

  // uart registers
  kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmap(PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
}
```

这个函数首先为根页表页分配一页物理内存，然后剩下的都是调用kvmmap函数将硬件资源映射到物理地址，这些资源包括内核的指令和数据， KERNBASE 到 PHYSTOP 0x86400000 的物理内存，以及实际上是设备的内存范围。

kvmmap函数调用mappages将虚拟地址映射到一个物理地址，将范围地址划分为页。

walk函数模拟硬件查找三级虚拟地址的过程。

main函数调用kvminit映射内核页表，将刚才生成的根页表页的物理地址写入寄存器satp中，然后CPU就可以利用内核页表翻译地址。

main函数调用procinit函数为每个进程分配一个内核栈，在trampoline下面，每一个进程分配两页，一页作为内核栈，一页作为溢出的保护措施每一个进程都调用kvmini将虚拟地址映射到申请的物理内存上。最后调用kvminithart将内核页表重新加载到satp中。

CPU会在TLB中缓存页表项，当xv6改变也表示，必须刷新TLB中的页表项。指令sfence.vma会刷新当前CPU的TLB，比如sv6在调用kvminithart中就会执行这恶执行。

**物理内存分配**

内核在运行时为页表、用户内存、内核堆栈和管道等分配和释放物理内存。每次分配和释放一个4096字节的页，这些页被组织成页链表，每次从空闲链表中取出一页分配。

## **代码：物理内存分配器**

分配器在kalloc.c中。分配器的数据结构是一个空闲页链表。每一页空闲链表使用结构体run表示，结构体里只有一个指向下一个run的指针。

main函数调用kinit初始化分配器。kinit首先初始化空闲链表的锁，然后调用freerange初始化从内核的end到PHYSTOP之间的内存。freerange对于每一页4096字节调用kfree函数。kfree把每个字节设置为1，并将页面从物理地址（每一页的首地址）转换为指向结构体run的指针，并挂在空闲链表上。

## **进程地址空间**

每一个进程的用户内存虚拟地址从0开始，最大可以增长到MAXVA，即256GB

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-5.png" alt="image-20230623163233227" style="zoom:80%;" />

## **代码：sbrk**

进程收缩或者增加内存的系统调用。

## **代码：exec**

exec创建用户地址空间。读取储存在文件系统上的文件来初始化用户地址空间。首先通过**namei**函数打开二进制文件。xv6应用程序使用ELF格式描述可执行文件，ELF文件格式在kernel/elf.h中，包括一个ELF头，elfhdr结构体，后面是一个程序段（*Program section header*）头序列，使用结构体progdr描述。每一个段头表述一个加载到内存中断程序段。

**exec**函数首先检查ELF头中是否有*ELF_MAGIC*标志，以此判断是否为ELF文件。

**exec**函数然后使用**proc_pagetable**函数分配一个新的页表，接着遍历所有的段头，为每一个段使用**uvmalloc**函数分配内存，通过**loadseg**将每一个段加载到内存中。
**proc_pagetable**函数如下，首先调用**uvmcreate**创建一个新的页，然后添加**trampoline**和**TRAPFRAME**的映射。

```c++
// Create a user page table for a given process,
// with no user memory, but with trampoline pages.
pagetable_t
proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe just below TRAMPOLINE, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  return pagetable;
}
```

**uvmalloc**函数如下：主要功能是将页表的物理内存**从**oldsz增加**至**newsz。**PGROUNDUP**的主要作用是求当前大小的前一个PAGESIZE整数倍。然后一页一页地申请物理内存，一直到申请到newsz，并将映射写入到页表中，

```c++
// Allocate PTEs and physical memory to grow process from oldsz to
// newsz, which need not be page aligned.  Returns new size or 0 on error.
uint64
uvmalloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  char *mem;
  uint64 a;

  if(newsz < oldsz)
    return oldsz;

  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += PGSIZE){
    mem = kalloc();
    if(mem == 0){
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
    memset(mem, 0, PGSIZE);
    if(mappages(pagetable, a, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
      kfree(mem);
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
  }
  return newsz;
}
```

**loadseg**函数使用 walkaddr 找到分配内存的**物理地址**，在该地址写入 ELF 段的每一页， 页的内容通过 readi 从文件中读取。

将程序段加载进内存之后，在sz后面申请两页，一页作为栈，栈下面的一页作为溢出检查页。下面一页的所有PTE经过**uvmclear**函数处理设置为非法的。初始化上面一页作为栈，并设置栈顶指针。

然后**exec**将会把参数字符串和参数地址入栈。

最后提交用户页表。

注意程序的大小p->sz是从0开始一直到栈的位置。





# print a page table

![image-20230623170047678](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-6.png)

![image-20230623170105165](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-7.png)

函数的参数是一个页表页的指针，首先要学习如何使用这个指针，因此需要阅读walk函数。

```c++
// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

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
  return &pagetable[PX(0, va)];
}
```

walk函数的处理过程简单描述为：**接收一个页表指针，把这个页表指针当作一个装有512个PTE的数组，页表指针就指向这个数组的首个PTE**。
根据虚拟地址取得每一级的PTE的地址，用一个指向PTE的指针表示。
从页表指针中获取PTE的转换：将页表指针看作数组，那么只需要通过虚拟地址va获取数组下标即可。假设为最高级（也就是二级页表，那么level=2），下标获取转换为
(va >> (12 + 2 * 9)) & 0X1FF
即向右移动12 + 18位，此时是最高级的9位，然后与1 1111 1111（9位1）进行与操作。
这个下标对应就是这个页表中的PTE。

如何从一个PTE转换位指向页表的指针：向向右移10位，清除最低10位的标志位（还剩低位44位有效位，高位10位没用），再向左移12位（**这里可以说明页面映射的是物理块号44位中的低27位**）

**判断有效位的方法**：使用1L<<0 和PTE的内容相与，如果结果为true则说明有效。

最后是关于不同级别如何表示：三重循环

```c++
int
vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);

  for (int i = 0; i < 512; ++i) {
    pte_t *pte = &pagetable[i];

    if (*pte & PTE_V) {
      // switch to pagetable
      pagetable_t pa_level1 = (pagetable_t)PTE2PA(*pte);
      // print level 2
      printf("..%d: pte %p pa %p\n", i, *pte, pa_level1);

      for (int j = 0; j < 512; ++j) {
        pte_t *pte_level1 = &pa_level1[j];

        if (*pte_level1 & PTE_V) {
          // switch to level 0
          pagetable_t pa_level0 = (pagetable_t)PTE2PA(*pte_level1);
          // print level 1
          printf(".. ..%d: pte %p pa %p\n", j, *pte_level1, pa_level0);

          for (int k = 0; k < 512; ++k) {
            pte_t *pte_level0 = &pa_level0[k];
            // switch to real pa
            pagetable_t pa = (pagetable_t)PTE2PA(*pte_level0);
            // print level 0
            if (*pte_level0 & PTE_V)
              printf(".. .. .. %d: pte %p pa %p\n", k, *pte_level0, pa);
          } // end of for k in range(0, 512)
        } // end of pte is valid
      } // end of for j in range(0, 512)
    } // end of pte is valid
  } // end of for i in range(0, 512)

  return 0;
}
```

![image-20230627095903765](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-8.png)

# A kernel page table pre process

![image-20230627110514127](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-9.png)

xv6在执行内核代码时有一张内核页表。内核页表直接映射到物理地址，因此内核虚拟地址x映射到物理地址x。xv6的每个进程都有自己的用户地址空间，只包含进程内存的映射，从虚拟地址0开始。用为内核页表没有包含这些映射，所以用户地址在内核空间是非法的。因此当内核需要使用用户通过系统调用传递的指针时，内核必须把指针翻译成物理地址。这一部分和下一部分的目标就是允许内核直接解引用用户指针。

第一个工作就是修改内核，使得每一个进程在内核中执行时使用自己的内核页表副本。修改struct proc，为每一个进程维护一个内核页表，并且修改调度程序（scheduler），使得切换进程时切换内核页表。每一个进程内核页表与已经存在的全局内核页表是一样的。

阅读本作业开头提到的书籍章节和代码；通过了解虚拟内存代码的工作原理，可以更容易地正确修改虚拟内存代码。页面表设置中的错误可能会由于缺少映射而导致陷阱，可能会导致加载和存储影响物理内存的意外页面，并可能导致从错误的内存页面执行指令。

- 为结构体struct proc中为每个进程增加一个内核页表
- 为每一个进程创建一个内核页表的合理方式是实现一个修改版本的kvminit（创建一个新的页表）而不是修改kernel_pagetable。你或许会在allocproc函数中调用这个函数
- 确保每一个进程的内核页表映射到进程内核栈。在未修改的xv6中，所有的内核栈在procinit中创建，您需要将这些功能中的部分或全部移动到allocproc
- 修改scheduler()，加载进程的内核页表到satp寄存器中（参考kvminithart函数）。不要在调用w_satp()之后调用sfence_vma()
- schedeler()应该使用kernel_pagerable当没有进程运行时
- 在freeproc中释放进程内核页表
- 你或许需要一种方式释放页表但是不释放叶子物理内存页
- 上一个vmprint可能会在调试页表时派上用场
- 可以修改xv6功能或添加新功能；您可能至少需要在kernel/vm.c和kernel/proc.c中这样做（但是，不要修改kernel/vmcopyin.c、kernel/stats.c、user/usertests.c和user/stats.c）
- 缺少页面表映射可能会导致内核遇到页面错误。它将打印一个错误，其中包括sepc=0x00000000XXXXXXXX。您可以通过在kernel/kernel.asm中搜索XXXXXXXX来找出故障发生的位置。

## 首先需要理解内核页表和进程页表

**理解内核页表启动过程：**

1.main函数首先使用kinit函数初始化分配器：分配器使用一个kmem的链表保存每一个空闲页，所以kinit函数就是初始化这个链表的锁和将从end（内核结束的地址）到PHYSTOP的所有物理内存划分成页进行保存。**freerange**每次取出4096字节作为一页，然后对于每一页调用**kfree**函数。**kfree**接收一个指针，将4096个字节都设置为1，然后将这个指针转换为结构体run保存在kmem链表上。

2.main函数调用kinit函数创建内核页表：**kernel_pagetable**是全局变量，表示唯一的内核页表指针。**kinit**函数首先为内核页表创建一页（使用kalloc函数，就是从kmem空闲页链表上取下一页），初始化为0。然后调用**kvmmap**函数将硬件资源映射到内核页表。

**kvmmap**第一个参数是虚拟地址，第二个参数是物理地址，第三个参数是大小，第四个参数是perm（表示权限，flag位）。由于内核页表物理地址和虚拟地址相同，所以前两个参数相同，都是硬件资源的物理地址，比如UART0就是0x10000000L。**kvmmap**调用**mappages**函数完成任务。

**mappages**函数多了一个参数是指向页表的指针，在kvmmap调用时传递的是内核页表指针。该函数将传入的物理地址转换为虚拟地址保存到内核页表中（保存方式就是首先调用walk根据虚拟地址pa找到pte指针，然后通过pte指针修改指向的物理地址为传入的物理地址，同时也把传入的权限flag位写进pte表项中）。

3.main函数初始化内核页表**kernel_pagetable**之后，调用**kvminithart**函数将内核页表地址写入satp寄存器，然后调用sfence_vma指令刷新TLB，即页表缓存。

4.main函数接着调用**procinit**函数为每个进程初始化页表：内核默认会创建NPROC（64））个PCB（进程控制块，描述进程信息），**procinit**的工作就是为每个进程分配一个物理页即物理地址，然后创建一个虚拟地址va，让每个PCB的kstack等于这个虚拟地址，然后将每个进程的内核栈和物理地址的映射写进内核页表中，最后刷新TLB完成内核页表修改。

## 每个进程的内核页表设计

首先是**创建进程的内核页表**：任务中要求进程的内核页表功能应该与全局唯一的内核页表一样。

接着是在创建进程（procinit时调用kvminit产生一个进程内核页表，需要确保进程内核页表有到进程内核栈的映射

最后还需要修改调度函数，以及如何释放进程内核页表。

根据HINT1：在proc结构体中创建一个内核页表：（kernel/proc.h）

```
  pagetable_t kernel_pagetable;// kernel page table
```

根据HINT2：仿照kvminit创建并初始化进程内核页表，并将函数在def.h文件中声明(kernel/vm.c)

```c++
void
uvmmap(pagetable_t proc_kernel_pagetabel, uint64 va, uint64 pa, uint64 sz, int perm) {
  if(mappages(proc_kernel_pagetabel, va, sz, pa, perm) != 0)
    panic("uvmmap");
}

// create processes's kernel pagetable
pagetable_t proc_kvminit() {
  pagetable_t proc_kernel_pagetable = (pagetable_t) kalloc();
  memset(kernel_pagetable, 0, PGSIZE);

  // uart registers
  uvmmap(proc_kernel_pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  uvmmap(proc_kernel_pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  uvmmap(proc_kernel_pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  uvmmap(proc_kernel_pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  uvmmap(proc_kernel_pagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  uvmmap(proc_kernel_pagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  uvmmap(proc_kernel_pagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
    
  return proc_kernel_pagetable;
}
```

根据HINT3：每个进程内核页表要有一个进程内核栈的映射，**procinit**函数的作用就是使得每一个proc结构体保存这个内核栈，现在需要将这部分功能转移到**allocproc**函数中。
**allocproc**函数的功能就是为进程申请一个proc结构体。首先遍历所有的proc结构体，如果发现状态为UNUSED（未使用的）就接着进行初始化，否则直接退出。初始化工作包括为这个结构体申请一个pid、申请一个trapframe页、申请一个用户页表、初始化寄存器信息结构体（context）。因此需要在这里为每一个进程初始化**进程内核页表**（调用proc_kvminit()函数）。
接着就需要保证在**allocproc**函数中将内核栈映射写入每个进程内核页表中，并初始化p->kstack。（kernel/proc.c中allocproc函数）

```c++
  // allocate and initialize the processes' kernel pagetable
  p->kernel_pagetable = proc_kvminit();
  char *pa = kalloc();
  if (pa == 0) 
    panic("kalloc");
  uint64 va = KSTACK((int)(p - proc));
  uvmmap(p->kernel_pagetable, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;
```

根据HINT4：修改**scheduler**函数，加载进程的内核页表到satp寄存器，并且刷新TLB（kernel/proc.c）

```
        // load processes's kernel pagetable to satp register
        w_satp(MAKE_SATP(p->kernel_pagetable));
        sfence_vma();
```

根据HINT5：在**freeproc**函数中释放进程内核页表。
释放页表是什么意思？参考**freewalk**函数：

```c++
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

对于一个页表，遍历512个PTE，如果PTE的（V无效）以及（V有效，并且R、W、X无效），说明这个pte指向下一个页表，则先释放下一级PTE，再释放本层PTE，释放手段就是将pte设置为0。否则如果V有效并且R、W、X其中一个有效，那么说明这个pte就是一个物理地址，报错。最后释放页表的内存。
因此在**freeproc**中，根据提示，不用管叶子结点，直接释放页表即可

```c++
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  if(p->kernel_pagetable)
    kfree(p->kernel_pagetable);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}


```

## 测试结果

运行发现无法启动。
调试发现遇到的问题是：在**scheduler**函数中，应该先将进程内核页表载入satp寄存器，再调用**swtch**(&c**->**context, &p**->**context)函数。这个函数是在干什么？明显是将进程p的寄存器信息调入CPU寄存器，到哪里去找进程p的内容？Save current registers in old. Load from new.	所以应该是需要找到内存中的内容载入到CPU寄存器中，而进程的内存的内容写在进程内核页表中，所以应该先将进程内核页表写入satp寄存器。**CPU与内存交互首先需要页表机制。**

修正之后遇到问题：panic：kvmpa
查看kvmpa函数：

```c++
// translate a kernel virtual address to
// a physical address. only needed for
// addresses on the stack.
// assumes va is page aligned.
uint64
kvmpa(uint64 va)
{
  uint64 off = va % PGSIZE;
  pte_t *pte;
  uint64 pa;
  
  pte = walk(kernel_pagetable, va, 0);
  if(pte == 0)
    panic("kvmpa");
  if((*pte & PTE_V) == 0)
    panic("kvmpa");
  pa = PTE2PA(*pte);
  return pa+off;
}
```

这个函数的功能是将一个虚拟地址翻译为物理地址，但是使用的是全局内核页表，所以尝试改为进程内核页表。
又遇到的问题是：invalid use of undefined type 'struct proc'，即编译器不知道结构体proc的实现，于是在kernel/vm.c文件增加#**include** "proc.h"。于是又报错：
![image-20230630174027470](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-10.png)
发现是proc.h中有用到spinloc结构体的定义，但是却没有包含这个文件，于是在#**include** "proc.h"增加\#**include** "spinlock.h"。

又出现问题：参看网上代码发现是scheduler函数的问题：

```c++
if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        proc_kvminithart(p->kernel_pagetable);

        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        kvminithart();
        c->proc = 0;
```

切换进程之后，需要**重新调用全局内核页表**。（`scheduler()` should use `kernel_pagetable` when no process is running.）

之后还是无法通过测试，经过与网上对比，发现是释放进程内核页表步骤太过简单，直接使用kfree将进程内核页表归还给空闲页表链。为什么要释放内核页表？

注意这里理解错误，不仅仅是要释放进程内核页表，而是要释放进程内核页表中申请的物理内存。每一个进程的内核页表只为内核栈分配了物理内存，而页表上的其它内容只是建立了映射，所以需要释放内核栈对应的物理内存。

参考释放进程页表的**proc_freepagetable**函数：

```c++
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmfree(pagetable, sz);
}
```

这个函数调用uvmunmap函数取消页表上关于TRAMPOLINE和TRAPFRAME的映射。**uvmunmap**函数接收参数为页表、虚拟地址、页面个数、是否释放物理地址。功能是删除从虚拟地址开始的n页。并将虚拟地址根据页表转换为物理地址，可以选择是否释放物理地址。

**uvmfree**函数将页表中的每一页的首地址取消映射，并调用**freewalk**函数释放页表。而**freewalk**函数会递归释放页表中每一个pte，不符合本题要求。所以需要编写新的释放进程内核页表函数：

首先是proc_free_kernelpagetable:

```c++
void
proc_free_kernelpagetable(pagetable_t pagetable, uint64 kstack) {
  uvmunmap(pagetable, kstack, 1, 1); // 取消对于kstack的映射并且释放kstack的物理内存

  kpgt_freewalk(pagetable);
}
```

这个函数仿照**proc_freepagetable**完成，即先取消kstack在也表上的映射并释放物理内存，然后取消页表上所有表项的映射，但是不释放叶子节点的物理内存。**kpgt_freewalk**仿照freewalk函数完成：

```c++
void
kpgt_freewalk(pagetable_t pagetable) {
  for (int i = 0; i < 512; ++i) {
    pte_t pte = pagetable[i];
    if ((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0) {
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      kpgt_freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } // do nothing to leaf
  }

  kfree((void *)pagetable);
}
```

这个函数遍历所有的页表的所有PTE，如果发现V标志有效但是R、W、X均无效，说明这个PTE是一个分支结点，指向下一个页表，所以需要递归。如果后面三个标志位任意一个有效，说明这个PTE就是一个叶子结点，指向物理内存，则不释放物理内存。最后释放页表的物理内存。

最终通过测试：

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-11.png" alt="image-20230703165524404" style="zoom:67%;" />



总结：1.为每个进程添加一个内核级页表：修改proc结构体定义

2.初始化这个内核级页表：模仿内核页表的初始化来初始化进程的内核页表kvminit，在allocproc中初始化内核页表

3.使用这个内核级页表：在schedule函数中切换进程时加载进程的内核页表，用完之后再切换为内核页表

4.销毁这个内核级页表以及释放相应的物理内存：在freeproc中释放页表



## 部分总结

这一部分的任务令我困惑的地方在于：应该向这个”所谓的**进程内核页表**“中填充什么内容？如果完全按照**全局内核页表**一样填充内容，那就不需要这个进程内核页表了。于是参考网上的内容，发现这个进程内核页表与全局内核页表内容几乎全都一样，唯一不同的是为其它所有的进程的内核栈（kstack）腾出空间，但是不写入映射，只在自己的内核栈（分配了物理内存）的位置写入映射，即需要将每个进程的内核栈的映射写入进程内核页表的最高虚拟地址的位置。
另外需要注意释放页表的过程，对于每个进程来说，进程内核页表中只对于内核栈分配了物理内存，其它的内容只有映射而没有分配内存，所以最后需要释放内核栈的内存和页表所占用的内存。

# Smplify copyin/copyinstr

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-12.png" alt="image-20230703165957525" style="zoom:80%;" />

内核函数**copyin**的功能是读取一个用户指针指向的内存。它将指针指向的地址（虚拟地址）翻译成内核能够直接解引用的物理地址。翻译过程是通过遍历进程的页表，将虚拟地址翻译成物理地址。这一部分的任务就是**增加用户映射到每一个进程的内核页表中**，（这里的意思应该是将用户页表中从0一直到p->sz的所有映射内容写到内核页表中），这样**copyin**函数就能直接解引用用户指针。

使用**copyin_new**函数（在kernel/vmcopyin.c中定义）替代**copyin**函数的函数体，同样的还有**copyinstr函数**。为每一个进程的内核页表增加用户地址的映射，这样两个函数才能工作。

这个方案依赖于用户虚拟地址范围没有与内核指令和数据的范围重叠。xv6的用户地址空间从0开始，内核地址从更高的地址开始。然而，这个方案限制了**用户进程的最高虚拟地址**小于**内核最低虚拟地址**。内核启动之后，那个地址是0xC00 0000，PLIC寄存器的地址。你需要修改xv6阻止用户进程地址增长到大于PLIC寄存器地址。

## 阅读copyin函数和copyintstr函数

```c++
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```

**copyin**函数主要功能是将srcva虚拟地址通过页表转化成源物理地址，然后将源物理地址的内容复制到目的物理地址dst。





## 操作步骤

根据HINT1：使用copyin_new函数替换copyin函数体，同时使用copyinstr_str替换copyinstr函数体。

根据HINT2：在每一个内核改变进程用户页表的位置，以同样的方式改变**进程内核页表**。包括fork、exec、sbrk。即在这些函数中，需要复制进程的页表，所以需要对进程内核页表做同样的操作，即将用户

比如在fork中，有一段代码将父进程的进程页表复制到子进程的页表：

```c++
  ...
  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  ...
```

**uvmcopy**函数的功能主要是复制页表，复制页表不仅仅是复制表项，还需要为表项申请物理内存。该函数按照页将p->sz以下的内容（也就是程序和数据内容）包括表项和物理页的内容复制到新的页表中。

```c++
// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

模仿上面的函数编写一个函数复制进程内核页表：这个函数的主要作用是将进程的用户页表内容复制到内核页表。

```c++
// copy kernel pagetable from pagetable
int
kernelpagetable_uvmcopy(pagetable_t pagetable, pagetable_t kernel_pagatable, uint64 oldsz, uint64 newsz) {
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for (i = oldsz; i < newsz; i += PGSIZE) {
    if ((pte = walk(pagetable, i, 0)) == 0)
      panic("kernelpagetable_uvmcopy: pte should exist");
    if ((*pte & PTE_V) == 0)
      panic("kernelpagetable_uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte) & (~PTE_U);
    if (mappages(kernel_pagatable, i, PGSIZE, pa, flags) != 0)
      goto err;
  }

  return 0;

  err:
  uvmunmap(kernel_pagatable, 0, i / PGSIZE, 0);
  return -1;
}
```



修改**sbrk**函数：**sbrk**函数调用**growproc**函数完成增长n个字节内存的任务。

**growproc**函数处理逻辑：如果n大于0，那么调用**uvmalloc**函数增加程序内存大小；如果n小于0，调用**uvmdealloc**函数减小内存大小。

**uvmalloc**函数的逻辑是：从oldsz增长到newsz（这两个均是虚拟内存）。首先对于oldsz向上取整，得到下一页的虚拟首地址，然后以页为单位，申请一页内存，然后将每一页的首地址作为虚拟地址，和物理地址一起写入页表。

```c++
// Allocate PTEs and physical memory to grow process from oldsz to
// newsz, which need not be page aligned.  Returns new size or 0 on error.
uint64
uvmalloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  char *mem;
  uint64 a;

  if(newsz < oldsz)
    return oldsz;

  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += PGSIZE){
    mem = kalloc();
    if(mem == 0){
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
    memset(mem, 0, PGSIZE);
    if(mappages(pagetable, a, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
      kfree(mem);
      uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
  }
  return newsz;
}
```

**uvmdealloc**函数的逻辑是对于大于oldsz部分的内存地址，调用**uvmunmap**函数取消映射并且释放物理内存。

因此需要修改**growproc**函数代码逻辑：当n大于0时，首先检查是否超过PLIC，然后调用**uvmalloc**函数申请新内存，此时sz已经是新的内存大小，然后调用**kernelpagetable_uvmcopy**函数复制用户页表到内核页表。当n小于0时，先调用**uvmdealloc**减小内存，然后计算需要释放几页，调用**uvmunmap**函数释放这几页。

```c++
// Grow or shrink user memory by n bytes.
// Return 0 on success, -1 on failure.
int
growproc(int n)
{
  uint sz;
  uint beg, end;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    if (sz + n > PLIC)
      panic("growproc: process's memory should less than PLIC");

    beg = sz;
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
    end = sz;
    // copy kerenl pagetable from user pagateable
    if (kernelpagetable_uvmcopy(p->pagetable, p->kernel_pagetable, PGROUNDUP(beg), end) < 0) 
      return -1;
  } else if(n < 0){
    beg = sz;
    sz = uvmdealloc(p->pagetable, sz, sz + n);
    end = sz;
    if (end < beg && PGROUNDUP(end) < PGROUNDUP(beg)) {
      int npages = (PGROUNDUP(beg) - PGROUNDUP(end)) / PGSIZE;
      uvmunmap(p->kernel_pagetable, PGROUNDUP(end), npages, 0);
    }
  }
  p->sz = sz;
  return 0;
}
```

然后是**exec**函数，在提交用户页表之前，先清除内核页表，然后将用户页表复制到这个内核页表。

```c++
  ...
  // clear old user mappings before copy
  uvmunmap(p->kernel_pagetable, 0, PGROUNDUP(oldsz) / PGSIZE, 0);
  if (kernelpagetable_uvmcopy(pagetable, p->kernel_pagetable, 0, sz) < 0) {
      // 内核页表已经被清除了, 此时复制新的用户映射出错了, 只能panic
      panic("exec: kernelpagetable_uvmcopy");
  }

  // Commit to the user image.
  ...
```

根据HINT3：需要修改**userinit**函数。

```c++
  ...
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // copy user pagetable to kernel pagetable
  if (kernelpagetable_uvmcopy(p->pagetable, p->kernel_pagetable, 0, PGSIZE) < 0)
    panic("userinit: kernelpagetable_uvmcopy");

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer
  ...
```

## 测试结果

发现出现remap的错误，这个错误是在**mappages**函数中出现的，即在重新写入映射时发现之前在这个位置已经存在一个有效的PTE。

出现问题的原因是：
<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-13.png" alt="img_4.png" style="zoom:80%;" />

在本任务中，只是不允许进程的内存增长到PLIC寄存器，但实际上进程内核页表会存在CLINT的硬件映射，所以目前采取的措施是进程内核页表不对这个硬件进行映射，因为实际上全局操作时还是使用的全局内核页表，所以不会使用进程的内核页表。

所以修改措施是在上一个任务中创建内核内核页表时不对CLINT进行映射。

kernel/vm.c中的**proc_kvminit**函数取消对于CLINT的映射。

最后回答两个问题。

问题1：实际上是问进程页表的内容是什么。进程的page0包含的是进程的代码段和数据段；page1其实是一个guard page，这一页的映射都是非法的，因此不可访问；page2是stack，进程在栈上保存的一些本地变量、参数。

问题2：参数srcva 和 len 都是64位无符号整数，无符号整数加法发生溢出时，得到的结果是正数，但会比srcva和len都要小。
因此srcva + len < srcva是用来检测溢出的。比如 srcva = 4096， len = UINT64MAX - 4095的情况；srcva >= p->sz(false)、src + len >= p->sz(发生溢出，0 >= p->sz， false)、srcva + len < srcva(0 < p->sz, true)。可见，如果没有这个溢出判断，就会复制用户空间之外的内存，可能导致内核被破坏。

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab3-14.png" alt="image-20230706170645210" style="zoom:80%;" />

# 总结

花费时间：20

以及参考了网上的教程。

## 第一个实验：打印内核页表

这个实验主要是考察对于一个页表的理解。一个页表指针实际上就是一个具有512个PTE的第一个PTE的首地址。遍历512个PTE，将PTE转换成物理地址，表示一个页表的位置或者页表目录的位置。xv6采用是三级页表机制，所以需要递归到第三级才是最终的页表物理地址。

## 第二个实验：每个进程增加一个内核页表

模仿全局唯一的内核页表，为每个进程添加一个内核页表。

首先是编写一个函数能够创建进程内核页表并返回。

接着是对于进程在初始化时创建自己的内核页表，这里需要注意的是之前进程的kstack是映射在全局内核页表上的，在这里修改为映射在自己的进程内核页表上。

然后是在进程调度的时候将进程内核页表调入satp寄存器，这样才能使用。没有进程时CPU使用的是全局内核页表。

最后是进程释放时也需要释放自己的内核页表。主要是需要释放掉申请的kstack物理内存，至于其它的映射由于没有分配内存所以不同释放，但是需要释放掉进程内核页表的内存。

## 第三个实验：将进程页表内容写入到进程内核页表

这样就能够在内核中直接使用指针解引用到进程的虚拟地址空间。

这一个实验主要是需要在进程的用户页表发生变化时要将用户页表内容复制到内核页表中，主要修改了fork、grwoproc、exec函数。

在这个实验中主要是加深了对于页表的理解。首先是页表目录也需要占用一页空间，然后是页表目录中并不是所有项都有效，更像是一页4096字节，每64位作为一个PTE，刚好512个PTE。使用walk函数在一个页表中取出物理地址时，首先是根据传入的虚拟地址va的高9位找第2级页表的物理地址，然后中间9位找到第1级页表的物理位置，最后根据最后9位找到第0级页表的位置，读出物理地址并加上va的最后12位。

exec在创建一个进程的页表时，按照页进行创建，然后把页的地址写进页表。

不清楚的地方就是为什么**uvmcopy**这些函数在复制页表是按照页的大小增加？

20230706答：复制时虚拟地址从0开始，一直到sz。进程的虚拟地址是从0增长到MAXVA，按照页写入页表。因此每次需要把一页的第一个虚拟地址在页表查找物理地址，复制其内容。





















