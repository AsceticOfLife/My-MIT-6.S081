# 任务1：Uthread: switching between threads

In this exercise you will design the context switch mechanism for a user-level threading system, and then implement it. To get you started, your xv6 has two files user/uthread.c and user/uthread_switch.S, and a rule in the Makefile to build a uthread program. uthread.c contains most of a user-level threading package, and code for three simple test threads. The threading package is **missing some of the code to create a thread and to switch between threads.**

Your job is to come up with a plan to create threads and save/restore registers to switch between threads, and implement that plan. When you're done, `make grade` should say that your solution passes the `uthread` test.

Once you've finished, you should see the following output when you run `uthread` on xv6 (the three threads might start in a different order):

![image-20230730161053026](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-1.png)

This output comes from the three test threads, each of which has a loop that prints a line and then yields the CPU to the other threads.

At this point, however, with no context switch code, you'll see no output.

You will need to **add code** to `thread_create()` and `thread_schedule()` in `user/uthread.c`, and `thread_switch` in `user/uthread_switch.S`. One goal is ensure that when `thread_schedule()` runs a given thread for the first time, the thread executes the function passed to `thread_create()`, on its own stack. Another goal is to ensure that `thread_switch` saves the registers of the thread being switched away from, restores the registers of the thread being switched to, and returns to the point in the latter thread's instructions where it last left off. You will have to decide where to save/restore registers; modifying `struct thread` to hold registers is a good plan. You'll need to add a call to `thread_switch` in `thread_schedule`; you can pass whatever arguments you need to `thread_switch`, but the intent is to switch from thread `t` to `next_thread`.

## 阅读代码

整体代码框架是三个函数作为线程，一个主函数作为一个线程。

首先是主函数：

```c++
volatile int a_started, b_started, c_started;
volatile int a_n, b_n, c_n;

int 
main(int argc, char *argv[]) 
{
  a_started = b_started = c_started = 0;
  a_n = b_n = c_n = 0;
  thread_init();
  thread_create(thread_a);
  thread_create(thread_b);
  thread_create(thread_c);
  thread_schedule();
  exit(0);
}
```

首先设置a_started等变量为0，接着调用thread_init函数初始化，接着调用thread_create函数分别创建三个进程，最后调用thread_schedule函数进程线程调度。

接着看一下三个线程函数：

```c++
void 
thread_a(void)
{
  int i;
  printf("thread_a started\n");
  a_started = 1;
  while(b_started == 0 || c_started == 0)
    thread_yield();
  
  for (i = 0; i < 100; i++) {
    printf("thread_a %d\n", i);
    a_n += 1;
    thread_yield();
  }
  printf("thread_a: exit after %d\n", a_n);

  current_thread->state = FREE;
  thread_schedule();
}
```

首先设置所属的a_started变量为1，当另外进程的变量为0时就让出CPU，即只有三个进程所属变量均被设置为1时才会继续执行。

接着循环体内打印一行之后然让出CPU。

循环体结束之后输出打印次数。

最后设置进程状态为FREE。

**thread_init**函数的功能是将current_thread指向主函数。

```c++
void 
thread_init(void)
{
  // main() is thread 0, which will make the first invocation to
  // thread_schedule().  it needs a stack so that the first thread_switch() can
  // save thread 0's state.  thread_schedule() won't run the main thread ever
  // again, because its state is set to RUNNING, and thread_schedule() selects
  // a RUNNABLE thread.
  current_thread = &all_thread[0];
  current_thread->state = RUNNING;
}
```

这里主函数的线程状态设置为RUNNING，而thread_schedule函数只会选择状态为RUNNABLE的线程进行切换。

**thread_create**函数：主要功能是根据传入的函数指针创建一个线程。

```c++
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
}
```

首先查找一个空的线程结构体，然后将其状态设置为RUNNABLE，



**thread_schedule**函数：

```c++
void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
  } else
    next_thread = 0;
}
```

首先在线程中找到一个RUNNABLE的线程。循环四次，如果三个线程都不是RUNNABLE，那么next_thread就会指向0，所以就会退出。

如果找到的下一个线程不是自己，那么就需要进行线程切换。

## 保存上下文结构

