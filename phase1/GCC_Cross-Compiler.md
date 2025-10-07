
[toc]


# GCC_Cross-Compiler

在这个章节，我们来试着创建适用于我们自己制作的操作系统的GCC交叉编译器。在此构建的编译器具有通用目标(i686-eif)，允许您脱离当前操作系统，这意味着不会适用宿主操作系统的头文件或库，在操作系统开发过程中，若不使用交叉编译器，也许会发生许多我们意想不到的事情，毕竟编译器不加任何参数，编译出来的程序默认是在宿主操作系统上运行的。

## 引入：什么是交叉编译器

一般来说，交叉编译器是一种在平台A(宿主机)上运行的编译器，但生成适用于平台B(目标平台)的可执行文件。这两个平台可能在CPU、操作系统和可执行格式上有所不同（但并非必须）。本文中，宿主机是我们当前使用的操作系统，而目标平台是我们自制的操作系统。这两个平台是实际不同的，不管怎么开发操作系统始终都和我们现在适用的操作系统有所不同。


## 为什么我们需要交叉编译器

主要文章:[Why do I need a Cross Compiler](https://wiki.osdev.org/Why_do_I_need_a_Cross_Compiler%3F)

简单来说，编译器必须了解正确的目标平台（CPU、操作系统）。随主机系统提供的编译器默认情况下并不知道它在编译完全不同的东西，除非传递给他传递大量存在问题的选项。这之后可能会引发各种各样的问题，解决方法就是构建一个交叉编译器。

## 选择什么样的编译器版本

文章：[Building_Gcc](https://github.com/MonsterGeo/my_os/blob/main/phase0/osdev_phase0.md)

我们推荐最新的GCC版本，旧版本也无所谓，但至少满足GCC 4.6.0。用如下命令打印GCC版本

```shell
gcc --version
```

（注：我的版本是14.2.0，使用的是最新的debian13，之后也以这个系统适配来写）
## 选择什么样的binutils版本？


主要文章：[Cross-Compiler Successful Builds](https://wiki.osdev.org/Cross-Compiler_Successful_Builds)

一样，也是使用最新最强的Binutils版本。但并非所有gcc和binutils的组合都能正常工作。如果遇到问题，那么可以尝试找出和你安装的编译器大致同时期发布的Binutils版本。使用下述命令查看binutils版本

```shell
ld --version
```

（目前我的版本是2.44）


## 确定目标平台

主要文章：[Target Triplet](https://wiki.osdev.org/Target_Triplet)

简短一点说：如果我们遵循osdev.org这个wiki上的Bare Bones教程，则我们实际上是在构建一个i686-elf的交叉编译器

## 关于arm-none-eabi-gcc的笔记

在Debian/Ubuntu系统中，apt管理器虽然提供了预编译的gcc-arm-none-eabi软件包，但因为其不含Libgcc.a库和stdint.h这样的独立C头文件，所以尽量不去使用他，相反，应该以arm-none-eabi作为$TARGET（目标平台）编译。

## 准备编译

为了构建gcc，需要如下的条件:

+ 类unix环境
+ 足够的内存和硬盘空间
+ GCC（希望替代的现有版本），或者其他C编译器
+ G++ 或其他的C++编译器
+ make、Bison、Flex、GMP、MPFR、MPC、Texinfo、ISL（可选）
+ 安装下述依赖：

```shell
sudo apt-get install bison flex libgmp3-dev libmpc-dev 	libmpfr-dev texinfo libisl-dev(可以选择不安装这个)
```

## 下载源码

您可以在如下网站下载到Binutils

[Binutils](https://ftp.gnu.org/gnu/binutils/)

我下载的是2.45

[GCC](https://ftp.gnu.org/gnu/gcc/)

下载的是15.2.0

## 编译

对于Debian，我们使用系统默认提供的gcc来构建一个可执行的交叉编译器。

将编译器装在系统目录下一般很危险，而且不太好，因此我们决定，只为自己安装，将编译器安装在/$HOME/opt/cross是一个不错的选择，若选择全局安装，则将安装在/usr/local/cross。

### 准备工作

首先，我们在安装前先添加一些东西到环境变量里面，在我们确保成功构建新的binutils后，编译器在构建过程能够检测到他们。

```shell
export PREFIX="$HOME/opt/cross"
export TARGET=i686-elf
export PATH="$PREFIX/bin:$PATH"
```

我们将交叉编译环境的所有文件存放于\$HOME/opt/cross。当然，您可以随意更改为你喜欢的任意前缀（例如/opt/cross或\$HOME/cross都是可以的）。如果您拥有管理员权限且希望将交叉编译工具链对所有用户可用，那么可以安装在/usr/local目录下，当然也可以直接安装在/usr上，这样交叉编译器和系统编译器同时存在同一环境中，但可能有其他的原因（比如目标设置错误，覆盖系统编译器或者和包管理器冲突），所以并不推荐安装在系统目录中。

我们先新建目录，叫src，它处于$HOME目录下。可以执行下面一系列命令来创建

```shell
mkdir $HOME/src #这将新建一个src文件夹在用户目录下
#解压源码
tar xvf gcc-15.2.0.tar.gz
tar xvf binutils-2.9.1.tar.gz
#使用cp 命令复制文件两个文件到src文件夹中
cp -r gcc-15.2.0 $HOME/src
cp -r binutils-2.9.1 $HOME/src

cd /
cd $HOME/src

mkdir build-binutils
cd build-binutils
../binutils-2.45/configure --target=$TARGET --prefix="$PREFIX" --with-sysroot --disable-nls --disable-werror
make
make install
```

对于2.45版本的binutils可以直接复制上述命令

这会编译 Binutils（汇编器、反汇编器以及其他各类有用的工具），这些工具可在你的系统上运行，但会处理由 $TARGET 指定格式的代码。
--disable-nls 选项会告知 Binutils 不要包含原生语言支持。这基本上是可选的操作，但能减少依赖项并缩短编译时间。而且，这样做还会让诊断信息以英语呈现，当你在论坛上提问时，论坛里的人能看懂这些信息。;-)
--with-sysroot 选项会让 Binutils 在交叉编译器中启用系统根目录（sysroot）支持，方法是将其指向一个默认的空目录。默认情况下，链接器会毫无合理技术缘由地拒绝使用系统根目录，而 GCC 能够在运行时处理这两种情况。这在后续会派上用场。

### GDB

如果你想用GDB，并且允许的计算机架构和操作系统不同，比如在x86_64开发arm。或者在arm上开发x86_64。则需要单独的交叉编译GDB，虽然从技术上来看这是Binutils的一部分，但是gdb还是位于一个独立的仓库中。安装也需要重新拉取源码

可以这样编译，和Binutils相似

```shell
../gdb.x.y.z/configure --target=$TARGET --prefix="$PREFIX" --disable-werror
make all-gdb
make install-gdb
```

### GCC

现在我们可以开始编译GCC了

编译交叉编译器需要一段时间，在多核CPU上我们可以使用并行来加快编译的速度，例如使用 ``` make -j8 all-gcc```命令。8表示同时运行的任务数。
如果您正在为x86-64平台构建交叉编译器，可能需要考虑编译不带“红区”的Libgcc（即Libgcc_without_red_zone）。


```shell

cd $HOME/src

# The $PREFIX/bin dir _must_ be in the PATH. We did that above.
which -- $TARGET-as || echo $TARGET-as is not in the PATH #检查交叉编译工具是否在环境里

mkdir build-gcc
cd build-gcc
../gcc-x.y.z/configure --target=$TARGET --prefix="$PREFIX" --disable-nls --enable-languages=c,c++ --without-headers --disable-hosted-libstdcxx

#开16个线程
make all-gcc -j16
make all-target-libgcc -j16
make all-target-libstdc++-v3 -j16
make install-gcc -j16
make install-target-libgcc -j16
make install-target-libstdc++-v3 -j16
```

我们构建了libgcc，这是一个底层支持库，编译器在编译时预期可用。链接到libgcc提供了整数、浮点数、十进制数、栈展开（对于异常处理很有用）以及其他支持函数。请注意，我们不仅仅是在运行make && make install，因为这将构建过多的内容，GCC的所有组件并非都准备好针对您未完成的操作系统。

--disable-nls与上面binutils中的用法相同。

--without-headers告诉GCC不要依赖目标上的任何C库（标准或运行时）。

--enable-languages告诉GCC不要编译它支持的所有其他语言前端，而只编译C语言（以及可选的C++）。

## 使用你刚安装的编译器

现在你有了一个 “粗糙”的交叉编译器，它目前无法访问C库或者C运行时库，所以你无法使用大多数包含标准头文件创建可执行的二进制文件。但是它却足以编译你即将制作的内核，记住。我们的工具安装在\$HOME/opt/cross(或者是你设置的$PREFIX指定的路径)

使用类似下述命令运行你的新编译器

```shell
$HOME/opt/cross/bin/$TARGET-gcc --version
```

若想简单的调用\$TARGET-gcc 调用编译器只需要添加环境变量即可

```shell
export PATH="$HOME/opt/cross/bin:$PATH"
```