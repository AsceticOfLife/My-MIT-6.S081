# 说明

mmap和munmap系统调用允许 UNIX 程序对其地址空间进行更为细致的控制。它们可用于在进程间共享内存，将文件映射到进程地址空间，并作为用户级page fault方案的一部分。在本实验室中，我们将在xv6中添加mmap和munmap系统调用，重点是memory-mapped files

mmap的 API 如下：

```c++
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
```

在xv6中，addr始终为 0，所以由kernel自行判断应当 map 的地址；prot表示了 mapped memory 的 R、W、X 权限，flags为MAP_SHARED或者MAP_PRIVATE，前者表示对 mapped memory 的修改应写回文件，后者则不需要；offset永远为 0，不用处理文件的偏移量；mmap 成功将返回对应内存起始地址；失败返回0xffffffffffffffff。

munmap(addr, length)需要将从 addr 开始的长度为length的内存unmap。实验指导书保证被munmap的这段内存位于mmap内存区间的头部/尾部或者是全部，munmap不会在中间挖一个洞；当内存是以MAP_SHARED模式被mmap时，需要先将修改写回文件。

看完了对于mmap和munmap的要求，发现其实测试没有一些比较难的case，为我们的实现提供了便利。

之后，就可以跟着 hints 完成实验：

- 首先添加`mmap`和`munmap`的系统调用声明，并且在`Makefile`中加入`_mmaptest`。

在文件makefile中增加生成mmaptest的指令：

```c++
	$U/_wc\
	$U/_zombie\
	$U/_mmaptest\
```

在文件user/user.h中增加用户调用的接口

```c++
void *mmap(void *addr, uint64 length, int prot, int flags, int fd, uint64 offset);
int munmap(void *addr, uint64 length);
```

在文件user/usys.pl中增加生成mmap和munmap系统调用汇编码的函数：

```c++
entry("mmap");
entry("munmap");
```

文件kernel/syscall.h，增加mmap和munmap的系统调用编号

```c++
#define SYS_mmap   22 
#define SYS_munmap  23
```

文件kernel/syscall.c，增加函数定义

```
extern uint64 sys_mmap(void); 
extern uint64 sys_munmap(void);


static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_mmap]    sys_mmap,
[SYS_munmap]  sys_munmap,
};
```



<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab10-1.png" alt="lab10-1" style="zoom:80%;" />

进程将用户文件内容映射到其虚拟地址空间内，vma（Virtual Memory Areas）的作用是记录文件（相关信息包含文件fd、offset、length）与虚拟地址空间的映射关系，每个VMA代表一段连续的虚拟内存。mmap的过程主要涉及4个过程，如图所示：

1.挑选可用VMA。每个内核进程都有一个VMA数组，在该实验里我们将该大小设置为16（也可以改成动态链表实现），在用户程序调用mmap后，内核进程在该数组内找到一个空余的VMA以供使用；
2.记录文件元数据信息。该VMA会记录文件的相关信息，包含文件的fd，文件指针，文件offset，文件长度等信息，同时，内核进程也会增加对该文件的引用计数；
3.分配虚拟地址空间。内核在用户虚拟地址空间内分配长度为length的空间，分配方法类似于sbrk函数。即通过增加proc的sz值（也就是占用heap空间）来实现。之后在VMA中记录好该虚拟地址。此时并未分配真正的物理内存。
4.物理内存分配。当用户真正需要读写该段内存时，会产生PageFault，此时我们才进行物理内存的分配，并将文件的内存读到内存中。

在文件kernel/proc.h中增加VMA的定义：

```c++
struct VMA { //增加关于VMA的定义
  uint64 addr; // VMA的开始地址
  uint64 length; // VMA的长度
  int prot; // VMA的保护位，这里可以为read/write/read_write
  int flags; // VMA的标志位，这里可以为MapShared和MapPrivate，如果为MapShared，则在内存中的脏数据需要写回至文件，否则，不需要写回文件
  int offset; // 调用mmap时，指定文件的offset值
  int valid; // 当前此VMA是否有效。1表示有效，即该VMA当前已经被使用，0表示无效，即该VMA当前还未被使用
  uint64 bitmap; // 此VMA代表的连续空间内，哪些页面已经被映射，哪些页面还未被映射
  struct file* f; // 映射的文件
};

// Per-process state
struct proc {
  struct spinlock lock;
  ...
  struct VMA vma[16]; //++++++++++++
};
```

