# 手写OS系列之中断


<!--more-->

# 手写OS系列之中断

### 1.中断相关概念

##### 1.什么是中断捏？

cpu获知到计算机中发生的某些事，暂停正在执行的程序，转而去执行**处理该事件的程序**，执行完毕后，继续执行刚才的程序，整个过程就叫做中断处理，也就是中断。

中断机制本质就是接收到一个中断信号后，调用相应的中断处理程序。一个中断信号就对应一个整数，作为中断信号的id，这个数字就是**中断向量** ，用于在中断描述符表（IDT）中索引，进而可以找到对应的中断处理程序。

##### 2.中断分类

- 外部中断：
  - 可屏蔽中断：通过INTR信号线进入cpu(外部设备)
  - 不可屏蔽中断：通过NMI信号线进入cpu(灾难性错误)
- 内部中断：
  - 软中断：由软件主动发出的中断，分类：
    - int	8位立即数
    - int3：断点调试的原理
    - into：中断溢出指令
    - bound：检查数组索引越界指令
    - ud2：未定义指令
  - 异常：有些异常会产生单独的错误码(error code)，在进入中断，cpu会压入栈中。分类
    - Fault：故障，最轻的异常
    - Trap：陷阱，例如int3指令引起的异常
    - Abort：终止，最终的异常

##### 3.中断描述符表

IDT是保护模式下用于存储中断处理程序**入口** 的表，每个表项都是终端描述符，也叫做**中断门**，大小为8byte，也就是64位

IDT结构：

| name                                              | bit             |
| ------------------------------------------------- | --------------- |
| 中断处理程序在目标代码段的偏移量的0~15位(低16位)  | 低32位的0~15位  |
| 中断处理程序目标代码段的选择子(16位)              | 低32位的16~31位 |
| 未使用(4位)                                       | 高32位的0~4位   |
| 默认为0(3位)                                      | 高32位的5~7位   |
| TYPE(IDT的这四位默认为011D)                       | 高32位的8~11位  |
| S字段(为0，代表是系统段)                          | 高32位的12位    |
| DPL(中断门的描述符特权级，将用作后续的特权级检查) | 高32位的13~14位 |
| P字段(表示段是否存在)                             | 高32位的15位    |
| 中断描述符在目标代码段的偏移量的16~31位(高32位)   | 高32位的16~31位 |

cpu内部有个中断描述符表寄存器(IDTR)，0~15位为表界限，16~47位为IDT基地址。

加载IDTR指令：

- lidt	48位内存数据

##### 4.中断处理流程

- 通过中断向量索引到idt中对应中断描述符。
- 进行特权级检查：目标代码段DPL>=CPL>=中断门DPL,数值上目标代码段DPL<=CPL<=中断门DPL。
- 执行中断处理程序：将目标代码段选择子加载到CS中，将32位的目标代码段偏移地址加载到EIP中，开始执行处理程序。

处理器提供了专门控制IF位的的指令：cli使IF为0，关中断，sti使IF为1，开中断。

*IF位只能限制外部设备的中断(可屏蔽中断)*

##### 5.中断发生时的压栈情况

###### 1.特权级变化时，CPL<目标代码段DPL：

- 压入旧的ss，低16位，高16位0补齐
- 压入旧的esp
- 压入EFLAGS
- 压入旧的cs，低16位，高16位0补齐
- 压入旧的eip
- 压入错误码ERROR_CODE

###### 2.无特权级变化，CPL=目标代码段DPL：

- 压入EFLAGS
- 压入旧的cs，低16位，高16位0补齐
- 压入旧的eip
- 压入错误码ERROR_CODE

**当栈中压入了错误码，cpu不会主动跳过，需要我们手动跳过**

### 2.可编程中断控制器8259A

8259A就是中断代理，用于管理和控制可屏蔽中断(外设中断)，决定哪个中断优先被cpu执行。

##### 1.结构

