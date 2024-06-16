# 手写OS系列之实现用户进程


<!--more-->

# 手写OS系列之实现用户进程

### 1.相关补充

#### TSS：

​	Task Status Segment，任务状态段，是处理器在硬件上原生支持多任务的一种实现方式，由TR(Task Register)寄存器加载，使用ltr指令。与其他段一样，本质上只是一片存储数据的内存区域，所以需要TSS描述符在GDT中注册，用来找到TSS。

#### LDT：

​	Local Descriptor Table,，局部描述符表，用于为每个程序单独赋予的一个结构来存储其私有的资源。

#### 任务切换方式：

​	cpu厂商原生提供了LDT和TSS这两种原生支持，LDT存放任务实体资源(可有可无，可以使用GDT来代替)，TSS保存任务上下文状态，三种特权级栈指针，I/O位图等信息。任务切换的方式有“中断+任务门”，“call或jmp+任务门” 和iretd。

​	由于每一次任务切换都要重新加载TR寄存器，效率低下，现代操作系统未采用原生提供的任务切换方式，但是由于 **CPU 向高特权级转移时所使用的栈地址，需要提前在TSS中写入 ** ，所以必须要使用TSS。转移到高特权级一般都是在用户模式发生，CPU由低特权级向高特权级转移时，会发生堆栈的转移，当发生中断时，处理器会从当前TSS的SS0和esp0获取用于处理中断的堆栈。因此我们必须创建一个TSS。

​	**Linux的任务切换方法**：为每个CPU创建一个TSS，在各个CPU上的任务共享一个TSS，每个CPU的TR寄存器保存对应的TSS，ltr指令加载TSS后，该TR寄存器永远指向这一个TSS。任务切换时，只把TSS的SS0和esp0更新为新任务的0特权级栈栈地址和栈指针。

### 2.定义并初始化TSS

##### 1.kernel/global.h：

增加tss描述符属性以及增加用户态代码段，数据段和栈段

~~~c
#ifndef	__KERNEL_GLOBAL_H
#define	__KERNEL_GLOBAL_H
#include "stdint.h"

//---------------------------macro in c progam --------------------------
#define NULL ((void*)0)
#define DIV_ROUND_UP(X,STEP) ((X + STEP -1 ) / (STEP))
#define bool int
#define true 1 
#define false 0 

//---------------------------- the attribute of SELECTOR----------------

#define	RPL0	0
#define RPL1	1
#define RPL2	2
#define RPL3	3

#define TI_GDT 0
#define TI_LDT 1

#define	SELECTOR_K_CODE	((1<<3) + (TI_GDT << 2 ) + RPL0)
#define	SELECTOR_K_DATA ((2<<3)+(TI_GDT << 2 ) + RPL0)
#define	SELECTOR_K_STACK  SELECTOR_K_DATA
#define SELECTOR_K_GS ((3<<3) + (TI_GDT << 2) + RPL0)
#define SELECTOR_U_CODE ((5<<3) + (TI_GDT << 2) + RPL3)
#define SELECTOR_U_DATA ((6<<3) + (TI_GDT << 2) + RPL3)
#define SELECTOR_U_STACK SELECTOR_U_DATA


//------------------------------- the attribute of GDT descriptor---------

#define DESC_G_4k 1
#define DESC_D_32 1
#define DESC_L 	0
#define DESC_AVL 0
#define DESC_P 1
#define DESC_DPL_0 0
#define DESC_DPL_1 1
#define DESC_DPL_2 2
#define DESC_DPL_3 3

#define DESC_S_CODE 1
#define DESC_S_DATA DESC_S_CODE
#define DESC_S_SYS 0

#define DESC_TYPE_CODE 8				//x=1,e=0,r=0,a=0
#define DESC_TYPE_DATA 2				//x=0,e=0,w=1,a=0
#define DESC_TYPE_TSS  9				//B = 0 ,not busy

#define GDT_ATTR_HIGH	\
	((DESC_G_4k << 7 ) + (DESC_D_32 << 6 ) + (DESC_L << 5) + (DESC_AVL << 4))

#define GDT_CODE_ATTR_LOW_DPL3	\
	((DESC_P << 7 ) + (DESC_DPL_3 << 5) + (DESC_S_CODE << 4) + DESC_TYPE_CODE )

