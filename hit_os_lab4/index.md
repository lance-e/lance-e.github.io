# Hit_os_lab4


<!--more-->

# hit os lab4

### 1.实验内容

现在的 Linux 0.11 采用 TSS（后面会有详细论述）和一条指令就能完成任务切换，虽然简单，但这指令的执行时间却很长，在实现任务切换时大概需要 200 多个时钟周期。

而通过堆栈实现任务切换可能要更快，而且采用堆栈的切换还可以使用指令流水的并行优化技术，同时又使得 CPU 的设计变得简单。所以无论是 Linux 还是 Windows，进程/线程的切换都没有使用 Intel 提供的这种 TSS 切换手段，而都是通过堆栈实现的。

本次实践项目就是将 Linux 0.11 中采用的 TSS 切换部分去掉，取而代之的是基于堆栈的切换程序。具体的说，就是将 Linux 0.11 中的 `switch_to` 实现去掉，写成一段基于堆栈切换的代码。

本次实验包括如下内容：

- 编写汇编程序 `switch_to`：
- 完成主体框架；
- 在主体框架下依次完成 PCB 切换、内核栈切换、LDT 切换等；
- 修改 `fork()`，由于是基于内核栈的切换，所以进程需要创建出能完成内核栈切换的样子。
- 修改 PCB，即 `task_struct` 结构，增加相应的内容域，同时处理由于修改了 task_struct 所造成的影响。
- 用修改后的 Linux 0.11 仍然可以启动、可以正常使用。
- （选做）分析实验 3 的日志体会修改前后系统运行的差别。

### 2.实验原理

##### 内核线程switch_to五段论

- ##### 中断进入阶段

- ##### 调用schedule引起PCB切换

- ##### 切换内核栈

- ##### 中断返回前

- ##### 中断返回

##### TSS切换

- 根据TR寄存器中的段描述符在GDT表中，找到当前进程的TSS段
- 将CPU中所有的寄存器信息保存到该内存地址
- 再根据目标进程的段描述符，在GDT表中找到目标进程的TSS段，再将该段信息“扣”到CPU上，就完成了进程切换

switch_to(linux/sched.h):

~~~c
#define switch_to(n) {\
struct {long a,b;} __tmp; \
__asm__("cmpl %%ecx,current\n\t" \
	"je 1f\n\t" \
	"movw %%dx,%1\n\t" \
	"xchgl %%ecx,current\n\t" \
	"ljmp *%0\n\t" \											//实际上switch_to就是一句ljmp指令
	"cmpl %%ecx,last_task_used_math\n\t" \
	"jne 1f\n\t" \
	"clts\n" \
	"1:" \
	::"m" (*&__tmp.a),"m" (*&__tmp.b), \
	"d" (_TSS(n)),"c" ((long) task[n])); \
}

#define _TSS(n) ((((unsigned long) n)<<4)+(FIRST_TSS_ENTRY<<3))
#define FIRST_TSS_ENTRY 4
~~~

### 3.实验开始

##### 1.修改switch_to相关

​	因为切换进程不再需要TSS进行切换了，而是采用内核栈切换的方式来进行切换，所以在新的switch_to需要当前进程的PCB，目标进程的PCB，当前进程的内核栈，目标进程的内核栈等信息。 内核栈信息与PCB位于同一页内存中，所以当知道PCB就可以获得内核栈信息，而当前进程的PCB位于全局变量current中，所以仅需要传递给switch_to目标进程的PCB指针。虽然不需要TSS(next)了，但是还需要_LDT(next)，所以还要传递一个_LDT(next)，每个进程都会有一个LDT，LDT是一个映射表，防止符号地址错乱。

对sched.c作出修改：

~~~c
void schedule(void)
{
	int i,next,c;
	struct task_struct ** p;
	...

	while (1) {
		...
		while (--i) {
			if (!*--p)
				continue;
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
				c = (*p)->counter, next = i;
		}
		...
	}
	switch_to(next);
}
~~~

改为

~~~c
sturct tss_struct * tss = &(init_task.task.tss)
//0 号进程的 tss，所有进程都共用这个 tss，任务切换时不再发生变化。
  ...
extern void switch_to(struct task_struct*,int );
void schedule(void)
{
	int i,next,c;
	struct task_struct ** p;
	struct task_struct * pnext = &(init_task.task); //表示下一个进程的相关信息

	while (1) {
		c = -1;
		next = 0;
		while (--i) {
			if (!*--p)
				continue;
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
				c = (*p)->counter, next = i,pnext=*p;
		}
		
	}
	switch_to(pnext,_LDT(next));
}
~~~

switch_to依次主要完成如下功能：

- 先处理栈帧，即处理`ebp`寄存器

- 取出下一个进程参数，并和`current`比较，相等就什么也不做，不等就开始进程切换

