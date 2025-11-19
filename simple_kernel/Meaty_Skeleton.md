# Meaty Skeleton

在阅读本章节之前，请先阅读Getting_started一文，或更多的OS理论

阅读了没用，本文充斥各种机翻

本章节是Bare_Bones的延续，我们目前要做的事情是建立一个最小模板的操作系统，以便我们进一步修改或作为初始操作系统版本的灵感。

Bare Bones教程中，我们只提供了绝对最少的代码来演示如何正确交叉编译内核，但这不适合作为操作系统示例，此外，本章还实现了满足ABI和编译器契约所需的必要ABI功能，以防止神必错误的出现

本章还可以作为有关“如何创建您自己的Libc（标准C库）”的初始模板教程。GCC文档明确指出libgcc需要独立环境来提供Memcmp、Memcpy、Memmove和memset函数，以及在某些平台上我们将通过创建一个特殊的内核C库来满足这些要求，该库包含独立的用户空间libc部分，而不是需要执行系统调用的托管libc功能

## 前言

本文章只是给出一个示例，本文会将一些比较重要的概念嵌入到我们构建的操作系统，例如libc的存在，以及间接或次要的Unix和ABI语义。本文构建的shell脚本和基于Make的构建系统会适用于Unix系统（我将修改为可以适配debian的脚本）。没有迫切需求，它将是可以在所有操作系统上移植的。我们把这个操作系统示例命名为myos，你也可以把myos换成其他名字。

## 准备工作

你需要看过Bare Bones的内容，并成功构建一个i686-elf的gcc编译器。GRUB，它原来grub-mkrescue命令以及运行时文件，xorriso(可以不需要)，他是grub-mkrescue使用的.iso创建引擎。需要GNU make4.0或更高版本，还有QEMU，用于运行测试系统。

我们可以在debian中执行如下命令安装需要的依赖

```shell
sudo apt-get install xorriso grub-pc-bin
```

## 系统根目录

一般来说，当我们运行编译器编译程序的时候，编译器首先会在系统目录中找到开发文件，即头文件和库，比如

```
/usr/include
/usr/lib
```

当然，你也可以在自己想要的目录下写这些文件。比如

```
/home/bwayne/myos/sysroot/usr/include
/home/bwayne/myos/sysroot/usr/lib
```

其中sysroot目录是我们自己写的操作系统中的假根目录，称其为系统根目录或sysroot。

您可以将sysroot视为操作系统的根目录，当构建时操作系统的时候，内核、标准库、程序都会逐步的安装到根目录中，最后，系统根将成为功能齐全的根文件系统，您将格式化一个分区并将文件复制到那里，添加适当的配置文件，配置引导加载程序从该处加载内核、并使用硬盘驱动器和文件系统驱动程序从该处读取文件。因此，系统跟目录是一个临时文件，最后在我们完成操作系统的编写时，他才会编程实际的根目录


接下来的内容中，交叉系统根位于sysroot目录下，它是由一个构建脚本创建并由make install目标填充的目录。makefile将系统头文件安装在sysroot/usr/include目录，系统库安装在sysroot/usr/lib目录中，而内核安装在sysroot/boot目录


-elf目标平台不具备用户态空间，本质上也无法支持用户态[^1]，我们为编译器配置系统根目录(sysroot)支持，则它会在\${SYSROOT}/usr/lib路径寻找依赖，在构建i686-elf-gcc时，我们通过--without-headers禁止编译器搜索标准库，因此它不会在\${SYSROOT}/usr/include下寻找头文件，在之后我们添加用户态空间和标准库(libc)之后，需要重新配置自定义交叉编译器并启用--with-sysroot选项，此时编译器就会在${SYSROOT}/usr/include寻找头文件，在完成这一步前，我们使用-isystem=/usr/include参数修正头文件查找路径作为临时解决寻找头文件的解决方案。

当然，你可以根据需求修改根目录(sysroot)的目录结构，但这需要你修改Binutils和GCC的部分源代码，是较高阶的操作，在我们实现完整的用户态空间之前，完全没必要这样做。需要注意的是，交叉编译器(cross-linker)默认在/lib、/usr/lib和/usr/local/lib路径下查找依赖，因此无需修改Binutils，直接将相关文件移到这些目录即可。其次，我们配置GCC没有指定头文件目录，目前是通过-isystem参数指定的，所以我们可以自由调整头文件目录的位置

## 系统头文件

