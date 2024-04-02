# 任务1：Large files

In this assignment you'll increase the maximum size of an xv6 file. Currently xv6 files are limited to 268 blocks, or 268*BSIZE bytes (BSIZE is 1024 in xv6). This limit comes from the fact that an xv6 inode contains 12 "direct" block numbers and one "singly-indirect" block number, which refers to a block that holds up to 256 more block numbers, for a total of 12+256=268 blocks.

The `bigfile` command creates the longest file it can, and reports that size:

```
$ bigfile
..
wrote 268 blocks
bigfile: file is too small
$
```

The test fails because `bigfile` expects to be able to create a file with 65803 blocks, but unmodified xv6 limits files to 268 blocks.

You'll change the xv6 file system code to support a "doubly-indirect" block in each inode, containing 256 addresses of singly-indirect blocks, each of which can contain up to 256 addresses of data blocks. The result will be that a file will be able to consist of up to 65803 blocks, or 256*256+256+11 blocks (11 instead of 12, because we will sacrifice one of the direct block numbers for the double-indirect block).

**Preliminaries**

The `mkfs` program creates the xv6 file system disk image and determines how many total blocks the file system has; this size is controlled by `FSSIZE` in `kernel/param.h`. You'll see that `FSSIZE` in the repository for this lab is set to 200,000 blocks. You should see the following output from `mkfs/mkfs` in the make output:

```
nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
```

This line describes the file system that `mkfs/mkfs` built: it has 70 meta-data blocks (blocks used to describe the file system) and 199,930 data blocks, totaling 200,000 blocks.
If at any point during the lab you find yourself having to rebuild the file system from scratch, you can run `make clean` which forces make to rebuild fs.img.

**What to Look At**

The format of an on-disk inode is defined by `struct dinode` in `fs.h`. You're particularly interested in `NDIRECT`, `NINDIRECT`, `MAXFILE`, and the `addrs[]` element of `struct dinode`. Look at Figure 8.3 in the xv6 text for a diagram of the standard xv6 inode.

<img src="D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab9-1.png" alt="image-20230811154620100" style="zoom:67%;" />

The code that finds a file's data on disk is in `bmap()` in `fs.c`. Have a look at it and make sure you understand what it's doing. `bmap()` is called both when reading and writing a file. When writing, `bmap()` allocates new blocks as needed to hold file content, as well as allocating an indirect block if needed to hold block addresses.

`bmap()` deals with two kinds of block numbers. The `bn` argument is a "logical block number" -- a block number within the file, relative to the start of the file. The block numbers in `ip->addrs[]`, and the argument to `bread()`, are disk block numbers. You can view `bmap()` as mapping a file's logical block numbers into disk block numbers.

**Your Job**

Modify `bmap()` so that it implements a doubly-indirect block, in addition to direct blocks and a singly-indirect block. You'll have to have only 11 direct blocks, rather than 12, to make room for your new doubly-indirect block; you're not allowed to change the size of an on-disk inode. The first 11 elements of `ip->addrs[]` should be direct blocks; the 12th should be a singly-indirect block (just like the current one); the 13th should be your new doubly-indirect block. You are done with this exercise when `bigfile` writes 65803 blocks and `usertests` runs successfully:

```
$ bigfile
..................................................................................................................
wrote 65803 blocks
done; ok
$ usertests
...
ALL TESTS PASSED
$ 
```

`bigfile` will take at least a minute and a half to run.

Hints:

- Make sure you understand `bmap()`. Write out a diagram of the relationships between `ip->addrs[]`, the indirect block, the doubly-indirect block and the singly-indirect blocks it points to, and data blocks. Make sure you understand why adding a doubly-indirect block increases the maximum file size by 256*256 blocks (really -1, since you have to decrease the number of direct blocks by one).
- Think about how you'll index the doubly-indirect block, and the indirect blocks it points to, with the logical block number.
- If you change the definition of `NDIRECT`, you'll probably have to change the declaration of `addrs[]` in `struct inode` in `file.h`. Make sure that `struct inode` and `struct dinode` have the same number of elements in their `addrs[]` arrays.
- If you change the definition of `NDIRECT`, make sure to create a new `fs.img`, since `mkfs` uses `NDIRECT` to build the file system.
- If your file system gets into a bad state, perhaps by crashing, delete `fs.img` (do this from Unix, not xv6). `make` will build a new clean file system image for you.
- Don't forget to `brelse()` each block that you `bread()`.
- You should allocate indirect blocks and doubly-indirect blocks only as needed, like the original `bmap()`.
- Make sure `itrunc` frees all blocks of a file, including double-indirect blocks.

## 设计目标

使用一个直接块实现二级引用块，即使用使用一个直接块号指向一个二级block，这个block中共有256个下标，每一个下标指向一个一级block，每一个一级block中有256个下标，这些下标指向一个装有文件内容的block。

不允许改变disk inode的大小。

前11个addrs表示direct block，第12个addr表示singly-indirect block，第13个表示doubly-indirect block。

HINT1：理解bmap函数。画出直接块、间接块等关系。

