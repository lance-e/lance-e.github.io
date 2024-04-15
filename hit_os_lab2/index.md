# Hit_os_lab2


<!--more-->

# hit os lab2 

### 1.实验内容

此次实验的基本内容是：在 Linux 0.11 上添加两个系统调用，并编写两个简单的应用程序测试它们。

#### （1）`iam()`

第一个系统调用是 iam()，其原型为：

```c
int iam(const char * name);
```

完成的功能是将字符串参数 `name` 的内容拷贝到内核中保存下来。要求 `name` 的长度不能超过 23 个字符。返回值是拷贝的字符数。如果 `name` 的字符个数超过了 23，则返回 “-1”，并置 errno 为 EINVAL。

在 `kernal/who.c` 中实现此系统调用。

#### （2）`whoami()`

第二个系统调用是 whoami()，其原型为：

```c
int whoami(char* name, unsigned int size);
```

它将内核中由 `iam()` 保存的名字拷贝到 name 指向的用户地址空间中，同时确保不会对 `name` 越界访存（`name` 的大小由 `size` 说明）。返回值是拷贝的字符数。如果 `size` 小于需要的空间，则返回“-1”，并置 errno 为 EINVAL。

也是在 `kernal/who.c` 中实现。

#### （3）测试程序

运行添加过新系统调用的 Linux 0.11，在其环境下编写两个测试程序 iam.c 和 whoami.c。最终的运行结果是：

```bash
$ ./iam lizhijun

$ ./whoami

lizhijun
```

#### （4）在实验报告中回答如下问题：

- 从 Linux 0.11 现在的机制看，它的系统调用最多能传递几个参数？你能想出办法来扩大这个限制吗？
- 用文字简要描述向 Linux 0.11 添加一个系统调用 foo() 的步骤。

### 2.系统调用原理

- 系统调用流程：
  - 应用程序调用库函数（API）
  - API把系统调用号传给EAX，中断调用进入内核态
  - 中断处理函数通过系统调用号，调用对应内核函数（系统调用）
  - 系统调用结束后，将返回值存入EAX，返回到中断处理函数
  - 中断处理函数返回到API
  - API将EAX返回给应用程序

##### 1.以write为例来分析系统调用实现：

write.c:

~~~c
#define __LIBRARY__
#include <unistd.h>

_syscall3(int,write,int,fd,const char *,buf,off_t,count)

~~~

unistd.h(截取关于write):

~~~c
#define __NR_write	4


#define _syscall3(type,name,atype,a,btype,b,ctype,c) \
type name(atype a,btype b,ctype c) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
	: "=a" (__res) \
	: "0" (__NR_##name),"b" ((long)(a)),"c" ((long)(b)),"d" ((long)(c))); \
if (__res>=0) \
	return (type) __res; \
errno=-__res; \
return -1; \
}

int write(int fildes, const char * buf, off_t count);
~~~

可以看到write是通过宏来展开成一段包含int中断的汇编代码，在这个宏中：type对应int，name对应write，atype对应int，a对应fd，btype对应const char * ，b对应buf，ctype对应off_t，c对应count。当宏展开后与下面这句代码向对应：

~~~c
int write(int fildes, const char * buf, off_t count);
~~~

在宏中，定义了返回值_res,然后是一段内嵌汇编，进行int 0x80中断调用。下一句是将EAX赋给__res，也就是通过将返回值放在EAX中进行返回。下一句的0指的依旧是EAX，name对应write，合起来就是 _ _ NR_write ，也就是把write的系统调用号4置给EAX。之后的就是将另外三个参数分别传给EBX，ECX，EDX，这三个传入参数也对应着syscall3的3。

##### 2.在内核中的write函数

是通过int 0x80进行中断处理函数，通过系统调用号来调用这个函数

read_write.c:

