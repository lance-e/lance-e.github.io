# 手写OS系列之bootloader


<!--more-->

# 手写OS系列之boot loader

### 1.操作系统是如何启动的呢？

首先先区别一下“载入内存”与“程序运行”区别：

- 载入内存：程序被加载器加载到内存某个区域
- 程序运行：CPU的cs:ip指向程序的起始地址

所以“载入内存” !=“程序运行”



背景知识：intel 8086有20条地址总线，可以访问2^20=1MB内存，也就是0x00000到0xFFFFF。BIOS(basic input & output system)处于0xF0000到0xFFFFF,大小为64kb。



开机(接电)一瞬间，CPU的cs:ip被强制初始化为0xF000:0xFFF0，因为处于实模式中，段基址左移四位，再加上段内偏移地址，可得cs:ip指向0xFFFF0，也就是BIOS的入口地址。bios入口在0xFFFF0~0xFFFFF，大小只有16b，这么小的空间，那么也只能是一条跳转指令，跳转到真正开始的位置。



BIOS执行结束之后，就该执行MBR(主引导记录main boot record)了。但在跳转之前，会先检验启动盘中的0盘0道1扇区（也就是最开始的扇区），该扇区的结尾必须是0x55，0xaa！BIOS才可以将MBR加载到0x7c00。



MBR再对硬盘操作，读取出硬盘中指定扇区里的loader程序加载到内存，结束后跳转到loader所在的内存继续取指执行。



loader的主要负责实模式向保护模式等过渡。

**为什么是0x7c00呢？** 书中提到：

- 早期的PC 5150 BIOS研发工程师为以32kb为最小内存的操作系统设计BIOS，32k = 0x8000
- MBR本身大小为512字节，再为所用的栈分配内存，一共分配了1kb
- 为了尽可能保护MBR不被过早破坏，所以放在最后1kb(0x400);
- 所以最后放在了0x8000-0x0400 = 0x7c00

### 2.手写MBR

mbr加载boader：对硬盘设备操作，进行读取和加载，把loader程序加载到指定内存位置。



对io端口操作：

- in: in ax, dx / in al,dx
- out: out dx, al /out dx,ax/ out 立即数,al / out 立即数,ax

硬盘操作基本流程：

- 先选择通道，想sector count寄存器写入要操作的扇区数
- 往该通道的三个LBA寄存器写入扇区的起始地址的低24位
- 然后往device寄存器中写入24～27位，并设置第四位选择主从盘，设置第六位选择LBA模式还是CHS模式
- 然后向该通道的command寄存器中写入操作命令，这里主要用到三个
  - Identify:0xEC,硬盘识别
  - read sector：0x20，读扇区
  - write sector：0x30，写扇区
- 读区status寄存器，判断硬盘工作是否完成
- 若是读磁盘就接着把数据读出，若是写磁盘就结束了

boot/mbr.S:这里就是我们写的mbr代码，前面部分是操作显存，然后打印我们定义的字符（无关紧要），后面就开始操作硬盘，

~~~assembly
%include	"boot.inc"
SECTION MBR vstart=0x7c00
	mov	ax,cs
	mov 	ds,ax
	mov 	es,ax
	mov 	ss,ax
	mov 	fs,ax
	mov 	sp,0x7c00
	
	mov 	ax,0xb800 		;add to operate the graphics memory,and 0xb8000 is the begining of the graphics memory 
	mov	gs,ax			

