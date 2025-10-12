[toc]

# Calling Global Constructors

本文是Bare Bones一文的补充。我们将讨论如何正确调用全局构造函数(例如全局C++对象的构造函数)。这类构造函数本应在main函数执行前运行，这也是程序入口点通常叫做_start的函数的原因。该函数负责解析命令行参数、初始化标准库（包括内存分配、信号处理）、执行全局构造函数、最终调用exit(main(argc,argv))来启动main函数。

若更换编译器，不同操作系统上的实现可能会有所不同；但若使用的是GNU编译器套件、遵循System V ABI（应用程序二进制接口），通常是一个更稳妥的选择。

在大多数平台上，全局构造函数 / 析构函数会存储在一个经过排序的函数指针数组中，调用它们的操作很简单：遍历该数组并执行每个数组元素（即调用对应的函数）。然而，编译器并非总能让开发者直接访问这个列表，部分编译器甚至将其视为内部实现细节。这种情况下，开发者需要与编译器协同工作 —— 强行 “对抗” 编译器只会引发问题。

## GUN编译器合集-System V ABI

System V ABI(适用于 i686-elf-gcc,x86_64-elf-gcc及其他ELF平台)规定使用五个不同的目标文件协同处理程序初始化，这些目标文件命名为crt0.o，crti.o，crtbegin.o,crtend.o和crtn.o，这五个文件实现了两个特殊函数：

+ _init ：负责执行全局构造函数及其它的初始化任务
+ _fini：负责执行全局析构函数及其他终止任务。

这种机制让编译器能够牢牢掌控程序初始化流程，也为开发者简化了操作，但你必须与编译器协同工作，否则可能会出现严重问题。你的交叉编译器会提供crtbegin.o和crtend.o这两个文件 —— 它们包含编译器希望对开发者隐藏、但又需要开发者间接使用的内部实现细节。若要利用这些内部逻辑（完成程序初始化），你需要自行实现crti.o和crtn.o。幸运的是，这一实现过程并不复杂，本教程会对此进行详细说明。
第五个文件crt0.o包含程序的入口点（通常是_start函数），它会调用特殊的_init函数 —— 而_init函数所执行的 “程序初始化任务”，正是由crti.o、crtbegin.o、crtend.o和crtn.o这四个文件共同构建的。此外，你的退出函数（负责程序终止）通常也会调用由这四个文件共同生成的_fini函数。不过，crt0.o的相关内容不在本文讨论范围内。（注：在内核场景中，包含_start函数的目标文件会承担crt0.o的角色。）
要理解这种看似复杂的机制，我们可以结合一个具体场景：假设某程序由foo.o和bar.o两个目标文件组成，且正处于链接阶段（如下所述）。

```shell

i686-elf-gcc foo.o bar.o -o program

```
编译器将重写命令行并传递给链接器

```
i686-elf-ld crt0.o ctri.o crtbegin.o foo.o bar.o crtent.o crtn.o
```

其核心原理是：在链接阶段，这些目标文件会共同构建出_init和_fini函数。具体实现方式是将_init函数存储在.init段（代码段的一部分）中，将_fini函数存储在.fini段中 —— 每个目标文件会向这两个段分别贡献一段代码片段，链接器则会根据命令行中指定的文件顺序，将这些片段 “拼接” 成完整的函数。

+ crti.o提供函数的头部框架（如函数入口的汇编指令、栈初始化代码）；
+ crtbegin.o和crtend.o提供函数的核心主体（如全局构造 / 析构函数的调用逻辑）；
+ crtn.o提供函数的尾部收尾（如函数返回的汇编指令、栈平衡代码）。

### 使用C中的全局构造函数

作为一个特殊的拓展，GCC允许将C作为全局构造函数运行。更多的信息请参考编译器。常规用法如下：

```c
__attribute__ ((constructor)) void foo(void)
{
	printf("foo is running and printf is available at this point\n");
}

int main(int argc, char* argv[])
{
	printf("%s: main is running with argc=%i\n", argv[0], argc);
}

```