./headers.sh 脚本的作用仅为将你的标准库（libc）和内核相关头文件（即系统头文件）安装到 sysroot/usr/include 目录，并不会实际交叉编译你的操作系统。这一操作的实用价值在于：让你能在真正编译系统之前，就为编译器提供一套完整的头文件副本。未来当你构建「支持用户态空间的宿主式 GCC 交叉编译器」（Hosted GCC Cross-Compiler）时，就需要依赖这些标准库头文件。
需要注意的是，你的交叉编译器本身已自带一系列完全符合「独立环境（freestanding）」规范的头文件，例如 stddef.h 和 stdint.h。这类头文件仅声明常用的数据类型和宏定义，不包含任何需要依赖操作系统的实现。而你的内核标准库（kernel standard library）会提供一系列实用函数（例如字符串长度计算函数 strlen）—— 这些函数无需依赖系统调用，本质上也属于「独立环境兼容型」，唯一的区别是它们需要你在某处提供具体的实现代码。

## Makefile设计

示例中的Makefile将不会忽略我们预设的系统环境变量，在我们没特别指定选项的情况下使用默认值。实现的方式是我们在Makefile中指定两个选项，一个用于设定默认值，一个用于添加所需的强制选项

```Makefile
#默认CFLAGS
CFLAGS?=-O2 -g

# 添加强制选项到CFLAGS
CFLAGS:=${CFLAGS} -Wall -Wextra

```
## 架构目录

我们把所有和架构相关的源代码存在arch/目录下，该目录下可能还包含专属的子Makefile。用于存放特定配置，这样设计的好处是能清晰的分离你所支持的各类系统架构，并且为未来移植到其他系统提供便利。

## 内核设计
我们将内核代码放置在kernel/目录中，但在之前，我们建议——若你的内核名称和操作系统完整发行版本名称不同，或许你需要将这个目录用其他名字命名，但采用kernel/命名的好处是，能够让其他开发者快速找到你这款新型操作系统的核心部分。

内核将其公共内核头文件安装到 sysroot/usr/include/kernel 路径下，若你希望后续开发支持模块的内核，那么这些模块都只需要包含主内核提供的公共头文件即可。

启动引导程序采用GNU GRUB，而内核则采用我们之前写的Bare Bones教程中的Multiboot规范实现引导

最后，内核还实现了全局构造函数的标准调用方式，以及使用__attribute__((constructor))特性的C代码。启动汇编代码会调用_init 函数，进而触发所有全局构造函数的执行。这些函数是用来在系统启动的极早期阶段运行，且不存在特定的执行顺序。它们的作用只是为了初始化那些无法在运行的时候初始化的全局变量。__is_kernel 可以让源代码检查自身是否是内核的一部分


## libc 和 libk 设计

libc和libk本质上是同一个库的两个版本，源代码储存在libc/目录下，标准库有两种版本，独立环境(freestanding)和宿主环境版(hosted)。二者的区别是：独立版(libk)不包含任何仅适用于用户态的代码（例如系统调用的相关实现）；同时libk将使用特殊的编译器构建，这与内核的编译方式相似。

你并非必须使用Libk方案，常规的libc也是可以的，同时在内核目录中单独维护一个极简的独立模块。但libk方案的核心优势是避免代码冗余——无需重复维护strlen等基础函数的多个版本（内核态和用户态可以用同一套核心实现）


示例中并不提供完整可用的libc，编译的libc.a本质上是一个空壳子，它用于后续添加用户态功能时提供基础架构的支持，并不具备实际功能

每个标准库函数均按照"头文件目录+函数名文件"的规则存放：例如, string.h 中的strlen函数，对应的路径是libc/string/strlen.c。而sys/stat.h中的stat函数，对应的文件路径就是libc/sys/stat/stat.c

头文件采用类BSD规范设计，sys/cdefs.h声明了一批供标准库内部使用的预处理宏，所有函数原型均包裹在 extern "C" {}块中，确保C++代码能正确编译libc（因为libc不使用C++链接规则），此外，编译器会无条件提供内部关键字 __restrict，这便于在C99之前的编译器模式或C++模式中为函数原型添加restrict关键字。专用宏__is_libc用于检查源代码是否属于libc库，__is_libk检查是否是libk二进制文件


## 源代码

你可以用git 从meaty Skeleton仓库下载到源代码，命令如下：

```git
git clone https://gitlab.com/sortie/meaty-skeleton.git
```

使用如下命令检查你的clone是否是源文件

```
git diff 084d1624bedaa9f9e395f055c6bd99299bd97f58..master
```

