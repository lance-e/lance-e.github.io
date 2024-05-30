# 手写OS系列之实现打印函数


<!--more-->

# 手写OS系列之实现打印函数

### 1.实现单字符打印

##### 处理流程：

- 备份寄存器现场
- 获取光标坐标值
- 获取待打印的字符
- 判断字符是否为控制字符
- 判断是否需要滚屏
- 更新光标位置
- 恢复寄存器，推出

##### 获取当前光标位置

通过显卡提供的cursor location high register和cursor location low regitster可以获得光标的高低8位。CRT controller registers的读写端口受到i/oas字段影响，但是i/oas默认为1，所以读写端口默认为3d4h和3d5h，获取步骤：

- 先往0x3d4的address register中写入寄存器索引
- 再从0x3d5的data register中读写数据

~~~assembly
;----------get the cursor position---------------------
	;get high 8 bit
	mov	dx,0x03d4				;operate the address register
	mov	al,0x0e
	out	dx,al
	mov	dx,0x03d5				;operate the data register
	in	al,dx	
	mov	ah,al

	;get low 8 bit
	mov	dx,0x03d4	
	mov	al,0x0f
	out	dx,al
	mov	dx,0x03d5
	in	al,dx

~~~



##### 实现滚屏操作：

​		80*25模式下屏幕可以显示2000个字符，一个字符占两个字节，高字节存放字符属性，低字节存放字符ascii码，所以一个屏幕可显示4000个字节内容。显存大小为32kb，所以共可显示约8个屏幕字符。显卡就为我们提供了两个设置屏幕显示字符起始位置的寄存器，CRT controller data registers索引为0ch和0dh的start address high register和start address low register，他们分别设置起始字符地址的高8位和低8位，屏幕自动向后显示2000字符。



​		但是这里为了简单采用另外一种方法：固定屏幕显示的起始字符始终为0，仅让屏幕显示最初的2000字符，当实现滚屏时，将1~24行内容移到0~23行，将24行全部设置为空格，将光标改成24行行首。虽然这种方法很有局限性，但是简单。

~~~assembly
.roll_screen:

        ;mov 1~24 to 0~23
        cld
        mov     ecx,960                                 ;960:   there are 2000-80 = 1920 char need to move , 1920 X 2 = 3820 byte, move 4 byte every time .so 3820 / 4 = 960

        mov     esi,0xc00b80a0                          ; second line
        mov     edi,0xc00b8000                          ; first line
        rep     movsd

        ;set the last line to space
        mov     ebx,3840
        mov     ecx,80
.cls:
        mov     word    [gs:ebx],0x0720                 ;0x0720 is 2 byte contain ascii and attribute
        add     ebx,2
        loop    .cls
        mov     bx,1920                                 ;make the cursor at the head of last line

~~~



##### 完整代码：

print.s:

~~~assembly
TI_GDT	equ	0
RPL0	equ	0
SELECTOR_VIDEO	equ	(0x0003 << 3) + TI_GDT + RPL0

[bits 32]
section	.text
;-----------------------------put char---------------------

global	put_char
put_char:
	pushad						;copy the register environment; there are 8 register
	mov	ax,SELECTOR_VIDEO
	mov	gs,ax

;----------get the cursor position---------------------
	;get high 8 bit
	mov	dx,0x03d4				;operate the address register
	mov	al,0x0e
	out	dx,al
	mov	dx,0x03d5				;operate the data register
	in	al,dx	
	mov	ah,al

	;get low 8 bit
	mov	dx,0x03d4	
	mov	al,0x0f
	out	dx,al
	mov	dx,0x03d5
	in	al,dx
	
;------------get char-----------------------------------
	mov	bx,ax					;save the cursor into bx
	mov	ecx,[esp+36]				;get the char from stack,36 = 4 * 8
	
;-------------judge control char-----------------------
	cmp	cl,0xd
	jz	.is_carriage_return 
	cmp	cl,0xa
	jz	.is_line_feed
	cmp	cl,0x8
	jz	.is_backspace
	jmp	.put_other
	
	
.is_backspace:
	dec	bx
	shl	bx,1					;move lefe 1 bit == X2 , now bx is the offset of cursor in graphics memory 

	mov	byte	[gs:bx],0x20			;0x20 is the space's ascii
	inc	bx
	mov	byte	[gs:bx],0x07			;0x07 is the attribute of char
	shr	bx,1
	jmp	.set_cursor

.put_other:
	shl 	bx,1					;now bx is the offset of cursor in graphics memory	
	mov	[gs:bx],cl				;
	inc	bx	
	mov	byte 	[gs:bx],0x07			;the attribute of char
	shr	bx,1					;recover the value of cursor
	inc	bx					;next cursor
	cmp	bx,2000
	jl	.set_cursor				;

.is_line_feed:
.is_carriage_return:
	xor	dx,dx					;dx is the high 16 bit
	mov	ax,bx					;ax is the low 16 bit
	mov	si,80
	
	div	si
	sub	bx,dx					;rounding 
	
.is_carriage_return_end:
	add	bx,80
	cmp	bx,2000
