# lab：locks

![image-20230803210313131](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab8-1.png)

# Memory allocator

The program user/kalloctest stresses xv6's memory allocator: three processes grow and shrink their address spaces, resulting in **many calls** to `kalloc` and `kfree`. `kalloc` and `kfree` obtain `kmem.lock`. kalloctest prints (as "#fetch-and-add") the number of loop iterations in `acquire` due to attempts to acquire a lock that another core already holds, for the `kmem` lock and a few other locks. The number of loop iterations in `acquire` is a rough measure of lock contention. The output of `kalloctest` looks similar to this before you complete the lab:

```c++
$ kalloctest
start test1
test1 results:
--- lock kmem/bcache stats
lock: kmem: #fetch-and-add 83375 #acquire() 433015
lock: bcache: #fetch-and-add 0 #acquire() 1260
--- top 5 contended locks:
lock: kmem: #fetch-and-add 83375 #acquire() 433015
lock: proc: #fetch-and-add 23737 #acquire() 130718
lock: virtio_disk: #fetch-and-add 11159 #acquire() 114
lock: proc: #fetch-and-add 5937 #acquire() 130786
lock: proc: #fetch-and-add 4080 #acquire() 130786
tot= 83375
test1 FAIL
```

`acquire` maintains, for each lock, the count of calls to `acquire` for that lock, and the number of times the loop in `acquire` tried but failed to set the lock. kalloctest calls a system call that causes the kernel to print those counts for the kmem and bcache locks (which are the focus of this lab) and for the 5 most contended locks. If there is lock contention the number of `acquire` loop iterations will be large. The system call returns the sum of the number of loop iterations for the kmem and bcache locks.

For this lab, you must use a dedicated unloaded machine with multiple cores. If you use a machine that is doing other things, the counts that kalloctest prints will be nonsense. You can use a dedicated Athena workstation, or your own laptop, but don't use a dialup machine.

The root cause of lock contention in kalloctest is that `kalloc()` has a single free list, protected by a single lock. To remove lock contention, you will have to redesign the memory allocator to avoid a single lock and list. **The basic idea is to maintain a free list per CPU, each list with its own lock.** Allocations and frees on different CPUs can run in parallel, because each CPU will operate on a different list. The main challenge will be to deal with the case in which one CPU's free list is empty, but another CPU's list has free memory; in that case, the one CPU must "steal" part of the other CPU's free list. Stealing may introduce lock contention, but that will hopefully be infrequent.

Your job is to implement per-CPU freelists, and stealing when a CPU's free list is empty. You must give all of your locks names that start with "kmem". That is, you should call `initlock` for each of your locks, and pass a name that starts with "kmem". Run kalloctest to see if your implementation has reduced lock contention. To check that it can still allocate all of memory, run `usertests sbrkmuch`. Your output will look similar to that shown below, **with much-reduced contention in total on kmem locks**, although the specific numbers will differ. Make sure all tests in `usertests` pass. `make grade` should say that the kalloctests pass.

```c++
$ kalloctest
start test1
test1 results:
--- lock kmem/bcache stats
lock: kmem: #fetch-and-add 0 #acquire() 42843
lock: kmem: #fetch-and-add 0 #acquire() 198674
lock: kmem: #fetch-and-add 0 #acquire() 191534
lock: bcache: #fetch-and-add 0 #acquire() 1242
--- top 5 contended locks:
lock: proc: #fetch-and-add 43861 #acquire() 117281
lock: virtio_disk: #fetch-and-add 5347 #acquire() 114
lock: proc: #fetch-and-add 4856 #acquire() 117312
lock: proc: #fetch-and-add 4168 #acquire() 117316
lock: proc: #fetch-and-add 2797 #acquire() 117266
tot= 0
test1 OK
start test2
total free number of pages: 32499 (out of 32768)
.....
test2 OK
$ usertests sbrkmuch
usertests starting
test sbrkmuch: OK
ALL TESTS PASSED
$ usertests
...
ALL TESTS PASSED
$
```

Some hints:

- You can use the constant `NCPU` from kernel/param.h
- Let `freerange` give all free memory to the CPU running `freerange`.
- The function `cpuid` returns the current core number, but it's only safe to call it and use its result when interrupts are turned off. You should use `push_off()` and `pop_off()` to turn interrupts off and on.
- Have a look at the `snprintf` function in kernel/sprintf.c for string formatting ideas. It is OK to just name all locks "kmem" though.

总体设计目标是：将原来多个CPU共享的一个空闲链表，划分为每一个CPU拥有一个空闲链表。并且当一个CPU自己的空闲链表为空时可以使用其它CPU的空闲链表。

