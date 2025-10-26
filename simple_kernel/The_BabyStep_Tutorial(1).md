# BabyStep 1

## 您的第一个引导扇区

下面的代码是从磁盘加载第程序的例子

```asm
hang:
    jmp hang
    times 512-($-$$) db 0
```

CPU在首次启动的时候是以实模式启动的，而bios会在地址0x7c00处加载上面的代码，而times 512($-$$) db 0 表示用0填充满512个字节，这是NASM汇编语言的语法。而partcopy会期望扇区是这个大小，里面十六进制的200就是十进制的512。修改这个填充指令会让partcopy运行出错

引导扇区的末尾通常有一个引导签名(0xAA55)。旧版本的BIOS通过识别该签名来找出引导扇区。若我们在传统的BIOS或者QEMU上运行这段代码，则我们必须在代码的最后一行改为 dw 0xaa55

```asm
hang:
    jmp hang

    times 510-($-$$) db 0 ; 2 bytes less now
    db 0x55
    db 0xAA
```
你可以会注意到电机并没有被关闭，且无法使用Ctrl+alt+del进行关机。


而删除循环，只添加0会导致BIOS启动错误，提示没有找到操作系统

## 创建硬盘镜像


使用NASM编译代码之后，可用partcopy,dd,debug复制到你的磁盘、优盘，然后启动即可。


### windows
```cmd
nasmw boot.asm -f bin -o boot.bin
partcopy boot.bin 0 200 -f0
OR
debug boot.bin
-W 100 0 0 1
-Q
```

### unix

```shell
nasm boot.asm -f bin -o boot.bin
dd if=boot.bin of=/dev/fd0
```

写入u盘可以这样
```shell
nasm boot.asm -f bin -o boot.bin
dd if=boot.bin of=/dev/sda
```


## 在QEMU上运行

如果你没有用磁盘的旧机器，你可以用QEMU模拟一个

```shell

qemu-system-i386 -fda boot.bin

```

使用QEMU模拟命令发送 Ctrl-Alt-Del给虚拟机

```shell

sendkey ctrl-alt-delete

```

您可能需要将速度调到1%才能注意到重新启动了。

# BabyStep 2

## 使用BIOS写入消息

快速回顾：
+ Boot加载的引导扇区为512字节
+ 磁盘引导扇区的代码由BIOS在0x7c00的地方加载
+ 机器将从实模式启动
+ 除非发送CLI汇编指令，否则计算机将陷入中断

许多(但并非所有)BIOS中断要求DS寄存器中存放实模式段值。这也是许多BIOS中断在保护模式下无法工作的原因，若你想使用int 10h/ah =0eh中断来输出内容到屏幕，就需要确保用于指向待打印字符的 seg:offset是正确的、

实模式下，地址的计算为段*16+偏移量。由于偏移量远大于16，所以会出现多对段和偏移对应一个地址的情况。例如，引导程序的加载地址可以这样得到：0000:7c00 = 07c0:0000

你使用哪个地址都没问题，但若你使用ORG，则你需要注意发生了什么，默认来说，起始文件的偏移量是0，但如果你确实需要，你可以将它改成其他值使得可以工作。例如下面的代码使用07c0的地址工作。

```asm
; boot.asm
   mov ax, 0x07c0
   mov ds, ax

   mov si, msg
   cld
ch_loop:lodsb
   or al, al ; zero=end of str
   jz hang   ; get out
   mov ah, 0x0E
   mov bh, 0
   int 0x10
   jmp ch_loop

hang:
   jmp hang

msg   db 'Hello World', 13, 10, 0
   times 510-($-$$) db 0
   db 0x55
   db 0xAA

```

接下来是ORG版本的，这次，将使用段0访问，不过注意的是你还是需要告诉DS是什么，因为它可以初始化为任意值

