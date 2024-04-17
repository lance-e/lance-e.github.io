# Hit_os_lab3


<!--more-->

# hit os lab3

### 1.实验内容

进程从创建（Linux 下调用 fork()）到结束的整个过程就是进程的生命期，进程在其生命期中的运行轨迹实际上就表现为进程状态的多次切换，如进程创建以后会成为就绪态；当该进程被调度以后会切换到运行态；在运行的过程中如果启动了一个文件读写操作，操作系统会将该进程切换到阻塞态（等待态）从而让出 CPU；当文件读写完毕以后，操作系统会在将其切换成就绪态，等待进程调度算法来调度该进程执行……

本次实验包括如下内容：

- 基于模板 `process.c` 编写多进程的样本程序，实现如下功能： + 所有子进程都并行运行，每个子进程的实际运行时间一般不超过 30 秒； + 父进程向标准输出打印所有子进程的 id，并在所有子进程都退出后才退出；
- 在 `Linux0.11` 上实现进程运行轨迹的跟踪。 + 基本任务是在内核中维护一个日志文件 `/var/process.log`，把从操作系统启动到系统关机过程中所有进程的运行轨迹都记录在这一 log 文件中。
- 在修改过的 0.11 上运行样本程序，通过分析 log 文件，统计该程序建立的所有进程的等待时间、完成时间（周转时间）和运行时间，然后计算平均等待时间，平均完成时间和吞吐量。可以自己编写统计程序，也可以使用 python 脚本程序—— `stat_log.py`（在 `/home/teacher/` 目录下） ——进行统计。
- 修改 0.11 进程调度的时间片，然后再运行同样的样本程序，统计同样的时间数据，和原有的情况对比，体会不同时间片带来的差异。

`/var/process.log` 文件的格式必须为：

```txt
pid    X    time
```

其中：

- pid 是进程的 ID；
- X 可以是 N、J、R、W 和 E 中的任意一个，分别表示进程新建(N)、进入就绪态(J)、进入运行态(R)、进入阻塞态(W) 和退出(E)；
- time 表示 X 发生的时间。这个时间不是物理时间，而是系统的滴答时间(tick)；

三个字段之间用制表符分隔。例如：

```txt
12    N    1056
12    J    1057
4    W    1057
12    R    1057
13    N    1058
13    J    1059
14    N    1059
14    J    1060
15    N    1060
15    J    1061
12    W    1061
15    R    1061
15    J    1076
14    R    1076
14    E    1076
......
```

完成实验后，在实验报告中回答如下问题：

- 结合自己的体会，谈谈从程序设计者的角度看，单进程编程和多进程编程最大的区别是什么？
- 你是如何修改时间片的？仅针对样本程序建立的进程，在修改时间片前后，log 文件的统计结果（不包括 Graphic）都是什么样？结合你的修改分析一下为什么会这样变化，或者为什么没变化？

### 2.实验开始

##### 1.编写process.c

~~~c
...
#include <sys/types.h>
#include <sys/wait.h>
int main(){
  pid_t id1;
  pid_t id2;
  pid_t id3;
  id1 = fork();
  if (id1 < 0 ){
    printf("error");
  }else if (id1 == 0 ){
    printf("child 1 \n");
    cpuio_bound(5, 0, 4);
  }
  id2 = fork();
  if (id2 < 0 ){
    printf("error");
  }else if (id2 == 0 ){
    printf("child 2 \n");
    cpuio_bound(5, 2, 2);
  }
  id3 = fork();
  if (id3 < 0 ){
    printf("error");
  }else if (id3 == 0 ){
    printf("child 3 \n");
    cpuio_bound(5, 4, 0);
  }
  printf("process id : %d\n",getpid());
  printf("child1 :%d\n",id1);
  printf("child2 :%d\n",id2);
  printf("child3 :%d\n",id3);
  wait(NULL);
  wait(NULL);
  wait(NULL);
  return 0;
}
...
~~~

##### 2.打开log文件

修改init/main.c,使操作系统启动时就能打开process.log文件,且已经加载文件系统和对应文件描述符(stdin,stdout,stderr)

将init函数中的这部分移动到main函数中