在文件kernel/sysfile.c中增加处理VMA的函数：

```c++
// 从进程拥有的VMA数组中中找一个合法的VMNA地址
struct VMA*
get_unused_vma()
{
  struct proc* p = myproc();
  for (int i = 0; i < 16; i++) {
    if (p->vma[i].valid == 0) {
      return &(p->vma[i]);
    }
  }
  return 0;
}

// 根据虚拟地址va找到相应的VMA
struct VMA*
get_vma_by_address(uint64 va)
{
  struct proc* p = myproc();
  for (int i = 0; i < 16; i++) {
    struct VMA* curr = &(p->vma[i]);
    if (curr->valid == 0)
      continue;
    // the address is covered by the vma
    if (curr->addr <= va && curr->addr + curr->length > va) {
      return curr;
    }
  }
  return 0;
}

// 找到进程顶端地址
// 然后增长进程大小
uint64
get_vma_start_addr(uint64 length)
{
  struct proc* p = myproc();
  uint64 addr = PGROUNDUP(p->sz);
  p->sz = addr + length;
  return addr;
}


uint64
sys_mmap()
{
  uint64 addr = 0, length = 0, offset = 0;
  int prot = 0, flags = 0;
  struct file *f;
  if (argaddr(0, &addr) < 0) // addr始终为 0，所以由kernel自行判断应当 map 的地址，用户无法指定
    return -1;
  if (argaddr(1, &length) < 0)
    return -1;
  if (argint(2, &prot) < 0)
    return -1;
  if (argint(3, &flags) < 0)
    return -1;
  if (argfd(4, 0, &f) < 0)
    return -1;
  if (argaddr(5, &offset) < 0)
    return -1;

  // check operation conflict
  if ((prot & PROT_WRITE) && !(f->writable) && !(flags & MAP_PRIVATE))
    return -1;
  if ((prot & PROT_READ) && !(f->readable))
    return -1;

  struct VMA* vma = get_unused_vma();
  if (0 == vma) // not enough vma
    return -1;

  // add file reference
  filedup(f);

  // init the vma
  vma->addr = get_vma_start_addr(length);
  vma->length = length;
  vma->f = f;
  vma->flags = flags;
  vma->prot = prot;
  vma->offset = offset;
  vma->valid = 1;

  int page_num = length / PGSIZE + ((length % PGSIZE == 0) ? 0 : 1);
  int mask = 1;
  for (int i = 0; i < page_num; i++) {
    vma->bitmap |= mask;
    mask = mask << 1;
  }
  return vma->addr;
}

uint64
unmap_vma(uint64 addr, uint64 length)
{
  struct VMA* vma = get_vma_by_address(addr);
  if (0 == vma)
    return -1;

  int total = sizeof (uint64);
  struct proc* proc = myproc();

  uint64 bitmap = vma->bitmap;
  for (int i = 0; i < total; i++) {
    if ((bitmap & 1) != 0) {
      uint64 addr = vma->addr + PGSIZE * i;
      uvmunmap(proc->pagetable, addr, 1, 0);
      bitmap = (bitmap >> 1);
    }
  }
  return 0;
}

uint64
flush_content(struct VMA* vma, uint64 addr, uint64 length)
{
  struct proc* proc = myproc();
  int page_number = length / PGSIZE + ((length % PGSIZE == 0) ? 0 : 1);
  int start = (addr - vma->addr) / PGSIZE;
  for (int i = start; i < page_number; i++) {
//    if (((vma->bitmap >> i) & 1) == 1) {
      vma->f->off = PGSIZE * i;
      // write in memory content to file
      filewrite(vma->f, vma->addr, PGSIZE);
//      uvmunmap(proc->pagetable, addr + i * PGSIZE, 1, 1);
//    }
  }
  uvmunmap(proc->pagetable, addr, page_number, 1);
  return 0;
}

uint64
sys_munmap_helper(uint64 addr, uint64 length)
{
  // get the address's vma
  struct VMA* vma = get_vma_by_address(addr);
  if (0 == vma)
    return -1;

  // no need to write back
  if (vma->flags & MAP_PRIVATE) {
    return clean_vma(vma, addr, length);
  }
  // we need to write in-memory content to file
  uint64 ret = flush_content(vma, addr, length);
  if (ret < 0)
    return ret;

  return clean_vma(vma, addr, length);
}

uint64
sys_munmap()
{
  uint64 addr = 0, length = 0;
  if (argaddr(0, &addr) < 0)
    return -1;
  if (argaddr(1, &length) < 0)
    return -1;

  return sys_munmap_helper(addr, length);
}


uint64
handle_mmap_page_fault(uint64 scause, uint64 va)
{
  struct VMA* vma = get_vma_by_address(va);
  if (0 == vma)
    return -1;

  // write
  if (scause == 15 && (!(vma->prot & PROT_WRITE)))
    return -1;
  if (scause == 13 && (!(vma->prot & PROT_READ)))
    return -1;

  // allocate new physical address
  void* pa = kalloc();
  if (0 == kalloc())
    return -1;
  memset(pa, 0, PGSIZE);

  struct proc* p = myproc();
  // set pages permission
  int flag = 0;
  if (vma->prot & PROT_WRITE)
    flag |= PTE_W;
  if (vma->prot & PROT_READ)
    flag |= PTE_R;
  flag |= PTE_U;
  flag |= PTE_X;

  // page align
  va = PGROUNDDOWN(va);
  // map the virtual address and physical address
  if (mappages(p->pagetable, va, PGSIZE, (uint64)pa, flag) < 0) {
    kfree(pa);
    return -1;
  }

  // read file content to memory
  ilock(vma->f->ip);
  readi(vma->f->ip, 0, (uint64)pa, vma->offset + va - vma->addr, PGSIZE);
  iunlock(vma->f->ip);
  return 0;
}
```