```asm
[ORG 0x7c00]

   xor ax, ax ; make it zero
   mov ds, ax

   mov si, msg
   cld
ch_loop:lodsb
   or al, al  ; zero=end of string
   jz hang    ; get out
   mov ah, 0x0E
   mov bh, 0
   int 0x10
   jmp ch_loop

hang:
   jmp hang

msg   db 'Hello World', 13, 10, 0

   times 510-($-$$) db 0
   db 0x55
   db 0xAA
```

为了节省空间，程序通常使用CALL和RET将代码分开


```asm
[ORG 0x7c00]
   xor ax, ax  ;make it zero
   mov ds, ax
   cld

   mov si, msg
   call bios_print

hang:
   jmp hang

msg   db 'Hello World', 13, 10, 0

bios_print:
   lodsb
   or al, al  ;zero=end of str
   jz done    ;get out
   mov ah, 0x0E
   mov bh, 0
   int 0x10
   jmp bios_print
done:
   ret

   times 510-($-$$) db 0
   db 0x55
   db 0xAA
```


出于某种原因，也许先加载SI寄存器再跳转到子程序的做法让你不舒服，不过NASM的宏定义允许我们模拟传递参数的操作（但宏定义必须放在调用的代码之前）

```asm

%macro BiosPrint 1
                mov si, word %1
ch_loop:lodsb
   or al, al
   jz done
   mov ah, 0x0E
   int 0x10
   jmp ch_loop
done:
%endmacro

[ORG 0x7c00]
   xor ax, ax
   mov ds, ax
   cld

   BiosPrint msg

hang:
   jmp hang

msg   db 'Hello World', 13, 10, 0

   times 510-($-$$) db 0
   db 0x55
   db 0xAA

```

注解：%macro BiosPrint 1是定义一个名为BiosPrint的宏，并接收一个参数。(1是参数量)，用于简化打印逻辑，```mov si, word %1```表示将参数（字符串地址存入si寄存器，%1是第一个参数）。```ch_loop:lodsb```的lodsb是将DS:SI指向的内存地址读取一个字节到AL寄存器中，然后根据DF调整SI寄存器的值。，若DF=0，则SI自增，若DF = 1，则SI--;

如果你的代码变得冗长不可读，你可以考虑将其拆分成不同的文件，最后在主函数的开头包含这些代码，例如：

```asm
jmp main
%include "othercode.inc"

main:
    ;...其余代码的位置
```

接着你可以直接编译，将上面代码保存为文件print.asm

```shell
nasm print.asm -f bin print.bin
qemu-system-i386 -fda print.bin
```

# BabyStep 4

## 无需BIOS打印到屏幕



我知道这看起来越来越像一个半成品的汇编教程，但我这种看似杂乱的讲解其实是有原因的。具体来说，在切换到保护模式等状态之前，尽可能多地解决掉更多问题，会大幅减少后续的困惑。

这个示例会打印一个字符串，以及某个内存地址的内容（也就是视频内存中该字符串的第一个字符）。它的目的是演示如何在文本模式下不使用 BIOS 进行屏幕打印，同时展示如何转换十六进制数以便显示 —— 这样我们就能查看寄存器和内存的值了。
代码中包含了栈，但仅在call和ret指令时使用。