#define GDT_DATA_ATTR_LOW_DPL3	\
	((DESC_P << 7) + (DESC_DPL_3 << 5) + (DESC_S_CODE << 4 ) + DESC_TYPE_DATA)

//-------------------------------the attribute of IDT descriptor-----------

#define	IDT_DESC_P	1
#define IDT_DESC_DPL0	0
#define	IDT_DESC_DPL3	3
#define	IDT_DESC_32_TYPE	0xE			//32 bit
#define IDT_DESC_16_TYPE	0x6			//16 bit , won't use ,just for compare
#define IDT_DESC_ATTR_DPL0	((IDT_DESC_P << 7 ) + (IDT_DESC_DPL0 << 5 ) + IDT_DESC_32_TYPE)
#define IDT_DESC_ATTR_DPL3	((IDT_DESC_P << 7 ) + (IDT_DESC_DPL3 << 5 ) + IDT_DESC_32_TYPE)



//------------------------------- the attribute of TSS descriptor----------

#define TSS_DESC_D 0

#define TSS_ATTR_HIGH	\
	((DESC_G_4k << 7 ) + (TSS_DESC_D << 6) + (DESC_L << 5) + (DESC_AVL << 4) + 0x0)

#define TSS_ATTR_LOW 	\
	((DESC_P << 7 ) + (DESC_DPL_0 << 5 ) + (DESC_S_SYS << 4) +DESC_TYPE_TSS)

#define SELECTOR_TSS ((4<< 3) + (TI_GDT << 2) + RPL0 )

// define the struct of GDT
struct gdt_desc{
	uint16_t limit_low_word;
	uint16_t base_low_word;
	uint8_t base_mid_byte;
	uint8_t attr_low_byte;
	uint8_t limit_high_attr_high;
	uint8_t base_high_byte;
};


//------------------------------ the attribute of eflags -------------------

#define EFLAGS_MBS (1<<1)
#define EFLAGS_IF_1 (1<<9)
#define EFLAGS_IF_0 0
#define EFLAGS_IOPL_3 (3<<12)
#define EFLAGS_IOPL_0 (0<<12)



#endif
~~~

##### 2.userprog/tss.c:

定义tss结构，同时声明一个实例

~~~c
// struct of tss
struct tss{
        uint32_t backlink;                                     		//pointer of last tss 
        uint32_t* esp0;
        uint32_t ss0;
        uint32_t* esp1;
        uint32_t ss1;
        uint32_t* esp2;
        uint32_t ss2;
        uint32_t* esp3;
        uint32_t ss3;
        uint32_t cr3;
        uint32_t (*eip) (void);
        uint32_t eflags;
        uint32_t eax;
        uint32_t ecx;
        uint32_t edx;
        uint32_t ebx;
        uint32_t esp;
        uint32_t ebp;
        uint32_t esi;
        uint32_t edi;
        uint32_t es;
        uint32_t cs;
        uint32_t ss;
        uint32_t ds;
        uint32_t fs;
        uint32_t gs;
        uint32_t ldt;
        uint32_t trace;
        uint32_t io_base;
};
static struct tss tss;

~~~

更新tss的esp0为指定任务的0特权级栈：

~~~c
//update tss's esp0 to the 0 privilege level stack
void update_tss_esp(struct task_struct* pthread){
	// pcb is occupy a page,and the top of address is the 0 privilege level stack
	tss.esp0 = (uint32_t*)((uint32_t)pthread + PG_SIZE);	
}
~~~

创建tss，3特权级下的代码段数据段栈段的描述符，并在GDT中添加，重新加载GDT：

~~~c
//create the gdt descriptor
static struct gdt_desc make_gdt_desc(uint32_t* desc_addr , uint32_t limit, uint8_t attr_low,uint8_t attr_high){
	//descriptor base address
	uint32_t desc_base = (uint32_t ) desc_addr;
	struct gdt_desc desc;
	desc.limit_low_word = limit & 0x0000ffff;
	desc.base_low_word = desc_base & 0x0000ffff;
	desc.base_mid_byte = ((desc_base & 0x00ff0000) >> 16);
	desc.attr_low_byte = (uint8_t)attr_low;
	desc.limit_high_attr_high =(((limit & 0x000f0000 ) >> 16 ) + (uint8_t)attr_high);
	desc.base_high_byte = desc_base >> 24;
	return desc;
}