HINT1：常量NCPU表示CPU的最大个数。

HINT2：freerange函数给运行freerange函数的CPU的所有空闲内存

HINT3：cpuid函数返回当前CPU的序号，但是必须关闭中断。

HINT4：查看sptintf函数查看string格式。

## 熟悉xv6多核CPU的启动过程

对于每一个CPU来说，都会从内存的0X100处执行跳转指令到0X800000处执行操作系统代码。

操作系统代码首先是执行start.c，完成寄存器的一些设置和定时器中断的初始化。注意这里每一个CPU都会执行这部分代码。

然后执行main函数，每一个CPU都会执行main函数，但是cpuid大于0的CPU会等待0号cpu完成一些初始化任务，比如页表的初始化、中断处理程序的初始化等等。然后其它CPU才会执行mian函数剩下的部分。

## 设计思路

每个CPU需要一个空闲页的链表，那么就使用一个kmem结构体数组实现，数组大小就是NCPU。

```c++
// struct {
//   struct spinlock lock;
//   struct run *freelist;
// } kmem;
struct Kmem {
  struct spinlock lock;
  struct run *freelist;
};
struct Kmem kmem[NCPU];
```

在kinit函数中需要对这个数组进行初始化，freerange函数就把当前所有空闲内存全都给运行freerange函数的CPU

```c++
void
kinit()
{
  for (int i = 0; i < NCPU; ++i) {
    push_off(); // shutdown interrupt
    int id = cpuid();
    if (id != i) {
      pop_off();
      continue;
    }

    initlock(&kmem[i].lock, "kmem");
    pop_off();
    freerange(end, (void*)PHYSTOP);
  } 
  // initlock(&kmem.lock, "kmem");
  // freerange(end, (void*)PHYSTOP);
}
```

kfree函数是将一个物理页添加到当前CPU的空闲链表上：

```c++
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  // get current cpuid
  push_off();
  int id = cpuid();
  pop_off();

  acquire(&kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&kmem[id].lock);
}
```

kalloc函数是向当前CPU申请一块空闲页，如果当前cpu没有空闲的，就像其它CPU申请一个。