```nasm
;使用 nasmw boot.asm -f bin -o boot.bin编译

[ORG 0x7c00] ;将程序加载到7c00地址
xor ax,ax    ;清空ax寄存器
mov ds,ax    ;数据段寄存器ds = 0
mov ss,ax    ;栈段寄存器SS=0
mov sp,0x9c00;栈指针设置在0x9c00，原理起始代码

cld          ;清除方向标志位，使得lodsb指令自动递增SI

mov ax,0xb800; 视频文本模式内存
mov es,ax    ;设置段寄存器指向视频内存

mov si,msg   ;SI指向字符串msg
call sprint  ;调用字符串打印函数

mov ax,0xb800
mov gs,ax
mov bx,0x0000 ;起始位，对应第一个字符位置
mov ax,[gs:bx] ;读取视频内存0xb800:0000处的16位数据

mov word [reg16] ,ax ; 读取寄存器
call printreg16

hang:
    jmp hang

dochar: call cprint ;打印一个字符
sprint: lodsb
    cmp al,0
    jne dochar  ;等价于else{goto done;}
    add byte[ypos],1;若是结束符号，则ypos+1移到下一行
    add byte[xpos],0;列号，重置为0
    ret ; 返回

cprint: mov ah,0x0F ;字符属性为白色字符加黑色背景

   mov cx,ax ;将字符(AL)和属性(AH)存到CX
   movzx ax,byte [ypos] ;将行号拓展为16位存入ax。
   mov dx,160;存入160字节，因为每行是80列加一个字符2字节
   mul dx;把dx和ax乘一起
   movzx bx, byte [xpos]
   shl bx,1 ;BX = 列号*2,表示左移一位

   mov di,0    ;视频内存起始偏移
   add di,ax   ;加上行偏移量
   add di,bx   ;加上列偏移量最终得到写入地址
   mov ax,cx   ;恢复字符和属性到ax
   stosw       ;将ax写入Es:di指向的视频内存
   add byte[xpos],1 ;列加一
   ret

printreg16:
   mov di,outstr16 ;打印16位数值的十六进制形式.di指向要输出的字符串的地址
   mov ax,[reg16]  ;ax = 要转换的16位数值
   mov si,hexstr ;si指向十六进制字符表
   mov cx,4 ;4个内容
hex loop:
   rol ax, 4   ;左移4为
   mov bx, ax   ; b暂存ax的值
   and bx, 0x0f   ; 保留BX的最低4位
   mov bl, [si + bx];从十六进制表取得对应字符
   mov [di], bl ;将字符存入输出缓冲区 
   inc di ;缓冲区指针+1
   dec cx ;循环计数-1
   jnz hexloop ;未完成计数就下一次

   mov si, outstr16 ;si指向转换后的十六进制字符窜
   call sprint ;调用sprint打印该字符串

   ret  

xpos   db 0
ypos   db 0
hexstr   db '0123456789ABCDEF'
outstr16   db '0000', 0  ;待打印寄存器的值初始化
reg16   dw    0  ; pass values to printreg16
msg   db "What are you doing, Dave?", 0
times 510-($-$$) db 0
db 0x55
db 0xAA
```

# BabyStep 5

## 中断

现在我们演示如何通过替换IVT中的seg:offset来处理按键产生的中断。该地址通常是指向BIOS例程，若要在找到在IVT的条目，则需要将中断号乘4. 每个按键处理程序仅显示扫描码，不会转为ASCII码，或进行处理或对拓展按键做出响应。这样做的目的是为了不混淆核心逻辑，即以最简单的方式同时实现输入输出功能

我不会深入讲解读取按键相关端口的具体原理和原因。只需知道，你是在与实际的芯片（或芯片的部分组件）进行通信，而非某个软件中介。我个人认为，无论你处于哪个抽象层级工作，最终都是在向硬件下达指令，记住这一点很有必要。
需要说明的是，通过 0x61 端口开启或关闭键盘的代码已完整呈现，但根据不同系统，其中部分代码可能并非必需。

