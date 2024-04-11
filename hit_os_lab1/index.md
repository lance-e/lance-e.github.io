# Hit_os_lab1


<!--more-->

# hit os lab1

### 1.实验内容

此次实验的基本内容是：

1. 阅读《Linux 内核完全注释》的第 6 章，对计算机和 Linux 0.11 的引导过程进行初步的了解；
2. 按照下面的要求改写 0.11 的引导程序 bootsect.s
3. 有兴趣同学可以做做进入保护模式前的设置程序 setup.s。

改写 `bootsect.s` 主要完成如下功能：

1. bootsect.s 能在屏幕上打印一段提示信息“XXX is booting...”，其中 XXX 是你给自己的操作系统起的名字，例如 LZJos、Sunix 等（可以上论坛上秀秀谁的 OS 名字最帅，也可以显示一个特色 logo，以表示自己操作系统的与众不同。）

改写 `setup.s` 主要完成如下功能：

1. bootsect.s 能完成 setup.s 的载入，并跳转到 setup.s 开始地址执行。而 setup.s 向屏幕输出一行"Now we are in SETUP"。
2. setup.s 能获取至少一个基本的硬件参数（如内存参数、显卡参数、硬盘参数等），将其存放在内存的特定地址，并输出到屏幕上。
3. setup.s 不再加载 Linux 内核，保持上述信息显示在屏幕上即可。

在实验报告中回答如下问题：

1. 有时，继承传统意味着别手蹩脚。x86 计算机为了向下兼容，导致启动过程比较复杂。请找出 x86 计算机启动过程中，被硬件强制，软件必须遵守的两个“多此一举”的步骤（多找几个也无妨），说说它们为什么多此一举，并设计更简洁的替代方案。

### 2.实验开始

##### 1.改写bootsect.s

使用到int 0x10中断进行屏幕打印,出入参数：


| 寄存器 | 参数                         |
| ------ | ---------------------------- |
| AL     | 显示模式                     |
| BH     | 视频页                       |
| BL     | 属性值（如果AL=0x00或0x01    |
| CX     | 字符串的长度                 |
| DH,DL  | 屏幕上显示起始位置的行、列值 |
| ES:BP  | 字符串的段:偏移地址          |

在屏幕上打印的源码：

 ~~~assembly
! Print some inane message

	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#24
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg1
	mov	ax,#0x1301		! write string, move cursor
	int	0x10
 ~~~

~~~assembly
msg1:
	.byte 13,10
	.ascii "Loading system ..."
	.byte 13,10,13,10

.org 508
root_dev:
	.word ROOT_DEV
boot_flag:
	.word 0xAA55

~~~

我们将源码修改并只保留**核心**部分

~~~assembly
entry _start
_start:

	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#26
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg1
	mov ax,#0x07c0
	mov es,ax					!
	mov	ax,#0x1301		! write string, move cursor
	int	0x10
my_loop:
	jmp	my_loop				! 死循环


msg1:
	.byte 13,10
	.ascii "LanceOS starting !!!"
	.byte 13,10,13,10

.org 510
boot_flag:
	.word 0xAA55

~~~

我们相对于源码进行修改的部分：

~~~assembly
	mov	cx,#26				!这里是因为我们修改了msg1中的ascii码数量，修改为了26
	mov ax,#0x07c0		！因为源码在这之前已经修改了es的值，所以我们去除无关代码后也要给es赋值，es是字符串的段地址
	mov es,ax					！
my_loop:
	jmp	my_loop				! 死循环

msg1:
	.byte 13,10
	.ascii "LanceOS starting !!!"			！修改为要修改的值
	.byte 13,10,13,10
.org 510						！这里是因为我们去除了无关的root_dev: .word ROOT_DEV,又要保证boot_flag:	.word 0xAA55在引导磁盘最后两个字节，所以508修改为510
~~~

打印成功

##### 2.改写setup.s

由于我们是重新开始写的bootsect.s，所以我们需要在新的bootsect.s中添加启动setup的代码

添加后的bootsect.s，成功加载setup

~~~assembly
SETUPLEN = 4				! nr of setup-sectors
SETUPSEG = 0x07e0			! 因为我们在新的bootsect中并没有将代码移动到0x90000，所以依旧还是在0x07c0,在0x07c0基础上移动521字节，所以改成0x07e0

entry _start
_start:

	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#26
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg1
	mov ax,#0x07c0
	mov es,ax
	mov	ax,#0x1301		! write string, move cursor
	int	0x10