~~~c
	setup((void *) &drive_info);					//加载文件系统
	(void) open("/dev/tty0",O_RDWR,0);		//打开/dev/tty0,让文件描述符0与之关联
	(void) dup(0);												//让文件描述符1与之关联
	(void) dup(0);												//让文件描述符2与之关联
~~~

像这样,再加上打开process.log文件的函数

~~~c
	move_to_user_mode();
	setup((void *) &drive_info);
	(void) open("/dev/tty0",O_RDWR,0);
	(void) dup(0);
	(void) dup(0);
	(void) open("/var/process.log",O_CREAT|O_TRUNC|O_WRONLY,0666);
	if (!fork()) {		/* we count on this going ok */
		init();
	}
~~~

##### 3.写log文件

根据实验手册指示，内核中无法使用write函数，考虑难度，手册直接给出了内核中可以使用的源码，放入到kernel/printk.c中：

~~~c
#include "linux/sched.h"
#include "sys/stat.h"

static char logbuf[1024];
int fprintk(int fd, const char *fmt, ...)
{
    va_list args;
    int count;
    struct file * file;
    struct m_inode * inode;

    va_start(args, fmt);
    count=vsprintf(logbuf, fmt, args);
    va_end(args);
/* 如果输出到stdout或stderr，直接调用sys_write即可 */
    if (fd < 3)
    {
        __asm__("push %%fs\n\t"
            "push %%ds\n\t"
            "pop %%fs\n\t"
            "pushl %0\n\t"
        /* 注意对于Windows环境来说，是_logbuf,下同 */
            "pushl $logbuf\n\t"
            "pushl %1\n\t"
        /* 注意对于Windows环境来说，是_sys_write,下同 */
            "call sys_write\n\t"
            "addl $8,%%esp\n\t"
            "popl %0\n\t"
            "pop %%fs"
            ::"r" (count),"r" (fd):"ax","cx","dx");
    }
    else
/* 假定>=3的描述符都与文件关联。事实上，还存在很多其它情况，这里并没有考虑。*/
    {
    /* 从进程0的文件描述符表中得到文件句柄 */
        if (!(file=task[0]->filp[fd]))
            return 0;
        inode=file->f_inode;

        __asm__("push %%fs\n\t"
            "push %%ds\n\t"
            "pop %%fs\n\t"
            "pushl %0\n\t"
            "pushl $logbuf\n\t"
            "pushl %1\n\t"
            "pushl %2\n\t"
            "call file_write\n\t"
            "addl $12,%%esp\n\t"
            "popl %0\n\t"
            "pop %%fs"
            ::"r" (count),"r" (file),"r" (inode):"ax","cx","dx");
    }
    return count;
}
~~~

记得在kernel.h中添加该函数定义,注意之后使用该内核函数需要将kernel.h加入到文件中：

~~~c
int fprintk(int fd, const char *fmt, ...)
~~~

##### 4.添加写log的代码

主要是对于这五个状态N,J,W,R,E,统一都是

~~~c
fprintk(3,"%d\t?\t%d\n",(*p)->pid,jiffies); //针对性的改变字符串中的?,改为对应对状态字符
~~~

###### 1.修改fork.c:

将copy_process

~~~c
...
p->start_time = jiffies;		//这里进程开始运行
...
p->state = TASK_RUNNING;		//进程设置为就绪态
~~~

改为

~~~c
...
p->start_time = jiffies;		//这里进程开始运行
fprintk(3,"%d\tN\t%d\n",p->pid,jiffies);
...
p->state = TASK_RUNNING;		//进程设置为就绪态
fprintk(3,"%d\tJ\t%d\n",p->pid,jiffies);
~~~



###### 2.修改sched.c

将schedule改为

~~~c
void schedule(void)
{
	int i,next,c;
	struct task_struct ** p;

/* check alarm, wake up any interruptible tasks that have got a signal */

	for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
		if (*p) {
			if ((*p)->alarm && (*p)->alarm < jiffies) {
					(*p)->signal |= (1<<(SIGALRM-1));
					(*p)->alarm = 0;
				}
			if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&
			(*p)->state==TASK_INTERRUPTIBLE){
        	(*p)->state=TASK_RUNNING;
        	fprintk(3,"%d\tJ\t%d\n",(*p)->pid,jiffies);						//在这里添加一个就绪态的记录
      }

		}