```c++
void *
kalloc(void)
{
  struct run *r;

  push_off();
  int id = cpuid();
  pop_off();

  acquire(&kmem[id].lock);
  r = kmem[id].freelist;
  if(r)
    kmem[id].freelist = r->next;
  release(&kmem[id].lock);

  // steal one page from other cpu
  int temp_id = id + 1;
  while (r == 0) {
    if (temp_id >= NCPU) temp_id = 0;
    if (temp_id == id) break; // 循环一遍其它的cpu均没有可用的空闲内存

    acquire(&kmem[temp_id].lock);
    r = kmem[temp_id].freelist;
    if (r) {
      // 如果偷到一块内存
      kmem[temp_id].freelist = r->next;
      release(&kmem[temp_id].lock);
      break;
    } else {
      // 释放temp_id的锁，继续寻找其它cpu的空闲内存
      release(&kmem[temp_id].lock);
      temp_id++;
    }
  }

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

## 测试与总结

未修改之前的争用情况：
![image-20230804101748312](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab8-2.png)

修改之后的争用情况：
![image-20230804101900796](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab8-3.png)

确实降低了争用情况

![image-20230804102320510](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab8-4.png)

...

![image-20230804102343816](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab8-5.png)

总结：

对于未修改版本：整个系统中所有CPU共享一个kmem，当一个CPU上的线程使用kmem时，其它CPU线程就必须等待。也就是存在很多争用的情况。

对于修改的版本：每一个CPU都有自己的kmem，因此每个线程的争用情况减少，提高了并行程度。



# Buffer cache

This half of the assignment is independent from the first half; you can work on this half (and pass the tests) whether or not you have completed the first half.

If multiple processes use the file system intensively, they will likely **contend for** `bcache.lock`, which protects the disk block cache in kernel/bio.c. `bcachetest` creates several processes that repeatedly read different files in order to generate contention on `bcache.lock`; its output looks like this (before you complete this lab):

```c++
$ bcachetest
start test0
test0 results:
--- lock kmem/bcache stats
lock: kmem: #fetch-and-add 0 #acquire() 33035
lock: bcache: #fetch-and-add 16142 #acquire() 65978
--- top 5 contended locks:
lock: virtio_disk: #fetch-and-add 162870 #acquire() 1188
lock: proc: #fetch-and-add 51936 #acquire() 73732
lock: bcache: #fetch-and-add 16142 #acquire() 65978
lock: uart: #fetch-and-add 7505 #acquire() 117
lock: proc: #fetch-and-add 6937 #acquire() 73420
tot= 16142
test0: FAIL
start test1
test1 OK
```

You will likely see different output, but the number of `acquire` loop iterations for the `bcache` lock will be high. If you look at the code in `kernel/bio.c`, you'll see that `bcache.lock` protects the list of cached block buffers, the reference count (`b->refcnt`) in each block buffer, and the identities of the cached blocks (`b->dev` and `b->blockno`).

Modify the block cache so that the number of `acquire` loop iterations for all locks in the bcache is close to zero when running `bcachetest`. Ideally the sum of the counts for all locks involved in the block cache should be zero, but it's OK if the sum is less than 500. **Modify `bget` and `brelse`** so that **concurrent lookups and releases for different blocks** that are in the bcache are **unlikely to conflict on locks** (e.g., don't all have to wait for `bcache.lock`). You must maintain the invariant that **at most one copy of each block is cached**. When you are done, your output should be similar to that shown below (though not identical). Make sure usertests still passes. `make grade` should pass all tests when you are done.

```
$ bcachetest
start test0
test0 results:
--- lock kmem/bcache stats
lock: kmem: #fetch-and-add 0 #acquire() 32954
lock: kmem: #fetch-and-add 0 #acquire() 75
lock: kmem: #fetch-and-add 0 #acquire() 73
lock: bcache: #fetch-and-add 0 #acquire() 85
lock: bcache.bucket: #fetch-and-add 0 #acquire() 4159
lock: bcache.bucket: #fetch-and-add 0 #acquire() 2118
lock: bcache.bucket: #fetch-and-add 0 #acquire() 4274
lock: bcache.bucket: #fetch-and-add 0 #acquire() 4326
lock: bcache.bucket: #fetch-and-add 0 #acquire() 6334
lock: bcache.bucket: #fetch-and-add 0 #acquire() 6321
lock: bcache.bucket: #fetch-and-add 0 #acquire() 6704
lock: bcache.bucket: #fetch-and-add 0 #acquire() 6696
lock: bcache.bucket: #fetch-and-add 0 #acquire() 7757
lock: bcache.bucket: #fetch-and-add 0 #acquire() 6199
lock: bcache.bucket: #fetch-and-add 0 #acquire() 4136
lock: bcache.bucket: #fetch-and-add 0 #acquire() 4136
lock: bcache.bucket: #fetch-and-add 0 #acquire() 2123
--- top 5 contended locks:
lock: virtio_disk: #fetch-and-add 158235 #acquire() 1193
lock: proc: #fetch-and-add 117563 #acquire() 3708493
lock: proc: #fetch-and-add 65921 #acquire() 3710254
lock: proc: #fetch-and-add 44090 #acquire() 3708607
lock: proc: #fetch-and-add 43252 #acquire() 3708521
tot= 128
test0: OK
start test1
test1 OK
$ usertests
  ...