由于线程切换需要保存前一个线程的上下文，并且加载新的线程的上下文。因此添加一个结构体保存CPU寄存器上下文。

```c++
// Saved registers
struct context {
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
struct context all_context[MAX_THREAD];
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context *context;      // current context
};
```

在thread_init函数中初始化每个线程的context指针：

```c++
void 
thread_init(void)
{
  for (int i = 0; i < MAX_THREAD; ++i)
    all_thread[i].context = &all_context[i];
  
  // main() is thread 0, which will make the first invocation to
  // thread_schedule().  it needs a stack so that the first thread_switch() can
  // save thread 0's state.  thread_schedule() won't run the main thread ever
  // again, because its state is set to RUNNING, and thread_schedule() selects
  // a RUNNABLE thread.
  current_thread = &all_thread[0];
  current_thread->state = RUNNING;
}
```

## thread_create函数

创建线程之后，需要保存该线程的返回地址和栈指针

```c++
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  memset(t->stack, 0, sizeof(char) * STACK_SIZE);
  t->context->ra = (uint64)(*func);
  // 注意栈的方向，否则会导致数据错乱  
  t->context->sp = (uint64)(&(t->stack[STACK_SIZE - 1]));
}
```

## thread_schedule函数

调用uthread_switch函数，从旧线程切换到新线程

```c++
...
if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)(t->context), (uint64)(next_thread->context));
  } else
    next_thread = 0;
    ...
```

## uthread_switch汇编程序

接收两个参数，第一个参数是旧线程的上下文指针，第二个参数是新线程的上下文指针。

复制kernel/swtch.S文件内容。

## 测试与总结

![image-20230731101027436](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-2.png)

总结：

首先，保存上下文信息，也就是保存返回地址、SP指针和callee寄存器。

接着，创建一个线程时，需要将context的ra即返回地址设置为线程对应的函数，因为切换时需要根据context的ra进行返回，从而能够返回新的线程的地址。另外需要保存SP指针，能够恢复线程上一次执行的状态。

最后，汇编程序switch函数主要就是将寄存器内容保存到context中，并从新的线程的context中加载寄存器内容。

# Using threads

In this assignment you will explore **parallel programming** with threads and locks using a hash table. You should do this assignment on a real Linux or MacOS computer (not xv6, not qemu) that has multiple cores. Most recent laptops have multicore processors.

This assignment uses the UNIX `pthread` threading library. You can find information about it from the manual page, with `man pthreads`, and you can look on the web, for example [here](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_mutex_lock.html), [here](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_mutex_init.html), and [here](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_create.html).

The file `notxv6/ph.c` contains a simple hash table that is correct if used from a single thread, but incorrect when used from multiple threads. In your main xv6 directory (perhaps `~/xv6-labs-2020`), type this:

```
make ph
$ ./ph 1
```

Note that to build `ph` the Makefile uses your OS's gcc, not the 6.S081 tools. The argument to `ph` specifies the number of threads that execute put and get operations on the the hash table. After running for a little while, `ph 1` will produce output similar to this:

```c++
100000 puts, 3.991 seconds, 25056 puts/second
0: 0 keys missing
100000 gets, 3.981 seconds, 25118 gets/second
```

The numbers you see may differ from this sample output by a factor of two or more, depending on how fast your computer is, whether it has multiple cores, and whether it's busy doing other things.

`ph` runs two benchmarks. First it adds lots of keys to the hash table by calling `put()`, and prints the achieved rate in puts per second. The it fetches keys from the hash table with `get()`. It prints the number keys that should have been in the hash table as a result of the puts but are missing (zero in this case), and it prints the number of gets per second it achieved.

You can tell `ph` to use its hash table from multiple threads at the same time by giving it an argument greater than one. Try `ph 2`:

You can tell `ph` to use its hash table from multiple threads at the same time by giving it an argument greater than one. Try `ph 2`:

```
$ ./ph 2
100000 puts, 1.885 seconds, 53044 puts/second
1: 16579 keys missing
0: 16579 keys missing
200000 gets, 4.322 seconds, 46274 gets/second
```