操作系统开发需要具备一些专业知识，请务必花时间阅读和理解仓库的代码，如果有不明白的地方，建议进一步查阅资料或者寻求帮助，本示例的代码较为精简，几乎所有设计都是经过慎重考虑、目的是为了提前规避可能出现的问题


### kernel

存放路径为: kernel/include/kernel/tty.h

```C
//函数原型
#ifndef _KERNEL_TTY_H
#define _KERNEL_TTY_H

#include <stddef.h>

void terminal_initialize(void);
void terminal_putchar(char c);
void terminal_write(const char* data, size_t size);
void terminal_writestring(const char* data);

#endif
```

注：这是一个内核态TTY(终端)模块的头文件声明，用于定义终端操作的接口，预处理指令避免头文件重复包含。


### kernel/Makefile

```Makefile line-numbers
DEFAULT_HOST!=../default-host.sh
HOST?=DEFAULT_HOST
HOSTARCH!=../target-triplet-to-arch.sh $(HOST)

CFLAGS?=-O2 -g
CPPFLAGS?=
LDFLAGS?=
LIBS?=

DESTDIR?=
PREFIX?=/usr/local
EXEC_PREFIX?=$(PREFIX)
BOOTDIR?=$(EXEC_PREFIX)/boot
INCLUDEDIR?=$(PREFIX)/include

CFLAGS:=$(CFLAGS) -ffreestanding -Wall -Wextra
CPPFLAGS:=$(CPPFLAGS) -D__is_kernel -Iinclude
LDFLAGS:=$(LDFLAGS)
LIBS:=$(LIBS) -nostdlib -lk -lgcc

ARCHDIR=arch/$(HOSTARCH)

include $(ARCHDIR)/make.config

CFLAGS:=$(CFLAGS) $(KERNEL_ARCH_CFLAGS)
CPPFLAGS:=$(CPPFLAGS) $(KERNEL_ARCH_CPPFLAGS)
LDFLAGS:=$(LDFLAGS) $(KERNEL_ARCH_LDFLAGS)
LIBS:=$(LIBS) $(KERNEL_ARCH_LIBS)

KERNEL_OBJS=\
$(KERNEL_ARCH_OBJS) \
kernel/kernel.o \

OBJS=\
$(ARCHDIR)/crti.o \
$(ARCHDIR)/crtbegin.o \
$(KERNEL_OBJS) \
$(ARCHDIR)/crtend.o \
$(ARCHDIR)/crtn.o \

LINK_LIST=\
$(LDFLAGS) \
$(ARCHDIR)/crti.o \
$(ARCHDIR)/crtbegin.o \
$(KERNEL_OBJS) \
$(LIBS) \
$(ARCHDIR)/crtend.o \
$(ARCHDIR)/crtn.o \

.PHONY: all clean install install-headers install-kernel
.SUFFIXES: .o .c .S

all: myos.kernel

myos.kernel: $(OBJS) $(ARCHDIR)/linker.ld
	$(CC) -T $(ARCHDIR)/linker.ld -o $@ $(CFLAGS) $(LINK_LIST)
	grub-file --is-x86-multiboot myos.kernel

$(ARCHDIR)/crtbegin.o $(ARCHDIR)/crtend.o:
	OBJ=`$(CC) $(CFLAGS) $(LDFLAGS) -print-file-name=$(@F)` && cp "$$OBJ" $@

.c.o:
	$(CC) -MD -c $< -o $@ -std=gnu11 $(CFLAGS) $(CPPFLAGS)

.S.o:
	$(CC) -MD -c $< -o $@ $(CFLAGS) $(CPPFLAGS)

clean:
	rm -f myos.kernel
	rm -f $(OBJS) *.o */*.o */*/*.o
	rm -f $(OBJS:.o=.d) *.d */*.d */*/*.d

install: install-headers install-kernel

install-headers:
	mkdir -p $(DESTDIR)$(INCLUDEDIR)
	cp -R --preserve=timestamps include/. $(DESTDIR)$(INCLUDEDIR)/.

install-kernel: myos.kernel
	mkdir -p $(DESTDIR)$(BOOTDIR)
	cp myos.kernel $(DESTDIR)$(BOOTDIR)

-include $(OBJS:.o=.d)
```

这是内核的makefile，最终生成一个可引导的内核文件 myos.kernel。并支持安装内核和头文件。适配不同架构

第一行，default-host.sh中输出的是宿主机的架构，注意的是,makfile中的!=是执行后面的脚本并赋值给变量，所以执行完之后DEFAULT_HOST被赋予当前主机的架构。

