# 手写OS系列之实现内核级线程


<!--more-->

# 手写OS系列之实现内核级线程

### 1.相关概念

##### 什么是执行流？

​	执行流就是一段逻辑上**独立**的指令区域，是人为给处理器安排的**处理单元(调度单元)**。

​	执行流可以大到是一整个程序文件(进程(单线程进程))，也可以小到是一个函数(线程，本质上就是函数)。

​	执行流是独立的，独立性体现在拥有自己的一套栈，寄存器映像和内存资源，也就是执行流的上下文环境。

##### 什么是线程？

​	线程是一套机制，为一般的代码块创造它依赖的上下文环境，从而使代码块具有独立性。让代码块能成为处理器的调度单元，处理器可以单独地，专门地执行该函数。

##### 进程和线程？

​	*进程 = 线程 + 资源* 

​	*线程 = 具有能动性，执行力，独立的代码块* 

​	

​	程序是指静态的，存储在文件系统上，未运行的指令代码，是实际运行时程序的映像。

​	进程是指正在运行的程序，即进行中的程序。(程序需要获取到运行中需要的资源(栈，寄存器等)，才能成为进程)。进程是一种控制流集合，至少包含一个执行流，执行流之间相互独立，共享线程的资源，(就是线程)。

​	进程拥有整个地址空间，从而拥有全部资源；线程没有自己独享的地址空间，进程中的所有线程共享同一个地址空间。**(说白了就是进程有页表，线程没有)**

​	显式创建线程时，执行流指的是粒度更小的线程，线程就是最小执行单元；未显式创建线程时，进程就是单线程进程，执行流便指的是该进程。进程和线程都有自己独立的栈空间和寄存器资源。

​	分页机制保证了进程之间是无法访问对方的资源的，但是进程中的线程是共享进程的地址空间的，所以会导致资源竞争的问题。



##### 线程是如何给进程提速的？

​	线程可以使函数单独地被处理器调度，本质上就是让处理器多执行进程中的代码达到提速的效果。另外线程还可以避免阻塞整个进程，也可以达到提速效果。

##### 什么是PCB？

​	PCB(process control block)，程序控制块：用来记录进程相关信息。通常以页为单位。

##### 用户级线程和内核级线程区别？

​	用户级线程则是操作系统不提供线程机制，由用户进程来实现线程的调度。内核级线程就是操作系统原生提供了线程机制。

​	用户级线程优缺点：

- 优点：
  - 调度算法由用户提供，可以针对某些场景定制，加权调度，自由度高
  - 将线程的寄存器映像装载到cpu时，无须陷入内核
- 缺点：
  - 单个线程阻塞，会阻塞整个进程(操作系统未实现线程机制，调度器的调度单元未进程，是无法感知到用户级线程的)。
  - 整个进程占用cpu的时间片仍然是有限的，再加上进程内线程调度器维护线程表，调度算法等耗时。

​	内核级线程优缺点：

- 优点：
  - 相较于用户级线程，内核级线程让进程多占用了cpu资源，达到了提速效果。
  - 单个线程阻塞了，操作系统可以调度到其他线程，达到了提速效果(操作系统原生提供线程进制，能感知到内核级线程)。
- 缺点：
  - 需要进入内核，增加了现场保护的栈操作。

### 2.实现内核级线程

##### 1.PCB和线程栈实现：

thread/thread.h:

线程状态：

~~~c
//status of process or thread
enum task_status{
	TASK_RUNNING,
	TASK_READY,
	TASK_BLOCKED,
	TASK_WAITING,
	TASK_HANGING,
	TASK_DIED
};
~~~



中断栈实现：

~~~c
/***************************	intr_stack	************************
 * used to save the context of process or thread when interrrupt happen
 * this stack is at the top of page
 * ********************************************************************/
struct intr_stack{
	uint32_t vec_no;				//kernel.S: VECTOR has push %1,the number of interrupt
	uint32_t edi;
	uint32_t esi;
	uint32_t ebp;
	uint32_t esp_dummy;
	uint32_t ebx;
	uint32_t edx;
	uint32_t ecx;
	uint32_t eax;
	uint32_t gs;
	uint32_t fs;
	uint32_t es;
	uint32_t ds;

	//there will push when cpu enter high privilege level from low privilege level
	uint32_t error_code;
	void (*eip) (void);
	uint32_t cs;
	uint32_t eflags;
	void* esp;
	uint32_t ss;
};
~~~

线程栈实现：

