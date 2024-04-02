# XV6

xv6是一个操作系统内核，是在ANSI C中针对多处理器x86系统的Unix第六版的现代重新实现。

RISC-V GNU 编译器工具链：我的理解是：RISC-V是一个精简指令集，即机器指令的集合，GNU编译器是针对C或者C++（后续好像还支持其它一些高级编程语言）的编译器，因此我理解的这个工具链可以将C/C++源代码编译成RISC-V指令。

QEMU：是模拟处理器软件，即可以模拟处理器运行机器指令。在这个实验中模拟的是RISC-V 64位的处理器。（应该就是能够运行这个指令集的处理器）

# 下载xv6-riscv-2020

```
git clone git://g.csail.mit.edu/xv6-labs-2020
cd xv6-labs-2020
```

编译源码

```
make qemu
```

遇到问题：

user/sh.c: In function 'runcmd':
user/sh.c:58:1: error: infinite recursion detected [-Werror=infinite-recursion]

即检测到无限递归，但是作为一个shell不就应该无限递归吗？

网上给出的方案是在这个函数的前面增加一个特殊的宏

```
__attribute__((noreturn))
```

之后就可以正常编译和运行内核。

# 书本第一章阅读

操作系统主要功能：以计算机硬件资源的抽象，管理和分配计算机资源，为用户提供接口。

xv6提供的系统调用。

进程和内存：fork的使用，exit的使用、wait的使用、exec的使用、

文件描述符：一个可被进程读写的对象，系统调用read和write、open与close、

管道：pipe，管道是一个小的内核缓冲区，作为一对文件描述符给进程，一个用于读，一个用于写。如果没有数据，读取管道的read就会等待数据被写入，或者等待所有指向写端的文件描述符被关闭。（注意，如果fork创建子进程，那么父子进程各自拥有一对管道的文件描述符，如果只关闭父进程或者子进程的写端描述符，read是不会终止的）

管道相对于临时文件的优势：1.管道会自动清理自己，临时文件需要手动删除；2.管道可以传递任意长的数据流，文件重定向需要磁盘上有足够的空间；3.管道可以分阶段并行执行，文件方式必须在第二个程序开始前完成第一个程序；4.进程间的通信，管道阻塞读写比文件的非阻塞语义更有效率

文件系统：

# sleep (easy)

![image-20230526202017876](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab1-1.png)

任务：1.接收一个字符串表示的整数参数；2.系统调用sleep。

源代码：

```c
#include "kernel/types.h" // for types define
#include "user/user.h" // for sleep(), write(), stoi()

int main(int argc, char *argv[]) {
	if (argc != 2) {
		write(2, "Pleaea input sleep time!\n", 25);
		exit(1);
	}
	int time = atoi(argv[1]);
	// system call
	sleep(time);

	exit(0);
}
```



# pingpong

![image-20230526202104976](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab1-2.png)

任务及收获：1.创建一个子进程；2.进程之间使用pipe进行通信；3.父进程使用wait等待子进程结束

源代码：

```c
#include "kernel/types.h"
#include "user/user.h"

int main() {
    // pipe file descriptor
    // fd[0] for input, fd[1] for output
    int fd1[2], fd2[2];
    // create two pipe 
    pipe(fd1);
    pipe(fd2);

    char c[1];
    int pid;

    // create child process
    // child process has the same memory(include fd)
    if (fork() == 0) {
        // receive one byte from parent process
        // wait for the one byte
        read(fd1[0], c, 1);
        pid = getpid();

        // print message
        printf("%d: received ping\n", pid);

        // send one byte to parent process
        write(fd2[1], c, 1);
    } else {
        // send one byte to child process
        c[0] = 'p';
        write(fd1[1], c, 1);

        // wait for the one byte from child process
        read(fd2[0], c, 1);
        pid = getpid();

        printf("%d: received pong\n", pid);
    }

    exit(0);
}

```



# *primes

![image-20230526202130315](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab1-3.png)

![image-20230526210221683](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab1-4.png)

这里的素数筛法的实现是：

每一个进程：左边获得的第一个数字一定是素数，然后剩下判断剩下传过来的数字是否能被素数整除，如果能够整除说明不是素数，直接抛弃；否则传递给下一个进程。（先去除区间内所有能被2整除的数，在剩下的数字中再除去所有能被3整除的数，在剩下的数字中再除去所有能被5整除的数，依次类推）

```c++
p = get a number from left neighbor
print p
loop:
    n = get a number from left neighbor
    if (p does not divide n)
        send n to right neighbor
```