第二行，?=的意思是默认赋值，当变量没定义的时候赋值DEFAULT_HOST给他。接着第三行，HOSTARCH的值为执行target-triplet-to-arch.sh $(HOST)后得到，这个作用只是输出简化后的架构名，比如把x86_64-pc-linux-gnu输出为x86_64。

从第五行开始，CFLAGS指的是编译器的参数，官邸给gcc/clang等c编译器，控制C代码的编译行为。其中-O2为第二优化。-g是生成调试信息，接着剩下三个参数都默认赋值为空

第10行开始，是Makefile中安装路径的默认配置。其定义头文件和内核文件的默认安装位置，第10行告诉编译器，不含有任何安装根目录前缀。11行的默认值是/usr/local。表示安装路径的根目录。接着，12行直接默认给usr/local。这是说把可执行文件安装在local目录下。13行是指定内核文件的安装目录，基本上就是/usr/local/boot。最后，就是指定内核头的安装目录，即是/usr/local/include

16行中,:=是立即赋值，即在原有的参数上拼接新的参数，其中-ffreestanding指的是独立编译模式，即不依赖任何用户态标准库，-wall开启所有警告，-Wextra 开启额外警告(空循环、多余的括号)，最后得到的是-O2 -g -ffreestanding -Wall -Wextra

下一行，-D__is_kernel -Iinclude第一个参数定义宏__is_kernel，即内核头文件可以通过#ifdef __is_kernel区分内核态代码和用户态。

而-Iinlcude是头文件搜索路径，它告诉预处理器include/下的头文件可以直接用。

28行
-nostdlib：链接时「不链接任何标准库」（如 libc.so）—— 内核不能依赖用户态的标准库（否则会导致内核崩溃，因为标准库需要操作系统支持，而内核本身就是操作系统）。
-lk：链接内核专属库 libk.a —— 这是你自己实现的 “迷你内核库”，提供 memcpy、strlen 等基础函数（替代标准库的同名函数）。
-lgcc：链接 GCC 内置库 —— 提供一些内核难以实现的底层函数（如 64 位除法、浮点运算辅助函数），GCC 会自动处理这些硬件相关逻辑。

之后，ARCHDIR=arch/$(HOSTARCH)是说定义架构相关代码的目录路径

23行开始，include语句的作用是引入$(ARCHDIR)/make.config配置文件。

之后,CFLAGS = -O2 -g -ffreestanding -Wall -Wextra -m64 -mno-red-zone -mcmodel=kernel -fno-pie -fno-pic

CPPFLAGS = -D__is_kernel -Iinclude ，启用is_kernel宏和添加引用文件路径

LDFLAGS = -m elf_x86_64 -z max-page-size=0x1000 告诉链接器（ld）“生成 x86_64 架构支持的 ELF64 二进制文件”，确保内核能被 Grub 引导，且内存分页对齐

LIBS = -nostdlib -lk -lgcc 链接时优先使用通用内核库，同时预留架构专属库的扩展接口，保证链接逻辑的兼容性。

=\是续行符号，所以这一段其实是一个完整的shell命令。他定义KERNEL_OBJS = ${KERNEL_ARCH_OBJS}/kernel/kernel.c

```
OBJS=\ # 1. OBJS：自定义变量（完整编译目标文件清单）；2. =：赋值符；3. \：续行符
$(ARCHDIR)/crti.o \  # 1. $()：引用变量；2. ARCHDIR：自定义变量（架构目录，值为 arch/x86_64）；3. /：路径分隔符；4. crti.o：目标文件（架构专属初始化汇编文件）；5. \：续行符
$(ARCHDIR)/crtbegin.o \  # 1. crtbegin.o：GCC 提供的启动文件；其他符号同上
$(KERNEL_OBJS) \  # 1. 引用上面定义的 KERNEL_OBJS 变量（内核核心文件）；2. \：续行符
$(ARCHDIR)/crtend.o \  # 1. crtend.o：GCC 提供的收尾文件；其他符号同上
$(ARCHDIR)/crtn.o \  # 1. crtn.o：架构专属收尾汇编文件；其他符号同上
```