The first line of this `ph 2` output indicates that when two threads concurrently add entries to the hash table, they achieve a total rate of 53,044 inserts per second. That's about twice the rate of the single thread from running `ph 1`. That's an excellent "parallel speedup" of about 2x, as much as one could possibly hope for (i.e. twice as many cores yielding twice as much work per unit time).

However, the two lines saying `16579 keys missing` indicate that a large number of keys that should have been in the hash table are not there. That is, the puts were supposed to add those keys to the hash table, but something went wrong. Have a look at `notxv6/ph.c`, particularly at `put()` and `insert()`.

```
Why are there missing keys with 2 threads, but not with 1 thread? Identify a sequence of events with 2 threads that can lead to a key being missing. Submit your sequence with a short explanation in answers-thread.txt

To avoid this sequence of events, insert lock and unlock statements in put and get in notxv6/ph.c so that the number of keys missing is always 0 with two threads. The relevant pthread calls are:

pthread_mutex_t lock;            // declare a lock
pthread_mutex_init(&lock, NULL); // initialize the lock
pthread_mutex_lock(&lock);       // acquire lock
pthread_mutex_unlock(&lock);     // release lock
You're done when make grade says that your code passes the ph_safe test, which requires zero missing keys with two threads. It's OK at this point to fail the ph_fast test.
```

Don't forget to call `pthread_mutex_init()`. Test your code first with 1 thread, then test it with 2 threads. Is it correct (i.e. have you eliminated missing keys?)? Does the two-threaded version achieve parallel speedup (i.e. more total work per unit time) relative to the single-threaded version?

There are situations where concurrent `put()`s have no overlap in the memory they read or write in the hash table, and thus don't need a lock to protect against each other. Can you change `ph.c` to take advantage of such situations to obtain parallel speedup for some `put()`s? Hint: how about a lock per hash bucket?

Modify your code so that some `put` operations run in parallel while maintaining correctness. You're done when `make grade` says your code passes both the `ph_safe` and `ph_fast` tests. The `ph_fast` test requires that two threads yield at least 1.25 times as many puts/second as one thread.

## 哈希桶

开散列法又叫链地址法(开链法)，首先对关键码集合用散列函数计算散列地址（index = x % array.length()-1），**具有相同地址的关键码归于同一子集合**，**每一个子集合称为一个哈希桶**，各个桶中的元素通过一个单链表链接起来，各链表的头结点存储在哈希表中。

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-3.png" alt="image-20230731150858966" style="zoom:67%;" />

如图，哈希是数组+链表结构，按照传入的元素放在通过散列函数计算出的散列地址处，**如果再加入新的相同散列地址则向后通过链表将其链接在一起**。

```c++
struct entry {
  int key;
  int value;
  struct entry *next;
};
```

每一个结点都包含key、value和指向下一个结点的指针

阅读并注释代码中关于哈希桶的代码。

```c++
#define NBUCKET 5
#define NKEYS 100000

// hash表中的结点
struct entry {
  int key;
  int value;
  struct entry *next;
};
struct entry *table[NBUCKET]; // hash table
int keys[NKEYS];
```

```
// 向hash表p中的结点n前插入新的结点
static void 
insert(int key, int value, struct entry **p, struct entry *n)
{
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
  e->next = n;
  *p = e;
}

// 向hash桶中插入元素
static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }

  // 上锁
  pthread_mutex_lock(&lock[i]);
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  // 解锁
  pthread_mutex_unlock(&lock[i]);
}

// 从hash桶中找到key对应的结点
// 没找到返回空节点
static struct entry*
get(int key)
{
  int i = key % NBUCKET;


  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key) break;
  }

  return e;
}
```



## main函数