kernel/trap.c，处理pagefault异常

```c++
void
usertrap(void)
{
  ...
  } else if (r_scause() == 15 || r_scause() == 13) {
    int ret = handle_mmap_page_fault(r_scause(), r_stval());
    if (ret < 0) {
      p->killed = 1;
    }
    // end+++++
  } 
  ....
}
```

修改文件kernel/proc.c，**procinit**进程初始化时初始化vma、fork时如果复制vma信息、exit时取消vma映射

```c++
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");
      p->kstack = KSTACK((int) (p - proc));
      memset(p->vma, 0, sizeof(p->vma)); //++++++
  }
}

int
fork(void)
{
    ...
    // init vma    
    for (int i = 0; i < 16; i++) {
    if (p->vma[i].valid == 1) {
      memmove(&(np->vma[i]), &(p->vma[i]), sizeof(struct VMA));
      filedup(p->vma[i].f);
    }
  }
    ...
}

int
exit(int status)
{
    ...
    // init vma    
    for (int i = 0; i < 16; i++) {
      unmap_vma(p->vma[i].addr, p->vma[i].length);
    }
    ...
}
```

文件kernel/vm.c，将懒加载导致的pannic注释掉

```c++
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
    if((pte = walk(pagetable, a, 0)) == 0) {
      panic("uvmunmap: walk");
    }
    if((*pte & PTE_V) == 0) {
      // panic("uvmunmap: not mapped");
      continue;
     }
    xxx
}

void 
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
    if((pte = walk(old, i, 0)) == 0) {
      panic("uvmcopy: pte should exist");
    }
    if((*pte & PTE_V) == 0) {
      // panic("uvmcopy: page not present");
      continue;
     }
    xxx
}
```

文件kernel/def.h，

```c++
uint64 handle_mmap_page_fault(uint64 scause, uint64 va);
uint64 unmap_vma(uint64 addr, uint64 length);
```



![image-20240319222916619](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab10-2.png)