**注：** C语言标准是没有全局构造函数，这是C++的原生特性，用于全局对象初始化。而__attribute__((constructor))是GCC为C语言提供的非标准拓展，能够让被修饰的函数在main函数执行之前自动运行。本质上是提供了在启动阶段自动执行代码的能力。其中__attribute__(())是GCC的属性声明语法，用于给编译器传递特殊指令。而constructor是属性名字，这里是告知编译器该函数作为全局构造函数，要在main前运行。

### 在内核中使用crti.o, crtbegin.o, crtend.o, crtn,o

在内核开发场景中，你不会使用用户态 C 库（user-space C library），可能会使用专门的内核版 “C 库”，甚至完全不依赖任何 C 库。编译器通常会提供crtbegin.o和crtend.o，但crti.o和crtn.o在正常情况下由 C 库提供 —— 而内核场景不存在这种 “正常情况”，因此内核需要自行实现crti.o和crtn.o（即便其实现内容与用户态 C 库中的版本完全一致）。

内核链接时会使用-nostdlib选项（该选项等价于同时传递-nodefaultlibs和-nostartfiles），此选项会禁用 “启动文件”（即crt*.o系列文件）—— 这些文件在常规程序链接时会被自动添加到链接命令行中。通过传递-nostartfiles，我们相当于向编译器承诺：由开发者自行负责调用crtbegin.o和crtend.o中包含的 “程序初始化任务”。这意味着，我们需要在链接命令行中手动添加crti.o、crtbegin.o、crtend.o和crtn.o这四个文件。

由于crti.o和crtn.o是我们自行实现的，将它们添加到内核链接命令行中很简单；但crtbegin.o和crtend.o存储在编译器专属的目录下，因此需要先确定它们的具体路径。幸运的是，GCC 提供了专门用于查询该路径的选项。

假设你的交叉编译器是i686-elf-gcc，$CFLAGS是你通常传递给编译器的编译选项，那么执行以下命令即可查询crtbegin.o的路径：

```shell
i686-elf-gcc $CFLAGS -print-file-name=crtbegin.o
```

该命令会让编译器将 “与$CFLAGS选项 ABI 兼容的正确crtbegin.o文件路径” 打印到标即终端界面。此方法同样适用于crtend.o（查询其路径时，只需将命令中的crtbegin.o替换为crtend.o即可）。

如果你使用GNU Make(常用的构建工具)，只需要在Makefile中配置即可获取该路径， 假设\$(CC)代表你的交叉编译器（如前文的i686-elf-gcc），\$(CFLAGS)代表你通常传递给编译器的编译选项，具体配置方式如下

```shell
CRTBEGIN_OBJ:=$(shell $(CC) $(CFLAGS) -print-file-name=crtbegin.o)
CRTEND_OBJ:=$(shell $(CC) $(CFLAGS) -print-file-name=crtend.o)

```

如何你可以这样使用他们构建你的脚本：

```MakeFile
OBJS:=foo.o bar.o

CRTI_OBJ=crti.o
CRTBEGIN_OBJ:=$(shell $(CC) $(CFLAGS) -print-file-name=crtbegin.o)
CRTEND_OBJ:=$(shell $(CC) $(CFLAGS) -print-file-name=crtend.o)
CRTN_OBJ=crtn.o

OBJ_LINK_LIST:=$(CRTI_OBJ) $(CRTBEGIN_OBJ) $(OBJS) $(CRTEND_OBJ) $(CRTN_OBJ)
INTERNAL_OBJS:=$(CRTI_OBJ) $(OBJS) $(CRTN_OBJ)

myos.kernel: $(OBJ_LINK_LIST)
	$(CC) -o myos.kernel $(OBJ_LINK_LIST) -nostdlib -lgcc

clean:
	rm -f myos.kernel $(INTERNAL_OBJS)
```
务必记住：这些目标文件必须严格按照特定顺序链接，否则可能会出现难以排查的诡异 bug