```c++
int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  double t1, t0;

  if (argc < 2) {
    fprintf(stderr, "Usage: %s nthreads\n", argv[0]);
    exit(-1);
  }
  nthread = atoi(argv[1]);  // 线程个数
  tha = malloc(sizeof(pthread_t) * nthread);  // 线程指针数组
  srandom(0);
  assert(NKEYS % nthread == 0);

  // 初始化keys数组为随机值
  for (int i = 0; i < NKEYS; i++) {
    keys[i] = random();
  }

  //
  // first the puts
  //
  t0 = now();
  // 对于每一个线程执行一次put函数
  for(int i = 0; i < nthread; i++) {
    assert(pthread_create(&tha[i], NULL, put_thread, (void *) (long) i) == 0);
  }
  for(int i = 0; i < nthread; i++) {
    assert(pthread_join(tha[i], &value) == 0);
  }
  t1 = now();

  printf("%d puts, %.3f seconds, %.0f puts/second\n",
         NKEYS, t1 - t0, NKEYS / (t1 - t0));

  //
  // now the gets
  //
  t0 = now();
  
  for(int i = 0; i < nthread; i++) {
    assert(pthread_create(&tha[i], NULL, get_thread, (void *) (long) i) == 0);
  }
  for(int i = 0; i < nthread; i++) {
    assert(pthread_join(tha[i], &value) == 0);
  }
  t1 = now();

  printf("%d gets, %.3f seconds, %.0f gets/second\n",
         NKEYS*nthread, t1 - t0, (NKEYS*nthread) / (t1 - t0));
}
```

## 修改

思路：需要加锁，因为hash table对于多个线程来说是共享资源，如果两个以上的线程同时进行修改操作，就会出现覆盖操作问题。

根据提示，只有修改同一个桶时才会出现覆盖操作，如果是修改不同的桶就不会出现覆盖问题，因此考虑为每一个桶加一个锁，当操作一个桶时必须先获取锁。

第一步：创建NBUCKET个锁

```c++
// lock
pthread_mutex_t lock[NBUCKET];

// 初始化锁
void InitLocks(void) {
  for (int i = 0; i < NBUCKET; ++i) 
    pthread_mutex_init(&lock[i], NULL);
}
```

第二步：在每个桶被修改时先上锁，修改之后释放锁（只有put操作会修改共享资源，get操作访问资源无需上锁）

```
// 向hash桶中插入元素
static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }

  // 上锁
  pthread_mutex_lock(&lock[i]);
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  // 解锁
  pthread_mutex_unlock(&lock[i]);
}
```

## 测试与总结

未修改之前的多个进程：
![image-20230731154920153](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-4.png)

修改之后的多个进程：
![image-20230731155018149](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-5.png)

对于未修改之前的单个线程：
![image-20230731155054161](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-6.png)

可以看到put速度提高了2倍，get速度提升不太明显。

![image-20230731155358258](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-7.png)

总结：

这个实验主要是说明：1.并发多线程可以提高程序速度，因为利用了多个核参与工作；2.并发会引发同步问题，即操作的顺序会影响操作的结果，主要是修改操作对于共享资源的影响。因此在修改时需要对于共享资源上锁，修改完成之后再释放锁。保证共享资源的不变性。

# Barrier

In this assignment you'll implement a [barrier](http://en.wikipedia.org/wiki/Barrier_(computer_science)): **a point in an application at which all participating threads must wait until all other participating threads reach that point too.** You'll use pthread condition variables, which are a sequence coordination technique similar to xv6's sleep and wakeup.

You should do this assignment on a real computer (not xv6, not qemu).

The file `notxv6/barrier.c` contains a broken barrier.

