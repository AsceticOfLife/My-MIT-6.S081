# lab: xv6 lazy page allocation

![image-20230717150330229](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab5-1.png)

O/S在使用页表硬件时可以使用的许多巧妙技巧之一是延迟分配用户空间堆内存。Xv6应用程序使用sbrk（）系统调用向内核请求堆内存。在我们给您的内核中，sbrk（）分配物理内存并将其映射到进程的虚拟地址空间中。内核为大型请求分配和映射内存可能需要很长时间。例如，考虑一个千兆字节由262144个4096字节的页面组成；这是一个巨大的分配数量，即使每个分配都很便宜。此外，一些程序分配的内存比实际使用的内存多（例如，用于实现稀疏阵列），或者在使用之前就分配好内存。在这些情况下，为了让sbrk（）更快地完成，复杂的内核会惰性地分配用户内存。也就是说，sbrk（）不分配物理内存，只是记住分配了哪些用户地址，并在用户页表中将这些地址标记为无效。当进程第一次尝试使用任何给定的延迟分配内存页面时，CPU会生成一个页面错误，内核会通过分配物理内存、将其归零并映射来处理该错误。您将在本实验室中将此延迟分配功能添加到xv6中。

# 任务1：Eliminate allocation from sbrk()

![image-20230717150817439](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab5-2.png)

第一个任务是删除系统调用sbrk中的页面分配（这个系统调用是sysproc.c中的sys_sbrk函数。这个系统调用使得进程内存增加n和字节，并且返回新分配区域的起点。你的新sbrk应该仅仅增加进程大小（myproc()->size）n个字节，并且返回之前的大小。不应该分配内存--所以需要删除对于growproc的调用。

## 查看之前的sbrk系统调用

kernel/sysproc.c:

```c++
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz = myproc()->sz + n;
  // if(growproc(n) < 0)
  //   return -1;
  return addr;
}
```

接收一个参数n，然后addr表示增长之前的起点，调用growproc增长n个字节之后，返回addr。

kernel/proc.c:

```c
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  p->sz = sz;
  return 0;
}
```

如果n大于0，调用uvmalloc函数增长内存；如果n小于0，调用uvmdealloc函数减少内存。

kernel/vm.c

```c
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

从页向上对齐地址开始，一次分配一页物理内存，然后初始化页面，最后将页面物理地址保存到页面中。

kernel/vm.c:

```c++
uint64
uvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 1);
  }

  return newsz;
}
```

调用uvmunmap函数清除从newsz开始到oldsz的页面的映射。就是把原来页面中的pte设置为0。

## 修改sys_sbrk函数

直接增加进程的sz，但是并不分配内存：

```c++
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz = myproc()->sz + n;
  // if(growproc(n) < 0)
  //   return -1;
  return addr;
}
```

测试结果：

![image-20230717152723516](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab5-3.png)

shell使用fork创建一个子进程，这个过程中会使用系统调用增加内存，但是实际上并没有分配，没有分配导致那些虚拟地址对应的表项V标志位为0，会引发page fault。

## 任务2：Lazy allocation

![image-20230717152949887](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab5-4.png)

修改trap.c中的代码以响应来自用户空间的页面错误，方法是在错误地址映射新分配的物理内存页面，然后返回到用户空间，让进程继续执行。您应该在生成“usertrap（）：…”消息的printf调用之前添加代码。为了让echo-hi正常工作，您需要修改任何其他xv6内核代码。

根据HINT1：检查发生页表错误对应的错误源对应的错误编号是多少。

```c++
  ...
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
  ...
