# Lab: Copy-on-Write Fork for xv6

![image-20230719104456067](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab6-1.png)

xv6中的fork（）系统调用将父进程的所有用户空间内存复制到子进程中。如果父对象很大，则复制可能需要很长时间。更糟糕的是，这项工作往往被大量浪费；例如，子级中的fork（）后面跟exec（）会导致子级丢弃复制的内存，而可能从未使用过大部分内存。另一方面，如果父级和子级都使用一个页面，并且其中一个或两个都写入该页面，则确实需要一个副本。

写时复制（COW）fork（）的目标是推迟为子级分配和复制物理内存页，直到实际需要副本（如果有的话）。

COW fork（）只为子级创建一个页面表，用户内存的PTE指向父级的物理页面。COW fork（）将父级和子级中的所有用户PTE标记为不可写。当任一进程试图写入其中一个COW页面时，CPU将强制执行页面故障。内核页面错误处理程序检测到这种情况，为出错进程分配一页物理内存，将原始页面复制到新页面中，并修改出错进程中的相关PTE以引用新页面，这一次PTE标记为可写。当页面错误处理程序返回时，用户进程将能够写入页面的副本。

COW fork（）使得释放实现用户内存的物理页面变得有点棘手。一个给定的物理页面可能被多个进程的页面表引用，并且只有当最后一个引用消失时才应该释放。

# 任务：实现COW

![image-20230719105221036](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab6-2.png)

Here's a reasonable plan of attack.

1. Modify uvmcopy() to map the parent's physical pages into the child, instead of allocating new pages. Clear `PTE_W` in the PTEs of both child and parent.
2. Modify usertrap() to recognize page faults. When a page-fault occurs on a COW page, allocate a new page with kalloc(), copy the old page to the new page, and install the new page in the PTE with `PTE_W` set.
3. Ensure that each physical page is freed when the last PTE reference to it goes away -- but not before. A good way to do this is to keep, for each physical page, a "reference count" of the number of user page tables that refer to that page. Set a page's reference count to one when `kalloc()` allocates it. Increment a page's reference count when fork causes a child to share the page, and decrement a page's count each time any process drops the page from its page table. `kfree()` should only place a page back on the free list if its reference count is zero. It's OK to to keep these counts in a fixed-size array of integers. You'll have to work out a scheme for how to index the array and how to choose its size. For example, you could index the array with the page's physical address divided by 4096, and give the array a number of elements equal to highest physical address of any page placed on the free list by `kinit()` in kalloc.c.
4. Modify copyout() to use the same scheme as page faults when it encounters a COW page.

Some hints:

- The lazy page allocation lab has likely made you familiar with much of the xv6 kernel code that's relevant for copy-on-write. However, you should not base this lab on your lazy allocation solution; instead, please start with a fresh copy of xv6 as directed above.
- It may be useful to have a way to record, for each PTE, whether it is a COW mapping. You can use the RSW (reserved for software) bits in the RISC-V PTE for this.
- `usertests` explores scenarios that `cowtest` does not test, so don't forget to check that all tests pass for both.
- Some helpful macros and definitions for page table flags are at the end of `kernel/riscv.h`.
- If a COW page fault occurs and there's no free memory, the process should be killed.

## 为每个物理页面添加引用计数

这里采取的方式是使用一个整数数组，数组的大小是所有物理页面的个数，可以使用(PHYSTOP) - (KERNBASE)) / (PGSIZE)计算，数组里每个值表示物理页面被引用的次数。一个物理页面在数组中的下标的计算方式为：((((uint64)pa) - KERNBASE) >> 12)

首先在riscv.h中添加宏：

```c++
#define PANUM (((PHYSTOP) - (KERNBASE)) / (PGSIZE))
#define INDEX(pa) ((((uint64)pa) - KERNBASE) >> 12)
```

然后在kernel/kalloc.c文件中增加一个整数数组：

```c++
int refNum[PANUM]; // reference times of each physical page
```

应该在kalloc一个物理页面时设置引用计数为1，在kfree一个物理页面时减少引用计数，只有当引用计数为1时才将物理页面添加到freelist中。

接着考虑对于引用数组的初始化：在kinit中会调用**freerange**函数对于所有的物理页面进行初始化，初始化包括对于每一个物理页面调用kfree函数，也就是会使得引用计数减一。所以应该在kinit函数中初始化数组每一个元素为1。这样才能保证调用kfree函数时能够使得每一个物理页面引用次数减一，从而添加进freelist。

```c++
void
kinit()
{
  initlock(&kmem.lock, "kmem");

  // initialize refNum
  // freerange need to call kalloc
  // kalloc firstly decrease reference, then try to free it when reference is 0
  for (int i = 0; i < PANUM; ++i)
    refNum[i] = 1;

  freerange(end, (void*)PHYSTOP);
}
```

接着在kfree函数中，先将一个物理页面的引用次数减一，然后如果减一之后引用次数为0，再将起添加到freelist列表中。
这里需要注意的是，这一个引用数组应该是一个共享变量，类似于freelist，在多处理器中，要考虑并发的问题，即同一时间间隔内只能有一个进程进行修改。所以先上锁。这里借用的是kmem的锁，因为想要归还一个物理页面，必须让数组拥有的锁释放。