;clear the screen: use the 0x10 interrupt
;input:
;AH = 0x06(function number)
;AL = the upScroll's row number
;BH = the attribute of the upScroll
;(CL,CH) = (x,y) in the upper left corner of windows
;(DL,DH) = (x,y) int the lower right corner of windows

	mov 	ax,0x0600
	mov 	bx,0x0700
	mov 	cx,0			;the upper left corner (0,0)
	mov 	dx,0x184f		;the lower right corner (24,79)  (0x18 = 24,0x4f=79)
	
	int 	0x10
	
	mov	byte	[gs:0x00],'L'
	mov	byte	[gs:0x01],0xA4
	mov	byte	[gs:0x02],'O'
	mov	byte	[gs:0x03],0xA4
	mov	byte	[gs:0x04],'O'
	mov	byte	[gs:0x05],0xA4
	mov	byte	[gs:0x06],'N'
	mov	byte	[gs:0x07],0xA4
	mov	byte	[gs:0x08],'G'
	mov	byte	[gs:0x09],0xA4
	mov	byte	[gs:0x0A],'-'
	mov	byte	[gs:0x0B],0xA4
	mov	byte	[gs:0x0C],'O'
	mov	byte	[gs:0x0D],0xA4
	mov	byte	[gs:0x0E],'S'
	mov	byte	[gs:0x0F],0xA4

	;begin to load the loader
	;operate the io
	mov	eax,LOADER_START_SECTOR
	mov	bx,LOADER_BASE_ADDR
	mov	cx,4
	call	rd_disk_m_16
	jmp	LOADER_BASE_ADDR
rd_disk_m_16:
	;copy the value
	mov	esi,eax
	mov	di,cx

	mov	al,cl
	mov	dx,0x1F2
	out	dx,al
	
	mov	eax,esi				;recover the value of ax

	mov	dx,0x1F3
	out	dx,al
	
	mov	cl,0x8
	shr	eax,cl
	mov	dx,0x1F4
	out	dx,al

	shr	eax,cl
	mov	dx,0x1F5
	out	dx,al

	shr	eax,cl
	and	al,0x0f
	or 	al,0xe0			;11100000 = 0xe0,
	mov	dx,0x1F6
	out	dx,al
	mov	ax,0x20		
	mov	dx,0x1F7
	out	dx,al

   .not_ready:
	nop
	in	al,dx
	and	al,0x88
	cmp	al,0x08
	jnz	.not_ready

	;begine to loop read
	;set the source 
	mov	ax,di
	mov	dx,256
	mul	dx
	mov	cx,ax
	mov	dx,0x1F0
   .loop_read_data:
	in	ax,dx
	mov	[bx],ax
	
	add	bx,2
	loop	.loop_read_data
	
	;read over
	ret

	times 510-($-$$) db 0
	db 	0x55,0xaa

	
~~~

include/boot.inc

~~~assembly
LOADER_BASE_ADDR equ 0x900
LOADER_START_SECTOR equ 0x2
~~~

### 3.手写BOADER

##### GDT结构：

<table>
<thead>
  <tr>
    <th>字段名</th>
    <th>位区域</th>
    <th>作用</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>段界限0~15</td>
    <td>低32位0~15</td>
    <td>表示段边界的扩展最值。<br>代码段和数据段的段扩展方向向上；栈段段段扩展方向向下。加上高32位中的16~19位共20位。<br>此段界限为单位量，还取决于G位的段界限粒度。<br>计算实际段界限边界值：(描述符中段界限+1) *(段界限的粒度大小)-1</td>
  </tr>
  <tr>
    <td>段基址0~15</td>
    <td>低32位16~31</td>
    <td>保护模式下，CPU寻址方式为段基址：段内偏移地址，段基址共32位，被分成三份。</td>
  </tr>
  <tr>
    <td>段基址16~23</td>
    <td>高32位0~7</td>
    <td>同上段基址</td>
  </tr>
  <tr>
    <td>TYPE</td>
    <td>高32位8~11</td>
    <td>指定本描述符类型，与S位共同使用。<br>例如S位为1表示非系统段时，type第0~3位在代码段分别表示A,R,C,X，在数据段分别表示A,W,E,X</td>
  </tr>
  <tr>
    <td>S</td>
    <td>高32位12</td>
    <td>指示是否为系统段，0表示系统段，1表示数据段</td>
  </tr>
  <tr>
    <td>DPL</td>
    <td>高32位13~14</td>
    <td>描述符特权级(descriptor privilege level)，这两位可表示4种级别:0，1，2，3</td>
  </tr>
  <tr>
    <td>P</td>
    <td>高32位15</td>
    <td>表示段是否存在</td>
  </tr>
  <tr>
    <td>段界限16~19</td>
    <td>高32位16~19</td>
    <td>同上段界限</td>
  </tr>
  <tr>
    <td>AVL</td>
    <td>高32位20</td>
    <td>对os来说随便用，对硬件来说，没有专门的用途</td>
  </tr>
  <tr>
    <td>L</td>
    <td>高32位21</td>
    <td>设置是否为64位代码段，1表示64位，0表示32位</td>
  </tr>
  <tr>
    <td>D/B</td>
    <td>高32位22</td>
    <td>用来表示指令中的有效地址和操作数大小。<br>对于代码段来说，为D位，D为0表示16位，指令有效地址用ip寄存器，D为1表示32位，指令有效地址用eip寄存器。<br>对于栈段来说，为B位，B为0使用sp寄存器，最大寻址范围为0xFFFF，B为1使用esp寄存器，最大寻址范围为0xFFFFFFFF</td>
  </tr>
  <tr>
    <td>G</td>
    <td>高32位23</td>
    <td>指定段界限单位大小(Granularity,粒度)，与段界限共同决定段大小，为0单位为1字节，为1单位为4kb</td>
  </tr>
  <tr>
    <td>段基址24~31</td>
    <td>高32位24~31</td>
    <td>同上段基址</td>
  </tr>