- 进程开始切换：

  - `PCB`切换

  - `TSS`内核栈指针重写

  - 内核栈切换

  - LDT切换

  - PC指针(CS:EIP)切换



syscall.s中修改：

~~~assembly
...
esp0 = 4 				！是因为TSS内核栈指针esp0偏移为4
kernel_stack = 12
signal = 16				！这些也要修改偏移量
sigaction = 20
blocked = (37*16)
...

.globl system_call,sys_fork,timer_interrupt,sys_execve
.globl hd_interrupt,floppy_interrupt,parallel_interrupt
.globl device_not_available, coprocessor_error
.globl switch_to,first_return_from_kernel
...
switch_to:
    pushl %ebp
    movl %esp,%ebp
    pushl %ecx
    pushl %ebx
    pushl %eax
    movl 8(%ebp),%ebx				!取出下一个进程PCB的参数,指的是传入的pnext参数
    cmpl %ebx,current				!与current当前进程做比较
    je 1f
! 切换PCB
		movl %ebx,%eax			!ebx是从参数中传入的目标进程的PCB指针
		xchgl %eax,current
! TSS中的内核栈指针的重写
	movl tss,%ecx
	addl $4096,%ebx
	movl %ebx,ESP0(%ecx)
! 切换内核栈
	movl %esp,KERNEL_STACK(%eax)    ! 再取一下 ebx，因为前面修改过 ebx 的值
	movl 8(%ebp),%ebx
	movl KERNEL_STACK(%ebx),%esp
! 切换LDT
    movl 12(%ebp),%ecx			！取出LDT(next)对应参数
    lldt %cx								！负责修改LDTR寄存器
    
    movl $0x17,%ecx					!这两句是重新取段寄存器fs值，非常重要
    mov %cx,%fs
! 和后面的 clts 配合来处理协处理器，由于和主题关系不大，此处不做论述
    cmpl %eax,last_task_used_math
    jne 1f
    clts

1:  popl %eax
    popl %ebx
    popl %ecx
    popl %ebp
ret

first_return_from_kernel:	
		popl %edx
		popl %edi
		popl %esi
		pop %gs
		pop %fs
	  pop %es
		pop %ds
		iret
~~~

这里引用实验手册原话：

​	关于 PC 的切换，和前面论述的一致，依靠的就是 `switch_to` 的最后一句指令 ret，虽然简单，但背后发生的事却很多：`schedule()` 函数的最后调用了这个 `switch_to` 函数，所以这句指令 ret 就返回到下一个进程（目标进程）的 `schedule()` 函数的末尾，遇到的是}，继续 ret 回到调用的 `schedule()` 地方，是在中断处理中调用的，所以回到了中断处理中，就到了中断返回的地址，再调用 iret 就到了目标进程的用户态程序去执行，和书中论述的内核态线程切换的五段论是完全一致的。

​	first_return_from_kernel要完成什么工作？PCB 切换完成、内核栈切换完成、LDT 切换完成，接下来应该那个“内核级线程切换五段论”中的最后一段切换了，即完成用户栈和用户代码的切换，依靠的核心指令就是 iret，当然在切换之前应该回复一下执行现场，主要就是eax,ebx,ecx,edx,esi,edi,gs,fs,es,ds等寄存器的恢复

然后是修改sched.h:

~~~c
struct task_struct {
/* these are hardcoded - don't touch */
	long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	long counter;
	long priority;
  long kernelstack					//在这里新增，所以signal，sigactioin,blocked在刚刚的syscall.s中也要修改
	long signal;
	struct sigaction sigaction[32];
	long blocked;	/* bitmap of masked signals */
/* various fields */
	
};

...
#define INIT_TASK \
/* state etc */	{ 0,15,15,PAGE_SIZE+(long)&init_task, \					//为新增的kernelstack初始化
/* signals */	0,{{},},0, \
  ...
}

...
//删除掉原来的switch_to
~~~

##### 2.修改fork相关

就是将进程的用户栈，内核栈，用户程序通过压入内核栈中的ss:esp,cs:eip相关联