~~~c
int sys_write(unsigned int fd,char * buf,int count)
{
	struct file * file;
	struct m_inode * inode;
	
	if (fd>=NR_OPEN || count <0 || !(file=current->filp[fd]))
		return -EINVAL;
	if (!count)
		return 0;
	inode=file->f_inode;
	if (inode->i_pipe)
		return (file->f_mode&2)?write_pipe(inode,buf,count):-EIO;
	if (S_ISCHR(inode->i_mode))
		return rw_char(WRITE,inode->i_zone[0],buf,count,&file->f_pos);
	if (S_ISBLK(inode->i_mode))
		return block_write(inode->i_zone[0],&file->f_pos,buf,count);
	if (S_ISREG(inode->i_mode))
		return file_write(inode,file,buf,count);
	printk("(Write)inode->i_mode=%06o\n\r",inode->i_mode);
	return -EINVAL;
}
~~~



##### 3.让我们看看0x80中断的处理：

这个函数就是在操作系统启动的时候就初始化好了，也就是设置好了0x80的中断处理

~~~c
void sched_init(void)
{
	//省略无关部分，这里可以看到int 0x80是执行system_call中断处理函数
	set_system_gate(0x80,&system_call);
}
~~~

下面这段代码就是设置好idt表，将0x80的中断号对应system_call中断处理函数的地址。同时这里也可以看到dpl被设置为了3

~~~assembly
#define set_system_gate(n,addr) \
	_set_gate(&idt[n],15,3,addr)

#define _set_gate(gate_addr,type,dpl,addr) \
__asm__ ("movw %%dx,%%ax\n\t" \
	"movw %0,%%dx\n\t" \
	"movl %%eax,%1\n\t" \
	"movl %%edx,%2" \
	: \
	: "i" ((short) (0x8000+(dpl<<13)+(type<<8))), \
	"o" (*((char *) (gate_addr))), \
	"o" (*(4+(char *) (gate_addr))), \
	"d" ((char *) (addr)),"a" (0x00080000))
~~~

下面就是int 0x80的中断处理函数

~~~assembly
system_call:
	cmpl $nr_system_calls-1,%eax
	ja bad_sys_call
	push %ds
	push %es
	push %fs
	pushl %edx
	pushl %ecx		# push %ebx,%ecx,%edx as parameters
	pushl %ebx		# to the system call
	movl $0x10,%edx		# set up ds,es to kernel space
	mov %dx,%ds
	mov %dx,%es
	movl $0x17,%edx		# fs points to local data space
	mov %dx,%fs
	call sys_call_table(,%eax,4)
	pushl %eax
	movl current,%eax
	cmpl $0,state(%eax)		# state
	jne reschedule
	cmpl $0,counter(%eax)		# counter
	je reschedule

~~~

着重看一下 **call sys_call_table(,%eax,4) ** 这句意思就是跳转到sys_call_table的地址+EAX*4，而EAX的值也就是通过最最开始的库函数传递给EAX的系统调用号。为什么是4呢，因为每个系统调用的函数占四个字节，因此可以知道sys_call_table是一个函数表。 

那么我们接着看看这个sys_call_table：

~~~c
//sched.h
typedef int (*fn_ptr)();

//sys.h
fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
sys_write,... };
~~~

与前面write调用号为4，完全对应。



### 3.实验开始

#####  1.添加iam和whoami系统调用号和库函数

unistd.h

**坑：需要修改虚拟机的unistd.h相同的地方，否则后面的gcc无法成功编译文件**

~~~c
#define __NR_iam 72
#define __NR_whoami 73

int iam(const char * name);
int whoami(char* name ,unsigned int size);
~~~

##### 2.在sys_call_table中添加系统调用函数

sys.h

~~~c
extern int sys_iam();
extern int sys_whoami();

fn_ptr sys_call_table[] = { ... ,sys_iam,sys_whoami};
~~~

##### 3.修改系统调用函数数量

system_call.s

~~~assembly
nr_system_calls = 74
~~~



##### 4.编写系统调用函数

kernel/who.c

