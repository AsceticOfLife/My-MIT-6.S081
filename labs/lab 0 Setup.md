# Ubuntu 虚拟机安装

官方Ubuntu版本：22.04.2

跟着教程一步步安装

https://blog.csdn.net/qq_43377653/article/details/126877889

# 配置vmtools

# 实验环境搭建

## 安装riscv-gnu-toolchain工具链

从gitee上下载与github上相同的主库：

```
git clone https://gitee.com/mirrors/riscv-gnu-toolchain.git
```

然后在cd进入**riscv-gnu-toolchain**目录，安装六个子模块：

```
git clone --recursive https://gitee.com/mirrors/riscv-newlib.git riscv-newlib
git clone --recursive https://gitee.com/mirrors/riscv-glibc.git riscv-glibc
git clone --recursive https://gitee.com/mirrors/riscv-gcc.git riscv-gcc
git clone --recursive https://gitee.com/mirrors/riscv-dejagnu.git riscv-dejagnu
git clone --recursive https://gitee.com/mirrors/riscv-binutils-gdb.git riscv-binutils
git clone --recursive https://gitee.com/mirrors/riscv-binutils-gdb.git riscv-gdb
```

接着安装编译所需程序：

```
sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
```

接着配置文件：

```
./configure --prefix=/usr/local
```

最后进行编译：

```
sudo make
```

需要大概一个小时

测试安装成功：

```
riscv64-unknown-elf-gcc --version
```

（由于之前配置时配置到 /usr/local，应该是加以恶搞前缀，所以应该在/usr/local/riscv64-unknown-elf-gcc能运行）

## 安装QEMU

~~下载QEMU：~~

```
git clone https://mirrors.tuna.tsinghua.edu.cn/git/qemu.git ./qemu
```

这个版本太高，无法配置成功。

找一个空文件夹，下载qemu5.1.0

```
wget https://download.qemu.org/qemu-5.1.0.tar.xz
tar xf qemu-5.1.0.tar.xz
cd qemu-5.1.0
```

安装过程可能会出错，先配置安装依赖包

如果报 `ERROR: glib-2.48 gthread-2.0 is required to compile QEMU` 错误，输入下面的命令安装工具进行解决。

```
apt-get install build-essential zlib1g-dev pkg-config libglib2.0-dev

```

如果报 `ERROR: pixman >= 0.21.8 not present. Please install the pixman devel package.` 错误，输入下面的两条命令安装工具进行解决。

```
apt-cache search pixman
apt-get install libpixman-1-dev
```

为 riscv64-softmmu 构建 QEMU。、

```
./configure --disable-kvm --disable-werror --prefix=/usr/local --target-list="riscv64-softmmu"
make
sudo make install
```

查看是否成功

```
qemu-system-riscv64 --version
```

也可以查看/usr/local/bin/目录

# 下载xv6源码



在github上下载qemu-risc-v源代码

```
git clone git://github.com/mit-pdos/xv6-riscv.git
```

或者直接下载

（下载课程中的xv6-labs-2020运行qemu会失败，可能是编译工具链的版本问题）

```
cd xv6-riscv
```

编译xv6源代码

```
make qemu
```