通过级联的方式，将多片串连到一起，分为一片主片master和其余的从片slave，从片的中断传递给主片，再由主片传递给cpu。



每个独立运行的外部设备都是一个中断源，它们发出的中断只有在IRQ(Interrupt ReQuest)信号线上才能被cpu知晓。



具体结构不赘述

##### 2.编程

内部分为两组寄存器：1.初始化命令寄存器组，用来保存初始化命令字ICW(initialization command word)，共四个ICW1~ICW4，2.操作命令寄存器组，用来保存操作命令字(operation command word)，共四个OCW1~OCW4。所以我们的编程分为初始化和操作：

- 用ICW做初始化：是否需要级联，设置起始中断向量号，设置中断结束模式
- 用OCW来操作8259A

具体结构不赘述

操作端口：

- ICW1，OCW2，OCW3是偶地址0x20(master),0xA0(slave)
- ICW2,ICW3,ICW4,OCW1是奇地址0x21(master),0xA1(slave)

### 3.编写中断处理程序

##### kernel/kernel.S:

interrupt.c中定义的idt_table才是中断处理函数的数组，保存了函数地址。intrXXentry通过call 数组的元素，来调用中断处理函数。

**定义了一个全局数组intr_entry_table,因为后面定义了数据段(section .data)，所以此数组元素为每个中断处理程序的地址**

~~~assembly
[bits 32]

%define	ERROR_CODE	nop				;if the cpu automic to push the error code so that don't do anything

%define	ZERO	push	0				;if the cpu wasn't automic push, so push 0 by ourself

extern idt_table					;in c file

section	.data
global	intr_entry_table
intr_entry_table:

%macro	VECTOR	2
section	.text
intr%1entry:						;%1 is the first param

	%2						;%2 is the second param
	;save the environment
	push 	ds
	push 	es
	push	fs
	push	gs
	pushad	

	mov	al,0x20					;interrupt end command:EOI
	out	0xa0,al
	out	0x20,al

	push	%1					;push the interrupt vector number
	
	call	[idt_table+ %1 * 4 ]			;call the interrupt function in c file
	jmp 	intr_exit
	


section	.data
	dd	intr%1entry				;in order to make the all element of intr_entry_table is the address of  every handle function
%endmacro

section .text
global intr_exit
intr_exit:
	add	esp,4					;skip the param(interrupt vector number)
	popad
	pop	gs
	pop	fs
	pop	es
	pop	ds
	add	esp,4					;skip the error code
	iretd


VECTOR	0x00,ZERO
VECTOR	0x01,ZERO
VECTOR	0x02,ZERO
VECTOR	0x03,ZERO
VECTOR	0x04,ZERO
VECTOR	0x05,ZERO
VECTOR	0x06,ZERO
VECTOR	0x07,ZERO
VECTOR	0x08,ERROR_CODE
VECTOR	0x09,ZERO
VECTOR	0x0a,ERROR_CODE
VECTOR	0x0b,ERROR_CODE
VECTOR	0x0c,ERROR_CODE
VECTOR	0x0d,ERROR_CODE
VECTOR	0x0e,ERROR_CODE
VECTOR	0x0f,ZERO
VECTOR	0x10,ZERO
VECTOR	0x11,ERROR_CODE
VECTOR	0x12,ZERO
VECTOR	0x13,ZERO
VECTOR	0x14,ZERO
VECTOR	0x15,ZERO
VECTOR	0x16,ZERO
VECTOR	0x17,ZERO
VECTOR	0x18,ZERO
VECTOR	0x19,ZERO
VECTOR	0x1a,ZERO
VECTOR	0x1b,ZERO
VECTOR	0x1c,ZERO
VECTOR	0x1d,ZERO
VECTOR	0x1e,ERROR_CODE
VECTOR	0x1f,ZERO
VECTOR	0x20,ZERO
~~~

##### kernel/interrupt.c：

用于创建IDT，安装中断程序

先初始化idt，一般中断处理函数注册和异常函数名称注册，然后初始化8259A，最后加载IDT