~~~c
/*****************************	thread_stack	********************************
 * stack of thread ,used to save the function which will execute in this thread
 * just for used in switch_to to save the environment of thread.
 * ****************************************************************************/
struct thread_stack{
	uint32_t ebp;
	uint32_t ebx;
	uint32_t edi;
	uint32_t esi;

	//when frist excute, eip point to the target function
	//other time , eip point to the return address of switch_to 
	void (*eip) (thread_func* func , void* func_arg);
	
	//there are used for first dispatched to cpu
	void (*unused_retaddr);			//unused_retaddr just for occupy a position  in stack
	thread_func* function;			//function name
	void* func_arg;				//function arguments
};

~~~

PCB实现：

**self_kstack就是内核栈栈顶指针，线程被创建时，初始化为PCB所在页的顶端 ** 。



**在线程被换下处理器时，线程的上下文信息会保存到0特权级栈中，此时self_kstack就保存新的0特权级栈顶；**



**当线程被换上处理器时，将self_kstack的值载入esp，就可以从0特权级栈中获取到线程的上下文信息，从而可以加载到处理器中执行**

~~~c
//the PCB 
struct task_struct {
	uint32_t self_kstack;
	enum task_status status;
	char name[16];
	uint8_t priority;
	uint8_t ticks;				//the time ticks of run in cpu

	uint32_t elapsed_ticks;			//record the all ticks after run in cpu

	struct list_elem general_tag;		//the node of general list

	struct list_elem all_list_tag;		//the node of thread list
	
	uint32_t* pgdir;				//the virtual address of process's page table,!:thread:NULL(thread don't have page table)

	uint32_t stack_magic;			//edge of stack
};

~~~

##### 2.线程的实现：

thread/thread.c:



~~~c
#define PG_SIZE 4096


//get the current thread's PCB pointer
struct task_struct* running_thread(){
	uint32_t esp;
	asm ("mov %%esp,%0":"=g"(esp));
	return (struct task_struct*)(esp & 0xfffff000);
}



//kernel_thread is to execute function(func_arg)
static void kernel_thread(thread_func* function ,void* func_arg){
	intr_enable();
	function(func_arg);
}


//thread_create :
//initial thread stack ,set the target function which to be executed  and some arguments
void thread_create(struct task_struct* pthread,thread_func function,void* func_arg){
	//reserve the space of intr_stack in kernel stack
	pthread->self_kstack -= sizeof(struct intr_stack);

	//reserve the space of thread_stack in kernel stack
	pthread->self_kstack -= sizeof(struct thread_stack);

	struct thread_stack* kthread_stack = (struct thread_stack*) pthread->self_kstack;
	kthread_stack->eip = kernel_thread;
	kthread_stack->function = function;
	kthread_stack->func_arg = func_arg;
	kthread_stack->ebp = kthread_stack->ebx = kthread_stack->esi = kthread_stack->edi = 0 ;
}

//initial the basic information of thread
void init_thread(struct task_struct* pthread,char* name,int prio){
	memset(pthread,0,sizeof(*pthread));
	strcpy(pthread->name ,name);
	if (pthread == main_thread){
		pthread->status = TASK_RUNNING;
	}else{
		pthread->status = TASK_READY;
	}
	pthread->self_kstack = (uint32_t*)((uint32_t)pthread + PG_SIZE);
	pthread->priority = prio;
	pthread->ticks = prio;
	pthread->elapsed_ticks= 0;
	pthread->pgdir= NULL;
	pthread->stack_magic = 0x88888888;			//magic number
}

//create a thread: priority is 'prio',name is 'name',target function is 'function(func_arg)'
struct task_struct* thread_start(char* name,int prio,thread_func function , void* func_arg){
	struct task_struct* thread = get_kernel_pages(1);
	init_thread(thread,name,prio);
	thread_create(thread,function,func_arg);
	
	asm volatile("movl %0,%%esp; pop %%ebp; pop %%ebx; pop %%edi; pop %%esi; ret" : : "g"(thread->self_kstack) :"memory");
	return thread;
}
~~~

##### 3.线程执行目标函数的原理^_^：

​	在我们的实现的线程中，线程执行目标函数不是靠的call来执行，我们靠的是ret指令**（为什么这里用ret指令呢？其实是为了后续多线程调度时的switch_to函数中的ret来进行线程的切换**。ret指令会从栈顶获取返回地址，然后跳转到返回地址。所以我们直接在执行ret指令时，直接在栈顶装入目标函数的地址，这里我们将kernel_thread的偏移地址(段基址=0)载入eip，到时候ret指令就会跳转到kernel_thread函数中。但在kernel_thread看来，是在调用kernel_thread。