```
LINK_LIST=\  # 1. LINK_LIST：自定义变量（链接顺序清单）；2. =：赋值符；3. \：续行符
$(LDFLAGS) \  # 1. $()：引用变量；2. LDFLAGS：自定义变量（链接器参数，如 -m elf_x86_64）；3. \：续行符
$(ARCHDIR)/crti.o \  # 同上面的 OBJS 中对应行，符号含义一致
$(ARCHDIR)/crtbegin.o \  # 同上面的 OBJS 中对应行，符号含义一致
$(KERNEL_OBJS) \  # 引用内核核心文件变量
$(LIBS) \  # 1. $()：引用变量；2. LIBS：自定义变量（依赖库参数，如 -nostdlib -lk -lgcc）；3. \：续行符
$(ARCHDIR)/crtend.o \  # 同上面的 OBJS 中对应行，符号含义一致
$(ARCHDIR)/crtn.o \  # 同上面的 OBJS 中对应行，符号含义一致
```

最后是编译环节。

首先，.PHONY是用于定义伪目标，若不使用，当目录中有和指令或其他文件同名的目标时，将会导致错误执行或不执行。所以我们需要告诉编译器当我们执行目标的时候，他应该被倒向正确的地方。

所以我们声明的all/clean/install等都是为了保证每次执行make的时候他们只会去做编译全部目标，清除编译文件和安装这些动作。而不是执行同名的操作

接着，.SUFFIXES告诉编译器，c代码和汇编代码(.c,.S)可以转为.o，即编译后的文件。

all myos.kernel指的是将所有.o编译为内核文件myos.kernel

myos.kernel检查linker.ld是否存在，若存在，则运行```$(CC) -T $(ARCHDIR)/linker.ld -o $@ $(CFLAGS) $(LINK_LIST)
	grub-file --is-x86-multiboot myos.kernel```并按照linker.ld定义的内存布局编译程序。

\$(CC)是指定你的gcc版本，-T xx.ld是指定链接脚本,-o是指定输出文件,\$(CFLAGS)是我们之前拼接的参数,\$(LINK_LIST)是之前定义的链接清单，他是如下完整的shell命令

```

gcc -T arch/x86_64/linker.ld -o myos.kernel -O2 -g -ffreestanding -Wall -Wextra -m64 ...（后续拼接 LINK_LIST 内容）

```

之后
```
$(ARCHDIR)/crtbegin.o $(ARCHDIR)/crtend.o:
	OBJ=`$(CC) $(CFLAGS) $(LDFLAGS) -print-file-name=$(@F)` && cp "$$OBJ" $@
```

编译器检查在$ARCHDIR目录下是否crtbegin.o和crtend.o，若没有，就运行后面的命令。在&&前的是首先要执行的shell命令，Gcc在使用默认的CFLAGS和LDFLAGS参数,\$@F是Makefile的变量，意思是替换目前规则的目标规则(即一开始那两个.o文件)，那么这段话就是,根据gcc的默认参数去寻找crtbegin.o文件，处理完后找crtend.o文件。并赋值给OBJs

```
.c.o:
	$(CC) -MD -c $< -o $@ -std=gnu11 $(CFLAGS) $(CPPFLAGS)

```
的意思是,自动将.c后缀的C语言文件编译成对应的.o文件，-MD是生成.d文件，记录C文件依赖的头文件,-c是仅编译不链接，即生成.o文件不生成可执行文件。\$<是一个makefile的自动变量，目的是指定第一个前置条件，在这里没有前置条件，这其实是一个默认的规则，若:后无条件，自动默认依赖和目标文件同名，即，若生成xx.o，则依赖默认为xx.c
。因为make → 要生成 myos.kernel → 需要 OBJS → OBJS 需要 KERNEL_OBJS → KERNEL_OBJS 需要 kernel/kernel.o → Makefile 必须生成 kernel/kernel.o。所以在这里\$<自动替换为kernel.c

### kernel/kernel/kernel.c

```c
#include <stdio.h>

#include <kernel/tty.h>

void kernel_main(void) {
	terminal_initialize();
	printf("Hello, kernel World!\n");
}
```

### kernel/arch/i386/tty.c

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <string.h>

#include <kernel/tty.h>

#include "vga.h"

static const size_t VGA_WIDTH = 80;
static const size_t VGA_HEIGHT = 25;
static uint16_t* const VGA_MEMORY = (uint16_t*) 0xB8000;

static size_t terminal_row;
static size_t terminal_column;
static uint8_t terminal_color;
static uint16_t* terminal_buffer;