</tbody>
</table>


##### 保护模式和实模式：

保护模式和实模式是32位cpu的两种运行模式，可以兼容实模式下的程序，但是不是cpu变成16位cpu。所以实模式与16位cpu无关，是32位cpu提出的概念。实模式下的内存寻址是通过段寄存器中的段基址左移四位+段内偏移地址；保护模式下的内存寻址是通过段寄存器中的选择子，查GDT/LDT，获取段描述符，再根据段描述符中的段基址直接+段内偏移地址。

###### 1.实模式的特点：

- 操作系统和用户程序的特权级相同
- 所指向的地址都是物理地址
- 用户程序可以自由修改段基址
- 访问超过64kb的地址，需要切换段基址
- 一次只能运行一个程序，资源利用率低
- 共20条地址线，最大可用内存为1mb

###### 2.进入保护模式前的三个步骤：

- 开启A20地址线
- 在gdtr寄存器中加载GDT的地址和偏移量(界限值)(lgdt指令)
- 将cr0寄存器的pe位置为1

###### 3.进入保护模式的两个需要解决的问题：

- 在实模式下的段描述符缓冲寄存器未更新：当每次计算段基址后，会将计算结果更新到段描述符缓冲寄存器，但是实模式下，段基址只有20位，所以此时段描述符缓冲寄存器只有低20位有效。进入保护模式后，很多属性位都是错误的，并且不重新引用一个段时，段描述符缓冲寄存器是不会更新的，所以需要马上更新其值，也就是想办法往相应寄存器中加载选择子。
- 流水线指令译码错误：流水线是cpu提高效率采用的一种工作方式，cpu将当前和后续指令同时放在流水线重叠执行，同时取指令，译码，执行。但是进入保护模式后，指令格式就发生了改变，译码就会错误，所以需要当进入保护模式后，清空以实模式的16位指令格式译码的流水线。

解决方法：使用无条件远跳转指令jmp。

##### 获取物理内存容量方法：

使用BIOS中断0x15

- EAX=0xE820,遍历主机上全部内存，迭代式查询，每次只返回一种类型的内存信息，存储内存信息的结构为地址范围描述符结构。
- AX=0xE801，分别检测15mb和16mb～4gb内存
- AH= 0x88，最多检测64mb内存

##### 分页机制

在保护模式中，寻址过程为在段寄存器中获取选择子，使用选择子来索引GDT/LDT表中的段描述符，从段描述符中可获取段基址，通过段基址+段内偏移进行寻址，这也叫做线性地址。在未开启分页机制时，线性地址就是物理地址，开启分页机制后，线性机制是虚拟地址，不是物理地址，需要查页表才能获取真实的物理地址。



为什么要有分页机制：

- 没有分页机制前，线性地址=物理地址，是连续且一一对应的，内存利用率低。所以提出了分页机制

一级页表相关知识：

