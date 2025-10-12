[toc]

# Bare_Bones

在该节，我们将编写一个32位的 x86内核并启动它，这是我们创建操作系统的第一步，我们的目的是创建一个最小系统，而并非如何正确构建项目。这些指导原则经过社区审查，遵循当前推荐方案（且这些方案的设计均有充分依据）。请警惕网上大量的同类教程，它们大多不遵守现代技术建议，且多由缺乏经验的开发者编写，可能存在兼容性、安全和功能性问题。

我们即将开始新操作系统的开发之旅，或许未来某一天，我们的操作系统能够在自身环境下完成自我开发——这一过程被称为自举(bootstrapping) 或实现自托管(going self-hosted)。不过今天，我们只需要搭建一个基础系统，能在现有操作系统（宿主系统）中编译你的新操作系统（目标系统）。这一过程就是“交叉编译”(cross-compiling)，这是操作系统开发的第一步。


本文将使用现有技术来帮助你入门并迅速进入内核开发的部分，而不是让您开发自己的编程语言、编译器、或bootloader，本文中我们将使用

+ 使用来自Binutils的GNU链接器将您的目标文件链接成最终的内核
+ 来自Binutils的GNU汇编器（或选择NASM）用于将指令汇编翻译为包含机器码的目标文件
+ 来自GNU的编译器合集将您的高等级代码编译为汇编
+ 使用C或C++编写你的高等级内核代码
+ GRUB引导加载程序使用Multiboot引导协议来引导你的内核，该协议将我们加载到禁用分页的32位保护模式下
+ ELF作为可执行格式。给予我们控制内核加载位置和方式的能力

我们假定你正在使用类似于Linux的Unix-like操作系统，该系统对操作系统提供了良好的支持，Windows用户可在WSL、MinGW或Cygwin软件包中完成相应的操作

在操作系统开发领域取得成功，需要成为专家、保持耐心以及非常仔细地阅读所有指令。在继续之前，您需要阅读本文中的所有内容。如遇问题，您需要更加细致地重新阅读本文，并且为了确保效果，再进行三次操作。如果您仍然遇到问题，OSDev社区经验丰富，将乐意在论坛或IRC上提供帮助。

## 编译你的交叉编译器

