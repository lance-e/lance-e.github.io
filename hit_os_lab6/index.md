# Hit_os_lab6


<!--more-->

# hit os lab6

### 1.实验内容

本次实验的基本内容是：

- 用 Bochs 调试工具跟踪 Linux 0.11 的地址翻译（地址映射）过程，了解 IA-32 和 Linux 0.11 的内存管理机制；
- 在 Ubuntu 上编写多进程的生产者—消费者程序，用共享内存做缓冲区；
- 在信号量实验的基础上，为 Linux 0.11 增加共享内存功能，并将生产者—消费者程序移植到 Linux 0.11。

#### 3.1 跟踪地址翻译过程

首先以汇编级调试的方式启动 Bochs，引导 Linux 0.11，在 0.11 下编译和运行 test.c。它是一个无限循环的程序，永远不会主动退出。然后在调试器中通过查看各项系统参数，从逻辑地址、LDT 表、GDT 表、线性地址到页表，计算出变量 `i` 的物理地址。最后通过直接修改物理内存的方式让 test.c 退出运行。

test.c 的代码如下：

```c
#include <stdio.h>

int i = 0x12345678;
int main(void)
{
    printf("The logical/virtual address of i is 0x%08x", &i);
    fflush(stdout);
    while (i)
        ;
    return 0;
}
```

#### 3.2 基于共享内存的生产者—消费者程序

本项实验在 Ubuntu 下完成，与信号量实验中的 `pc.c` 的功能要求基本一致，仅有两点不同：

- 不用文件做缓冲区，而是使用共享内存；
- 生产者和消费者分别是不同的程序。生产者是 producer.c，消费者是 consumer.c。两个程序都是单进程的，通过信号量和缓冲区进行通信。

Linux 下，可以通过 `shmget()` 和 `shmat()` 两个系统调用使用共享内存。

#### 3.3 共享内存的实现

进程之间可以通过页共享进行通信，被共享的页叫做共享内存，结构如下图所示：