- cr3控制寄存器存储**页表**物理地址
- 一级页表是用一个页表放入1M个标准页，一个页表表示4GB空间
- 32位地址表示4GB空间，高20位代表内存块数量，用于索引页，低12位代表内存块大小(4kb)，用于页内寻址
- 虚拟地址向物理地址转换的过程中，页表和页表项(**PTE,4字节**)的地址都是物理地址(防止无限套娃)
- **物理地址=线性地址高20位在页表中索引的页表项中的物理地址+线性地址低12位**
- 注意：高20位为索引值，需要x4(页表项大小4字节)，加上页表物理地址,才能求出目标页表项的物理地址

二级页表相关知识：

- cr3控制寄存器存储**页目录表**物理地址
- 二级是用1k个页表，每个页表就只放入1k个标准页，一个页表表示4mb空间
- 一级页表大小为4mb，二级页表大小为4kb (= 标准页大小)
- 新增页目录表来存储1k个页表，页表物理地址在页目录表中以页目录项(PDE)形式存储，大小也是**4字节**，所以页目录表大小也是4kb
- 页目录表共1k项= 2^10,页表共1k项=2^10,每页大小4kb=2^12,所以将32位虚拟地址分成了10位，10位，12位
- **物理地址=(线性地址高10位在页目录表中索引到的页目录项中获取页表的物理地址)线性地址中间10位在页表中索引的页表项中的物理地址+线性地址低12位**
- 注意：高10位和中间10位分别为PDE和PTE索引值，需要x4(页目录项和页表项4字节)，加上页目录表和页表的物理地址，才能求出目标物理地址



##### 加载内核：

loader需要进行的操作：

- 加载内核：将处在磁盘中的内核文件加载到内存中
- 初始化内核：需要在分页之后，将加载的elf内核文件放置到对应虚拟内存地址，然后跳转执行，从此loader工作结束

内核被加载到内存中，loader会通过其elf结构展开到新的位置，所以说内核在内存中有两份，一份是初始的elf内核文件，一份是解析elf格式后生成的内核映像（真正运行的内核）



##### 完整代码：

loader.S:

~~~assembly
%include	"boot.inc"
section	loader	vstart=LOADER_BASE_ADDR
LOADER_STACK_TOP	equ	LOADER_BASE_ADDR

GDT_BASE:	dd	0x00000000			;this is 0th descriptor
		dd	0x00000000
CODE_DESC:	dd	0x0000FFFF
		dd	DESC_CODE_HIGH4
DATA_STACK_DESC: dd	0x0000FFFF
		dd	DESC_DATA_HIGH4
VIDEO_DESC:	dd	0x80000007			;limit=(0xbffff - 0xb8000) / 4k = 0x7
		dd	DESC_VIDEO_HIGH4
GDT_SIZE	equ	$ - GDT_BASE
GDT_LIMIT	equ	GDT_SIZE - 1

times	60	dq	0				;reserve 60 descriptor (8 byte);

SELECTOR_CODE	equ	(0x0001 << 3 ) + TI_GDT + RPL0
SELECTOR_DATA	equ	(0x0002 << 3 ) + TI_GDT + RPL0
SELECTOR_VIDEO	equ	(0x0003 << 3 ) + TI_GDT + RPL0

total_mem_bytes	dd	0				;!!!save the total memory capacity

gdt_ptr		dw	GDT_LIMIT			;the pointer of GDT
		dd	GDT_BASE
	
ards_buf	times	244	db	0		;in order to memory alignment 
ards_nr		dw	0				;record the number of ARDS
loader_start:
	;get the physic memory
	xor	ebx,ebx					;EBX:set the ebx = 0 
	mov	edx,0x534d4150				;EDX:the ascii of SMAP
	mov	di,ards_buf				;DI:set the buffer of ARDS

;---------------------int 0x15 eax=0xe820------------------------
.e820_mem_get_loop:
	mov	eax,0x0000e820				;EAX:set the function number is 0xe820 per loop,because of the EAX will update to the 0x534d4150 after eachinterruption 
	mov	ecx,20					;ECX:set the size of ARDS
	int	0x15				
	jc	.e820_failed_so_try_e801	
	
	add	di,cx					;add 20 byte that make the di point to the next ARDS
	inc	word	[ards_nr]			;record the number of the ARDS
	cmp	ebx,0					;if the ebx = 0 and the cf != 1 mean that the ARDS are all return.this one is the last one
	jnz	.e820_mem_get_loop

	;calculate the memory capacity 
	mov	cx,[ards_nr]
	mov	ebx,ards_buf
	xor	edx,edx