.is_line_feed_end:
	jl	.set_cursor


.roll_screen:

	;mov 1~24 to 0~23
	cld	
	mov	ecx,960					;960:	there are 2000-80 = 1920 char need to move , 1920 X 2 = 3820 byte, move 4 byte every time .so 3820 / 4 = 960
	
	mov	esi,0xc00b80a0				; second line
	mov	edi,0xc00b8000				; first line
	rep	movsd

	;set the last line to space
	mov	ebx,3840
	mov	ecx,80
.cls:
	mov	word	[gs:ebx],0x0720			;0x0720 is 2 byte contain ascii and attribute
	add	ebx,2
	loop	.cls
	mov	bx,1920					;make the cursor at the head of last line

.set_cursor:
	;set high 8 bit
	mov	dx,0x03d4
	mov	al,0x0e
	out	dx,al
	mov	dx,0x03d5
	mov	al,bh
	out	dx,al
	;set low 8 bit
	mov	dx,0x03d4
	mov	al,0x0f
	out	dx,al
	mov	dx,0x03d5
	mov	al,bl
	out	dx,al

.put_char_done:
	popad
	ret
	
~~~

print.h:

~~~c
#ifndef __LIB_KERNEL_PRINT_H
#define __LIB_KERNEL_PRINT_H
#include "stdint.h"
void put_char(uint8_t char_ascii);
#endif
~~~

stdint.h

~~~c
#ifndef	__LIB_STDINT_H
#define	__LIB_STDINT_H
typedef	signed	char	int8_t;
typedef	signed	short	int	int16_t;
typedef	signed	int	int32_t;
typedef	signed	long	long	int	int64_t;
typedef	unsigned	char	uint8_t;
typedef unsigned	short	int	uint16_t;
typedef	unsigned	int	uint32_t;
typedef	unsigned	long	long 	int	uint64_t;
#endif;
~~~

### 2.打印字符串

就是对put_char的简单封装：

~~~assembly
;------------------------put_str----------------------------------
global	put_str
put_str:
	push	ebx
	push	ecx
	xor	ecx,ecx
	mov	ebx,[esp+12]				;get the position of string ,stack had the value of ebx and ecx (8 byte) and the address of function to return (4 byte)	
		
.goon:
	mov	cl,[ebx]
	cmp	cl,0					;0 is '\0'
	je	.put_over
	push	ecx					;push the param of put_char 
	call	put_char				;
	add	esp,4
	inc	ebx
	jmp	.goon

.put_over:
	pop	ecx
	pop	ebx
	ret
~~~

### 3.打印整数

实现逻辑：将32位整数的每四位二进制数转换成一个16进制数，共转换成8个数。所以定义了8字节的缓冲区，用于存储转换后的字符。每一次操作一个16进制数，从低位到高位处理，也就是将数字and与运算保留最后四位，每一次操作完再将操作的数字右移4位，覆盖掉上一次操作的数字。将数字转换为字符，就是该数字到初始数字的偏移量加上初始数字的ascii码(例：3转换过程：3-0 得偏移为3，再加上‘0’ascii码48，最终得51)，这里需要分两种情况，0~9和A~F。最后打印也是对put_char的封装，因为打印的顺序是从数字对应的高位开始打印的,所以要判断是否为0字符，是就跳到下一个字符，不是就put_char。

~~~assembly
;-------------------------put_int-----------------------------
global	put_int
put_int:
	pushad
	mov	ebp,esp
	mov	eax,[ebp+4*9]				;the address of return and value of 8 register
	mov	edx,eax
	mov	edi,7					;the offset in buffer
	mov	ecx,8
	mov	ebx,put_int_buffer
	
.based16_4bits:
	
	and	edx,0x0000000F
	
	cmp	edx,9
	jg	.is_A2F					; >
	add	edx,'0'
	jmp	.store

.is_A2F:
	sub	edx,10					;sub 10: is the offset to A
	add	edx,'A'					;get the ascii
	
.store:
	mov	[ebx+edi],dl				;ebx is the base address of buffer,edi is the offset of buffer,dl is the ascii of char
	dec	edi

	shr	eax,4					;deal next number
	mov	edx,eax
	loop	.based16_4bits
	
.ready_to_print:
	inc	edi
.skip_prefix_0:
	cmp	di,8
	je	.full0

.go_on_skip:
	mov	cl,[put_int_buffer+edi]
	inc	edi					;next char
	cmp	cl,'0'
	je	.skip_prefix_0
	dec	edi					;due to the char isn't '0' , so resume the edi that point to the char and print
	jmp	.put_each_num

.full0:
	mov	cl,'0'	
.put_each_num:
	push	ecx
	call	put_char
	add	esp,4					;clear the param stack in put_char
	inc	edi					;next char
	mov	cl,[put_int_buffer+edi]
	cmp	edi,8
	jl	.put_each_num
	popad
	ret
	
~~~

### 4.总结

print函数最主要的还是put_char的实现，其他的打印都是基于它进行的封装，我们的重心是放在写os，所以print函数稍显简陋。