//create tss in GDT and reload GDT 
void tss_init(){
	put_str("tss_init start\n");
	uint32_t tss_size = sizeof(tss);
	memset(&tss , 0 , tss_size);

	tss.ss0 = SELECTOR_K_STACK;
	tss.io_base = tss_size;							//mean no io bitmap

	//add tss descriptor 
	*((struct gdt_desc * ) 0xc0000920 ) = make_gdt_desc(	\
		(uint32_t*) &tss,	\
		tss_size -1 ,		\
		TSS_ATTR_LOW,		\
		TSS_ATTR_HIGH);

	//add code descritor and data descritor (dpl = 3 )
	*((struct gdt_desc * )0xc0000928 ) = make_gdt_desc(	\
		(uint32_t*)0,		\
		0xfffff,			\
		GDT_CODE_ATTR_LOW_DPL3, \
		GDT_ATTR_HIGH);
	*((struct gdt_desc * )0xc0000930 ) = make_gdt_desc(	\
		(uint32_t*)0,		\
		0xfffff,			\
		GDT_DATA_ATTR_LOW_DPL3, \
		GDT_ATTR_HIGH);

	//load gdt and tss
	uint64_t gdt_operand = ((8 * 7 - 1) | ((uint64_t)(uint32_t)0xc0000900 << 16));

	asm volatile("lgdt %0" : : "m"(gdt_operand));
	asm volatile("ltr %w0" : : "r"(SELECTOR_TSS));
	put_str("tss_init and ltr done\n");
}
~~~

### 3.实现用户进程

基于之前实现的内核级线程实现用户进程，

#### 1.用户进程虚拟内存空间：

由于采用的是平坦模型，每个进程都拥有4GB虚拟空间，这意味着每个进程都有自己的页目录表(管理4GB虚拟内存)，还要有自己的虚拟内存池(基于位图)，用来管理内存：

thread/thread.h:

pcb中新增：pgdir代表进程的页目录表，userprog_vaddr代表进程的虚拟内存池。

**线程与进程的区别就是线程没有自己的地址空间，也就是没有自己的页表，所以线程的pgdir=NULL**

~~~c
//the PCB 
struct task_struct {
	uint32_t self_kstack;
	pid_t pid;
	enum task_status status;
	char name[16];
	uint8_t priority;
	uint8_t ticks;				//the time ticks of run in cpu

	uint32_t elapsed_ticks;			//record the all ticks after run in cpu

	struct list_elem general_tag;		//the node of general list

	struct list_elem all_list_tag;		//the node of thread list
	
	uint32_t* pgdir;			//the virtual address of process's page table,!:thread:NULL(thread don't have page table)

	struct virtual_addr userprog_vaddr;	// virtual address pool of user process 

	uint32_t stack_magic;			//edge of stack
};

~~~

#### 2.创建页目录表和3特权级栈：

kernel/memory.c:

- 给物理内存池增加一个互斥锁lock
- 为vaddr_get新增用户级部分，可以向用户虚拟内存池申请虚拟内存
- get_user_pages：在用户物理内存池中申请多个页
- get_a_page:将指定的一页虚拟内存，映射到pf对应的物理内存池。如果是用户进程申请用户内存，就修改用户进程的虚拟地址位图；如果是内核线程申请内核内存，就修改kernel_vaddr实例。
- addr_v2p：将虚拟内存转换为物理内存