文章:[GCC Cross-Compiler, Why do I need a Cross Compiler?](https://wiki.osdev.org/Why_do_I_need_a_Cross_Compiler%3F)

翻译：[编译GCC交叉编译器](https://github.com/MonsterGeo/my_os/blob/main/phase1/GCC_Cross-Compiler.md)

你首先要做的是为 i686-elf 架构搭建一个 GCC 交叉编译器（GCC Cross-Compiler）。由于你尚未对编译器进行修改，使其能识别你的操作系统，因此需要使用一个名为 i686-elf 的通用目标架构 —— 它会为你提供一个面向 System V ABI（应用程序二进制接口）的工具链。该配置在操作系统开发（osdev）社区中经过了充分的测试和验证，能让你借助 GRUB（多引导加载程序）和 Multiboot（多引导规范）轻松搭建可引导的内核。（注意：如果你当前使用的是 Linux 等 ELF 平台，可能已经拥有能生成 ELF 格式程序的 GCC 编译器，但它并不适用于操作系统开发工作。因为这种编译器生成的程序是针对 Linux 系统的，而你的操作系统无论与 Linux 有多相似，都不是 Linux。如果不使用交叉编译器，后续必然会遇到问题。）
没有交叉编译器，你将无法正确编译自己的操作系统。
使用 x86_64-elf 架构的交叉编译器也无法完成本教程 —— 因为 GRUB 仅支持加载 32 位的 Multiboot 内核。如果这是你的第一个操作系统开发项目，应先从 32 位内核开始。若你改用 x86_64 编译器，且即便通过了后续的完整性检查，最终得到的内核也无法被 GRUB 识别并引导。

## 综述

现在，我们已经设置了编译器为i686-elf平台的格式，我们继续最小内核的示例（当然，也是基于x86操作系统的）。在示例中，你只需要三个输入文件：

+ boot.s 内核入口点，用于设置处理器环境
+ kernel.c 你的内核例程
+ linker.ld 链接上述文件用

## 启动操作系统

在操作系统被启动之前，将会先运行一段程序，这段程序被用来加载操作系统的，也被称为引导加载程序。等一下我们将看看怎么使用GRUB，配置/编写自己的引导加载程序。

你需要处理的第一个任务，就是了解引导加载程序如何启动内核。操作系统开发者（OSDevers）其实很幸运，因为存在一套多引导规范（Multiboot Standard）—— 它定义了引导加载程序与操作系统内核之间的简易交互接口。该规范的实现原理是：在一些全局变量中存入特定的 “魔法数字”（这些变量集合被称为多引导头部（multiboot header）），引导加载程序会主动搜索这些魔法数字。当检测到魔法数字时，引导加载程序就会识别出该内核符合多引导规范，进而知道如何加载它；甚至还能向内核传递内存映射（memory maps）等重要信息，不过目前你暂时用不到这些信息。
由于此时栈尚未建立，且你需要确保全局变量的配置无误，因此这部分工作需要通过汇编语言（assembly） 来完成

我们将使用nasm来作为汇编编译器。

现在，我们创建一个叫boot.s的文件并讨论其，我们使用的是GNU汇编器，这是之前构建的交叉编译工具链的一部分，该汇编器与GNU工具链的其他部分相容。

该文件中最关键的部分是多重引导头部(multiboot header)，因为其必须位于内核二进制文件的前部，否则引导加载器无法识别我们的系统。

```asm
/* 声明多引导头部的常量 */
.set ALIGN,    1<<0             /* 把加载的模块对其到页面边界 */
.set MEMINFO,  1<<1             /* 提供内存映射 */
.set FLAGS,    ALIGN | MEMINFO  /* 这是多引导的'标志'字段 */
.set MAGIC,    0x1BADB002       /* ‘魔法数字’的作用是让bootloader找到引导头 */
.set CHECKSUM, -(MAGIC + FLAGS) /* 校验、检查是否为多引导*/


/* 
声明一个将程序标记为内核的多重引导（multiboot）头部。这些是在多重引导标准中有明确文档说明的 “魔法数字”（magic values）。引导加载程序（bootloader）会在内核文件的前 8 千字节（8 KiB）范围内、按 32 位边界对齐的位置搜索这个签名。该签名位于独立的段（section）中，这样就能确保头部被强制放置在内核文件的前 8 千字节内。
*/
.section .multiboot
.align 4
.long MAGIC
.long FLAGS
.long CHECKSUM

/*
多重引导标准尚未定义栈寄存器(esp)指针值，它必须由内核提供栈空间。通过下述方式为小型栈分配空间：首先在底部定义一个符号，接着为栈分配16384字节（16kb）的内存，然后在栈的顶部定义另一个符号，在x86架构下，栈是向下生长的，即栈指针从高位往地位移动。

该栈位于独立的段（section）内，这样便可将其标记为nobits（无数据）类型——意味着内核文件体积会更小，因为它无需包含未初始化的栈数据（因为没初始化必须存储在二进制文件中，加载到内存后由系统自动初始化为0）

根据System V应用程序二进制接口（ABI）标准及实际应用中的拓展规范，x86架构下的栈必须满足16字节对齐要求，若栈未按要求对齐，将可能导致未定义行为（比如程序崩溃，数据损坏等不可预测的结果）

.section .bss
.align 16
stack_bottom:
.skip 16384 # 16 KiB
stack_top:
*/

链接脚本指定了 _start 作为进入内核的入口并且当内核加载完成时BootLoader会跳转到此，由于加载完毕BootLoader就会消失，因此该函数的返回是无意义的。

.section .text
.global _start
.type _start, @function
_start:
    /*bootloader已经在x86机器的32位保护模式下加载_start程序。中断将被禁用，分页功能也未被启用。处理器状态符合引导规范(Multiboot Standard)的定义，内核完全掌控cpu。

    内核只能使用硬件自身具备的功能，以及其自身所包含的代码，当前环境中是不存在printf函数的，除非内核自行提供专属的<stdio.h>头文件以及printf实现。此外，也不存在任何的安全限制、保护机制或调试工具，这些功能需要内核自己实现。此时内核对计算机拥有绝对、完整的控制权
    */

    /*
    为了设置堆栈，我们设置esp寄存器指向栈顶（并随着系统向下找），这些都需要使用汇编完成，因为C语言这类需要堆栈才能运行。
    */
    mov $statck_top ,%esp

    /*
    这是在进入高级别内核之前，初始化关键处理器的状态的理想位置。应尽量缩短关键功能处于未启用状态的早期环境时长。注意的是，此时处理器也尚未完全初始化，诸如浮点指令和指令集拓展等功能尚未启用。全局描述符（GDT）也应该在此处加载，分页功能也在此处启用，C++的特性（如全局构造函数和异常处理）同样需要运行时支持才能正常工作。
    */

    /*
    进入高级别内核，应用程序二进制接口（ABI）要求在执行call指令时，栈必须保持16字节对齐（call执行后会将4字节的返回地址压栈）
    目前，栈已经初始化为16字节对齐状态，并且从那时起我们向栈压入数据量都是16bit的整数倍(虽然现在没有压任何数据)，因此栈是以保持对齐的状态存在的，此时是可以执行call指令，因为符合规范。
    */

    call kernel_main
    /* 
    若系统没有其他任务，则计算机将进入如下的无限循环：
    1) 使用cli指令禁用中断（清eflags寄存器的允许中断标志）。不过引导程序已经禁用了中断，所以这一步并非必须。但注意，之后你可能会启用中断并从kernel_main函数返回（其实这步没什么意义）
    2) 使用hlt指令（停机指令）等待下一个中断到来，由于此时中断被禁用，计算机进入停滞状态

    3) 如果因不可屏蔽中断(NMI)发生或系统管理模式(SMM)触发被唤醒，就回到Hlt指令继续停机
    cil
1:
    hlt
    jmp 1b
/*
将_start符号大小设置为当前位置'.'减去其起始位置。这在调试中非常有用
*/
.size _start, . - _start
```

接着，上一节我们安装了i686的gcc，使用链接包内的i686-elf-as来编译汇编命令

```shell
$HOME/opt/cross/bin/i686-elf-as boot.s -o boot.o
```

## 实现内核

我们刚才编写了引导汇编存根(bootstrap assembly stub)，用来初始化处理器状态，使c等高级语言能够运行，实际上，也可以使用其他的语言(比如c++)来编写代码。

### 独立和托管环境

如果你在用户空间做过c/c++编程，那你肯定使用过所谓的宿主环境(Hosted Environment)。“宿主”意味着存在C标准库以及其他实用的运行时特性。与之对应的是独立环境(Freestanding)，也就是现在正在使用的这种。“独立”意味着没有C标准库，只有你自己提供的东西。

不过，有一些头文件并不属于C标准库，而是编译器的一部分，即便独立的C源代码中，这些头文件仍然可以使用，在这种情况下，你可以使用
+ <stdbool.h>来获取bool数据类型
+ <stddef.h>来获取size_t类型和NULL
+ <stdint.h>获取Int_x和uintx_t类型。

此外，你还能使用 <float.h>、<iso646.h>、<limits.h> 和 <stdarg.h> 这些头文件，因为它们也支持独立环境。GCC 实际上还附带了更多头文件，但那些是特殊用途的。

## 在C上写内核

下面我们展示如何使用C写一个简单的内核，该内核使用VGA文本缓冲区（位于地址0xB8000处）作为输出设备，它会初始化一个简易驱动，该驱动记录缓冲器中下一个字符的输出位置并提供一个用于添加新字符的功能。

需要注意的是，目前不支持换行符('\n')（写入该字符会显示某个VGA特定的字符而非实现换行），也不支持屏幕填满后的滚动功能，这两项功能是我们要实现的第一个任务，请花点时间理解下述代码：

重要提示：VGA 文本模式（以及 BIOS）在较新的机器上已被弃用，UEFI 仅支持像素缓冲区（framebuffer）。为了向前兼容，你可能需要从像素缓冲区开始着手。可以通过适当的 Multiboot 标志让 GRUB 设置一个帧缓冲区，或者自己调用 VESA VBE（视频电子标准协会的视频 BIOS 扩展）来实现。
与 VGA 文本模式不同，帧缓冲区由像素构成，因此你需要自行绘制每个字符的字形。这意味着你需要一个不同的terminal_putchar函数，还需要一套字体（每个字符对应的位图图像）。所有 Linux 发行版都附带了 PC 屏幕字体（PC Screen Fonts）可供使用，相关维基百科文章中也有一个简单的putchar()实现示例。
除此之外，本文中描述的其他所有内容仍然适用（比如你仍需要跟踪光标位置、实现换行和滚动功能等）。

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

/*检查编译器是否认为你的目标系统有误*/
#if defined(__linux__)
#error " you are not using a cross-compiler, you will most certainly run into trouble"
#endif

/*该教程仅使用x86的32位系统*/
#if !defined(__i386__)
#error "This tutorial needs to be compiled with a ix86-elf compiler"
#endif 


/*硬件文本模式颜色常量定义*/
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

// 组合颜色属性
static inline uint8_t vga_entry_color(enum vga_color fg, enum vga_color bg) 
{
	return fg | bg << 4;  // 前景色占低4位，背景色占高4位
}
// VGA 文本模式中，每个字符的颜色属性用 1 字节表示：高 4 位是背景色，低 4 位是前景色。例如，vga_entry_color(VGA_COLOR_RED, VGA_COLOR_BLACK) 会返回 0x04（红色文字 + 黑色背景）

//构建VGA缓冲器条目


static inline uint16_t vga_entry(unsigned char uc, uint8_t color) 
{
	return (uint16_t) uc | (uint16_t) color << 8;
}

//VGA 文本模式的显存中，每个字符占 2 字节：低 8 位是 ASCII 字符，高 8 位是颜色属性。这个函数将字符和颜色组合成一个 16 位的值，直接写入显存即可显示。

//辅助函数和常量
size_t strlen(const char* str) 
{
	size_t len = 0;
	while (str[len])  // 遍历到字符串结束符'\0'
		len++;
	return len;
}
//标准库的strlen函数在裸机环境中不可用，因此手动实现：遍历字符串直到遇到'\0'，返回长度。

//VGA文本模式常量
#define VGA_WIDTH   80   // 屏幕宽度（列数）
#define VGA_HEIGHT  25  // 屏幕高度（行数）
#define VGA_MEMORY  0xB8000  // VGA文本模式显存起始地址

//x86 架构中，VGA 文本模式的默认分辨率是 80 列 ×25 行，显存映射到物理地址0xB8000，直接读写该地址即可控制屏幕显示。

//终端状态变量
size_t terminal_row;       // 当前光标行号
size_t terminal_column;    // 当前光标列号
uint8_t terminal_color;    // 当前字符颜色属性
uint16_t* terminal_buffer = (uint16_t*)VGA_MEMORY;  // VGA显存指针

//这些全局变量记录终端的状态：光标位置、当前颜色，以及指向 VGA 显存的指针（用于直接操作屏幕）。

//终端初始化
void terminal_initialize(void) 
{
	terminal_row = 0;          // 光标初始位置：第0行
	terminal_column = 0;       // 第0列
	terminal_color = vga_entry_color(VGA_COLOR_LIGHT_GREY, VGA_COLOR_BLACK);  // 初始颜色：浅灰文字+黑背景
	
	// 清空屏幕：将所有位置填充为空格（用当前颜色）
	for (size_t y = 0; y < VGA_HEIGHT; y++) {
		for (size_t x = 0; x < VGA_WIDTH; x++) {
			const size_t index = y * VGA_WIDTH + x;  // 计算显存索引（行×列数 + 列）
			terminal_buffer[index] = vga_entry(' ', terminal_color);
		}
	}
}

/*
作用：设置光标初始位置为左上角（0, 0）；
设置默认颜色（浅灰文字 + 黑背景）；
清空屏幕（用空格覆盖所有位置）。*/

//终端操作函数

//设置颜色
void terminal_setcolor(uint8_t color) 
{
	terminal_color = color;  // 更新当前颜色属性
}

//在指定位置输出字符
void terminal_putentryat(char c, uint8_t color, size_t x, size_t y) 
{
	const size_t index = y * VGA_WIDTH + x;  // 计算显存中的位置索引
	terminal_buffer[index] = vga_entry(c, color);  // 写入字符和颜色
}
//直接操作 VGA 显存，在指定（x,y）坐标输出字符（x 是列，y 是行）。

// 输出单个字符并更新光标
void terminal_putchar(char c) 
{
	// 在当前光标位置输出字符
	terminal_putentryat(c, terminal_color, terminal_column, terminal_row);
	
	// 更新光标位置：列号+1；若达到最大列数（80），则换行
	if (++terminal_column == VGA_WIDTH) {
		terminal_column = 0;  // 列号重置为0
		if (++terminal_row == VGA_HEIGHT)  // 行号+1；若达到最大行数（25）
			terminal_row = 0;  // 行号重置为0（循环覆盖，无滚动）
	}
}
// 在当前光标位置输出字符；
// 自动更新光标：列满则换行，行满则回到第 0 行（覆盖屏幕顶部，没有滚动功能）；
// 注意：当前不支持'\n'换行符（输入'\n'会显示一个乱码字符）。

//输出字符
void terminal_write(const char* data, size_t size) 
{
	for (size_t i = 0; i < size; i++)
		terminal_putchar(data[i]);  // 逐个字符输出
}

//内核入口函数

void kernel_main(void) 
{
	terminal_initialize();  // 初始化终端
	terminal_writestring("Hello, kernel World!\n");  // 输出测试字符串
}
// 首先初始化终端（清屏、设置光标和颜色）；
// 输出字符串"Hello, kernel World!\n"，但由于terminal_putchar不支持'\n'，换行符会显示为乱码
```

注意这段代码中一个细节：你本想使用常用的 C 函数strlen，但该函数属于 C 标准库 —— 而你当前的（内核）环境中并没有 C 标准库可用。因此，你转而依赖独立环境头文件<stddef.h>提供size_t类型，并自行实现了strlen函数。
对于所有你想使用的（标准库）函数，都需要这样做 —— 因为独立环境头文件（freestanding headers）仅提供宏定义和数据类型（如<stddef.h>的size_t、<stdbool.h>的bool），并不会提供函数的具体实现。

用下述命令编译：

```shell
$HOME/opt/cross/bin/i686-elf-gcc -c kernel.c -o kernel.o -std=gnu99 -ffreestanding -O2 -Wall -Wextra
```
c++的编译方法：

```shell
$HOME/opt/cross/bin/i686-elf-g++ -c kernel.c++ -o kernel.o -ffreestanding -O2 -Wall -Wextra -fno-exceptions -fno-rtti
```

## 链接你的内核

现在你可以汇编boot.s文件并编译kernel.c文件了，这一步会生成两个目标文件，每个文件包含内核的一部分，要创建完整的最终内核，你需要将这两个目标文件链接起来，生成最终的内核程序，这个程序才能被BootLoader使用。

在开发用户空间程序时，你的工具链会自带用于链接这类程序的默认脚本。但这些脚本并不适用于内核开发，因此你需要提供自己定制的链接脚本。请将以下内容保存到 linker.ld 文件中：

```asm
/*引导程序将查看该映像并从程序指定的进入点开始运行。*/

ENTRY(_start)
/*告知目标文件中的每个段最终被放置在内核镜像的什么位置*/
SECTIONS
{
	/*过去，人们普遍使用1MB作为起始偏移量，因为在BIOS系统下，这个位置实际上知识保证可用的。

	但在UEFI下，情况更为复杂， 实验表明2MB彩色更安全的加载位置，2016年，multiboot2的规范中引入了一项新功能

	该功能允许内核告知引导加载程序：内核可以被加载到某个地址范围内的任意位置，并且能够重定位自身，以便从加载程序所选择的地址运行。

	这样做是为了让加载程序能够自由选择一段经固件验证可用的内存区域，从而解决上述问题。

	由于本内核不使用该功能，因此选择了 2MB 作为起始位置，它相比传统的 1MB 是更安全的选项*/

	. = 2M

	/*首先放置multiboot头部，因为其必须放在镜像的开头位置，否则引导加载程序无法识别该文件格式。接下来我们放置.text段.*/

	.text BLOCK(4k) : ALIGN(4k)
	{
		*(.multiboot)
		*(.text)
	}

	//注：这是因为所有段均按照4kb对齐(block(4k)和align(4k))，符合内存管理的页对齐需求

	/*星号表示对所有文件启用，上述代码表示text段内放置text和multiboot的代码段。编译器（如 gcc）或汇编器（如 as）生成目标文件（.o 文件）时，
	会将代码、数据等按 “段”（Section）划分，并在目标文件中记录段的名称（如 .text、.rodata）、大小、内容等信息，存放在目标文件的 “段表”（Section Table）中。
	例如，boot.s 汇编后会生成包含 .text 段（汇编代码）和 .multiboot 段（multiboot 头部）的目标文件；
	kernel.c 编译后会生成包含 .text（C 代码）、.rodata（只读常量）、.data（已初始化变量）、.bss（未初始化变量）等段的目标文件。*/

	/*链接器（ld）在处理脚本的时候遇到*(.xxx)的语法则会：
	链接器（如 ld）处理链接脚本时，遇到 *(.xxx) 这样的语法，会：
	遍历所有输入的目标文件（如 boot.o、kernel.o）。
	读取每个目标文件的段表，查找名称为 .xxx 的段。
	将找到的所有同名段，按目标文件传入链接器的顺序（或脚本指定的优先级）合并，放置到脚本中定义的位置（如 .text 段区域）。
	*/ 

	/*例如：对于 *(.text)：链接器会从 boot.o 中找到 .text 段（汇编代码），从 kernel.o 中找到 .text 段（C 代码），然后将这两个段的内容合并，统一放在链接脚本中 .text 所定义的内存区域（2MB 起始位置附近，4KB 对齐）。*/


	/* 只读数据段 */
	.rodata BLOCK(4K) : ALIGN(4K)
	{
		*(.rodata)
	}

	/* 可读写数据段（初始化完毕） */
	.data BLOCK(4K) : ALIGN(4K)
	{
		*(.data)
	}

	/* 可读写段（未初始化）和栈 */
	.bss BLOCK(4K) : ALIGN(4K)
	{
		*(COMMON)
		*(.bss)
	}

	/* 编译器可能会生成其他额外的段（非.text、.rodata 等默认常见段），默认情况下，编译器会将这些额外段归类到与其名称对应的段中（比如生成一个叫.foo 的段，就会单独归为同名段）。如果需要让这些额外段按特定规则放入最终内核镜像，只需根据需求在这里添加对应的配置（比如像写 *(.foo) 这样的语句）即可。 */
}
```

注意：注意：有些教程建议使用 i686-elf-ld（链接器）而非编译器进行链接，但这样会阻止编译器在链接过程中执行各种必要操作。
现在，myos.bin 文件就是你的内核了（其他所有文件都不再需要）。需要注意的是，我们链接了 libgcc 库，它实现了交叉编译器所依赖的各种运行时例程。如果遗漏这个库，未来会出现问题。如果你在构建和安装交叉编译器时没有包含 libgcc，现在应该返回去重新构建一个包含 libgcc 的交叉编译器。编译器依赖这个库，无论你是否显式提供它，编译器都会使用它。

## 验证多重引导

若安装了GRUB,可以检查某个文件是否包含有效的 Multiboot 版本 1标头—— 你的内核就应该是这种情况。需要注意的是，Multiboot 头部必须位于实际程序文件的前 8KiB 范围内，且按 4 字节对齐。如果你在启动汇编代码、链接脚本或其他环节出现错误，这一点可能会被破坏。如果头部无效，当你尝试启动时，GRUB 会报错说找不到 Multiboot 头部。以下代码可以帮助你诊断这类问题：

```shell
grub-file --is-x86-multiboot myos.bin
```
grub-file 命令本身不会输出太多信息，但如果检查的是有效的 Multiboot 内核，它会以 0 退出（表示成功）；否则以 1 退出（表示失败）。你可以在执行该命令后立即在 shell 中输入 echo $? 来查看退出状态。你可以将这个 grub-file 检查添加到构建脚本中作为一项完整性测试，以便在编译时就能发现问题。
对于 Multiboot 版本 2，可以使用 --is-x86-multiboot2 选项进行检查。如果你在 shell 中手动执行 grub-file 命令，将其包装在条件语句中能更直观地查看状态。现在可以运行以下命令：

```shell
if grub-file --is-x86-multiboot myos.bin; then
  echo multiboot confirmed
else
  echo the file is not multiboot
fi
```

## 引导内核

再等个几分钟，就能看到内核运行了

### 构建可启动映像

你可以使用 grub-mkrescue 程序轻松创建一个包含 GRUB 引导加载程序和你的内核的可引导光盘（CD-ROM）镜像。在此之前，你可能需要安装 GRUB 工具程序以及 xorriso 程序（版本需为 0.5.6 或更高）。
首先，你需要创建一个名为 grub.cfg 的文件，并在其中写入以下内容：

```
menuentry "myos"{
	multiboot /boot/myos.bin 
}


#告诉您的grub如何加载你的内核，首先menuentry "myos"创建一个名字叫myos的启动菜单，当grub运行时会看到，如何multiboot /boot/myos.bin是说指定使用multiboot规范加载内核文件，文件在boot/myos.bin中。当用户选择myos的时候，grub就会按照这个配置加载你的myos.bin内核并启动他
```
需要注意的是，大括号({})的放置必须与此处所示的格式一致（需严格遵循语法要求，不能随意换行或遗漏）。
现在，你可以通过输入以下命令来创建操作系统的可引导镜像：

```shell
mkdir -p isodir/boot/grub
cp myos.bin isodir/boot/myos.bin
cp grub.cfg isodir/boot/grub/grub.cfg
grub-mkrescue -o myos.iso isodir
```

恭喜！你现在已经创建出了名为 myos.iso 的文件，这个文件中包含了你编写的 “Hello World” 操作系统。
如果你的电脑上还没有安装 grub-mkrescue 程序，现在就是安装 GRUB 的好时机 ——Linux 系统通常已经预装了 GRUB。而 Windows 用户，如果没有原生的 grub-mkrescue 程序可用，一般建议使用 Cygwin（一种在 Windows 上模拟类 Unix 环境的工具）来获取该功能。

**警告：** grub-mkrescue 所使用的引导程序 GNU GRUB 是在 GNU 通用公共许可证（GPL）下授权的。你的 iso 文件包含受该许可证保护的有版权材料，若违反 GPL 进行再分发，将构成版权侵权。
GPL 要求你公开与该引导程序对应的源代码。你需要从你的发行版中获取与调用 grub-mkrescue 时所安装的 GRUB 包完全对应的源代码包（因为发行版的包偶尔会更新）。然后，你需要将该源代码与你的 ISO 一同发布，以符合 GPL 要求。
或者，你可以自己从源代码构建 GRUB：将源代码解压到某个位置，从中构建 GRUB，并将其安装到 PATH 中的独立前缀目录下，确保使用其 grub-mkrescue 程序生成你的 iso。之后，你可以将官方的 GRUB 压缩包与你的操作系统发行版一同发布。
需要注意的是，你完全无需公开自己操作系统的源代码，只需公开 iso 中包含的引导程序的代码即可。

## 测试你的操作系统(Qemu)

虚拟机在操作系统开发中非常实用，因为它们能让你快速测试代码，并且在程序运行时还能查看源代码。否则，你就得陷入无休止的重启循环，这只会让你不胜其烦。虚拟机启动速度很快，尤其是搭配你正在开发的这种小型操作系统时。
在本教程中，我们将使用 QEMU。你也可以根据自己的喜好使用其他虚拟机，只需将 ISO 文件添加到空虚拟机的光驱中即可。
从你的软件仓库中安装 QEMU，然后使用以下命令来启动你的新操作系统。

```shell
qemu-system-i386 -cdrom myos.iso
```

这样应该会启动一个包含你的iso作为启动介质的虚拟机，若一切正常，你会看到引导程序提供的菜单。选择myos之后等待运行完毕，你应该就能看到你的快乐老家 "Hello,kernel World!" 跟着一些神秘字符

另一方面，qemu也支持直接启动多引导内核，无需任何可启动介质

```shell
qemu-system-i386 -kernel myos.bin
```


## 更进一步

恭喜，你已经能运行这款全新的操作系统了。当然，这是否只是一个开始，取决于你对它的兴趣程度。以下是一些帮助你开启后续使用的建议：

### 为终端添加换行支持

要处理终端中的换行符（\n），你需要在 terminal_putchar 函数中添加对换行符的判断逻辑。当检测到 \n 时，不需要向 VGA 缓冲区写入任何字符，而是直接调整终端的行（terminal_row）和列

```shell
void terminal_putchar(char c) 
{
	// 在当前光标位置输出字符

	if(c == '\n')
	{
		terminal_column = 0;
		terminal_row = ++;
	}
	terminal_putentryat(c, terminal_color, terminal_column, terminal_row);
	
	// 更新光标位置：列号+1；若达到最大列数（80），则换行
	if (++terminal_column == VGA_WIDTH) {
		terminal_column = 0;  // 列号重置为0
		if (++terminal_row == VGA_HEIGHT)  // 行号+1；若达到最大行数（25）
			terminal_row = 0;  // 行号重置为0（循环覆盖，无滚动）
	}
}
```

### 实现终端滚动

若终端内容已填满，它只会回到屏幕顶部（继续输出）。这在正常使用中是不可接受的。相反，终端应将所有行向上移动一行并丢弃最顶部的行，同时在底部保留一行空白行，以备输入字符。请实现此功能。

```c

static void terminal_scroll()
{
	//将n-1行的内容复制到n-2行上即可，其中n是最大的行数
	for(size_t y = 1; y< VGA_HEIGHT; y++)
	{
		for(size_t x = 0; x< VGA_WIDTH; x++)
		{
			size_t src_idx = y * VGA_WIDTH +x;
			size_t dest_idx = (y-1) * VGA_WIDTH +x;
			terminal_buffer[dest_idx] = terminal_buffer[src_idx]
		}
	}
}

```

然后只需要清空最底部的新行，即VGA_HEIGHT -1行即可。

```c
size_t last_row_idx = (VGA_HEIGHT -1) * VGA_WIDTH

for(size_t x = 0; x< VGA_WITDT;x++)
{
	terminal_buffer[last_row_idx+x] = vga_entry(' ', terminal_color);
}
```

这样就实现了一个清屏函数

### 调用全局函数

主要文章:[调用全局构造函数](https://github.com/MonsterGeo/my_os/blob/main/simple_kernel/more_and_more/Calling_Global_Constructors.md)


本教程展示了一个为 C 和 C++ 内核创建最小运行环境的简单示例。但遗憾的是，你的环境尚未完全配置妥当。例如，包含全局对象的 C++ 代码中，这些对象的构造函数不会被调用 —— 因为你从未执行过调用它们的操作。
编译器通过crt*.o系列目标文件，实现了一种在程序初始化阶段执行特定任务的特殊机制，即便对于 C 语言开发者而言，这种机制也可能很有价值。如果你能正确组合这些crt*.o文件，就会生成一个_init函数，该函数会触发所有程序初始化任务。之后，你的boot.o目标文件可以在调用kernel_main之前调用_init。
本节课到此结束