.find_max_mem_area:
	mov	eax,[ebx]				;base_add_low
	add	eax,[ebx+8]				;length_low
	add	ebx,20					;point to the next ARDS
	cmp	edx,eax
	jge	.next_ards
	mov	edx,eax					;edx = the all memory
.next_ards:
	loop	.find_max_mem_area
	jmp	.mem_get_ok

;-----------------------int 0x15 ax=0xe801---------------------
.e820_failed_so_try_e801:
	mov	ax,0xe801
	int	0x15
	jc	.e801_failed_so_try88
	
	;calculate the memory capacity
	;1.the memory less than 15mb
	mov	cx,0x400
	mul	cx
	shl	edx,16
	and	eax,0x0000FFFF
	or 	edx,eax
	add	edx,0x100000
	mov	esi,edx
	;2.the memory more than 16mb
	xor	eax,eax
	mov	ax,bx
	mov	ecx,0x10000
	mul	ecx
	add	esi,eax
	mov	edx,esi					;edx = the all memory
	jmp	.mem_get_ok

;------------------------int 0x15 ah=0x88-------------------------
.e801_failed_so_try88:
	mov	ah,0x88
	int	0x15
	jc	.error_hlt
	and	eax,0x0000FFFF
	mov	cx,0x400
	mul	cx
	shl	edx,16
	or	edx,eax
	add	edx,0x100000
	
.mem_get_ok:
	mov	[total_mem_bytes],edx
	
	
	;enter to the pretect mode
	;1.open the A20 address line
	;2.load the gdt
	;3.set the cr0's pe to 1
	
	in	al,0x92
	or	al,0000_0010B
	out	0x92,al

	lgdt	[gdt_ptr]
	
	mov	eax,cr0
	or	eax,0x00000001
	mov	cr0,eax

	;over
	jmp	dword	SELECTOR_CODE:p_mode_start
.error_hlt:
	jmp	$

[bits 32]
p_mode_start:
	mov	ax,SELECTOR_DATA
	mov	ds,ax
	mov	es,ax
	mov	ss,ax
	mov	esp,LOADER_STACK_TOP
	mov	ax,SELECTOR_VIDEO
	mov	gs,ax
	
	mov	byte	[gs:160], 'P'

;--------------------------load kernel--------------------------------

	mov	eax,KERNEL_START_SECTOR			;the sector of kernel	
	mov	ebx,KERNEL_BIN_BASE_ADDR		;the target address of load the kernel
	mov	ecx,200					;the number of kernel's sector
	
	call	rd_disk_m_32


;------------------------turn on the page machanism-----------------------------
	call	setup_page
	
	sgdt	[gdt_ptr]
	
	mov	ebx,[gdt_ptr+2]				;the video descriptor
	or	dword	[ebx+0x18+4],0xc0000000
	add	dword 	[gdt_ptr+2],0xc0000000
	add	esp,0xc0000000
	
	mov	eax,PAGE_DIR_TABLE_POS			;set cr3 = the position of page directory 
	mov	cr3,eax
	
	mov	eax,cr0
	or 	eax,0x80000000				;set the pg position of cr0
	mov	cr0,eax

	lgdt 	[gdt_ptr]				;reload the new gdt

;---------------enter kernel--------------------------------

	jmp	SELECTOR_CODE:enter_kernel		;refresh assembly line,just in case
enter_kernel:	
	call	kernel_init
	mov	esp,0xc009f000
	mov	byte	[gs:160], 'K'
	jmp	KERNEL_ENTRY_POINT



;---------------setup page ----------------------------
setup_page:
	mov	ecx,4096				;the size of page directory is 4kb , so set the ecx= 4096
	mov	esi,0
	;clear the memory