void terminal_initialize(void) {
	terminal_row = 0;
	terminal_column = 0;
	terminal_color = vga_entry_color(VGA_COLOR_LIGHT_GREY, VGA_COLOR_BLACK);
	terminal_buffer = VGA_MEMORY;
	for (size_t y = 0; y < VGA_HEIGHT; y++) {
		for (size_t x = 0; x < VGA_WIDTH; x++) {
			const size_t index = y * VGA_WIDTH + x;
			terminal_buffer[index] = vga_entry(' ', terminal_color);
		}
	}
}

void terminal_setcolor(uint8_t color) {
	terminal_color = color;
}

void terminal_putentryat(unsigned char c, uint8_t color, size_t x, size_t y) {
	const size_t index = y * VGA_WIDTH + x;
	terminal_buffer[index] = vga_entry(c, color);
}

void terminal_scroll(int line) {
	int loop;
	char c;

	for(loop = line * (VGA_WIDTH * 2) + 0xB8000; loop < VGA_WIDTH * 2; loop++) {
		c = *loop;
		*(loop - (VGA_WIDTH * 2)) = c;
	}
}

void terminal_delete_last_line() {
	int x, *ptr;

	for(x = 0; x < VGA_WIDTH * 2; x++) {
		ptr = 0xB8000 + (VGA_WIDTH * 2) * (VGA_HEIGHT - 1) + x;
		*ptr = 0;
	}
}

void terminal_putchar(char c) {
	int line;
	unsigned char uc = c;

	terminal_putentryat(uc, terminal_color, terminal_column, terminal_row);
	if (++terminal_column == VGA_WIDTH) {
		terminal_column = 0;
		if (++terminal_row == VGA_HEIGHT)
		{
			for(line = 1; line <= VGA_HEIGHT - 1; line++)
			{
				terminal_scroll(line);
			}
			terminal_delete_last_line();
			terminal_row = VGA_HEIGHT - 1;
		}
	}
}

void terminal_write(const char* data, size_t size) {
	for (size_t i = 0; i < size; i++)
		terminal_putchar(data[i]);
}

void terminal_writestring(const char* data) {
	terminal_write(data, strlen(data));
}
```

### kernel/arch/i386/crtn.S

```
.section .init
	/* gcc will nicely put the contents of crtend.o's .init section here. */
	popl %ebp
	ret

.section .fini
	/* gcc will nicely put the contents of crtend.o's .fini section here. */
	popl %ebp
	ret
```


### kernel/arch/i386/vga.h

```c
#ifndef ARCH_I386_VGA_H
#define ARCH_I386_VGA_H

#include <stdint.h>

enum vga_color {
	VGA_COLOR_BLACK = 0,
	VGA_COLOR_BLUE = 1,
	VGA_COLOR_GREEN = 2,
	VGA_COLOR_CYAN = 3,
	VGA_COLOR_RED = 4,
	VGA_COLOR_MAGENTA = 5,
	VGA_COLOR_BROWN = 6,
	VGA_COLOR_LIGHT_GREY = 7,
	VGA_COLOR_DARK_GREY = 8,
	VGA_COLOR_LIGHT_BLUE = 9,
	VGA_COLOR_LIGHT_GREEN = 10,
	VGA_COLOR_LIGHT_CYAN = 11,
	VGA_COLOR_LIGHT_RED = 12,
	VGA_COLOR_LIGHT_MAGENTA = 13,
	VGA_COLOR_LIGHT_BROWN = 14,
	VGA_COLOR_WHITE = 15,
};

static inline uint8_t vga_entry_color(enum vga_color fg, enum vga_color bg) {
	return fg | bg << 4;
}

static inline uint16_t vga_entry(unsigned char uc, uint8_t color) {
	return (uint16_t) uc | (uint16_t) color << 8;
}

#endif
```

### kernel/arch/i386/make.config

```c
KERNEL_ARCH_CFLAGS=
KERNEL_ARCH_CPPFLAGS=
KERNEL_ARCH_LDFLAGS=
KERNEL_ARCH_LIBS=

KERNEL_ARCH_OBJS=\
$(ARCHDIR)/boot.o \
$(ARCHDIR)/tty.o \
```

### kernel/arch/i386/crti.S

```
.section .init
.global _init
.type _init, @function
_init:
	push %ebp
	movl %esp, %ebp
	/* gcc will nicely put the contents of crtbegin.o's .init section here. */

.section .fini
.global _fini
.type _fini, @function
_fini:
	push %ebp
	movl %esp, %ebp
	/* gcc will nicely put the contents of crtbegin.o's .fini section here. */
```

[^1]: 因为我们的目标是裸机，用户态是操作系统之后的事情。



