~~~c
#include<errno.h> //错误代码
#include<string.h> //字符串操作
#include<asm/segment.h> //用于提供get_fs_byte和put_fs_byte
char data[24];
int sys_iam(const char* name){
  int i = 0 ;
  char temp[26];
  for (;i<26;i++){
    temp[i] = get_fs_byte(name+i);
    if (temp[i] == '\0') break;
  }
  if (i > size) return -(EINVAL);
  strcpy(data,temp);
  return i;
}
int sys_whoami(char * name,unsigned int size){
  int i = 0 ;
  while (data[i] != '\0') i++;
  if (i >size) return -(EINVAL);
  i = 0;
  for (;i<size;i++){
    put_fs_byte(data[i],name+i);
    if (data[i] == '\0') break;
  }
  return i;
}
~~~

##### 5.修改makefile

参考实验手册

- 第一处

~~~makefile
OBJS  = sched.o system_call.o traps.o asm.o fork.o \
        panic.o printk.o vsprintf.o sys.o exit.o \
        signal.o mktime.o
~~~

改为

~~~makefile
OBJS  = sched.o system_call.o traps.o asm.o fork.o \
        panic.o printk.o vsprintf.o sys.o exit.o \
        signal.o mktime.o who.o
~~~

- 第二处

~~~makefile
### Dependencies:
exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
  ../include/asm/segment.h
~~~

改成

~~~makefile
### Dependencies:
who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h
exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
  ../include/asm/segment.h
~~~

##### 6.重新编译

~~~ba
make all
~~~

##### 7.在虚拟机中新增iam.c和whoami.c文件

[参考](https://blog.csdn.net/weixin_45666853/article/details/104543695)

~~~bash
sudo ./mount-hdc
cd ./hdc/usr/root
~~~

创建iam.c

~~~c
#define __LIBRARY__ 
#include <unistd.h>
_syscall1(int,iam,const char*,name);

int main(int argc,char* argv[])
{
    iam(argv[1]);
    return 0;
}
~~~

创建whoami.c文件

~~~c
#define __LIBRARY__
#include <unistd.h>
#include <stdio.h>

_syscall2(int, whoami, char*, name, unsigned int, size);

int main(int argc, char ** argv)
{
    char t[30];
    whoami(t, 30);
    printf("%s\n", t);
    return 0;
}

~~~

##### 8.在虚拟机中添加测试文件

(关闭虚拟机文件系统时，要记得切换到非虚拟机文件，否则会损坏文件系统)

~~~bash
cd ~/oslab
sudo ./mount-hdc
cd ./hdc/usr/root
cp /home/teacher/testlab2.c ./
cp /home/teacher/testlab2.sh ./
~~~

##### 9.开始测试

~~~bash
gcc -o iam iam.c 
gcc -o whoami whoami.c
gcc -o testlab2 testlab2.c
./iam lance	
./whoami 			//返回lance
./testlab2
./testlab2.sh
~~~

实验成功。。。

##### 10.回答实验报告中的问题

- 从 Linux 0.11 现在的机制看，它的系统调用最多能传递几个参数？你能想出办法来扩大这个限制吗？
  - 最多传递三个参数，也就是unistd.h中定义的那几个_syscalln宏定义，n最大为三，可以再定义更多参数的宏定义来扩大限制，但是也会收到寄存器数量限制。
- 用文字简要描述向 Linux 0.11 添加一个系统调用 foo() 的步骤。
  - 添加foo的系统调用号
  - 编写foo系统调用函数
  - 将系统调用函数添加到sys_call_table中
  - 修改相应Makefile

### 4.总结

很多地方参考[LOVE6的博客](https://blog.csdn.net/qq_37500516/article/details/117229377?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522171291048816800182173631%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=171291048816800182173631&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-3-117229377-null-null.nonecase&utm_term=%E5%93%88%E5%B7%A5%E5%A4%A7&spm=1018.2226.3001.4450)以及[这个](https://blog.csdn.net/weixin_45666853/article/details/104543695),对c语言不熟练，写函数实现磕磕碰碰的，但是对系统调用有了更深的认识。