```c++
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  // 如果bn是直接块的内容
  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT; // 减去直接块的序号，在间接块中的序号

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    // 如果间接块没有分配
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);  // 获取buf缓冲块
    a = (uint*)bp->data;  // buf->data: 1024个uchar元素的数组，转换成uint元素的数组，于是变为256个元素，每个元素4个uchar
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev); // 申请一个块并且修改buf内容  
      log_write(bp);  // 记录到日志中
    }
    brelse(bp); // 释放buf缓冲块
    return addr;
  }

  panic("bmap: out of range");
}
```

HINT2：考虑如何使用逻辑块编号对双间接块及其指向的间接块进行索引。

对于直接块，逻辑块号[0, NDIRECT)，即[0, 11)；

对于一级间接块，逻辑块号首先减去NDIRECT, 然后逻辑块号为[0, NINDIRECT)，即[0, 256)

对于二级间接块，逻辑块号再次减去NINDIRECT，首先整除 NINDIRECT，得到二级引用块中的序号，接着mod NINDIRECT，得到一级引用块中的序号。

HINT3：如果您更改NDIRECT的定义，您可能需要更改文件.h中结构inode中addrs[]的声明。请确保结构inode和结构dinode的addrs[]数组中的元素数量相同。

HINT4：如果更改了NDIRECT的定义，请确保创建一个新的fs.img，因为mkfs使用NDIRECT来构建文件系统。

HINT5：如果您的文件系统进入坏状态，可能是崩溃，请删除fs.img（从Unix而不是xv6执行此操作）。make将为您构建一个新的干净的文件系统映像。

HINT6：别忘了brelse（）你遍历的每个块。

HINT7：应该仅根据需要分配间接块和双重间接块，如原始的bmap（）。

HINT8：确保itrunc释放文件的所有块，包括双间接块。

## 实现

修改dinode中addrs的结构，以及inode结构体内容：

fs.h

```c++
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};
```

file.h

```c++
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+2];
};
```

修改bmap函数，增加对于二级引用块的处理

```c++
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  // 如果bn是直接块的内容
  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT; // 减去直接块的序号，在间接块中的序号

  // 处理一级引用块
  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    // 如果间接块没有分配
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);  // 获取buf缓冲块
    a = (uint*)bp->data;  // buf->data: 1024个uchar元素的数组，转换成uint元素的数组，于是变为256个元素，每个元素4个uchar
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev); // 申请一个块并且修改buf内容  
      log_write(bp);  // 记录到日志中
    }
    brelse(bp); // 释放buf缓冲块
    return addr;
  }
  bn -= NINDIRECT;

  // 处理二级引用块
  if (bn % NINDIRECT < NINDIRECT) {
    if ((addr = ip->addrs[NDIRECT + 1]) == 0) 
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
    // 获取二级buf块，buf块内容是256个指向另一个buf的下标
    bp = bread(ip->dev, addr);
    a = (uint *)bp->data;
    if ((addr = a[bn % NINDIRECT]) == 0) {
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);

    // 获取一级buf块，buf块内容是256个指向data数据块的下标
    bp = bread(ip->dev, addr);
    a = (uint *)bp->data;
    if ((addr = a[bn / NINDIRECT]) == 0) {
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);

    return addr;    
  }

  panic("bmap: out of range");
}
```

修改itrunc，删除二级引用块

```c++
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  struct buf *bp2;
  uint *a;
  uint *a2;

  // 释放直接块
  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  // 释放一级引用块
  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  // 释放二级引用块
  if (ip->addrs[NDIRECT + 1]) {
    // 获取二级引用
    bp2 = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a2 = (uint *)bp2->data;
    for (i = 0; i < NINDIRECT; ++i) {
      if (a2[i]) {
        // 获取一级引用
        bp = bread(ip->dev, a2[i]);
        a = (uint *)bp->data;
        for (j = 0; j < NINDIRECT; ++j) {
          // 遍历释放data块
          if (a[j]) bfree(ip->dev, a[j]);
        }
        // 释放一级引用块
        brelse(bp);
        bfree(ip->dev, a2[i]);
      }
    }
    brelse(bp2); // 释放二级引用块
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  } 

  ip->size = 0;
  iupdate(ip);
}
```

## 测试与总结

测试发现失败，查阅资料发现是需要修改MAXFILE的定义

```
#define NDOUBLE ((NINDIRECT) * (NINDIRECT)) // 两次跳转索引可增加的Block引用数量
#define MAXFILE (NDIRECT + (NINDIRECT) + (NDOUBLE))
```

![image-20230812162658591](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab9-2.png)

![image-20230812163201367](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab9-3.png)



总结：通过设计二级引用块增加文件大小。

# 任务2：Symbolic links

In this exercise you will add symbolic links to xv6. Symbolic links (or soft links) refer to a linked file by pathname; **when a symbolic link is opened, the kernel follows the link to the referred file.** Symbolic links resembles hard links, but hard links are restricted to pointing to file on the same disk, while symbolic links can cross disk devices. Although xv6 doesn't support multiple devices, implementing this system call is **a good exercise to understand how pathname lookup works.**