.clear_page_dir:
	mov	byte	[PAGE_DIR_TABLE_POS+esi],0
	inc	esi
	loop	.clear_page_dir				

	;create the page directory entry
.create_pde:	
	mov	eax,PAGE_DIR_TABLE_POS
	add 	eax,0x1000
	mov	ebx,eax					;ebx is the base address of pte
	or	eax,PG_US_U | PG_RW_W | PG_P		;eax=0x101111 ,this is position and attribute of first page table
	mov	[PAGE_DIR_TABLE_POS + 0x0],eax
	mov	[PAGE_DIR_TABLE_POS + 0xc00],eax	;0xc00 is the boundary between user program  and operation system
	sub	eax,0x1000
	mov	[PAGE_DIR_TABLE_POS+4092],eax
	
	;create the page table entry
	mov	ecx,256
	mov	esi,0
	mov	edx,PG_US_U | PG_RW_W | PG_P		;edx = 0x111, the attribute is 7 , the position is 0 
.create_pte:
	mov	[ebx+esi*4],edx
	add	edx,4096				;4k
	inc	esi
	loop	.create_pte
	

	;create the kernel page directory entry
	mov	eax,PAGE_DIR_TABLE_POS
	add	eax,0x2000				;eax is the position of second page table
	or	eax,PG_US_U | PG_RW_W | PG_P
	mov	ebx,PAGE_DIR_TABLE_POS
	mov	ecx,254					;the number of all pde between 769 ~ 1022
	mov	esi,769
.create_kernel_pde:
	mov	[ebx+esi*4],eax
	inc	esi
	add	eax,0x1000
	loop	.create_kernel_pde
	ret

;--------------------copy the segment of kernel.bin to the address of compile kernel -------------

kernel_init:
	xor	eax,eax
	xor	ebx,ebx					;ebx:record the address of program header table(e_phoff)
	xor	ecx,ecx					;cx:record the number of program header in program header table(e_phnum)
	xor	edx,edx					;dx:record the size of program header(e_phentsize)
	
	mov	dx,[KERNEL_BIN_BASE_ADDR+42]		;42 is the e_phentsize's offset
	mov	ebx,[KERNEL_BIN_BASE_ADDR+28]		;28 is the e_phoff's offset. Here is mean the first program header
	add	ebx,KERNEL_BIN_BASE_ADDR
	mov	cx,[KERNEL_BIN_BASE_ADDR+44]		;44 is the e_phnum's offset

.each_segment:
	cmp	byte	[ebx+0],PT_NULL			;compare between  p_type and PT_NULL,
	je	.PTNULL					;if == is that the program header isn't use, so jump
	
	;push the params for function memcpy

	;first param:size
	push	dword	[ebx+16]			;16 is the p_filesz's offset

	;scond param:source address 
	mov	eax,[ebx+4]				;4 is the p_offset's offset
	add 	eax,KERNEL_BIN_BASE_ADDR		;add the kernel.bin's load address.
	push 	eax					;Now the eax is the physic memory of this section,

	;third param;target address
	push	dword	[ebx+8]				;8 is the p_vaddr's offset
	
	call 	mem_cpy
	add	esp,12					;clear the three params in the stack

.PTNULL:
	add	ebx,edx					;edx is the size of program header .So here is make the ebx point to the next program header
	loop	.each_segment
	ret

;-------------------------mem_cpy(dst,src,size)---------------------------;
;			(byte by byte copy)				  ;
;-------------------------------------------------------------------------;
mem_cpy:
	;save the environment
	cld
	push	ebp
	mov	ebp,esp
	push	ecx					
	
	mov	edi,[ebp+8]				;dst
	mov	esi,[ebp+12]				;src
	mov	ecx,[ebp+16]				;size
	rep	movsb					;byte by byte copy
	
	;recovery the environment 
	pop	ecx
	pop	ebp
	ret
			