ALL TESTS PASSED
$
```

Please give all of your locks names that start with "bcache". That is, you should call `initlock` for each of your locks, and pass a name that starts with "bcache".

Reducing contention in the block cache is more tricky than for kalloc, because bcache buffers are truly shared among processes (and thus CPUs). For kalloc, one could eliminate most contention by giving each CPU its own allocator; that won't work for the block cache. We suggest you look up block numbers in the cache with a hash table that has a lock per hash bucket.

There are some circumstances in which it's OK if your solution has lock conflicts:

- When two processes concurrently use the same block number. `bcachetest` `test0` doesn't ever do this.
- When two processes concurrently miss in the cache, and need to find an unused block to replace. `bcachetest` `test0` doesn't ever do this.
- When two processes concurrently use blocks that conflict in whatever scheme you use to partition the blocks and locks; for example, if two processes use blocks whose block numbers hash to the same slot in a hash table. `bcachetest` `test0` might do this, depending on your design, but you should try to adjust your scheme's details to avoid conflicts (e.g., change the size of your hash table).

`bcachetest`'s `test1` uses more distinct blocks than there are buffers, and exercises lots of file system code paths.

Here are some hints:

- Read the description of the block cache in the xv6 book (Section 8.1-8.3).
- It is OK to use a fixed number of buckets and not resize the hash table dynamically. Use a prime number of buckets (e.g., 13) to reduce the likelihood of hashing conflicts.
- Searching in the hash table for a buffer and allocating an entry for that buffer when the buffer is not found must be atomic.
- Remove the list of all buffers (`bcache.head` etc.) and instead time-stamp buffers using the time of their last use (i.e., using `ticks` in kernel/trap.c). With this change `brelse` doesn't need to acquire the bcache lock, and `bget` can select the least-recently used block based on the time-stamps.
- It is OK to serialize eviction in `bget` (i.e., the part of `bget` that selects a buffer to re-use when a lookup misses in the cache).
- Your solution might need to hold two locks in some cases; for example, during eviction you may need to hold the bcache lock and a lock per bucket. Make sure you avoid deadlock.
- When replacing a block, you might move a `struct buf` from one bucket to another bucket, because the new block hashes to a different bucket. You might have a tricky case: the new block might hash to the same bucket as the old block. Make sure you avoid deadlock in that case.
- Some debugging tips: implement bucket locks but leave the global bcache.lock acquire/release at the beginning/end of bget to serialize the code. Once you are sure it is correct without race conditions, remove the global locks and deal with concurrency issues. You can also run `make CPUS=1 qemu` to test with one core.

总体设计目标是：解决多个进程对于bcache的争用。

疑问1：为什么与kalloc不同？不能让每一个CPU有一个cache？为什么使用每个哈希桶都有锁的哈希表在缓存中查找块号？





HINT1：阅读关于block cache的描述

HINT2：使用固定大小的buckets也是可以的，不用动态变化哈希表的大小；使用素数个buckets降低哈希冲突的可能性。

HINT3：在哈希表中查找一个buffer和当没有找到一个buffer从而申请一个entry的操作必须是原子的。

HINT4：



## 了解bcache缓冲块

磁盘被划分成块，每一个磁盘块对应了一个内存中的缓冲区块buffer。

```
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
};
```

buffer保证：1.磁盘块在内存中只有一个buffer缓存，一次只能有一个内存线程使用该buffer缓存。2.缓存使用较多的块。

若干个buf组成bcache：

```c++
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;
```

bread返回一个可以在内存中读取和修改的块副本buf，bwrite将修改后的buf写到磁盘上。内核线程使用完一个buf之后，必须使用brelse释放。

bcache有固定数量的buf，使用双端链表表示，但是物理内存上使用静态数组实现。

 buf结构体valid字段表示是否从磁盘读取了数据，字段disk表示缓冲区内容已经被修改需要重新写入磁盘。

bget函数：根据dev号和blockno号查找bcache中是否存在相应的buf，如果存在就返回已经上锁的buf；否则，寻找一个引用次数为0（表示没有缓冲磁盘块数据的buf）。

每一个buf的sleep-lock保护的是缓冲内容的读写，而bcache.lock保护的是缓存块buf的信息。

brelse会把使用完的buf称为head后面第一个buf。

## 设计思路

总体设计思路：使用blocknp搜索块号时，使用hash桶的形式，即定位到一个桶，这个桶上的所有号都是一样的，只是设备号不同，在这个桶上再寻找指定设备号的buf。

这样设计的好处是，一次只会获取一个桶上buf的锁，而不会获取所有buf的锁。即查找时只有块号相同时才会发生争用。

根据HINT1：使用固定大小的桶的hash表也是可以的，最好使用素数个桶，这样能够降低争用的可能性。

HINT2：在hash表中查找一个buf和buf不存在时需要创建一个entry必须是原子的。

HINT3：删除所有缓冲区的列表（bcache.head等），改为使用上次使用的时间标记缓冲区（即，使用kernel/trap.c中的tick）。通过此更改，brelse不需要获取bcache锁，bget可以根据时间戳选择最近使用最少的块。

HINT4：可以在bget中串行化驱逐（即，当查找在缓存中未命中时，bget中选择要重用的缓冲区的部分）。

HINT5：在某些情况下，您的解决方案可能需要持有两个锁；例如，在驱逐过程中，您可能需要持有bcache锁和每个bucket一个锁。确保避免死锁。

HINT6：当替换一个块时，您可能会将结构buf从一个bucket移动到另一个bucket，因为新的块会散列到不同的bucket。您可能会遇到一个棘手的情况：新块可能会与旧块哈希到同一个bucket。在这种情况下，一定要避免死锁。

HINT7：一些调试技巧：实现bucket锁，但将全局bcache.lock获取/释放留在bget的开始/结束处，以序列化代码。一旦您确定它在没有竞争条件的情况下是正确的，就删除全局锁并处理并发问题。您还可以运行makeCPUS=1qemu来使用一个核心进行测试。

## 实现

首先定义固定大小的bucket的bcache：

```c++
#define NBUCKETS 97 // number of buckets
#define NSIZE 2 // size of bucket