/* this is the scheduler proper: */

	while (1) {
		c = -1;
		next = 0;
		i = NR_TASKS;
		p = &task[NR_TASKS];
		while (--i) {
			if (!*--p)
				continue;
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
				c = (*p)->counter, next = i;
		}
		if (c) break;
		for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
			if (*p)
				(*p)->counter = ((*p)->counter >> 1) +
						(*p)->priority;
	}
  //!!!这部分是新增，参考别人的博客，
  /*
 实验手册提示：schedule() 找到的 next 进程是接下来要运行的进程（注意，一定要分析清楚 next 是什么）。如果 next 恰好是当前正处于运行态的进程，swith_to(next) 也会被调用。这种情况下相当于当前进程的状态没变。
  */
  if(current->pid != task[next] ->pid)
    {
        if(current->state == TASK_RUNNING)
            fprintk(3,"%d\tJ\t%d\n",current->pid,jiffies);
        fprintk(3,"%d\tR\t%d\n",task[next]->pid,jiffies);
    }
  //!!!
	switch_to(next);
}
~~~

将syspause

~~~c
current->state = TASK_INTERRUPTIBLE;
...
~~~

改为

~~~c
current->state = TASK_INTERRUPTIBLE;
fprintk(3,"%d\tW\t%d\n",current->pid,jiffies);
...
~~~

将sleep_on

~~~c
...	
current->state = TASK_UNINTERRUPTIBLE;
	schedule();
	if (tmp)
		tmp->state=0;
...
~~~

改为

~~~c
...	
	current->state = TASK_UNINTERRUPTIBLE;
	fprintk(3,"%d\tW\t%d\n",current->pid,jiffies);
	schedule();
	if (tmp){
    tmp->state=0;
		fprintk(3,"%d\tJ\t%d\n",tmp->pid,jiffies);
  }
...
~~~

将interruptible_sleep_on

~~~c
...
	repeat:	current->state = TASK_INTERRUPTIBLE;
	schedule();
	if (*p && *p != current) {
		(**p).state=0;
		goto repeat;
	}
	*p=NULL;
	if (tmp)
		tmp->state=0;
..
~~~

改为

~~~c
...
	repeat:	current->state = TASK_INTERRUPTIBLE;
	fprintk(3,"%d\tW\t%d\n",current->pid,jiffies);
	schedule();
	if (*p && *p != current) {
		(**p).state=0;
    fprintk(3,"%d\tJ\t%d\n",(*p)->pid,jiffies);
		goto repeat;
	}
	*p=NULL;
	if (tmp){
    tmp->state=0;
    fprintk(3,"%d\tJ\t%d\n",tmp->pid,jiffies);
  }

..
~~~

将wake_up

~~~c
	if (p && *p) {
		(**p).state=0;
		*p=NULL;
	}
~~~

改为

~~~c
	if (p && *p) {
		(**p).state=0;
		fprintk(3,"%d\tJ\t%d\n",(*p)->pid,jiffies);
    *p=NULL;
	}
~~~



###### 3.修改exit.c

修改do_exit

~~~c
...
	current->state = TASK_ZOMBIE;
...
~~~

改为

~~~c
...
  current->state = TASK_ZOMBIE;
	fprintk(3,"%d\tE\t%d\n",current->pid,jiffies);
...
~~~

修改sys_waitpid

~~~c
...
schedule();
...
~~~

改为

~~~c
...
  fprintk(3,"%d\tW\t%d\n",current->pid,jiffies);
schedule();
...
~~~

##### 5.测试

##### 6.回答实验问题

- 结合自己的体会，谈谈从程序设计者的角度看，单进程编程和多进程编程最大的区别是什么？
  - 提高cpu利用率：单进程编程面临io密集型任务，cpu利用率接近0%,多进程编程通过cpu的调度可以更好的利用cpu
  - 数据同步：单进程只有一个进程，改变数据，会影响到该进程，多进程编程中，子进程是拷贝父进程的数据等，改变数据不会影响其他进程

### 3.总结

这个实验主要是重头在于分析fork.c,sched.c,exit.c中那些地方对进程状态做出来改变，然后在改变的时候进行对log文件的写入，这些写入的位置[参考](https://blog.csdn.net/weixin_45666853/article/details/104727107?spm=1001.2014.3001.5501)，修改时间片的部分就不做了。