rd_disk_m_32:
	;copy the value
	mov	esi,eax
	mov	edi,ecx

	mov	al,cl
	mov	dx,0x1F2
	out	dx,al
	
	mov	eax,esi				;recover the value of ax

	mov	dx,0x1F3
	out	dx,al
	
	mov	cl,0x8
	shr	eax,cl
	mov	dx,0x1F4
	out	dx,al

	shr	eax,cl
	mov	dx,0x1F5
	out	dx,al

	shr	eax,cl
	and	al,0x0f
	or 	al,0xe0			;11100000 = 0xe0,
	mov	dx,0x1F6
	out	dx,al
	mov	ax,0x20		
	mov	dx,0x1F7
	out	dx,al

   .not_ready:
	nop
	in	al,dx
	and	al,0x88
	cmp	al,0x08
	jnz	.not_ready

	;begine to loop read
	;set the source 
	mov	ax,di
	mov	dx,256
	mul	dx
	mov	cx,ax
	mov	dx,0x1F0
.loop_read_data:
	in	ax,dx
	mov	[ebx],ax
	
	add	ebx,2
	loop	.loop_read_data
	
	;read over
	ret
~~~

include/boot.inc

~~~assembly
;--------------------loader and kernel -----------------

LOADER_BASE_ADDR equ 0x900
LOADER_START_SECTOR equ 0x2

PAGE_DIR_TABLE_POS	equ	0x100000			;the position of page directory table


;--------------------gdt:the attribute of descriptor----

DESC_G_4K	equ	1_00000000000000000000000b
DESC_D_32	equ	1_0000000000000000000000b
DESC_L		equ	0_000000000000000000000b		;L : set to the 64 bit code snippet
DESC_AVL	equ	0_00000000000000000000b
DESC_LIMIT_CODE2 equ 	1111_0000000000000000b
DESC_LIMIT_DATA2 equ	DESC_LIMIT_CODE2
DESC_LIMIT_VIDEO2 equ	0000_0000000000000000b
DESC_P		equ	1_000000000000000b
DESC_DPL_0	equ	00_0000000000000b
DESC_DPL_1	equ	01_0000000000000b
DESC_DPL_2	equ	10_0000000000000b
DESC_DPL_3	equ	11_0000000000000b
DESC_S_CODE	equ	1_000000000000b
DESC_S_DATA	equ	DESC_S_CODE
DESC_S_sys	equ	0_000000000000b
DESC_TYPE_CODE	equ	1000_00000000b				;code segment:x=1,c=0,r=0,a=0
DESC_TYPE_DATA	equ	0010_00000000b				;data segment:x=0,e=0,w=1,a=0

DESC_CODE_HIGH4	equ	(0x00 << 24) + DESC_G_4K + DESC_D_32 + \
DESC_L + DESC_AVL + DESC_LIMIT_CODE2 + \
DESC_P + DESC_DPL_0 + DESC_S_CODE + \
DESC_TYPE_CODE + 0x00

DESC_DATA_HIGH4 equ	(0x00 << 24) + DESC_G_4K + DESC_D_32 +\
DESC_L + DESC_AVL + DESC_LIMIT_DATA2 +\
DESC_P + DESC_DPL_0 + DESC_S_DATA + \
DESC_TYPE_DATA+ 0x00

DESC_VIDEO_HIGH4 equ	(0x00 << 24) + DESC_G_4K + DESC_D_32 +\
DESC_L + DESC_AVL + DESC_LIMIT_VIDEO2 +\
DESC_P + DESC_DPL_0 + DESC_S_DATA + \
DESC_TYPE_DATA+ 0x0B						;the begining of graphics memory is 0xb8000

;----------------------selector: the attribute of selector-------
RPL0	equ	00b
RPL1	equ	01b
RPL2	equ	10b
RPL3	equ	11b
TI_GDT	equ	000b
TI_LDT	equ	100b

;---------------------page table : the attribute of page table -------------

PG_P	equ	1b
PG_RW_R	equ	00b
PG_RW_W	equ	10b
PG_US_S	equ	000b
PG_US_U	equ	100b

;----------------------kernel ------------------------------------
KERNEL_START_SECTOR	equ	0x9
KERNEL_BIN_BASE_ADDR	equ	0x70000
KERNEL_ENTRY_POINT	equ	0xc0001500


PT_NULL	equ	0x0
~~~

### 4.总结

迈入内核，以后再来梳理bootloader全流程^_^


