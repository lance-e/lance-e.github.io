# Hit_os_lab8


<!--more-->

# hit os lab8

### 1.实验内容

在 Linux 0.11 上实现 procfs（proc 文件系统）内的 psinfo 结点。当读取此结点的内容时，可得到系统当前所有进程的状态信息。例如，用 cat 命令显示 `/proc/psinfo` 的内容，可得到：

```bash
$ cat /proc/psinfo
pid    state    father    counter    start_time
0    1    -1    0    0
1    1    0    28    1
4    1    1    1    73
3    1    1    27    63
6    0    4    12    817
$ cat /proc/hdinfo
total_blocks:    62000;
free_blocks:    39037;
used_blocks:    22963;
...
```

`procfs` 及其结点要在内核启动时自动创建。

相关功能实现在 `fs/proc.c` 文件内。

### 2.实验开始

##### 1.添加proc类型

include/sys/stat.h

~~~c
#define S_IFPROC 0070000
#define S_ISPROC(m) (((m) & S_IFMT) == S_IFPROC)
~~~

##### 2.修改mknod使能创建新的文件类型

fs/namei.c

~~~c
if (S_ISBLK(mode) || S_ISCHR(mode) || S_ISPROC(mode))
~~~

##### 3.在操作系统初始化时自动生成proc文件系统

init/main.c:

首先在init函数相应位置改成：

~~~c
setup((void *) &drive_info);
mkdir("/proc",0755);
mknod("/proc/psinfo",S_IFPROC|0444,0);
~~~

添加这两个系统调用的api：

~~~c
_syscall2(int,mkdir,const char*,name,mode_t,mode)
_syscall3(int,mknod,const char*,filename,mode_t,mode,dev_t,dev)
~~~

##### 4.让proc文件可读

fs/read_write.c

~~~c
extern int proc_handle(int dev, off_t * pos, char * buf, int count);

if(S_ISPROC(inode->i_mode)) {
		return proc_handle(inode->i_zone[0], &file->f_pos, buf, count);
	}
~~~

##### 5.proc的处理函数

proc.c:

~~~c
#include <sys/types.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <asm/segment.h>
#include <string.h>
#include <stdarg.h>
int sprintf(char* buf,const char * fmt,...)
{
        va_list args;int i;
        va_start(args,fmt);
        i = vsprintf(buf,fmt,args);
        va_end(args);
        return i;
}
int proc_handle(int dev,unsigned long * pos,char * buf,int count)
{
        struct task_struct **p;
        int i ;
        char * tmp = (char *)malloc(sizeof(char)*512);
        memset(tmp,'\0',sizeof(char)*512);
        char * ptr = tmp;
        
        offset = sprintf(ptr,"pid\tstate\tfather\tcounter\tstart_time\n");
        tmp+= offset;
        for (p = &FIRST_TASK;p<= &LAST_TASK;p++)
        {
                if (*p  && (!(*p)->pid || (*p)->counter))
                {
                        offset = sprintf(tmp,"%s\t%s\t%s\t%s\t%s\n",(*p)->pid,(*p)->state,(*p)->father,(*p)->counter,(*p)->start_time);
                        tmp += offset;
                }
        }
        for (i = 0 ;i< 512;i++){
                if (ptr[*pos+i]=='\0') break;
                put_fs_byte(ptr[*pos+i],buf+i);
        }
        *pos += i;
        free(tmp);
        return i;
}

~~~

##### 6.修改makefile

~~~makefile
OBJS=	open.o read_write.o inode.o file_table.o buffer.o super.o \
	block_dev.o char_dev.o file_dev.o stat.o exec.o pipe.o namei.o \
	bitmap.o fcntl.o ioctl.o truncate.o proc.o
...
proc.o :proc.c ../include/linux/kernel.h ../include/linux/sched.h \
  ../include/asm/segment.h   ../include/sys/types.h \
  ../include/stdarg.h ../include/string.h

~~~

##### 7.结果

(之前也没看到bochs可以截图成文件😭)

~~~bash
[/usr/root]# cat /proc/psinfo
pid     state   father  counter start_time
0       1       -1      0       0
1       1       0       28      1
4       1       1       3       73
3       1       1       27      63
6       0       4       12      4008
~~~

### 3.总结

我这个lab就做了psinfo，hdinfo就懒得做了，整体难度不大，但是我实在没啥耐心了。最后proc.c文件中老是出bug，找了参考博客一直比对了好久，才发现是自己瞎，导致少了个括号😡。哈工大的操作系统lab终于是全做完了，完结撒花🎉，接下来目标是照着书写一个简易os，跟上辣个男人的脚步🫡