~~~c
struct pool{
	struct bitmap pool_bitmap;			//to manage the physic memory
	uint32_t phy_addr_start;			//the start of physic memory to manage
	uint32_t pool_size;				//the size of pool
	struct lock lock;				//lock when apply for address

};
//apply for 'pg_cnt' virtual page in the 'pf' virtual memory pool
//success :return the begin of address , failed: return NULL
static void* vaddr_get(enum pool_flags pf,uint32_t pg_cnt){
	int vaddr_start =  0 , bit_idx_start = -1;
	uint32_t cnt = 0;
	if (pf == PF_KERNEL){
		//kernel address pool
		bit_idx_start = bitmap_scan(&kernel_vaddr.vaddr_bitmap,pg_cnt);
		if (bit_idx_start == -1 ){
			return NULL;
		}
		while (cnt < pg_cnt){
			bitmap_set(&kernel_vaddr.vaddr_bitmap,bit_idx_start+cnt++,1);
		}
		vaddr_start = kernel_vaddr.vaddr_start + bit_idx_start * PG_SIZE;
	}else {
		//user address pool
		struct task_struct* cur = running_thread();
		bit_idx_start = bitmap_scan(&cur->userprog_vaddr.vaddr_bitmap,pg_cnt);
		if (bit_idx_start == -1 ){
			return NULL;
		}
		while (cnt < pg_cnt){
			bitmap_set(&cur->userprog_vaddr.vaddr_bitmap,bit_idx_start+cnt++,1);
		}
		vaddr_start = cur->userprog_vaddr.vaddr_start + bit_idx_start * PG_SIZE;
		// (0xc0000000 - PG_SIZE ) has been allocate for user process's 3 level stack
		ASSERT((uint32_t) vaddr_start < (0xc0000000 - PG_SIZE));
	}
	return (void*) vaddr_start;
}



//apply for 'pg_cnt'  page in user physic memory pool
void* get_user_pages(uint32_t pg_cnt){
	lock_acquire(&user_pool.lock);
	void* vaddr = malloc_page(PF_USER,pg_cnt);
	if (vaddr != NULL){
		memset(vaddr , 0 , pg_cnt * PG_SIZE);
	}
	lock_release(&user_pool.lock);
	return vaddr;
}


//make a map of 'vaddr' and physic address in 'pf' pool ,just can map only a page
// (it is like malloc_page , but don't need apply virtual address again);
void* get_a_page(enum pool_flags pf, uint32_t vaddr){
	struct pool* mem_pool = (pf & PF_KERNEL)? &kernel_pool : &user_pool;
	lock_acquire(&mem_pool->lock);

	// first : set virtual address's bitmap to 1
	struct task_struct* cur = running_thread();
	int32_t bit_idx = -1;
	if (cur->pgdir != NULL &&  pf == PF_USER){
		bit_idx = (vaddr - cur->userprog_vaddr.vaddr_start) / PG_SIZE;
		ASSERT(bit_idx > 0);
		bitmap_set(&cur->userprog_vaddr.vaddr_bitmap,bit_idx, 1);
	}else if (cur->pgdir == NULL && pf == PF_KERNEL){
		bit_idx = (vaddr - kernel_vaddr.vaddr_start) / PG_SIZE;
		ASSERT(bit_idx > 0 );
		bitmap_set(&kernel_vaddr.vaddr_bitmap,bit_idx , 1);
	}else {
		PANIC("get_a_page: not allow kernel alloc userspace or user alloc kernelsapce by get_a_page\n");
	}

	// second : apply for physic address
	void* page_phyaddr = palloc(mem_pool);
	if (page_phyaddr == NULL){
		lock_release(&mem_pool->lock);
		return NULL;
	}
	// third : make map
	page_table_add((void*)vaddr,page_phyaddr);
	lock_release(&mem_pool->lock);
	return (void*)vaddr;
}

//get the physic address of 'vaddr'
uint32_t addr_v2p (uint32_t vaddr){
	uint32_t* pte = pte_ptr(vaddr);
	return ((*pte & 0xfffff000) + (vaddr & 0x00000fff));
}
	
~~~

#### 创建用户进程(重头戏^_^)：

##### 线程的相关流程：

​	由于用户进程是基于内核级线程，所以先回忆下内核级线程创建流程：

``thread_start()  --> init_thread()初始化pcb属性  --> thread_create()初始化0特权级栈 --> 把线程添加到就绪队列和全部任务队列 ``

​	当轮到该线程执行时，调度器就会从就绪队列中取出线程开始执行，流程：

``  时钟中断  --> 上一个任务的时间片耗尽 -->schedule() 将当前任务改为就绪态，重新添加到就绪队列，取出next任务 -->  switch_to() 保存当前任务上下文，加载next任务的上下文 ``

##### 进程的创建流程：

​	由于进程的pcb新增了页目录表的地址和用户进程的虚拟内存池，所以需要额外进行这两项的初始化，创建过程：

