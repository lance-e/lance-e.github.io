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