```
$ make barrier
$ ./barrier 2
barrier: notxv6/barrier.c:42: thread: Assertion `i == t' failed.
```

The 2 specifies **the number of threads** that synchronize on the barrier ( `nthread` in `barrier.c`). Each thread executes a loop. In each loop iteration a thread calls `barrier()` and then sleeps for a random number of microseconds. The assert triggers, because one thread leaves the barrier before the other thread has reached the barrier. The desired behavior is that each thread blocks in `barrier()` until all `nthreads` of them have called `barrier()`.

Your goal is to achieve the desired barrier behavior. In addition to the lock primitives that you have seen in the `ph` assignment, you will need the following new pthread primitives; look [here](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_cond_wait.html) and [here](https://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_cond_broadcast.html) for details.

```
pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex, acquiring upon wake up
pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond
```

Make sure your solution passes `make grade`'s `barrier` test.

`pthread_cond_wait` releases the `mutex` when called, and re-acquires the `mutex` before returning.

We have given you `barrier_init()`. Your job is to implement `barrier()` so that the panic doesn't occur. We've defined `struct barrier` for you; its fields are for your use.

There are two issues that complicate your task:

- You have to deal with a succession of barrier calls, each of which we'll call a round. `bstate.round` records the current round. You should increment `bstate.round` each time all threads have reached the barrier.
- You have to handle the case in which one thread races around the loop before the others have exited the barrier. In particular, you are re-using the `bstate.nthread` variable from one round to the next. Make sure that a thread that leaves the barrier and races around the loop doesn't increase `bstate.nthread` while a previous round is still using it.

Test your code with one, two, and more than two threads.

## barrier结构体

```c++
struct barrier {
  pthread_mutex_t barrier_mutex;
  pthread_cond_t barrier_cond;
  int nthread;      // Number of threads that have reached this round of the barrier
  int round;     // Barrier round
} bstate;
```

初始化函数：

```c++
static void
barrier_init(void)
{
  assert(pthread_mutex_init(&bstate.barrier_mutex, NULL) == 0);
  assert(pthread_cond_init(&bstate.barrier_cond, NULL) == 0);
  bstate.nthread = 0;
}
```

首先初始化barrier的锁，接着初始化cond，最后设置当前到达barrier的线程数为0.

## 线程函数

```c++
static void *
thread(void *xa)
{
  long n = (long) xa; // 当前线程序数
  long delay;
  int i;

  for (i = 0; i < 20000; i++) {
    int t = bstate.round; // 当前轮数
    // 如果当前轮数与i不一致，说明round没有等待所有线程完成一次循环就增加
    // 即没有在一个统一位置等待所有进程完成操作，所以程序退出
    assert (i == t);   
    barrier();  // 调用barrier函数，需要等待所有的线程都调用这个函数之后才能继续进行
    usleep(random() % 100);
  }

  return 0;
}
```

## main函数

```c++
int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  long i;
  double t1, t0;

  if (argc < 2) {
    fprintf(stderr, "%s: %s nthread\n", argv[0], argv[0]);
    exit(-1);
  }
  nthread = atoi(argv[1]);  // 线程数
  tha = malloc(sizeof(pthread_t) * nthread);  // 线程指针数组
  srandom(0);

  barrier_init();

  // 对于每个线程都执行thread函数
  for(i = 0; i < nthread; i++) {
    assert(pthread_create(&tha[i], NULL, thread, (void *) i) == 0);
  }
  for(i = 0; i < nthread; i++) {
    assert(pthread_join(tha[i], &value) == 0);
  }
  printf("OK; passed\n");
}
```

## 修改barrier函数

所有线程都需要执行这个函数，并且只有所有线程都执行到这个函数时才能继续运行，否则这个线程就需要阻塞自己。当所有线程都执行到这个函数时round加一。

```c++
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  // 首先获取锁
  pthread_mutex_lock(&bstate.barrier_mutex);
  // 此时需要将到达barrier的线程数加1
  bstate.nthread++;

  // 如果当前到达线程数小于总的线程数就阻塞自己
  if (bstate.nthread != nthread) {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  } else {
    // 当前达到进程数已经达到总的线程数
    bstate.round++;
    bstate.nthread = 0;
    // 解锁所有被阻塞的线程
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  
  // 释放锁
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

pthread_cond_wait(&cond, &mutex)
函数的主要作用是阻塞当前线程，并且释放拥有的锁。这里标记这个线程是因为cond而阻塞的。

pthread_cond_broadcast(&cond)
函数的主要作用是唤醒所有因cond阻塞的线程。

## 测试与总结

![image-20230731164831435](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-8.png)

（这里循环次数太长修改为100）

总结：

这个实验主要是理解线程同步，在所有线程到达barrier之前需要阻塞，当所有线程都到达了barrier之后就唤醒所有的线程。

# 总结

![image-20230731165242256](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab7-9.png)



总结：

这次试验主要是理解多个线程之间的切换、同步等关系。

1.第一个实验主要是关注在进行用户级别的线程切换时需要保存和加载那些CPU寄存器(上下文信息)；

2.第二个实验主要是关注不同线程之间访问共享资源时需要上锁和解锁的操作；

3.第三个实验主要是关注不同线程之前如何同步达到一个执行点。