遇到的问题以及收获：

1.子进程会反复执行相同的一段操作，因此需要将操作整理成函数进行递归调用。（之前写了太久只用一次的函数，忘记了函数的复用性）

2.关于pipe的读写：pipe设置了一对文件描述符，但是fork一个子进程时，子进程也拥有一对新的文件描述符，关闭父进程的两个文件描述符不会关闭子进程的；同理，关闭子进程的文件描述符不会关闭父进程的。

使用read读取pipe时，如果没有数据，那么read会等待；但是如果这个pipe的所有**写入端口**的文件描述符都被关闭，那么read返回0。另外，如果尝试读取一个不存在的pipe，那么read返回-1.

3.严重的错误是：由于xv6资源有限，因此需要注意申请的文件描述符数量。递归时应该在需要的时候再申请pipe。

错误为：

递归函数逻辑为：

（1）申请一个新的pipe；

（2）（**先关闭本进程对于旧pipe的写端口**）尝试从旧的pipe中读取一个字符，如果为0，表示上级进程已经完成向pipe中写端口，因此这是**递归的终点**，应该**关闭申请的新pipe**之后退出；否则执行（3）；

（3）处理上级进程写入旧pipe的数据，并经过一些处理之后，创建子进程，并向新pipe中写入数据；

（4）等待子进程返回。

逻辑上不合适的地方在于应该先判断是否为递归的重点，应该先判断如果不是递归的终点再去申请新的pipe，这样就会尽量减少不必要的资源消耗。

真正导致出现问题的原因是xv6支持的文件描述符不够，目前看来最多支持14，因为测试中子进程pid从4~13都能申请出一个pipe，加上主进程1个，以及默认的3个。

![image-20230526203558626](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab1-3-1.png)

上图打印出申请pipe是否成功、本次打印的prime值、尝试read的返回值

能够看出pid14的进程并没有申请成功pipe，此时进程13向右边pipe（也就是与14通信的pipe，这个pipe已经申请成功）写入数字31，由于14可以读取左边的pipe（也就是13写入的pipe），所以接下来pid14打印出prime。

~~接着按照代码逻辑，pid14创建子进程pid15，15申请成功了一个pipe，可能是由于13写完之后释放了与14之间的pipe。但是pid15 尝试read的返回值是-1，（14并没有与15的pipe有关联，因为14申请失败了），那么按道理来说**一个进程读取自己申请的pipe，并且在读取之前关闭了写端口，应该返回0才对**，测试如下：~~

```c
#include "kernel/types.h"
#include "user/user.h"

int main() {
	int fd[2];
	pipe(fd);
	
	close(pipe[1]);
	int n = 1;
	printf("%d\n", read(pipe[0], &n, 4));
	
	exit(0);
}
```

确实返回0。

进程1虽然申请成功了一个pipe，但是它是从旧pipe中读取，但是旧的pipe是创建失败的，所以read会返回-1。

**pipe创建失败时返回的文件描述符会是什么？**猜测应该是一个不可能为文件描述符范围的值，当read检测文件描述符不正确时返回-1。

修正之后的逻辑：刚好创建14个文件描述符，进程id14不创建pipe，不会发生错误。

将（1）（2）互换位置。

源代码：

```c
#include "kernel/types.h"
#include "user/user.h"

/**
 * @description: every child process should do
 * @param {int} *fd: parent process's file descriptor
 * @return {*}
 */
void func(int *old_fd) {
    int p; // the first input number from parent
    close(old_fd[1]); // close write pipe
    if (read(old_fd[0], &p, 4) == 0) {
        // can't read from parent, so and exit
        exit(0);
    }

    // new file descriptor to connect to child process
    int new_fd[2];
    pipe(new_fd);

    // first one must be prime
    printf("prime %d\n", p);

    // create child process and the child process will start from this position
    if (fork() == 0) {
        func(new_fd);
    } else {
        close(new_fd[0]); // close read pipe

        int n;
        // try to read from parent pipe
        // if pipe is already closed then read() will return 0
        while (read(old_fd[0], &n, 4) != 0) {
            // if p is not a factor of n, then pass it to child process
            // else drop it
            if (n % p != 0) {
                write(new_fd[1], &n, 4);
            }
        }
        close(old_fd[0]);
        close(new_fd[1]);
        
        wait(0); // wait for child process return
    }

}

int main() {
    // create pipe to communicate with child
    int fd[2];
    pipe(fd);

    // create child process
    if (fork() > 0) {
        close(fd[0]);

        // parent process sends 2 ~ 35
        for (int i = 2; i <= 35; i++) 
            write(fd[1], &i, 4);

        // close pipe's write
        close(fd[1]);

        // wait
        wait(0);
    } else {
        func(fd);
    }

    exit(0);
}
```