`` process_execute()    -->  init_thread() --> create_user_vaddr_bitmap() 创建用户进程的虚拟内存池  --> thread_create()  -->  create_page_dir() 创建用户进程的页目录表 -->  把进程添加到就绪队列和全部任务队列 ``

​	可以看到用户进程的创建过程与线程创建过程几乎一致，唯一的区别就是进程需要多进行页目录表的创建和虚拟内存池的初始化。



创建流程用到的函数：注意thread_create中的参数start_process，将会在下一节介绍，用于开始进程。

!create_page_dir!:	

- **实现了让所有进程共享内核空间，也就是通过将内核页目录表中的的内核态页目录项(第768~1023项，最后的256项)，复制到所有进程的内核态页目录项 **
- **将页目录表的最后一项初始化为页目录表的物理地址，用于找到页目录表的位置，因为可能进程运行过程中存在页表操作**

~~~c
//create page directory table
uint32_t* create_page_dir(void){

	//user can't get the user program 's page directory table , so create at kernel 
	uint32_t* page_dir_vaddr = get_kernel_pages(1);
	
	if (page_dir_vaddr == NULL){
		put_str("create_page_dir : get_kernel_pages failed \n");
		return NULL;
	}
	
	/********************** all process share kernel **************************************
	 *
	 *	make all process share kernel is to copy the page directory table of
	 *	kernel to every process
	 *
	 *	0x300 * 4 : 0x300 is the 768th pde(begin of kernel pde) , 4 is 4 byte per pde
	 *	0xfffff000 : the last pde ,is the address of page directory table
	 *	1024 : 1 GB of kernel is 256 th pde, so all is 256 * 4 = 1024 */

	memcpy((uint32_t*)((uint32_t)page_dir_vaddr + 0x300 * 4),(uint32_t*)(0xfffff000 + 0x300*4),1024 );

	/****************************************************************************************/

	/****************************  update address of page directory ***********************
	 *
	 * 	the last pde is the physic address of page directory table
	 * 	so update the last pde to the new physic address       */

	uint32_t new_page_dir_phy_addr = addr_v2p((uint32_t)(page_dir_vaddr)); 
	page_dir_vaddr[1023] = new_page_dir_phy_addr | PG_US_U | PG_RW_W | PG_P_1 ;

	/*************************************************************************************/

	return page_dir_vaddr;
}

//chreate user process's virtual address bitmap
void create_user_vaddr_bitmap(struct task_struct* user_prog){
	//0x804800
	user_prog->userprog_vaddr.vaddr_start  = USER_VADDR_START ;
	
	// the length of bitmap
	uint32_t bitmap_bytes_len =(uint32_t)((0xc0000000 - USER_VADDR_START ) / PG_SIZE / 8);
	//get the number of page 
	uint32_t pg_cnt = DIV_ROUND_UP(bitmap_bytes_len, PG_SIZE);
	
	//apply for page, return the start address  of page 
	user_prog->userprog_vaddr.vaddr_bitmap.bits = get_kernel_pages(pg_cnt);
	user_prog->userprog_vaddr.vaddr_bitmap.btmp_bytes_len = bitmap_bytes_len;

	bitmap_init(&user_prog->userprog_vaddr.vaddr_bitmap);
}


// create user process
void process_execute(void* filename , char* name) {
	// get a pcb
	struct task_struct* thread = get_kernel_pages(1);
	init_thread(thread,name,default_prio);
	create_user_vaddr_bitmap(thread);
	thread_create(thread, start_process,filename);
	thread->pgdir = create_page_dir();

	enum intr_status old_status = intr_disable();

	ASSERT(!elem_find(&thread_ready_list,&thread->general_tag));
	list_append(&thread_ready_list,&thread->general_tag);
	ASSERT(!elem_find(&thread_all_list,&thread->all_list_tag));
	list_append(&thread_all_list,&thread->all_list_tag);

	intr_set_status(old_status);
}
	
~~~

##### 进程的执行流程：

​	创建过程只是创建了进程的pcb，要执行用户进程还要包括从内核态的0特权级跳转到用户态的3特权级等等，进程的执行流程：