load_setup:
	mov	dx,#0x0000		! drive 0, head 0
	mov	cx,#0x0002		! sector 2, track 0
	mov	bx,#0x0200		! address = 512, in INITSEG
	mov	ax,#0x0200+SETUPLEN	! service 2, nr of sectors
	int	0x13			! read it
	jnc	ok_load_setup		! ok - continue
	mov	dx,#0x0000
	mov	ax,#0x0000		! reset the diskette
	int	0x13
	j	load_setup

ok_load_setup:
	jmpi	0,SETUPSEG


msg1:
	.byte 13,10
	.ascii "LanceOS starting !!!"
	.byte 13,10,13,10

.org 510
boot_flag:
	.word 0xAA55

~~~

新的setup.s，成功打印"Now we are in SETUP"

~~~assembly
entry _start
_start:

	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#25
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg2
	mov ax,cs
	mov es,ax
	mov	ax,#0x1301		! write string, move cursor
	int	0x10
	
	
	
my_loop:
	jmp	my_loop				! 死循环


msg2:
	.byte 13,10
	.ascii "Now we are in SETUP"
	.byte 13,10,13,10

.org 510
boot_flag:
	.word 0xAA55
~~~

获取硬件参数，并打印到屏幕，这部分代码比较复杂，直接看实验给出的答案吧：

~~~assembly
INITSEG  = 0x9000
entry _start
_start:
! Print "NOW we are in SETUP"
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#25
    mov bx,#0x0007
    mov bp,#msg2
    mov ax,cs
    mov es,ax
    mov ax,#0x1301
    int 0x10

    mov ax,cs
    mov es,ax
! init ss:sp
    mov ax,#INITSEG
    mov ss,ax
    mov sp,#0xFF00

! Get Params
    mov ax,#INITSEG
    mov ds,ax
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov [0],dx
    mov ah,#0x88
    int 0x15
    mov [2],ax
    mov ax,#0x0000
    mov ds,ax
    lds si,[4*0x41]
    mov ax,#INITSEG
    mov es,ax
    mov di,#0x0004
    mov cx,#0x10
    rep
    movsb

! Be Ready to Print
    mov ax,cs
    mov es,ax
    mov ax,#INITSEG
    mov ds,ax

! Cursor Position
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#18
    mov bx,#0x0007
    mov bp,#msg_cursor
    mov ax,#0x1301
    int 0x10
    mov dx,[0]
    call    print_hex
! Memory Size
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#14
    mov bx,#0x0007
    mov bp,#msg_memory
    mov ax,#0x1301
    int 0x10
    mov dx,[2]
    call    print_hex
! Add KB
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#2
    mov bx,#0x0007
    mov bp,#msg_kb
    mov ax,#0x1301
    int 0x10
! Cyles
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#7
    mov bx,#0x0007
    mov bp,#msg_cyles
    mov ax,#0x1301
    int 0x10
    mov dx,[4]
    call    print_hex
! Heads
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#8
    mov bx,#0x0007
    mov bp,#msg_heads
    mov ax,#0x1301
    int 0x10
    mov dx,[6]
    call    print_hex
! Secotrs
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#10
    mov bx,#0x0007
    mov bp,#msg_sectors
    mov ax,#0x1301
    int 0x10
    mov dx,[12]
    call    print_hex

inf_loop:
    jmp inf_loop

print_hex:
    mov    cx,#4
print_digit:
    rol    dx,#4
    mov    ax,#0xe0f
    and    al,dl
    add    al,#0x30
    cmp    al,#0x3a
    jl     outp
    add    al,#0x07
outp:
    int    0x10
    loop   print_digit
    ret
print_nl:
    mov    ax,#0xe0d     ! CR
    int    0x10
    mov    al,#0xa     ! LF
    int    0x10
    ret

msg2:
    .byte 13,10
    .ascii "NOW we are in SETUP"
    .byte 13,10,13,10
msg_cursor:
    .byte 13,10
    .ascii "Cursor position:"
msg_memory:
    .byte 13,10
    .ascii "Memory Size:"
msg_cyles:
    .byte 13,10
    .ascii "Cyls:"
msg_heads:
    .byte 13,10
    .ascii "Heads:"
msg_sectors:
    .byte 13,10
    .ascii "Sectors:"
msg_kb:
    .ascii "KB"

.org 510
boot_flag:
    .word 0xAA55
~~~



##### 5.多此一举的操作

- bootsect代码先加载到0x7c00 ，然后又复制到0x90000处
- system 代码先加载到0x10000,然后又复制到内存的最开始0处