​	正常的函数调用都是通过call指令，函数中的栈顶应该是返回地址，栈顶之上(高地址)是参数，我们调用kernel_thread函数不是通过call指令，而是通过ret指令，所以不会自动将函数的返回地址压栈，但是kernel_thread中调用``function(func_arg)``，需要kernel_thread的两个参数，这时就会到栈中的esp+4和esp+8处寻找参数，所以在线程栈中有一个``void (*unused_retaddr);``项，就是负责占返回地址那个位置。然后正确获取到参数，就跳转到function。

​	线程执行目标函数靠的就是这句内联汇编：

`` asm volatile("movl %0,%%esp; pop %%ebp; pop %%ebx; pop %%edi; pop %%esi; ret" : : "g"(thread->self_kstack) :"memory");`` 首先将内核栈栈顶加载到esp，获取到线程的上下文信息，pop掉低地址的四个寄存器信息，ret指令跳转到此时栈顶(eip)指向的kernel_thread。

### 3.实现多线程调度

#### 1.核心数据结构--双向链表：

简单，不再赘述

lib/kernel/list.h:

~~~c
#ifndef __LIB_KERNEL_LIST_H
#define __LIB_KERNEL_LIST_H
#include "global.h"



//the struct of list element 
struct list_elem{
	struct list_elem* prev;
	struct list_elem* next;
};

//the struct of list
struct list{
	//head is point to the first node, so the first node is head.next
	struct list_elem head;
	struct list_elem tail;
};


//self define function : used for callback function in 'list_traversal'
typedef bool (function)(struct list_elem* , int arg);

void list_init(struct list*);
void list_insert_before(struct list_elem* before,struct list_elem* elem);
void list_push(struct list* plist, struct list_elem* elem);
void list_iterate(struct list* plist);
void list_append(struct list* plist,struct list_elem* elem);
void list_remove(struct list_elem* pelem);
struct list_elem* list_pop(struct list* plist);
bool list_empty(struct list* plist);
uint32_t list_len(struct list* plist);
struct list_elem* list_traversal(struct list* plist,function func, int arg);
bool elem_find(struct list* plist , struct list_elem* obj_elem);

#endif
~~~

lib/kernel/list.c:

~~~c
#include "list.h"
#include "interrupt.h"


//initial list
void list_init(struct list* list){
	list->head.prev = NULL;
	list->head.next = &list->tail;
	list->tail.prev = &list->head;
	list->tail.next = NULL;
}

//insert 'elem' into the front of 'before' 
void list_insert_before(struct list_elem* before , struct list_elem* elem){
	enum intr_status old_status = intr_disable();
	before->prev->next = elem;
	elem->prev = before->prev;
	elem->next = before;
	before->prev = elem;
	intr_set_status(old_status);
}

//push to the head
void list_push(struct list* plist , struct list_elem* elem){
	list_insert_before(plist->head.next,elem);
}

//append to the tail
void list_append(struct list* plist,struct list_elem* elem){
	list_insert_before(&plist->tail,elem);
}

//remove the target element
void list_remove(struct list_elem* pelem){
	enum intr_status old_status = intr_disable();
	pelem->prev->next = pelem->next;
	pelem->next->prev = pelem->prev;
	intr_set_status(old_status);
}

//pop the first elem and return the element
struct list_elem* list_pop(struct list* plist){
	struct list_elem* elem = plist->head.next;
	list_remove(elem);
	return elem;
}


//search the target element in 'plist' 
bool elem_find(struct list* plist, struct list_elem* obj_elem){
	struct list_elem* elem = plist->head.next;
	while(elem != &plist->tail){
		if (elem == obj_elem){
			return true;
		}
		elem = elem->next;
	}
	return false;
}

//traversal all element in this list ,and judge elem is it eligible (meet a condition)
struct list_elem* list_traversal(struct list* plist, function func , int arg){
	struct list_elem* elem = plist->head.next;
	if (list_empty(plist)){
		return NULL;
	}
	while (elem != &plist->tail){
		if (func(elem,arg)){
			return elem;
		}
		elem = elem->next;
	}
	return NULL;
}



//return the length of list
uint32_t list_len(struct list* plist){
	struct list_elem* elem = plist->head.next;
	uint32_t length = 0;
	while (elem != &plist->tail){
		length ++;
		elem = elem->next;
	}
	return length;
}