注意这里我们初始化8259A完成后，仅开启了时钟中断，屏蔽了其他中断。

~~~c
#include   	"interrupt.h"
#include	"stdint.h"
#include	"global.h"
#include	"io.h"


#define PIC_M_CTRL 0x20  			//master chip contrl port
#define PIC_M_DATA 0x21				//master chip data port
#define PIC_S_CTRL 0xa0				//slave chip contrl port 
#define PIC_S_DATA 0xa1				//slave chip data port



//the number of interrupt
#define	IDT_DESC_CNT	0x21				


// struct of interrupt gate describtor
struct 	gate_desc{
	uint16_t	func_offset_low_word;
	uint16_t	selector;
	uint8_t		dcount;
	uint8_t		attribute;
	uint16_t	func_offset_high_word;
};

static	void make_idt_desc(struct gate_desc* p_gdesc,uint8_t attr,intr_handler function);
//interrupt descriptor table
static	struct 	gate_desc	idt[IDT_DESC_CNT];

//save the name 
char* intr_name[IDT_DESC_CNT];

//save the interrupt function
intr_handler idt_table[IDT_DESC_CNT];

extern intr_handler intr_entry_table[IDT_DESC_CNT];

static void pic_init(void){
	//initial main chip
	outb(PIC_M_CTRL , 0x11);			//ICW1
	outb(PIC_M_DATA , 0x20);			//ICW2
	outb(PIC_M_DATA , 0x04);			//ICW3
	outb(PIC_M_DATA , 0x01);			//ICW4

	//initial slave chip
	outb(PIC_S_CTRL , 0x11);			//ICW1
	outb(PIC_S_DATA , 0x28);			//ICW2
	outb(PIC_S_DATA , 0x02);			//ICW3
	outb(PIC_S_DATA , 0x01);			//ICW4

	//open the IR0 in the main chip , mean that just accept the  clock interrupt
	outb(PIC_M_DATA , 0xfe);
	outb(PIC_S_DATA , 0xff);

	put_str(" pic_init done\n");
}


static	void 	make_idt_desc(struct gate_desc* p_gdesc,uint8_t	attr,intr_handler function){
	p_gdesc->func_offset_low_word = (uint32_t)function & 0x0000FFFF;
	p_gdesc->selector = SELECTOR_K_CODE;
	p_gdesc->dcount = 0 ;
	p_gdesc->attribute = attr;
	p_gdesc->func_offset_high_word=((uint32_t)function & 0xFFFF0000)>> 16;
}

static	void	idt_desc_init(void){
	int 	i;
	for (i = 0 ; i < IDT_DESC_CNT;i++){
		make_idt_desc(&idt[i],IDT_DESC_ATTR_DPL0,intr_entry_table[i]);
	}
	put_str(" idt_desc_init	done\n");
}

// general  interrupt function 
static void general_intr_handler(uint8_t vec_nr){
	//IRQ7 and IRQ15 will make spurious interrupt ,don't need handle
	//0x2f is the slave chip 's last IRQ ,is reserved items
	if (vec_nr == 0x27 || vec_nr == 0x2f){
		return ;
	}
	put_str("int vector : 0x");
	put_int(vec_nr);
	put_char('\n');
}	