完成链接后，你的内核中将包含_init和_fini函数，可在将控制权传递给kernel_main（或内核主函数的其他命名）之前，从boot.o（或内核入口点所在的目标文件）中调用这两个函数。需要注意的是，在调用这些函数的阶段，内核可能尚未完成任何初始化，因此全局构造函数中只能执行一些简单操作。此外，_fini函数可能永远不会被调用 —— 因为操作系统会持续运行，而当需要关机时，几乎所有清理工作都可通过处理器重置完成，无需额外操作。

///**注意:** 内核链接时，crti.o→crtbegin.o→内核目标文件→crtend.o→crtn.o的顺序是不可颠倒的，其底层原因是：这些文件向.init段和.fini段贡献的代码片段存在 “依赖关系”—— 例如crti.o的代码（函数头部）必须位于crtbegin.o的代码（主体逻辑）之前，否则_init函数会先执行主体逻辑（如调用全局构造函数），再执行栈初始化，导致栈访问错误（如读取未初始化的栈基址）。
若顺序错误，可能出现的 “诡异 bug” 包括：
全局构造函数未执行（因crtbegin.o的逻辑被跳过）；
程序计数器（PC）跳转到错误地址（因函数片段拼接混乱）；
栈溢出或数据损坏（因栈初始化与主体逻辑顺序颠倒）///


或许值得设计一个kernel_early_main函数，用于初始化堆、日志系统及其他核心内核功能。之后，boot.o可以先调用kernel_early_main，再调用_init，最后将控制权传递给真正的kernel_main。这与用户态程序的工作机制类似：在用户态中，crt0.o会先调用_initialize_c_library（或其他类似名称的函数），然后调用_init，最后通过exit(main(argc, argv))启动main函数。

### 在用户空间中使用crti.o, crtbegin.o, crtend.o, crtn.o

我们假设你已经有了这些文件，没有？没关系，看看这篇文章：