```nasm
; nasm boot.asm -f bin -o boot.bin
; partcopy boot.bin 0 200 -f0
[ORG 0x7c00]
   jmp start 
   %include "print.inc" ;引用print库文件
start: 
   xor ax,ax ;初始化
   mov ds,ax ;将段寄存器初始化为0
   mov ss,ax ;栈寄存器初始化为0
   mov sp,0x9c00 ;设置栈指针sp为0x0c00

   cli ;关闭中断

   mov bx,0x09 ;硬件中断
   shl bx,2    ;乘4（左移2位）
   xor ax,ax   ;ax清零
   mov gs,ax   ;段寄存器从0开始
   mov [gs:bx], word keyhandler ;把中断处理程序偏移地址写入中断向量表
   mov [gs:bx+2], ds;将中断处理程序的段地址(Ds = 0)写入中断向量表
   sti      ;开启中断
   jmp $    ;$表示当前指令地址，无限循环的意思

keyhandler: ;处理程序
   in al,0x60  ;从60端口读取键盘扫描码，存入AL寄存器
   mov bl,al   ;存到bl
   mov byte [port60],al ;扫描码存入内存变量port60中
   in al,0x61  ;从0x61端口读取键盘控制状态器，存入al

   mov ah,al   ;暂存ah
   or al,0x80  ;Al第7位置1（禁用键盘，通知控制器已接收扫描吗）
   out 0x61,al ;将修改的状态写回61
   xchg ah,al  ; 交换AH和al的值
   out 0x61,al ; 将原始状态写回61端口
   mov al,0x20 ;中断结束命令

   out 0x20,al ;发送到主中断控制器端口0x20告知中断处理完毕
   and bl,0x80 ;判断原始扫描码是否释放，释放时按第七位为1
   jnz done    ;非零则跳转到done，否则继续打印
   mov ax,[port60]
   mov word[reg16],ax
   call printreg16

done:
   iret  ; 恢复断点返回中断程序
port60 dw 0 ;定义变量port60
   times 510-($-$$) db 0
   dw 0xAA55
```

为了方便演示，我们将babystep4的打印函数移动到一个文件print.inc

```nasm

dochar:  call cprint ;打印一个字符

sprint:  lodsb 
   cmp al,0 ;al用于储存打印字符串
   jne dochar ;等价 else{goto done}
   add byte [ypos],1 ;ypos+1表示该行打印完成准备打印下一行
   mov byte [xpos],0 ;初始化到0列
   ret

cprint: mov ah,0x0F; 调整字体白色背景黑色
   mov cx,ax; 保存ax
   movzx ax,byte [ypos];拓展指令，用0填充高位
   mov dx,160  ;存入160个字节，因为1个字节2byte，一行80个
   mul dx   ;计算当前行在视频内存中偏移量 mul dx = ax * dx，而ax是行号
   movzx bx,byte [xpos]
   shl bx,1 ;列号*2，因为一个字占2byte，*2获得正确的位置

   mov di,0 ;初始化偏移量
   add di,ax ; 将行的偏移量传进去
   add di,bx ; 传入列的偏移量即可得到当前字符的位置

   mov ax,cx   ;恢复之前保存的ax
   stosw ;将ax中的数据写入es:di指向的内存单元，因为之后ax用来储存行号，一开始的字符属性需要恢复写入才能显示。这样是写入字符属性
   add byte [xpos],1 ;准备下一个字符打印
   ret

printreg16:
   mov di, outstr 16; 传入outstr16地址
   mov ax,[reg16] ;把reg16传入
   mov si,hexstr ;指向十六进制字符表
   mov cx,4 ;循环计数器，16位数据是4byte，一次处理一字节，所以是4
hexloop:
   rol ax,4 ;左移ax4位，即把最高4位移到低4位
   mov bx,ax   ;提取低4位
   and bx,0x0f ;与运算保留bx的低4位，0x0f=00001111
   mov bl,[si+bx] ;以bx为索引，从hexstr取出对应字符存入bl
   mov [di],bi ;取出的字符存入输出缓冲区outstr16的位置
   inc dl;自增1，指向缓冲区下一个位置
   dec cx;循环计数器-1
   jnz hexloop ;若cx不为0，则继续循环

   mov si,outstr16
   call sprint
   ret

xpos db 0;初始化为0
ypos db 0
hexstr db '0123456789ABCDF'
outstr16 db '0000',0
reg16 dw 0;初始化
```