//judge this list is it empty
bool list_empty(struct list* plist){
	return (plist->head.next == &plist->tail ? true : false);
}
~~~

#### 2.多线程调度

**注意我这一小节就截取重要部分，完整请看项目地址或后面的完整代码部分** 

##### a.注册时间中断函数

device/timer.c:

我们用ticks来代表时钟中断次数，也就是代表任务的时间片，我们设置的线程的优先级=线程的ticks。每一次时钟中断，使当前线程的ticks-1，当ticks=0时，就会开始线程的调度。

~~~c
//timer interrupt handler
static void intr_timer_handler(void){
	struct task_struct* cur_thread = running_thread();
	ASSERT(cur_thread->stack_magic == 0x88888888);
	++cur_thread->elapsed_ticks;		//record time of current thread  had run in cpu
	++ticks;				//from first time interrupt to now 
						
	if (cur_thread->ticks == 0){
		schedule();
	}else {
		--cur_thread->ticks;
	}
}
~~~

##### b.调度器schedule()

thread/thread.c:

采用的轮询的调度算法(FIFO),最简单^_^

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
	switch_to(cur,next);

}
~~~

##### c.任务切换函数switch_to

thread/switch_to.S:

注意最后的ret，跟上面那个ret使用方法是一样的：都是ret到kernel_thread。

~~~assembly
[bits 32]
section .text
global switch_to
switch_to:
	;save context of current thread
	;<- in stack :here is the return address
	push	esi
	push 	edi
	push	ebx
	push	ebp

	mov	eax,[esp+20]				;get the cur(current thread)
	mov	[eax],esp				;save the esp(stack of current thread)

	;resume context of next thread
	mov	eax,[esp+24]
	mov	esp,[eax]				;reload new esp(stack of next thread)
	
	pop	ebp
	pop	ebx
	pop	edi
	pop	esi
	ret						
~~~

### 4.部分文件完整代码

全部代码可以项目地址看[loong-OS](https://github.com/lance-e/loong-OS),这里我贴一下重头的一些文件：

##### thread/thread.c:

从开机到现在，都有一个主线程在执行，所以我们还要使用make_main_thread为主线程增加PCB信息。

~~~c
#include "thread.h"
#include "stdint.h"
#include "string.h"
#include "global.h"
#include "memory.h"
#include "interrupt.h"
#include "debug.h"
#include "print.h"
#include "list.h"

#define PG_SIZE 4096

struct task_struct* main_thread;			//main thread PCB
struct list thread_ready_list;				//ready queue
struct list thread_all_list;				//all thread queue node
//static struct list_elem* thread_tag;			//save the thread node in queue



extern void switch_to(struct task_struct* cur,struct task_struct* next);

//get the current thread's PCB pointer
struct task_struct* running_thread(){
	uint32_t esp;
	asm ("mov %%esp,%0":"=g"(esp));
	return (struct task_struct*)(esp & 0xfffff000);
}



//kernel_thread is to execute function(func_arg)
static void kernel_thread(thread_func* function ,void* func_arg){
	intr_enable();
	function(func_arg);
}


//thread_create :
//initial thread stack ,set the target function which to be executed  and some arguments
void thread_create(struct task_struct* pthread,thread_func function,void* func_arg){
	//reserve the space of intr_stack in kernel stack
	pthread->self_kstack -= sizeof(struct intr_stack);

	//reserve the space of thread_stack in kernel stack
	pthread->self_kstack -= sizeof(struct thread_stack);

	struct thread_stack* kthread_stack = (struct thread_stack*) pthread->self_kstack;
	kthread_stack->eip = kernel_thread;
	kthread_stack->function = function;
	kthread_stack->func_arg = func_arg;
	kthread_stack->ebp = kthread_stack->ebx = kthread_stack->esi = kthread_stack->edi = 0 ;
}

//initial the basic information of thread
void init_thread(struct task_struct* pthread,char* name,int prio){
	memset(pthread,0,sizeof(*pthread));
	strcpy(pthread->name ,name);
	if (pthread == main_thread){
		pthread->status = TASK_RUNNING;
	}else{
		pthread->status = TASK_READY;
	}
	pthread->self_kstack = (uint32_t*)((uint32_t)pthread + PG_SIZE);
	pthread->priority = prio;
	pthread->ticks = prio;
	pthread->elapsed_ticks= 0;
	pthread->pgdir= NULL;
	pthread->stack_magic = 0x88888888;			//magic number
}

//create a thread: priority is 'prio',name is 'name',target function is 'function(func_arg)'
struct task_struct* thread_start(char* name,int prio,thread_func function , void* func_arg){
	struct task_struct* thread = get_kernel_pages(1);
	init_thread(thread,name,prio);
	thread_create(thread,function,func_arg);
	

	//make sure is not in thread_ready_list 
	ASSERT(!elem_find(&thread_ready_list,&thread->general_tag));
	//append into ready queue
	list_append(&thread_ready_list,&thread->general_tag);

	//make sue is not in thread_all_list
	ASSERT(!elem_find(&thread_all_list,&thread->all_list_tag));
	//append into thread all list
	list_append(&thread_all_list,&thread->all_list_tag);

	return thread;
}


//initial main thread:
//don't need apply for address of PCB,because in loader.S had initial the esp = 0xc009f000
//so the main thread's PCB is at 0xc009e000
static void make_main_thread(void){
	main_thread = running_thread();
	init_thread(main_thread,"main",31);

	ASSERT(!elem_find(&thread_all_list,&main_thread->all_list_tag));
	list_append(&thread_all_list,&main_thread->all_list_tag);
}

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
	switch_to(cur,next);

}

