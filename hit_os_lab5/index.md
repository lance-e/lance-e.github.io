# Hit_os_lab5


<!--more-->

# hit os lab5

### 1.实验内容

本次实验的基本内容是：

- 在 Ubuntu 下编写程序，用信号量解决生产者——消费者问题；
- 在 0.11 中实现信号量，用生产者—消费者程序检验之。

#### 3.1 用信号量解决生产者—消费者问题

在 Ubuntu 上编写应用程序“pc.c”，解决经典的生产者—消费者问题，完成下面的功能：

- 建立一个生产者进程，N 个消费者进程（N>1）；
- 用文件建立一个共享缓冲区；
- 生产者进程依次向缓冲区写入整数 0,1,2,...,M，M>=500；
- 消费者进程从缓冲区读数，每次读一个，并将读出的数字从缓冲区删除，然后将本进程 ID 和 + 数字输出到标准输出；
- 缓冲区同时最多只能保存 10 个数。

一种可能的输出效果是：

```txt
10: 0
10: 1
10: 2
10: 3
10: 4
11: 5
11: 6
12: 7
10: 8
12: 9
12: 10
12: 11
12: 12
……
11: 498
11: 499
```

其中 ID 的顺序会有较大变化，但冒号后的数字一定是从 0 开始递增加一的。

`pc.c` 中将会用到 `sem_open()`、`sem_close()`、`sem_wait()` 和 `sem_post()` 等信号量相关的系统调用，请查阅相关文档。

《UNIX 环境高级编程》是一本关于 Unix/Linux 系统级编程的相当经典的教程。如果你对 POSIX 编程感兴趣，建议买一本常备手边。

#### 3.2 实现信号量

Linux 在 0.11 版还没有实现信号量，Linus 把这件富有挑战的工作留给了你。如果能实现一套山寨版的完全符合 POSIX 规范的信号量，无疑是很有成就感的。但时间暂时不允许我们这么做，所以先弄一套缩水版的类 POSIX 信号量，它的函数原型和标准并不完全相同，而且只包含如下系统调用：

```c
sem_t *sem_open(const char *name, unsigned int value);
int sem_wait(sem_t *sem);
int sem_post(sem_t *sem);
int sem_unlink(const char *name);
```

- `sem_open`的功能是创建一个信号量，或打开一个已经存在的信号量。
  - `sem_t` 是信号量类型，根据实现的需要自定义。
  - `name` 是信号量的名字。不同的进程可以通过提供同样的 name 而共享同一个信号量。如果该信号量不存在，就创建新的名为 name 的信号量；如果存在，就打开已经存在的名为 name 的信号量。
  - `value` 是信号量的初值，仅当新建信号量时，此参数才有效，其余情况下它被忽略。当成功时，返回值是该信号量的唯一标识（比如，在内核的地址、ID 等），由另两个系统调用使用。如失败，返回值是 NULL。
- `sem_wait()` 就是信号量的 P 原子操作。如果继续运行的条件不满足，则令调用进程等待在信号量 sem 上。返回 0 表示成功，返回 -1 表示失败。
- `sem_post()` 就是信号量的 V 原子操作。如果有等待 sem 的进程，它会唤醒其中的一个。返回 0 表示成功，返回 -1 表示失败。
- `sem_unlink()` 的功能是删除名为 name 的信号量。返回 0 表示成功，返回 -1 表示失败。

在 `kernel` 目录下新建 `sem.c` 文件实现如上功能。然后将 pc.c 从 Ubuntu 移植到 0.11 下，测试自己实现的信号量。

### 2.实验开始

##### 1.使用信号量解决生产者-消费者问题

`sem_open()`：打开或创建一个信号量

`sem_close()`：删除一个信号量

`sem_wait()` ：消费资源，对应P

`sem_post()`：生产资源，对应V