```c++
  // try to free the pa
  acquire(&kmem.lock);
  // decrease reference count
  if (--refNum[INDEX(pa)] > 0) {
    release(&kmem.lock);
    return;
  }
  release(&kmem.lock);
```

接着修改kalloc函数，当初次申请一个物理页面时，应该将引用计数设置为1：

```c++
  ...
  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    kmem.freelist = r->next;
    refNum[INDEX(r)] = 1;
  }
    
  release(&kmem.lock);
  ...
```

最后增加两个函数，分别用于增加和减少引用次数：

```c++
void decreaseRef(uint64 pa) {
  acquire(&kmem.lock);
  --refNum[INDEX(pa)];
  release(&kmem.lock);
}

void increaseRef(uint64 pa) {
  acquire(&kmem.lock);
  ++refNum[INDEX(pa)];
  release(&kmem.lock);
}
```



## 修改fork

fork函数就是调用allocproc函数申请一个新的进程控制块，然后将父进程的内容都复制给子进程。

涉及到内存复制的操作就是调用**uvmcopy**函数将父进程页表的内容复制给子进程的页表。

**uvmcopy**函数遍历父进程页表的表项，检查pte是否为0和是否合法，然后为子进程页表申请一页内存，将父进程的物理内存的内容复制进子进程的新的物理内存。

目前的要求是：子进程并不立即分配物理内存，而是虚拟地址的pte指向父进程的物理地址。此外，这块物理地址变成只读，并且使用一位标志位表示这些物理地址变成了COW。
首先在riscv.h中声明COW标志位：

```
#define PTE_COW (1L << 8)
```

接着修改uvmcopy函数，使得新的页表的虚拟地址的PTE指向的是旧的页表的物理地址，并且两个PTE均设置为不可写以及COW标志位。

```c++
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  // char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");

    // modify flags
    (*pte) &= (~PTE_W); // forbid write
    (*pte) |= PTE_COW;  // COW
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);

    if (mappages(new, i, PGSIZE, pa, flags) != 0)
      goto err;
    increaseRef(pa);
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

然后每当一个物理页面与一个页表解绑时，需要调用uvmunmap函数，所以需要修改这个函数：

```c++
    ...
    uint64 pa = PTE2PA(*pte);
    if(do_free)
      kfree((void*)pa);
    else decreaseRef(pa);
	...
    *pte = 0;
```



## 修改usertrap

在usertrap函数中捕捉页表错误，根据之前设置的RSW位判断是否为一个cow页面引发的错误.如果是一个cow页面错误，需要进行的操作是：1.申请一个新的物理页面；2.将旧的物理页面的内容复制到新页面中；3.将新的物理页面安装到页表中。

新增一个cow handler函数：（需要在def.h中声明walk函数原型）

```c++
int cowhandler(pagetable_t pagetable, uint64 va) {
  // get the va(must aligned to PGSIZE)
  va = PGROUNDDOWN(va);
  
  // get pte
  pte_t *pte = walk(pagetable, va, 0);
  if (pte == 0) return -1; // not existed
  if ((*pte & PTE_COW) == 0) return -1; // not cow
  if ((*pte & PTE_V) == 0) return -1; // invalid
  
  // translate to physical address
  uint64 pa = PTE2PA(*pte);
  if (refNum[(pa - KERNBASE) / PGSIZE] == 1) {
    // only belongs to parent or child process
    (*pte) |= (PTE_W);
    (*pte) &= (~PTE_COW);

    return 0;
  }

  // allocate a new physical page
  char *mem;
  if ((mem = kalloc()) == 0) return -1;

  // copy memory to new from old
  memmove(mem, (void *)pa, PGSIZE);
  // decrease reference of old pa
  kfree((void *)pa);

  // get and modify flags
  uint64 flags = (PTE_FLAGS(*pte) | (PTE_W)) & (~PTE_COW);
  *pte = PA2PTE((uint64)mem) | flags;

  return 0;
}
```

## 修改copyout函数

copyout函数由内核调用，将一个地址的内容复制到一个页面的虚拟地址中。所以当需要复制一个COW的页面时，应该调用cow handler函数为其分配物理内存。

这里需要单独写一个返回PTE的函数，因为在原来的walk函数中，如果遇到va大于MAXVA的时候，会panic，这里如果va大于MAXVA，直接返回0.

```c++
pte_t* walkpte(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  if((*pte & PTE_U) == 0)
    return 0;
  return pte;
}

```

修改copyout函数：

```c++
  ...
  uint64 n, va0, pa0;
  pte_t *pte;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pte = walkpte(pagetable, va0);
    if (pte == 0) return -1;
    // if pte is a cow
    if ((*pte & PTE_COW)) {
      // cow handler
      if (cowhandler(pagetable, va0) < 0) {
        panic("copyout: cow handler failed");
      } else pte = walkpte(pagetable, va0); // turn into non-cow page
    }

    pa0 = PTE2PA(*pte);
    if(pa0 == 0)
    ...
```



## 测试结果

![image-20230720210531875](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab6-3.png)

# 总结

实现COW，首先是fork时，使得子进程的va映射到父进程的pa。这样就会使得一个物理地址有两个虚拟地址引用。因此需要标志这是一个COW页面。

接着是需要一个数组标识每个物理页面到底有多少次引用，第一次分配时引用次数为1，只有当引用次数为0时才能释放。

最后是捕捉COW页面错误，这个时候cow handler应该为其真正分配物理内存。























