static void exception_init(void){
	int i ;
	for (i =  0 ; i < IDT_DESC_CNT; i++){
		idt_table[i] = general_intr_handler; 			//here set the interrupt function is the default general_intr_handler
		intr_name[i] = "unknown";
	}
	//initial exception name
	intr_name[0] = "#DE Divide Error";
	intr_name[1] = "#DB Debug Exception";
	intr_name[2] = "NMI Interrupt";
	intr_name[3] = "#BP Breakpoint Exception";
	intr_name[4] = "#OF overflow Exception";
	intr_name[5] = "#BR BOUND Range Exceeded Exception";
	intr_name[6] = "#UD Invalid Opcode Exception";
	intr_name[7] = "#NM Device Not Availible Exception";
	intr_name[8] = "#DF Double Fault Exception";
	intr_name[9] = "Coprocessor Segment Overrun";
	intr_name[10] = "#TS Invalid TSS Exception ";
	intr_name[11] = "#NP Segment Not Present";
	intr_name[12] = "#SS Stack Fault Exception";
	intr_name[13] = "#GP General Protection Exception";
	intr_name[14] = "#PF Page-Fault Exception";
	//intr_name[15] ; 15 is the reserved items,not used
	intr_name[16] = "#MF x87 FPU Floationg-Point Error";
	intr_name[17] = "#AC Alignment Check Exception";
	intr_name[18] = "#MC Machine-Check Exception";
	intr_name[19] = "#XF SIMD Floating-Point Exception";
}

void	idt_init(){
	put_str("idt_init start\n");
	idt_desc_init();		//initial interrupt descriptor table	
	exception_init();
	pic_init();			//initial 8259A

	
	//load idt
	uint64_t idt_operand = ((sizeof(idt)-1) | ((uint64_t)(uint32_t)idt << 16 ));
	asm volatile("lidt %0": : "m"(idt_operand));
	put_str("idt_init done\n");
}
~~~

##### kernel/interrupt.h：

~~~c
#ifndef __KERNEL_INTERRUPT_H
#define __KERNEL_INTERRUPT_H
typedef void* intr_handler;
#endif
~~~

##### kernel/global.h:

~~~c
#ifndef	__KERNEL_GLOBAL_H
#define	__KERNEL_GLOBAL_H
#include "stdint.h"

#define	RPL0	0
#define RPL1	1
#define RPL2	2
#define RPL3	3

#define TI_GDT 0
#define TI_LDT 1

#define	SELECTOR_K_CODE	((1<<3) + (TI_GDT << 2 ) + RPL0)
#define	SELECTOR_K_DATA ((2<<3)+(TI_GDT << 2 ) + RPL0)
#define	SELECTOR_k_STACK  SELECTOR_K_DATA
#define SELECTOR_K_GS ((3<<3) + (TI_GDT << 2) + RPL0)

//-------------------------------the attribute of IDT descriptor-----------
#define	IDT_DESC_P	1
#define IDT_DESC_DPL0	0
#define	IDT_DESC_DPL3	3
#define	IDT_DESC_32_TYPE	0xE			//32 bit
#define IDT_DESC_16_TYPE	0x6			//16 bit , won't use ,just for compare
#define IDT_DESC_ATTR_DPL0	((IDT_DESC_P << 7 ) + (IDT_DESC_DPL0 << 5 ) + IDT_DESC_32_TYPE)
#define IDT_DESC_ATTR_DPL3	((IDT_DESC_P << 7 ) + (IDT_DESC_DPL3 << 5 ) + IDT_DESC_32_TYPE)


#endif
~~~

##### lib/kernel/io.h:

使用内联汇编实现端口操作，这里用到了c语法static inline：建议cpu将函数编译成内嵌形式，在函数调用处原封不动展开，编译后的代码中不再包含call等指令，减少了函数调用和返回时的现场保护和恢复过程，提高了效率。

~~~c
/*-------------------------mathine_mode--------------------
	b --(QImode)low 8bit of register:[a-d]l
	w --(HImode) 2 bit of register :[a-d]x
-------------------------------------------------------------*/

#ifndef	__LIB_IO_H
#define	__LIB_IO_H
#include "stdint.h"

//write a byte into port
static inline void outb(uint16_t port,uint8_t data){
	//N is represent 0~255, d represent dx,%b0 represent al , %w1 represent dx
	asm volatile("outb %b0,%w1" : : "a" (data),"Nd" (port));
}

//write word_cnt byte from the begin of addr into port
static inline void outsw(uint16_t port,const void* addr, uint32_t word_cnt){
	// + is represent read and write ;outsw is write the 16 bit content in ds:esi into port
	asm volatile("cld;rep outsw": "+S" (addr) , "+c" (word_cnt) : "d" (port));
}