参考[主函数实现](https://blog.csdn.net/weixin_45666853/article/details/104947170?spm=1001.2014.3001.5501)

~~~c
#include <stdio.h>
#include <string.h>
#include<unistd.h>
#include<semaphore.h>
#include"stdio.h"
#include "sys/wait.h"
sem_t * empty , *full ,*mutex;
FILE * fp;
#define ALLNUMS 550
#define BUFFERSIZE 10 
#define CUSTOMERNUM 5
int Inpos  = 0;
int Outpos = 0;
void Producter(FILE * fp);
void Customer(FILE * fp);
void Producter(FILE* fp){
    for (int i = 0 ; i < ALLNUMS;i++){
        sem_wait(empty);
        sem_wait(mutex);
        fseek(fp,Inpos * sizeof(int), SEEK_SET);//把fp指针移动到离文件开头Inpos个整数处（Inpos从0开始）
        fwrite(&i,sizeof(int),1,fp);
        fflush(fp);
        Inpos = (Inpos+1)%BUFFERSIZE;
        sem_post(mutex);
        sem_post(full);
    }
}
void Customer(FILE * fp){
    int productid;
    for (int j = 0 ; j < ALLNUMS / CUSTOMERNUM;j++){
        sem_wait(full);
        sem_wait(mutex);
        fseek(fp,10*sizeof(int),SEEK_SET);
        fread(&Outpos, sizeof(int), 1,fp);
        //读取最后一个位置存储的消费位置
        fseek(fp, Outpos*sizeof(int), SEEK_SET);
        fread(&productid,sizeof(int),1, fp);
        //读取对应消费位置的值
        
        printf("%d: %d\n",j,productid);
        fflush(stdout);

        Outpos = (Outpos+1) %BUFFERSIZE;//更新消费位置
        fseek(fp, 10*sizeof(int), SEEK_SET);
        fwrite(&Outpos, sizeof(int), 1, fp);
        fflush(fp);

        sem_post(mutex);
        sem_post(empty);
    }
}
int main(){
    pid_t producter;
    pid_t customer;
    empty  = sem_open("empty",O_CREAT,0777,10);
    full = sem_open("full",O_CREAT,0777, 0);
    mutex = sem_open("mutex", O_CREAT,0777,1 );
    fp = fopen("products.txt","wb+");

    //现将最初的消费位置写入10的位置
    fseek(fp, 10*BUFFERSIZE, SEEK_SET);
    fwrite(&Outpos, sizeof(int),1,fp);
    fflush(fp);
    producter = fork();
    if (producter){
        Producter(fp);
    }else {
        for (int i= 0 ; i < CUSTOMERNUM;i++){
            customer = fork();
            if (!customer){
                Customer(fp);
                break;
            }
        }
    }
    wait(NULL);
    wait(NULL);
    wait(NULL);
    wait(NULL);
    wait(NULL);
    sem_unlink("empty");
    sem_unlink("full");
    sem_unlink("mutex");
    fclose(fp);
    return 0;
}
~~~

##### 2.实现信号量

增加信号量的系统调用，这里的相关原理与lab2紧密相关，

自己老是写错，这里就直接参考别人的博客实现：

kernel/sem.c:

~~~c
#include <asm/io.h>
#include <asm/segment.h>
#include <asm/system.h>
#include <linux/fdreg.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/sem.h>
#include <linux/tty.h>
#include <unistd.h>
// #include <string.h>

sem_t semtable[SEMTABLE_LEN];
int cnt = 0;

sem_t *sys_sem_open(const char *name, unsigned int value)
{
    char kernelname[SEM_NAME_LEN] = {'\0'};
    int isExist = 0;
    int i = 0;
    int name_cnt = 0;
    while (get_fs_byte(name + name_cnt) != '\0')
        name_cnt++;
    if (name_cnt > SEM_NAME_LEN)
        return NULL;
    for (i = 0; i < name_cnt; i++)
        kernelname[i] = get_fs_byte(name + i);
    int name_len = strlen(kernelname);
    int sem_name_len = 0;
    sem_t *p = NULL;
    for (i = 0; i < cnt; i++)
    {
        sem_name_len = strlen(semtable[i].name);
        if (sem_name_len == name_len)
        {
            if (!strcmp(kernelname, semtable[i].name)) 
            {
                isExist = 1;
                break;
            }
        }
    }
    if (isExist == 1)
    {
        p = (sem_t *)(&semtable[i]);
        // printk("find previous name!\n");
    }
    else
    {
        i = 0;
        for (i = 0; i < name_len; i++)
        {
            semtable[cnt].name[i] = kernelname[i];
        }
        semtable[cnt].value = value;
        p = (sem_t *)(&semtable[cnt]);
        // printk("creat name!\n");
        cnt++;
    }
    return p;
}

int sys_sem_wait(sem_t *sem)
{
    cli();
    while (sem->value <= 0) 
        sleep_on(&(sem->queue));  
    sem->value--;
    sti();
    return 0;
}
int sys_sem_post(sem_t *sem)
{
    cli();
    sem->value++;
    if ((sem->value) <= 1)
        wake_up(&(sem->queue));
    sti();
    return 0;
}

int sys_sem_unlink(const char *name)
{
    char kernelname[100];
    int isExist = 0;
    int i = 0;
    int name_cnt = 0;
    while (get_fs_byte(name + name_cnt) != '\0')
        name_cnt++;
    if (name_cnt > SEM_NAME_LEN)
        return NULL;
    for (i = 0; i < name_cnt; i++)
        kernelname[i] = get_fs_byte(name + i);
    int name_len = strlen(name);
    int sem_name_len = 0;
    for (i = 0; i < cnt; i++)
    {
        sem_name_len = strlen(semtable[i].name);
        if (sem_name_len == name_len)
        {
            if (!strcmp(kernelname, semtable[i].name))
            {
                isExist = 1;
                break;
            }
        }
    }
    if (isExist == 1)
    {
        int tmp = 0;
        for (tmp = i; tmp <= cnt; tmp++)
        {
            semtable[tmp] = semtable[tmp + 1];
        }
        cnt = cnt - 1;
        return 0;
    }
    else
        return -1;
}
~~~

这里就不将之前的pc.c移植到linux0.11中了

### 3.回答问题

完成实验后，在实验报告中回答如下问题：

在 `pc.c` 中去掉所有与信号量有关的代码，再运行程序，执行效果有变化吗？为什么会这样？

- 会导致进程之间无法同步协作，进行资源竞争。

 实验的设计者在第一次编写生产者——消费者程序的时候，是这么做的：

```c
Producer()
{
    // 生产一个产品 item;
		// 互斥信号量
    P(Mutex);
    // 空闲缓存资源
    P(Empty);

    

    // 产品资源
    V(Full);
    // 将item放到空闲缓存中;
    V(Mutex);

}

Consumer()
{
    P(Mutex);
    P(Full);

    // 消费产品item;
    V(Empty);
    //从缓存区取出一个赋值给item;
    V(Mutex);


}
```

这样可行吗？如果可行，那么它和标准解法在执行效果上会有什么不同？如果不可行，那么它有什么问题使它不可行？

- 不可行，会导致死锁

### 4.总结

信号量是为了多进程同步协作，在该lab中，进行了信号量的实现与应用，实现的部分参考[信号量的实现](https://github.com/NaChen95/Linux0.11/commit/4a6a351f933e6e969843ff34b19af0c7993ef783#diff-b2e1ab2f039bf6e283545365d1c4a11c7e46fad5887fa0a53222f7297724dc89),应用部分参考[信号量的应用](https://blog.csdn.net/weixin_45666853/article/details/104947170?spm=1001.2014.3001.5501),感觉自己的代码能力有待提高😫