文章：[Creating a C Library](https://wiki.osdev.org/Creating_a_C_Library)


在用户态环境中使用这些目标文件（crt0.o、crti.o、crtbegin.o、crtend.o、crtn.o）非常简单，因为交叉编译机会自动按正确顺序将它们链接到最终程序中。与前文一致，编译器会提供crtbegin.o和crtend.o，而你的 C 库（如用户态的 GNU C 库glibc）会提供crt0.o（含程序入口点的文件）、crti.o和crtn.o。


如果你使用的是 “操作系统专用工具链”（OS Specific Toolchain），则可以通过修改STARTFILE_SPEC和ENDFILE_SPEC这两个配置项，自定义以下内容：

程序入口点的名称（默认是_start）；
编译器搜索crt{0,i,n}.o等文件的路径；
具体使用哪些文件（甚至可以用其他名称的文件替代默认的crt*.o）及它们的链接顺序。

因此，当你开始构建用户态环境时，创建一个 “操作系统专用工具链” 往往是很有价值的 —— 它能让你对上述所有流程（程序启动、目标文件链接）拥有高度控制权。

在 x86 架构下实现这一机制（构建_init和_fini函数）非常简单：你只需在crti.o中定义两个函数（_init和_fini）的头部框架，在crtn.o中定义它们的尾部收尾代码，然后将这两个目标文件用于你的 C 库或内核即可。之后，你就可以直接调用_init来执行初始化任务，调用_fini来执行终止（清理）任务 —— 这两个函数的调用通常来自crt0.o（用户态场景）或 “内核启动目标文件”（如my-kernel-boot-object.o，内核场景）。


#### x86架构的crti.s代码

```asm
.section .init          ; 声明当前代码属于.init段（存放_init函数代码）
.global _init           ; 声明_init为全局符号，供链接器识别
.type _init, @function  ; 标记_init为函数类型（符合ELF规范）
_init:
    push %ebp           ; 保存调用者的栈基址（%ebp）
    movl %esp, %ebp     ; 建立当前函数的栈基址（将栈顶指针%esp的值赋给%ebp）
    /* gcc会自动将crtbegin.o的.init段内容插入到这里 */

.section .fini          ; 声明当前代码属于.fini段（存放_fini函数代码）
.global _fini           ; 声明_fini为全局符号
.type _fini, @function  ; 标记_fini为函数类型
_fini:
    push %ebp           ; 保存调用者的栈基址
    movl %esp, %ebp     ; 建立当前函数的栈基址
    /* gcc会自动将crtbegin.o的.fini段内容插入到这里 */
```



#### x86架构的crtn.s代码

```asm
.section .init          ; 继续补充.init段内容
    /* gcc会自动将crtend.o的.init段内容插入到这里 */
    popl %ebp           ; 恢复调用者的栈基址（与crti.s中的push %ebp对应）
    ret                 ; 函数返回（跳回调用者）

.section .fini          ; 继续补充.fini段内容
    /* gcc会自动将crtend.o的.fini段内容插入到这里 */
    popl %ebp           ; 恢复调用者的栈基址
    ret                 ; 函数返回
```

///**注意：**

代码中的注释/* gcc会自动将xxx的内容插入到这里 */揭示了核心机制：链接器会按以下顺序拼接各文件的.init段和.fini段，形成完整函数

以_init函数为例（.init 段拼接顺序）：
crti.o的.init段：提供_init的头部（push %ebp → movl %esp, %ebp）；
crtbegin.o的.init段：插入全局构造函数的核心逻辑（如遍历全局构造函数列表并调用）；
crtend.o的.init段：插入全局构造的收尾逻辑（如清理临时数据）；
crtn.o的.init段：提供_init的尾部（popl %ebp → ret）。
拼接后的_init函数汇编逻辑如下（简化版）：

```asm
_init:
    # 1. crti.o提供的头部
    push %ebp
    movl %esp, %ebp

    # 2. crtbegin.o提供的核心逻辑（示例）
    call __do_global_ctors_aux  # 调用所有全局构造函数

    # 3. crtend.o提供的收尾逻辑（示例）
    movl $0, (%esp)
    call __cxa_atexit  # 注册全局析构函数（供_fini调用）

    # 4. crtn.o提供的尾部
    popl %ebp
    ret

```
///

### CTOR/DTOR

执行全局函数/析构函数的另一种方法是手动执行.ctors/./dtors符号（假设您有自己的ELF加载器）。 将每个ELF文件加载进内存后全部解析并重新定位，接着就可以使用.ctors/.dtors去执行全局构造/析构函数，为此您必须先找到.ctors/.dtors头

```c
for (i = 0; i < ef->ehdr->e_shnum; i++)
{
    char name[250];
    struct elf_shdr *shdr;

    // 获取第i个段的段头
    ret = elf_section_header(ef, i, &shdr);
    if (ret != ELF_SUCCESS)
        return ret;

    // 获取该段的名称
    ret = elf_section_name_string(ef, shdr, &name);
    if (ret != BFELF_SUCCESS)
        return ret;

    // 匹配".ctors"段并记录
    if (strcmp(name, ".ctors") == 0)
    {
        ef->ctors = shdr;
        continue;
    }

    // 匹配".dtors"段并记录
    if (strcmp(name, ".dtors") == 0)
    {
        ef->dtors = shdr;
        continue;
    }
}
```

**注意：**
在 ELF 文件中，.ctors（constructors）和.dtors（destructors）是两个特殊的数据段，用于存储全局构造函数和析构函数的指针列表，其作用与前文提到的.init/.fini段类似，但实现方式更直接

+ .ctors段：存放全局构造函数的函数指针数组，数组以0（空指针）结尾，确保遍历时有明确的终止条件；
+ .dtors段：存放全局析构函数的函数指针数组，同样以0结尾

编译器（如 GCC）会自动将被__attribute__((constructor))修饰的函数指针放入.ctors段，将被__attribute__((destructor))修饰的函数指针放入.dtors段。

既然已经获取了.ctors/.dtors段的段头，就可以通过以下方式解析每个构造函数（或析构函数）。需要注意的是，.ctors/.dtors是一个指针表（ELF32 架构下为 32 位指针，ELF64 架构下为 64 位指针），表中的每个指针都指向一个需要被执行的函数。

```c
// 定义函数指针类型：指向指向无参数、无返回值的构造函数
typedef void(*ctor_func)(void);

// 遍历.ctors段中的所有构造函数并执行
for(i = 0; i < ef->ctors->sh_size / sizeof(void *); i++)
{
    ctor_func func;  // 声明构造函数指针变量
    elf64_addr sym = 0;  // 存储ELF文件中记录的函数地址（文件内偏移）

    // 从ELF文件的.ctors段中读取第i个函数的地址（文件内偏移）
    sym = ((elf64_addr *)(ef->file + ef->ctors->sh_offset))[i];
    // 将文件内偏移转换为内存中的实际执行地址（已重定位）
    func = ef->exec + sym;
    // 调用构造函数
    func();

    /* 注释说明：
       ef->file 是存储ELF文件（可能是二进制可执行文件或共享库）的char*指针
       ef->exec 是存储ELF文件加载到内存中并完成重定位后的起始地址的char*指针
    */
}
```
如果你的.ctors/.dtors段中只有一个条目，不必感到惊讶。至少在 x86_64 架构下，GCC 编译器似乎会只添加一个条目，指向一组名为_GLOBAL__sub_I_XXX（构造相关）和_GLOBAL__sub_D_XXX（析构相关）的函数 —— 这些函数会调用_Z41__static_initialization_and_destruction_0ii，而后者才会实际为你调用每个构造函数 / 析构函数。此时，即使你添加更多全局定义的构造函数 / 析构函数，增长的也会是这个中间函数（_Z41__static_initialization_and_destruction_0ii），而非.ctors/.dtors段本身的条目数量。

### 稳定性问题

如果你不调用 GCC 提供的构造函数 / 析构函数，在 x86_64 架构下，GCC 生成的代码可能会在特定条件下发生段错误（segfault）。举个例子：

```c
class A
{
    public: 
        A() {}
};

A g_a;

void foo(void)
{
    A *p_a = &g_a;
    p_a->anything();     // <---- segfault
}
```

**注意：**全局对象g_a的实例存储在.data或.bss段中，但它的地址（&g_a）可能需要经过重定位才能正确映射到内存。
若未执行.ctors段中的初始化函数（如_GLOBAL__sub_I_g_a），g_a的地址可能仍为 “ELF 文件内的相对偏移” 而非 “内存中的绝对地址”，导致p_a成为野指针。
即使地址正确，某些编译器（尤其 C++ 环境）会为全局对象添加 “初始化标记”（如是否已构造的标志），未调用构造函数时该标记未设置，访问对象成员（p_a->anything()）可能触发内部校验失败。

看起来，GCC 所利用的构造函数 / 析构函数初始化程序，其功能远不止于简单调用每个全局定义类的构造函数 / 析构函数。执行.ctors/.dtors 中定义的函数，不仅会初始化所有的构造函数 / 析构函数，还能解决这类段错误（上述例子只是众多可解决问题中的一个）。据我观察，当存在全局定义的对象时，GCC 可能还会创建 “.data.rel.ro” 段 —— 这是 GCC 需要处理的另一个重定位表。该段被标记为 PROGBITS（程序数据），而非 REL/RELA（重定位表），这意味着 ELF 加载器不会为你执行这些重定位操作。相反，执行.ctors 中定义的函数会调用_Z41__static_initialization_and_destruction_0ii，而这个函数似乎会为我们完成这些重定位。更多信息请参考：[1](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=68738)