struct {
  struct spinlock lock;
  char name[10]; // name of buckets
  struct buf buf[NSIZE];

  // // Linked list of all buffers, through prev/next.
  // // Sorted by how recently the buffer was used.
  // // head.next is most recent, head.prev is least.
  // struct buf head;
} bcache[NBUCKETS];
```

在每个buf中添加一个时间戳：

```c++
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
  uint timestamp; // 时间戳
};
```

初始化bcache：

```c++
void
binit(void)
{
  struct buf *b;

  for (int i = 0; i < NBUCKETS; i++) {
    // 对每个桶进行初始化
    snprintf(bcache[i].name, 10, "bcache%d", i);
    initlock(&bcache[i].lock, bcache.name);

    // 初始化每个桶中的buf
    for (int j = 0; j < NSIZE; j++) {
      b = &bcache[i].buf[j];
      initsleeplock(&b->lock, "buffer");
      b->refcnt = 0; // 引用次数为0
      b->timestamp = ticks;
    }
  }

}
```

修改bget根据设备号和块号查找一个buf的过程，根据块号找到相应的桶，如果桶中存在buf，则可以直接返回；如果不存在，需要在桶中找到一个引用次数为0并且时间戳最小的buf：

```c++
static struct buf*
bget(uint dev, uint blockno) {
  struct buf *b;

  // 根据块号计算出hash表中的序号
  int id = blockno % NBUCKETS;
  // 获取序号对应的桶的锁
  acquire(&bcache[id].lock);

  // 在桶中查找是否存在块
  for (int i = 0; i < NSIZE; i++) {
    b = &bcache[id].buf[i];
    if (b->dev == dev) {
      b->refcnt++; // 引用次数加1
      release(&bcache[id].lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // 如果桶子中没有已经缓存的块
  // 那么就需要在桶子中找到一个时间戳最小的块
  // 时间戳最小说明最久没有使用过
  uint min = -1;
  struct buf *temp = 0;
  for (int i = 0; i < NSIZE; i++) {
    b = &bcache[id].buf[i];
    if (b->refcnt == 0 && B->timestamp < min) {
      min = b->timestamp;
      temp = b;
    }
  }

  // 如果找到了一个合适的buf
  if (min != -1) {
    temp->dev = dev;
    temp->blockno = blockno;
    temp->valid = 0;
    temp->refcnt = 1;
    release(&bcache[id].lock);
    acquiresleep(&temp->lock);
    return temp;
  }

  panic("bget: no buffers");
}
```

修改brelse函数，释放区块的sleep锁，并且需要获取所在桶的锁之后才能修改这个buf的引用次数，当引用次数为0时，需要更新其时间戳。

```c++
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  // 根据区块号获取这个buf对应的桶
  int id = b->blockno % NBUCKETS;
  acquire(&bcache[id].lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // 更新未被使用的时间戳
    b->timestamp = ticks;
  }
  release(&bcache[id].lock);
}
```

修改最后两个函数获取桶子级别的锁：

```c++
void
bpin(struct buf *b) {
  int id = b->blockno % NBUCKETS;
  acquire(&bcache[id].lock);
  b->refcnt++;
  release(&bcache[id].lock);
}

void
bunpin(struct buf *b) {
  int id = b->blockno % NBUCKETS;
  acquire(&bcache[id].lock);
  b->refcnt--;
  release(&bcache[id].lock);
}
```

## 测试与总结

![image-20230807103344704](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab8-6.png)

![image-20230807103625945](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab8-7.png)



总结：

在修改代码之前，BufferCache是通过双向链表连接在一起的，使用时通过最近最少使用(LRU)的算法进行buffer的分配。因为整个双向链表就一把锁（锁的粒度太大），导致高并发时BufferCache使用效率较低。

该实验的目标，是将BufferCache以HashTable的形式组织在一起，并在桶级别进行加锁，从而提升高并发下的BufferCache使用的效率。

但是注意，上述方案仍然可能出现以下争用：
两个进程使用相同的块号
当两个进程同时在缓存中未命中，并且需要找到一个未使用的块来替换时
当两个进程同时使用冲突的块时，无论您使用什么方案来划分块和锁。

# 测试与总结

![image-20230807120139114](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab8-8.png)

总结：锁主要是解决同步问题的一种方式。（同步问题指的是一些操作步骤应该具有时间上的先后顺序）

但是锁其实本质上是将一些操作串行化，因此会降低并发程度。

如果锁的粒度较大，那么整个程序都是串行的，效率就会低；

锁的粒度越小，并发程度就可以越高。