```

![image-20230717153441425](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab5-5.png)

可以看到r_scause()寄存器内容是15，说明发生上面的缺页中断对应的错误编号是15.

根据HINT2：r_stval()函数返回stval寄存器的内容，包含了产生错误的虚拟地址。



根据HINT3：模仿vm.c中的uvmalloc函数，实际上sbrk函数就是通过growproc函数调用这个函数。（需要使用kalloc和mappages）

根据HINT4：使用PGROUNDDOWN计算出虚拟地址对应的页面的边界。

修改trap.c中的usertrap函数：

```c++
...  
} else if((which_dev = devintr()) != 0){
    // ok
  } else if (r_scause() == 15) {
    // get va which results in page fault
    uint64 va = r_stval();
    printf("page fault: %p\n", va);
    // try to allocate a new page
    uint64 ka = (uint64)kalloc();

    if (ka == 0) p->killed = 1;
    else {
      memset((void *)ka, 0, PGSIZE);
      va = PGROUNDDOWN(va);
      if ((mappages(p->pagetable, va, PGSIZE, ka, PTE_W|PTE_U|PTE_R)) != 0) {
        kfree((void *)ka);
        p->killed = 1;
      }
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
...
```

根据HINT5：uvmunmap函数会panic。出现这种情况的原因是需要释放进程内存时，是根据页表中的映射进行释放：
freeproc函数调用**proc_freepagetable**函数释放进程内存。
**proc_freepagetable**根据页表和sz进行释放进程占用的物理内存。

```c++
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmfree(pagetable, sz);
}
```

**uvmunmap**函数解除进程页表对于TRAMPOLINE和TRAPFRAME的映射，但是没有释放物理内存。（因为实际上这里映射的是与内核页表一样的物理内存，方便进行trap）。然后调用**uvmfree**函数释放页表中sz大小的内存。

```c++
void
uvmfree(pagetable_t pagetable, uint64 sz)
{
  if(sz > 0)
    uvmunmap(pagetable, 0, PGROUNDUP(sz)/PGSIZE, 1);
  freewalk(pagetable);
}
```

**uvmfree**会调用uvmunmap函数解除整页映射，并释放物理内存。最后调用freewalk函数释放页表占用的内存。
**uvmunmap**函数中，如果发现最底层的pte的V标志位为0，说明没有为虚拟地址分配内存，对应了lazy allocation没有为虚拟地址分配内存，但是增加了虚拟内存大小。

```c++
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

### 修改uvmunmap函数

kernel/vm.c中的uvmunmap函数，当遇见没有分配物理内存的pte时，跳过不做处理。

```c++
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      // panic("uvmunmap: not mapped");
      continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

### 测试结果

![image-20230717160609779](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab5-6.png)

正常执行。

第一个页表错误是引发缺页中断的页面虚拟地址，与上一个实验结果一致。

第二个错误不确定是什么。

## 任务3：Lazytests and Usertests

![image-20230717161103940](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab5-7.png)

- 处理sbrk中n为负数的情况

对于n为正数的情况，采取的策略是暂时不分配新的内存，当使用到增加的那一部分虚拟内存时，发生缺页中断，再为其分配内存；
但是对于n为负数的情况，如果不立即取消这部分虚拟地址的映射，那么该进程之后不会再使用这部分内存，因此也就无法标识该进行什么操作。

综上，对于n为负数的情况就是立即调用uvmdealloc函数删除内存映射。修改sbrk函数：

```c++
  if (n < 0) 
    uvmdealloc(myproc()->pagetable, addr, addr + n);
```

- 如果进程在高于sbrk（）能够分配的虚拟内存地址上出现页面错误，则终止该进程。

对于一个进程来说，其能够分配的最高虚拟内存是MAXVA，减去两页TRAMPOLINE和TRAPFRAME，所以最高虚拟内存应该是宏定义TRAPFRAME减一页的地址。因此va不应该超过这个虚拟地址。此外，va不应该超过p->sz。

修改trap.c中的usertrap函数：

```c++
  ...
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if (r_scause() == 15 || r_scause() == 13) {
    // get va which results in page fault
    uint64 va = r_stval();
    printf("page fault: %p\n", va);
    if (va >= (TRAPFRAME - PGSIZE))
      p->killed = 1;
    else {
      if (uvmalloc(p->pagetable, PGROUNDDOWN(va), PGSIZE + PGROUNDDOWN(va)) == 0)
      p->killed = 1;
    }
    
  } else {
  ...
```

- 正确处理fork系统调用中的父进程到子进程的内存复制

在fork中，将父进程的sz复制给子进程的sz，并调用**uvmcopy**函数将父进程的页表中的内容复制到子进程的列表中。
**uvmcopy**函数为old页表中的每一个有效的虚拟地址在new页表中分配一个新的物理地址并写入相同虚拟地址和物理地址的映射。

```c++
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

这里有一点，对于old页表中（也就是父进程的页表中）全部为0的pte，也就是存在虚拟地址，但是没有分配物理内存的pte，这里是采取报错。

按照lazy allocation的策略，这里应该也为子进程的页表保存这个虚拟地址，但是不分配物理内存。所以直接continue就可以。（一位内在分配一页时，进行了初始化，默认值都是0）

```c++
    ...
    if((pte = walk(old, i, 0)) == 0)
      // panic("uvmcopy: pte should exist");
      continue;
    if((*pte & PTE_V) == 0)
      // panic("uvmcopy: page not present");
      continue;
    pa = PTE2PA(*pte);
    ...
```

- 处理当进程传递了一个使用sbrk增长的合法虚拟地址给系统调用，比如read或者write，但是这个合法虚拟地址的物理内存还没分配。

类似与read和write这些系统调用最终都需要操作虚拟地址，也就是使用vm.c/copyin或者copyout函数，而这两个函数会调用**walkaddr**函数将虚拟地址va翻译成物理地址pa。

```c++
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
```

**walkaddr**函数首先将虚拟地址va根据页表得到pte，如果pte为0或者pte的V标志位为0，这里采取的操作是直接返回0，说明没有物理地址。但是根据lazy allocation策略，这些物理地址虽然不存在，但是虚拟地址是合法的，所以应该在使用时为这些虚拟地址分配物理内存。采取和usertrap中一样的处理策略：（需要在vm.c文件中先包含"spinlock.h"再包含"proc.h"）

```c++
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if((pte == 0) || ((*pte & PTE_V) == 0)) {
    struct proc *p = myproc();

    if (va >= p->sz || va >= (TRAPFRAME - PGSIZE)) 
      return 0;
    else {
      if (uvmalloc(p->pagetable, PGROUNDDOWN(va), PGSIZE + PGROUNDDOWN(va)) == 0) 
        return 0;
      
    }
  }
  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
```

- 处理out-of-memory的情况：如果kalloc在page fault handler中失败，杀死进程。

在之前的handler中，如果出现了kalloc失败的情况，需要设置p->killed为1。分配只会发生在需要读写这块增加的内存时，不发生在复制时，因此需要在usertrap函数以及walkaddr函数中处理kalloc的情况。这两者主要是使用了uvmalloc函数，在kalloc失败时，uvmalloc返回值为0.因此当uvmalloc返回值为0时，需要设置killed为1。

- 处理发生在用户栈下面的非法错误。

也就是处理发生了访问了不属于栈的page fault。
查资料得知：page fault会产生trap，trapframe中保存用户栈的字段是sp，因此可以通过判断va是否在sp也就是用户栈的下面来判断是否为非法va。

因此需要修改usertrap和wakladdr函数的va判断条件为：

```
if (va >= p->sz || va >= (TRAPFRAME - PGSIZE) || va < PGROUNDUP(p->trapframe->s0))
```



### 测试结果

![image-20230717181158702](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab5-8.png)



# 总结

主要是针对sbrk进行lazy allocation策略，即在进程申请增加内存时，并不立即分配物理内存，而是将sz扩大。

这样当用户需要使用增加部分的内存时，就会引发page fault，从而进入trap，然后内核通过捕捉这个page fault，进行响应的处理。



