``  时钟中断  --> 上一个任务的时间片耗尽 -->schedule() 将当前任务改为就绪态，重新添加到就绪队列，取出next任务，这里多了激活用户进程页目录表和更新tss的esp0 -->  switch_to() 保存当前任务上下文，加载next任务的上下文  --> kernel_thread()也就是start_process()用于重新初始化intr_stack  --> intr_exit通过iretd从0特权级跳转到3特权级 --> 跳转到待执行的用户程序地址 --> 开始执行用户程序``

thread/thread.c:

schedule中的改动：多了激活用户进程页目录表和更新tss的esp0

~~~c
//task scheduling
void schedule(){

	ASSERT(intr_get_status() == INTR_OFF);

	struct task_struct* cur = running_thread();
	if (cur->status == TASK_RUNNING){
		ASSERT(!elem_find(&thread_ready_list,&cur->general_tag));
		list_append(&thread_ready_list,&cur->general_tag);
		cur->ticks=cur->priority;
		cur->status = TASK_READY;
	}else{
		//maybe block
		//don't need append to list,because current thread isn't at ready list
	}
	

	ASSERT(!list_empty(&thread_ready_list));

	struct list_elem *thread_tag;
	thread_tag = NULL;
	
	//begin to schedule next thread
	thread_tag = list_pop(&thread_ready_list);
	
	//get the PCB, like the 'running_thread'
	struct task_struct* next = (struct task_struct*)((uint32_t)thread_tag & 0xfffff000);

	next->status = TASK_RUNNING;

	//activate task's page table and tss (process, not thread)
	process_activate(next);

	switch_to(cur,next);

}
~~~

userprog/process.c:

通过假装从中断返回的方式，间接使filename_运行：

- 创建用户进程，初始化上下文，为iretd做准备：eip初始化为3特权级下的目标用户程序地址，将cs初始化为3特权级的代码段选择子，ds,es,fs,ss都初始化为3特权级的数据段选择子... ,然后直接跳转到intr_exit
- 激活页目录表：因为每个用户程序与内核线程都有独立的页目录表，所以需要更新cr3寄存器中的页目录表地址，保证分页机制正确。
- 更新tss的esp0

~~~c
//create the context of process , will invoke in 'kernel_thread'
void start_process(void* filename_) {
	void* function = filename_;
	struct task_struct* cur = running_thread();
	cur->self_kstack += sizeof(struct thread_stack);			//jump thread_stack,now self_kstack point to the top of intr_stack(lowest)
	struct intr_stack* proc_stack = (struct intr_stack*)(cur->self_kstack);
	proc_stack->edi = proc_stack->esi = proc_stack->ebp = proc_stack->esp_dummy = 0 ;
	proc_stack->ebx = proc_stack->edx = proc_stack->ecx = proc_stack->eax = 0;
	proc_stack->gs = 0 ;						// user mode don't use
	proc_stack->fs = proc_stack->es = proc_stack->ds = SELECTOR_U_DATA;
	proc_stack->eip = function;					// address of target process
	proc_stack->cs = SELECTOR_U_CODE;
	proc_stack->eflags = ( EFLAGS_IOPL_0 | EFLAGS_IF_1  |  EFLAGS_MBS );	
	proc_stack->esp = (void*) ((uint32_t)get_a_page(PF_USER,USER_STACK3_VADDR )+PG_SIZE);
	proc_stack->ss = SELECTOR_U_DATA;
	asm volatile ("movl %0 , %%esp ; jmp intr_exit" : : "g" (proc_stack) : "memory" );
}

//activate page table
void page_dir_activate(struct task_struct* p_thread){
	uint32_t pagedir_phy_addr = 0x100000;
	if (p_thread->pgdir != NULL ){
		pagedir_phy_addr = addr_v2p((uint32_t)p_thread->pgdir);
	}
	//update cr3
	asm volatile(" movl %0 , %%cr3" : : "r" (pagedir_phy_addr) : "memory");
}

//activate process :
// 1. activate page table;
// 2. update esp0 of tss to user process's 0 level stack
void process_activate(struct task_struct* p_thread){
	ASSERT(p_thread != NULL);
	
	//first step;
	page_dir_activate(p_thread);

	//second step:
	if (p_thread->pgdir){
		update_tss_esp(p_thread);
	}
}
~~~

### 4.总结

***进程如磅礴大河,浩荡奔腾不息***



*** 线程如清泉细流,曲折穿行灵动***