![图片描述信息](https://doc.shiyanlou.com/userid19614labid573time1424086247964)

图 1 进程间共享内存的结构

本部分实验内容是在 Linux 0.11 上实现上述页面共享，并将上一部分实现的 producer.c 和 consumer.c 移植过来，验证页面共享的有效性。

具体要求在 `mm/shm.c` 中实现 `shmget()` 和 `shmat()` 两个系统调用。它们能支持 `producer.c` 和 `consumer.c` 的运行即可，不需要完整地实现 POSIX 所规定的功能。

- shmget()

```c
int shmget(key_t key, size_t size, int shmflg);
```

`shmget()` 会新建/打开一页内存，并返回该页共享内存的 shmid（该块共享内存在操作系统内部的 id）。

所有使用同一块共享内存的进程都要使用相同的 key 参数。

如果 key 所对应的共享内存已经建立，则直接返回 `shmid`。如果 size 超过一页内存的大小，返回 `-1`，并置 `errno` 为 `EINVAL`。如果系统无空闲内存，返回 -1，并置 `errno` 为 `ENOMEM`。

`shmflg` 参数可忽略。

- shmat()

```c
void *shmat(int shmid, const void *shmaddr, int shmflg);
```

`shmat()` 会将 `shmid` 指定的共享页面映射到当前进程的虚拟地址空间中，并将其首地址返回。

如果 `shmid` 非法，返回 `-1`，并置 `errno` 为 `EINVAL`。

`shmaddr` 和 `shmflg` 参数可忽略。

### 2.实验原理

​	80X86内存寻址依靠的是段的寻址方式，也就是将内存分为不同的段，对一个数据对象的寻址就需要段的起始位置(段地址)+段内偏移地址，段地址由16位的段描述符指定，段内偏移地址由32位值指定。80X86提供了段寄存器CS,DS,SS,ES,FS,GS，段寄存器存放了段选择子，段选择子用于索引段描述符，CS专门存放代码段，SS专门存放堆栈段。

​	段选择子的结构为：段描述符索引+TI（表指示标记）+RPL（请求特权级）。TI为1代表段描述符在LDT中，为0代表段描述符在GDT中；RPL为请求特权级，访问一个段时，处理器要比较RPL和CPL来判断是否权限足够（CPL存放在CS的0位和1位）。

​	段描述符的结构为段基址+段限长。段基址+段内偏移就是线性地址。

### 3.实验开始

##### 3.1跟踪地址翻译流程

- 通过汇编级调试，获取代码的汇编
- 通过汇编代码可以知道对象的虚拟地址（段寄存器：段内偏移地址）（例：ds：0x3004）
- 通过对应的段寄存器，获取段选择子，可知段描述符索引，以及是在GDT还是LDT
- 再到LDT或者GDT中找段描述符，可以获得段基址
- 段基址+段内偏移地址（这个偏移地址就是指上面例子中的0x3004），可得线性地址
- 再通过线性地址查页目录表，页表，去获取页目录号，页表号，再加上页内偏移获取到完整物理地址

##### 3.2-3.3

这里仅展示共享内存的实现，参考[地址映射与共享](https://nachen95.github.io/2023/07/09/HITOS/#%E5%AE%9E%E9%AA%8C%E5%85%AD-%E5%9C%B0%E5%9D%80%E6%98%A0%E5%B0%84%E4%B8%8E%E5%85%B1%E4%BA%AB)实现。

shm.h:

~~~c
#ifndef _SHM_H
#define _SEM_H

#include <linux/sched.h>

#define SHM_SIZE 64

typedef struct shm_ds
{
    unsigned int key;
    unsigned int size;
    unsigned long page;
}shm_ds;

int sys_shmget(unsigned int key,size_t size);
void * sys_shmat(int shmid);

#endif
~~~



shm.c:

~~~c
static shm_ds shm_list[SHM_SIZE] = {{0, 0, 0}}; 

int sys_shmget(unsigned int key, size_t size)
{
    int i;
    void *page;
    if (size > PAGE_SIZE || key == 0)
        return -EINVAL;
    for (i = 0; i < SHM_SIZE; i++) 
    {
        if (shm_list[i].key == key)
        {
            return i;
        }
    }
    page = get_free_page(); 
    decrease_mem_map(page);

    if (!page)
        return -ENOMEM;
    for (i = 0; i < SHM_SIZE; i++) 
    {
        if (shm_list[i].key == 0)
        {
            shm_list[i].page = page;
            shm_list[i].key = key;
            shm_list[i].size = size;
            return i;
        }
    }
    return -1;
}

void *sys_shmat(int shmid)
{
    if (shmid < 0 || SHM_SIZE <= shmid || shm_list[shmid].page == 0 || shm_list[shmid].key == 0)
        return (void *)-EINVAL;
    put_page(shm_list[shmid].page, current->brk + current->start_code);

    increase_mem_map(shm_list[shmid].page);
    current->brk += PAGE_SIZE;
    return (void *)(current->brk - PAGE_SIZE);
}
~~~

memory.c:据说这里是因为多进程退出时都会对共享内存进行释放，导致panic。

~~~c
void increase_mem_map(unsigned long page){
    page -= LOW_MEM;
    page >>= 12;
    mem_map[page]++;
}
void decrease_mem_map(unsigned long page){
    page -= LOW_MEM;
    page <<= 12;
    mem_map[page]--;
}
~~~

### 3.回答问题

完成实验后，在实验报告中回答如下问题：

- 对于地址映射实验部分，列出你认为最重要的那几步（不超过 4 步），并给出你获得的实验数据。
  - 通过汇编代码获取到数据对象的虚拟地址（段选择子+段内偏移）
  - 查段表（GDT或LDT）得到段描述符，获取段基址，组合成线性地址
  - 通过线性地址，通过查页目录表，页表，获取到物理地址

### 4.总结

最后没有耐心做了，不知道为啥两个进程同时运行bochs直接黑屏了😫，就不做了。因为lab5我的完成度也不高，所以这个lab做得很糟糕，没有达到完整的实验结果，共享内存实现参考[地址映射与共享](https://nachen95.github.io/2023/07/09/HITOS/#%E5%AE%9E%E9%AA%8C%E5%85%AD-%E5%9C%B0%E5%9D%80%E6%98%A0%E5%B0%84%E4%B8%8E%E5%85%B1%E4%BA%AB)，代码能力真依托，但起码还是学到了虚拟内存到物理内存的推理过程🥺