//initial thread environment
void thread_init(void){
	put_str("thread_init start\n");
	list_init(&thread_ready_list);
	list_init(&thread_all_list);
	make_main_thread();
	put_str("thread_init done\n");
}
~~~

##### device/timer.c:

~~~c
#include "timer.h"
#include "io.h"
#include "print.h"
#include "interrupt.h"
#include "debug.h"
#include "thread.h"
#include "stdint.h"

#define IRQ0_FREQUENCY 100000
#define INPUT_FREQUENCY 1193180
#define COUNTER0_VALUE INPUT_FREQUENCY / IRQ0_FREQUENCY				//the initial value of counter 0 
#define COUNTER0_PORT 0x40
#define COUNTER0_NO 0 								//No.1 counter 
#define COUNTER0_MODE 2							
#define READ_WRITE_LATCH 3
#define PIT_CONTROL_PORT 0x43

uint32_t ticks;

//write the control word into the control word register and set the initial value 
static void frequency_set(uint8_t counter_port, uint8_t counter_no, uint8_t rwl, uint8_t counter_mode, uint16_t counter_value){
	//write into control word register
	outb(PIT_CONTROL_PORT,(uint8_t)(counter_no << 6 | rwl << 4 | counter_mode << 1 ));
	//write the low bit of counter_value
	outb(counter_port,(uint8_t)counter_value);
	//writhe the hight bit of counter_value
	outb(counter_port,(uint8_t)(counter_value >> 8));										//!!!!! attention: heare is diffrent from book !!!!!
}



//timer interrupt handler
static void intr_timer_handler(void){
	struct task_struct* cur_thread = running_thread();
	ASSERT(cur_thread->stack_magic == 0x88888888);
	++cur_thread->elapsed_ticks;		//record time of current thread  had run in cpu
	++ticks;				//from first time interrupt to now 
						
	if (cur_thread->ticks == 0){
		schedule();
	}else {
		--cur_thread->ticks;
	}
}


//initial PIT8253
void timer_init(){
	put_str("timer_init start \n");
	//set the frequency of counter
	frequency_set(COUNTER0_PORT,COUNTER0_NO,READ_WRITE_LATCH, COUNTER0_MODE, COUNTER0_VALUE);
	register_handler(0x20,intr_timer_handler);
	put_str("timer_init done \n");
}
~~~

##### thread/switch.S:

~~~assembly
[bits 32]
section .text
global switch_to
switch_to:
	;save context of current thread
	;<- in stack :here is the return address
	push	esi
	push 	edi
	push	ebx
	push	ebp

	mov	eax,[esp+20]				;get the cur(current thread)
	mov	[eax],esp				;save the esp(stack of current thread)

	;resume context of next thread
	mov	eax,[esp+24]
	mov	esp,[eax]				;reload new esp(stack of next thread)
	
	pop	ebp
	pop	ebx
	pop	edi
	pop	esi
	ret	
~~~

### 5.总结

这部分收获非常大，也很绕。中途也因为与书上的效果不同，不断排查，发现了书上的一个小错误：在增加时钟中断的频率那一节，少加一个括号，导致高8位一定会总是为0。重新修改之后，模拟书上之前设置时钟中断频率，我的多线程调度速度仍然是达不到书上的调度速度的，有点难崩，所以我提高了时钟中断的频率，大概较书上的频率，提高了10倍，调度速度才跟书上的调度速度差不多。