**Your job**

You will implement the `symlink(char *target, char *path)` system call, which creates a new symbolic link at path that refers to file named by target. For further information, see the man page symlink. To test, add symlinktest to the Makefile and run it. Your solution is complete when the tests produce the following output (including usertests succeeding).

```
$ symlinktest
Start: test symlinks
test symlinks: ok
Start: test concurrent symlinks
test concurrent symlinks: ok
$ usertests
...
ALL TESTS PASSED
$ 
```

Hints:

- First, create a new system call number for symlink, add an entry to user/usys.pl, user/user.h, and implement an empty sys_symlink in kernel/sysfile.c.
- Add a new file type (`T_SYMLINK`) to kernel/stat.h to represent a symbolic link.
- Add a new flag to kernel/fcntl.h, (`O_NOFOLLOW`), that can be used with the `open` system call. Note that flags passed to `open` are combined using a bitwise OR operator, so your new flag should not overlap with any existing flags. This will let you compile user/symlinktest.c once you add it to the Makefile.
- Implement the `symlink(target, path)` system call to create a new symbolic link at path that refers to target. Note that target does not need to exist for the system call to succeed. You will need to choose somewhere to store the target path of a symbolic link, for example, in the inode's data blocks. `symlink` should return an integer representing success (0) or failure (-1) similar to `link` and `unlink`.
- Modify the `open` system call to handle the case where the path refers to a symbolic link. If the file does not exist, `open` must fail. When a process specifies `O_NOFOLLOW` in the flags to `open`, `open` should open the symlink (and not follow the symbolic link).
- If the linked file is also a symbolic link, you must recursively follow it until a non-link file is reached. If the links form a cycle, you must return an error code. You may approximate this by returning an error code if the depth of links reaches some threshold (e.g., 10).
- Other system calls (e.g., link and unlink) must not follow symbolic links; these system calls operate on the symbolic link itself.
- You do not have to handle symbolic links to directories for this lab.

## 设计思路

实现符号链接的系统调用。

## 实现步骤

**根据HINT1：增加系统调用**

修改kernel/syscall.h文件：

```
#define SYS_symlink 22
```

修改kernel/syscall.c文件

```c++
extern uint64 sys_symlink(void);

...
[SYS_close]   sys_close,
[SYS_symlink] sys_symlink,
};
```

修改kernel/sysfile.c文件，增加一个空的sys_symlink函数

```
uint64 sys_symlink(void) {
  
}
```

修改user/user.h文件，增加一个系统调用

```
int symlink(char *target, char *path);
```

修改user/usys.pl文件，增加entry

```
entry("symlink");
```

**根据HINT2：在kernel/stat.h中增加一种新的文件类型**

```
#define T_SYMLINK 4
```



**根据hINT3：在kernel/fcntl.h中增加一个新的flag，被用于open系统调用。**

请注意，传递给open的标志是使用逐位OR运算符组合的，因此您的新标志不应与任何现有标志重叠。这将允许您在将user/symlinktest.c添加到Makefile后编译它。



**根据HINT4：**实现symlink（target，path）系统调用，在引用target的路径上创建一个新的符号链接。请注意，系统调用成功不需要存在目标。您将需要选择存储符号链接的目标路径的位置，例如，在inode的数据块中。symlink应该返回一个整数，表示成功（0）或失败（-1），类似于link和unlink。

逻辑为：新建一个inode，并且设置类型为SYS_SYMLINK，并且inode的data block内容是target，最后将inode内容写入磁盘，再删除inode。

```c++
uint64 sys_symlink(void) {
  char target[MAXPATH], path[MAXPATH];
  struct inode *ip;

  // 从寄存器中获取参数
  if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0) 
    return -1;

  begin_op();
  // 创建一个新的inode
  ip = create(path, T_SYMLINK, 0, 0);
  if (ip == 0) {
    end_op();
    return -1;
  }

  // 将目标路径写入inode的data block
  if (writei(ip, 0, (uint64)target, 0, MAXPATH) != MAXPATH) 
    return -1;
  
  // inode已经使用完毕
  iunlockput(ip);
  end_op();
  return 0;
}
```



**HINT5：**修改open系统调用，处理关于类型为symlink的情况。如果文件不存在，open失败。当进程在要打开的标志中指定O_NOFOLLOW时，open应该打开符号链接（而不是跟随符号链接）。

**HINT6：**如果链接的文件也是符号链接，则必须递归地跟随它，直到到达非链接文件。如果链接形成一个循环，则必须返回一个错误代码。如果链接的深度达到某个阈值（例如，10），您可以通过返回错误代码来近似这一点。

```

```

## 测试与总结

![image-20230813172624981](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab9-4.png)

![image-20230813173054157](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab9-5.png)

总结：

link类型的文件相当于对于其它文件的引用，可以根据link类型文件去找到相应的文件内容。

link类型文件保存的是真正文件的路径名。使用路径名去找到真正的文件。

# 测试

![image-20230813173939579](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab9-6.png)

