~~~c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
{
	struct task_struct *p;
	int i;
	struct file *f;

	p = (struct task_struct *) get_free_page();
	if (!p)
		return -EAGAIN;
	task[nr] = p;
	*p = *current;	/* NOTE! this doesn't copy the supervisor stack */
	p->state = TASK_UNINTERRUPTIBLE;
	p->pid = last_pid;
	p->father = current->pid;
	p->counter = p->priority;
	p->signal = 0;
	p->alarm = 0;
	p->leader = 0;		/* process leadership doesn't inherit */
	p->utime = p->stime = 0;
	p->cutime = p->cstime = 0;
	p->start_time = jiffies;
	//tss相关部分全部删除
  
  krnstack = (long *) (PAGE_SIZE + (long) p);
  *(--krnstack) = ss & 0xffff;
	*(--krnstack) = esp;
	*(--krnstack) = eflags;
	*(--krnstack) = cs & 0xffff;
	*(--krnstack) = eip;
  
  *(--krnstack) = ds & 0xffff;
 	*(--krnstack) = es & 0xffff;
 	*(--krnstack) = fs & 0xffff;
 	*(--krnstack) = gs & 0xffff;
 	*(--krnstack) = esi;
 	*(--krnstack) = edi;
 	*(--krnstack) = edx;
	*(--krnstack) = (long) first_return_from_kernel;
 	//初始化地址，switch_to的ret就会跳转到first_return_form_kernel的地方
 

  //为了完成switch_to最后的弹栈
  *(--krnstack) = ebp;
	*(--krnstack) = ecx;
	*(--krnstack) = ebx;
	// 这里的 0 最有意思。
	*(--krnstack) = 0;
  
  p->kernelstack = stack;
  
	if (last_task_used_math == current)
		__asm__("clts ; fnsave %0"::"m" (p->tss.i387));
	if (copy_mem(nr,p)) {
		task[nr] = NULL;
		free_page((long) p);
		return -EAGAIN;
	}
	for (i=0; i<NR_OPEN;i++)
		if ((f=p->filp[i]))
			f->f_count++;
	if (current->pwd)
		current->pwd->i_count++;
	if (current->root)
		current->root->i_count++;
	if (current->executable)
		current->executable->i_count++;
	set_tss_desc(gdt+(nr<<1)+FIRST_TSS_ENTRY,&(p->tss));
	set_ldt_desc(gdt+(nr<<1)+FIRST_LDT_ENTRY,&(p->ldt));
	p->state = TASK_RUNNING;	/* do this last, just in case */
	return last_pid;
}
~~~

fork会调用first_return_form_kernel,所以还需要添加：

~~~c
extern void first_return_from_kernel(void);
~~~

### 4.回答问题

回答下面三个题：

#### 问题 1

针对下面的代码片段：

```
movl tss,%ecx
addl $4096,%ebx
movl %ebx,ESP0(%ecx)
```

回答问题：

- （1）为什么要加 4096；
  - 参考答案：因为ebx为下一个进程的task_struct指针，内核栈栈顶需要设置为与该进程task_struct相同物理页页顶，一页为4k(4096);

- （2）为什么没有设置 tss 中的 ss0。
  - 参考答案：所有进程的SS0都是0x10，切换前后不改变，所以不需要设置


#### 问题 2

针对代码片段：

```c
*(--krnstack) = ebp;
*(--krnstack) = ecx;
*(--krnstack) = ebx;
*(--krnstack) = 0;
```

回答问题：

- （1）子进程第一次执行时，eax=？为什么要等于这个数？哪里的工作让 eax 等于这样一个数？
  - 答：eax=0，为了区分子进程和父进程的区别，注意修改后的fork.c中`*(--krnstack) = 0;`，这里将eax赋值为0。
- （2）这段代码中的 ebx 和 ecx 来自哪里，是什么含义，为什么要通过这些代码将其写到子进程的内核栈中？
  - 答：来自父进程在调用fork系统调用时传递进来的参数，即是通过system_call压入了内核栈，为了使子进程完全拷贝父进程的信息。通过在fork函数中写入到子进程的内核栈中，顺序不能发生变化，因为在进程切换时，由switch_to等函数弹出内核栈，必须与pop的顺序严格对应。
- （3）这段代码中的 ebp 来自哪里，是什么含义，为什么要做这样的设置？可以不设置吗？为什么？
  - 答：来自于父进程调用fork系统调用传递进来的参数，是父进程fork函数栈帧的基指针。因为在switch_to中设置了ebp的pop，所以必须设置，如果switch_to不pop，就可以不设置。

#### 问题 3

为什么要在切换完 LDT 之后要重新设置 fs=0x17？而且为什么重设操作要出现在切换完 LDT 之后，出现在 LDT 之前又会怎么样？

难懂，跳过，

### 5.总结

参考博客：[哈工大操作系统实验四——基于内核栈切换的进程切换（极其详细）](https://blog.csdn.net/lyj1597374034/article/details/111033682?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522162246292016780255217157%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=162246292016780255217157&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_v2~rank_v29-6-111033682.first_rank_v2_pc_rank_v29&utm_term=%E5%9F%BA%E4%BA%8E%E5%86%85%E6%A0%B8%E6%A0%88%E7%9A%84%E8%BF%9B%E7%A8%8B%E5%88%87%E6%8D%A2&spm=1018.2226.3001.4187)

这个lab是我目前来说做过最难的一个lab，基本上都是参考其他优秀博客，很多晦涩难懂的概念以及过程。对于基于内核栈切换线程的基本流程了解了个大概，应该会在之后写小内核会再深入理解一下。