# find

![image-20230527170610728](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab1-5.png)

遇到的问题及收获：

1.在xv6的文件系统中，打开一个文件的过程为：（ls的实现流程）

（1）尝试使用open获取一个文件的file descriptor；

（2）尝试使用fstat(fd, &st)获取一个文件的信息，文件的信息使用一个结构体表示：（感觉上可以当作《操作系统》中的文件结点来看）

```c
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device

struct stat {
  int dev;     // File system's disk device
  uint ino;    // Inode number
  short type;  // Type of file
  short nlink; // Number of links to file
  uint64 size; // Size of file in bytes
};
```

（3）根据文件描述信息中的type字段决定采取哪种操作。

2.xv6系统中文件目录也是一种特殊的文件，即上面的T_DIR类型，这种文件的记录格式是固定的结构体：（即文件目录作为一种文件其中存放的是一条条dirent记录）

```c
struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```

DIRSIZE表示文件目录的大小，为14+1（感觉与上一题的最大文件描述符14个有关联）

因此可以遍历文件目录的记录项，根据name字段查找文件（当然文件也有可能是另一个目录）。

3.问题1：使用完一个fd之后一定要及时释放，不然会出现无法fd不够用的情况。如果在每次函数调用之后都不释放fd，那么函数最多递归14次就把fd资源耗尽。

4.问题2：注意不要递归当前目录文件中的.和..记录项。

源代码：

```c++
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/stat.h" //for struct stat
#include "kernel/fs.h" // for struct dirent

void find(char *path, char *targetfile) {
    static char buf[512];

    // root is a directory, which is a special file
    // try to open root
    int fd;
    if ((fd = open(path, 0)) < 0) {
        // fprintf(2, "Couldn't open %s\n", path);
        return;
    }

    // try to get state
    struct stat st;
    if (fstat(fd, &st) < 0) {
        // fprintf(2, "Couldn't stat %s\n", path);
        close(fd);
        return;
    }

    if (st.type == T_FILE) {
        // if root is a file as same as targetfile
        // compare string after last slash
        char *p;
        for (p = path + strlen(path); p >= path && *p != '/'; p--)
            ;
        p++;

        if (strcmp(p, targetfile) == 0) {
            printf("%s\n", path);
        }

    } else if (st.type == T_DIR) {
        // try every dirent in the root
        struct dirent de;

        // save current path
        strcpy(buf, path);
        // point to the current directory
        char *p = buf + strlen(buf);
        *p++ = '/'; // p指向路径后一个空字符

        while (read(fd, &de, sizeof(de)) == sizeof(de)) {
            // de is a dirent of a file
            if (de.inum == 0) continue;
            if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0) continue;

            // add de.name to path
            memmove(p, de.name, DIRSIZ);
            p[DIRSIZ] = '\0';
            // recursion
            find(buf, targetfile);
        }

    }

    close(fd);
    return;
}

int main(int argc, char **argv) {
    if (argc < 2) {
        printf("Usage: ls -s[root directory] -s[filename]\n");
        exit(1);
    }
    char *root = argv[1];
    char *targetfile = argv[2];
    
    find(root, targetfile);
    
    exit(0);
}
```

find的两个参数是：path 和 filename 。当根据path打开的是一个文件时，这里采取的做法是判断path最后一个/之后的字符是否于filename相等；当打开的是一个目录时，遍历目录文件中所有记录，将目录中的文件名添加到path后，递归调用find。

# xargs

![image-20230528115626201](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab1-6.png)

步骤：

1.调用fork和exec实现调用指令：

xargs echo abc

```c
int main(int argc, char **argv) {
    if (argc <= 1) {
        printf("Usage: xargs command1 command2...\n");
    }

    if (fork() == 0) {
        exec(argv[1], argv + 1);
        fprintf(2, "exec failed\n");
    } else {
        wait(0);
    }

    exit(0);
}
```

熟悉exec命令的格式。

2.使用管道符号|：管道符号将前一条指令的输出放在一个管道中传递给下一指令作为输入。意味着第二条指令可以使用read从fd 0 中读取前一条指令的输出。

比如：echo 123 | xargs echo abc

xargs可以从一个管道中读取123作为输入

