# 手写OS系列之实现同步机制


<!--more-->

# 手写OS系列之实现同步机制

### 1.相关概念

​	公共资源：就是被所有任务共享的一套资源

​	临界区：若多个任务都想访问同一公共资源，那么各任务中**访问公共资源的指令代码组成的区域** 就称为临界区。(强调是指令，不是受访的静态公共资源)

​	互斥：也可称为排他，即某一时刻公共资源只能被1个任务共享，不允许多个任务同时出现在自己的临界区。

​	竞争条件：多个任务以互斥态同时进入临界区，对公共资源的访问是以竞争的方式并行的。因此公共资源的最终状态依赖于这些任务的临界区的微操作执行次序

​	信号量：就是个0以上的整数，当信号量为0时，表示已无可用信号，或者说条件不允许，因此它表示某种信号的累积量，所以称作信号线。当信号量为1时，称作二元信号量或互斥信号量，可以作为锁的实现

### 2.实现线程的阻塞与唤醒

thread/thread.c:

​	阻塞就是将当前运行的线程状态改为阻塞态，并调度新的任务。

​	唤醒就是将目标线程的重新添加到就绪队列中(这里直接添加到队列头，下一次调度就可以直接执行该线程)，因为调度器每一次调度都是从就绪队列中选取就绪的任务开始执行。

~~~c
//block thread
void thread_block(enum task_status stat){
	ASSERT(((stat == TASK_BLOCKED )||(stat ==TASK_WAITING) || (stat == TASK_HANGING)));
	enum intr_status old_status  = intr_disable();
	struct task_struct* cur_thread = running_thread();
	cur_thread->status = stat;
	schedule();
	intr_set_status(old_status);
}

//unblock thread
void thread_unblock(struct task_struct* pthread){
	enum intr_status old_status = intr_disable();
	ASSERT(((pthread->status == TASK_BLOCKED) || (pthread->status == TASK_WAITING )||(pthread->status == TASK_HANGING)));
	if (pthread->status != TASK_READY){
		ASSERT(!elem_find(&thread_ready_list,&pthread->general_tag));
		if (elem_find(&thread_ready_list,&pthread->general_tag)){
			PANIC("thread_unblock: blocked thread in ready_list\n");
		}
		list_push(&thread_ready_list,&pthread->general_tag);
		pthread->status = TASK_READY;
	}
	intr_set_status(old_status);
}

~~~

### 3.实现信号量和互斥锁

锁和信号量的定义：

thread/sync.h:锁的实现就是互斥信号量，所以锁包含一个初始值为1的信号量

~~~c
//struct of semaphore
struct semaphore{
	uint8_t value;
	struct list waiters;
};


//struct of lock
struct lock{
	struct task_struct * holder;				//holder of lock
	struct semaphore semaphore;				//mutex semaphore; semaphore = 1 
	uint32_t holder_repeat_nr;				//the number of holder had apply for lock
};

~~~

操作：

thread/sync.c:

​	分别是初始化信号量，初始化锁，信号量P操作，信号量V操作，获取锁，释放锁

​	sema_down(P):信号量-1，当信号量=0时，此时就会阻塞线程，并将当前线程添加到等待队列中，直到信号量不为0。

​	sema_up(V):信号量+1，判断信号量的等待队列是否为空，为空直接+1，不为空代表此时有线程在等待信号量，所以先获取等待队列头的阻塞线程，然后唤醒该线程

​	lock_acquire:获取锁，若当前线程已经持有该锁，就让holder_repeat_nr+1，这是防止嵌套申请同一把锁导致死锁；若当前线程未持有该锁，就对该锁进行P操作，如果锁在其他线程那里未释放，就暂时阻塞，当上一个持有者释放了锁，就将锁的持有者改为当前线程。

​	lock_release:释放锁，判断holder_repeat_nr大小，大于1代表当前持有者多次申请锁，不能直接释放锁(V操作)，只能将holder_repeat_nr-1；等于1代表仅申请一次，可以直接释放锁(V操作)。

~~~c
//initial semaphore 
void sema_init(struct semaphore* psema,uint8_t value){
	psema->value = value;
	list_init(&psema->waiters);
}


//initial lock
void lock_init(struct lock* plock){
	plock->holder = NULL;
	plock->holder_repeat_nr = 0;
	sema_init(&plock->semaphore,1);
}


//semaphore down(-1) :P
void sema_down(struct semaphore* psema){
	enum intr_status old_status = intr_disable();
	while(psema->value == 0){
		ASSERT(!elem_find(&psema->waiters,&running_thread()->general_tag));
		if (elem_find(&psema->waiters,&running_thread()->general_tag)){
			PANIC("sema_down: thread blocked has been in waiters_list\n");
		}
		list_append(&psema->waiters,&running_thread()->general_tag);
		thread_block(TASK_BLOCKED);			//block thread,until other thread wake up this blocked thread
	}
	psema->value--;
	ASSERT(psema->value == 0);
	intr_set_status(old_status);
}

//semaphore up(+1) :V
void sema_up(struct semaphore* psema){
	enum intr_status old_status = intr_enable();
	ASSERT(psema->value = 0 );
	if (!list_empty(&psema->waiters)){
		struct task_struct* thread_blocked = (struct task_struct*)((uint32_t)list_pop(&psema->waiters) & 0xfffff000);
		thread_unblock(thread_blocked);
	}
	psema->value++;
	ASSERT(psema->value ==1 );
	intr_set_status(old_status);
}


//get lock
void lock_acquire(struct lock* plock){
	if (plock->holder != running_thread){
		sema_down(&plock->semaphore);					//semaphore :P
		plock->holder = running_thread();
		ASSERT(plock->holder_repeat_nr ==  0 );
		plock->holder_repeat_nr= 1;
	}else{
		plock->holder_repeat_nr++;
	}
}

//release lock
void lock_release(struct lock* plock){
	ASSERT(plock->holder == running_thread());
	if (plock->holder_repeat_nr > 1 ){
		plock->holder_repeat_nr--;
		return ;
	}
	ASSERT(plock->holder_repeat_nr == 1);

	plock->holder = NULL;
	plock->holder_repeat_nr = 0;
	sema_up(&plock->semaphore);						//semaphore :V
}
~~~