//read a byte from port
static inline uint8_t inb(uint16_t port) {
	uint8_t data;
	asm volatile ("inb %w1,%b0" : "=a"(data) :"Nd"(port));
	return data;
}

//read wrod_cnt byte from port into addr
static inline void insw(uint16_t port,const void* addr , uint32_t word_cnt){
	asm volatile("cld;rep insw" :"+D" (addr) , "+c"(word_cnt):"d"(port) :"memory");
}

#endif
~~~

##### device/timer.c:

由于IRQ0引脚上的时钟中断频率太低，通过对8253定时器编程，加快时钟中断的频率.

- 1.向8253的控制字寄存器写入控制字，端口为0x43，用于选择计数器，设置工作模式，读写格式，数制。

- 2.向8253的计数器0写入计算初值，端口为0x40，决定**时钟中断**信号的频率。



~~~c
#include "timer.h"
#include "io.h"
#include "print.h"

#define IRQ0_FREQUENCY 100
#define INPUT_FREQUENCY 1193180
#define COUNTER0_VALUE INPUT_FREQUENCY / IRQ0_FREQUENCY				//the initial value of counter 0 
#define COUNTER0_PORT 0x40
#define COUNTER0_NO 0 								//No.1 counter 
#define COUNTER0_MODE 2							
#define READ_WRITE_LATCH 3
#define PIT_CONTROL_PORT 0x43

//write the control word into the control word register and set the initial value 
static void frequency_set(uint8_t counter_port, uint8_t counter_no, uint8_t rwl, uint8_t counter_mode, uint16_t counter_value){
	//write into control word register
	outb(PIT_CONTROL_PORT,(uint8_t)(counter_no << 6 | rwl << 4 | counter_mode << 1 ));
	//write the low bit of counter_value
	outb(counter_port,(uint8_t)(counter_value));
	//writhe the hight bit of counter_value
	outb(counter_port,(uint8_t)(counter_value >> 8));
}

//initial PIT8253
void timer_init(){
	put_str("timer_init start \n");
	//set the frequency of counter
	frequency_set(COUNTER0_PORT,COUNTER0_NO,READ_WRITE_LATCH, COUNTER0_MODE, COUNTER0_VALUE);
	put_str("timer_init done \n");
}
~~~

##### device/timer.h:

~~~c
#ifndef __DEVICE_TIME_H
#define __DEVICE_TIME_H
void timer_init();
#endif
~~~



##### kernel/init.c:

init_all负责所有的初始化工作

~~~c
#include "init.h"
#include "print.h"
#include "interrupt.h"
#include "../device/timer.h"

//initial all module
void init_all(){
	put_str("init_all\n");
	idt_init();
	timer_init();
}
~~~

##### kernel/init.h:

~~~c
#ifndef __KERNEL_INIT_H
#define __KERNEL_INIT_H
void init_all();
#endif
~~~

##### kernel/main.c:

这里我们开启中断查看效果，之前我们开启了时钟中断

~~~c
#include "print.h"
#include "init.h"
int main(void){
	put_str("Kernel Starting!\n");
	init_all();
	asm volatile ("sti");
	while(1);
	return 0;
}
~~~

### 4.流程总结

- main函数调用init_all，初始化所有
- 此时先初始化idt，然后加强时钟中断timer_init
- idt的初始化中：
  - 先通过kernel.S中的全局数组intr_entry_table提供中断处理函数的入口，初始化中断描述符表
  - 再初始化一般处理函数和异常函数名称注册。**这里的函数才是真正的中断处理函数，intr_entry_table变项对应的函数中只是调用这里初始化的处理函数，所以只是中断处理函数的入口**
  - 然后开始8259A(中断代理)的初始化
  - 使用内联汇编加载idt
- timer_init加强时钟中断,通过编程8253，增加时钟中断频率
- 初始化结束