```c
int main(int argc, char **argv) {
    if (argc <= 1) {
        printf("Usage: xargs command1 command2...\n");
    }

    char *params[MAXARG]; // MAXARG个元素的数组，每个数组元素是一个指向cha *的指针
    for (int i = 0; i < argc; ++i) 
        params[i] = argv[i];
	
    // 从管道文件中读取3个字符
    char buf[4];
    if (read(0, buf, 3) == 3) {
        params[argc++] = buf;
    }
    
    if (fork() == 0) {
        // exec("echo", "echo abc 123")
        exec(argv[1], params);
        fprintf(2, "exec failed\n");
    } else {
        wait(0);
    }

    exit(0);
}
```

**疑问：如果是把后者的输入绑定到前者的管道中，这个操作是在哪里实现的？**

3.接收前者输出为多个参数，参数与参数之间使用换行符隔开。错误：**echo 打印\n时不会打印换行符，而是打印\和n两个字符。**但是echo输出到pipe文件之后会在末尾打印一个换行符，这个是可以读出来并检测的。

4.收获：预先申请一块静态char数组作为内存缓冲区，作为本程序可能用于保存数据的地方。

最终程序逻辑确定为：从pipe中读取所有字符，然后根据其中的换行符（没有检测\和n的组合）划分命令，最后将原始命令加上新读到的命令调用exec执行。

源代码如下：

```c++
#include "kernel/types.h"
#include "kernel/param.h" // for MAXARG
#include "user/user.h"


int main(int argc, char **argv) {
    static char buf[512]; // 缓冲区用于保存pipe中的输入

    if (argc <= 1) {
        printf("Usage: xargs command\n");
        exit(1);
    }

    if (argc > MAXARG) {
        fprintf(2, "two many parameters including addational params(at most %d)\n", MAXARG);
        exit(1);
    } 

    char *params[argc]; // parameters and one additional parameter

    // save the parameters from argv
    // the first parameter is xargs which is noneed
    for (int i = 1; i < argc; ++i) 
        params[i - 1] = argv[i];

    // read additional parameters from another input
    char *p = buf;  // point to empty space of buf
    // read all the character of pipe
    while (read(0, p, 1) == 1) {
        p++;
    }
    *p = '\0'; // 缓冲区末尾添加空字符


    char *additonal[MAXARG];
    int count = 0; // 缓冲区中指令个数
    // if there is no additional params, then error and exit
    // else move on
    if (p == buf) {
        fprintf(2, "There is no additonal param!\n");
        exit(1);
    }
    char *command = buf; // 缓冲区中是pipe文件中的指令

    p = buf;
    while (*p != '\0') {
        if (*p == '\n') {
            additonal[count++] = command; // 划分缓冲区作为一条指令起点

            *p++ = '\0';
            command = p;
        } else {
            p++;
        }
    } 

    // printf("%d\n", count);
    // // output params
    // for (int i = 0; i < count; ++i) {
    //     printf("%s\n", additonal[i]);
    // }

    for (int i = 0; i < count; ++i) {
        params[argc - 1] = additonal[i]; // 在每一条xargs 后添加从pipe中读取的每一行指令
        // 创建子进程完成任务
        if (fork() == 0) {
            exec(params[0], params);
        } else {
            // wait for one command to complete
            wait(0);
        }
    }

    exit(0);
}
```

# 总结

添加time.txt文件，耗费14个小时（实际上可能更多）

make grade打分

![image-20230529161758595](D:\RegularFile\CSLearning\mit-6.S081\labs\pic\lab1-7.png)

总结这五个程序的意义；

- sleep：1.了解系统调用；2.熟悉kernel/types.h中的类型声明和user/user.h中的函数声明。
- pingpong：1.了解fork调用创建进程；2.使用pipe在父子进程之间传输信息；3.使用wait等待子进程返回
- primes：1.使用进程递归调用；2.创建多个pipe进行通信；3.注意不使用pipe时及时关闭文件描述符，文件描述符目前最多有14个；4.注意适用fork创建子进程时，父子进行各有自己的文件描述符，关闭各自的文件描述符不会影响别人；
- find：1.了解文件组织形式，目录和文件都是文件，使用open打开作为一个文件描述符；2.使用fstat根据文件描述符获取文件结点信息，结构体的形式；3.如果文件类型是目录，那么目录文件的每个记录是固定的形式，可以使用结构体获取文件目录项信息，包含name和一个常数。
- xargs：1.了解管道符号|机制是将前一个命令的输入传输到一个pipe中，后一个文件的默认文件描述符为0，是这个pipe，可以接收作为参数；2.进一步了解exec运行原理。









