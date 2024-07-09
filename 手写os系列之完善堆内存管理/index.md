# 手写OS系列之完善堆内存管理


<!--more-->

# 手写OS系列之完善堆内存管理

### 1.实现动态分配内存

##### a.相关数据结构



​	引入新名词arena(舞台)：将一大块内存划分成多个小内存，每个小内存之间互不干涉，可以分别管理，这些众多的小内存块就叫做arena，可以理解为内存仓库。arena占一个物理页框，包含元数据和arena中的内存池，内存池中以内存块为单位进行划分内存。

​	arena元数据数据结构：

- desc：内存块描述符指针可以间接获取内存块信息。
- cnt：代表内存块数量或页框数
- large：用于判断申请的内存是否是大内存

~~~c
//memory repository
struct arena {
	struct mem_block_desc* desc;
	
	// if large =true , cnt is the number of page 
	// if large =false, cnt is the number of memory block
	uint32_t cnt;					

	bool large;
};
~~~

​	内存块mem_block数据结构：

只有一个free_elem成员，用来配合内存块描述符中的链表使用。

~~~c
//memory block
struct mem_block{
	struct list_elem free_elem;
};
~~~



​	当不断地内存分配之后，将会产生大量的arena，为了更好地管理arena集群，引入内存块描述符(mem_block_desc)数据结构。用于管理同种规格的arena集群，mem_block_desc只包含两个属性，一个是内存块规模，一个是包含所有空闲内存块的链表。

​	mem_block_desc 数据结构：

- block_size：内存块规模
- block_per_arena：每个arena中的内存块容量
- free_list：空闲内存块链表

~~~c
//memory block descriptor
struct mem_block_desc{
	uint32_t block_size;				//size of memory block
	uint32_t block_per_arena;			//the number of memory block in per arena
	struct list free_list;
};
~~~

##### b.初始化

​	针对不同的内存申请，我们有不同的处理策略，当申请的内存小于1024字节时，我们使用arena中的内存块，当申请内存大于1024字节时，我们将不会再拆分大内存，而是直接把整块内存分配出去，代码易维护。

​	(为什么是1024字节？因为arena占一页，元数据会占用内存，所以内存池必定小于4kb，内存块是平均分配的，所以最大的内存块肯定是小于2kb的，我们又是以2的指数来划分，所以最大为1024字节，这里我们仅支持7种规格内存块，16，32，64，128，256，512，1024。)

​	kernel/memory.c:初始化**内核**7种规格内存块描述符

~~~c
struct mem_block_desc k_block_descs[DESC_CNT];

//prepare for malloc :initial the memory block descriptor  
void block_desc_init(struct mem_block_desc* desc_array){
	uint16_t index ,size = 16;
	for (index = 0 ; index < DESC_CNT ; index ++ ){
		desc_array[index].block_size = size;

		//initial the number of memory block in arena
		desc_array[index].block_per_arena = (PG_SIZE - sizeof(struct arena)) / size;

		list_init(&desc_array[index].free_list);

		size *= 2;
	}
}

void mem_init(){
	//...
	block_desc_init(k_block_descs);
	//...
}
~~~

​	初始化**用户进程**7种规格内存块描述符:

thread/thread.h:添加用户进程的内存块描述符，因为用户进程有自己专属的虚拟内存空间，所以每个进程有自己的内存块描述符，用来管理自己的虚拟内存。

~~~c
//the PCB 
struct task_struct {
	//...
						
	struct mem_block_desc u_block_desc[DESC_CNT]; //memory block descriptor of user process

	//...
};
~~~

userprog/process.c:创建用户进程时，进行初始化

~~~c
// create user process
void process_execute(void* filename , char* name) {
	//...
	block_desc_init(thread->u_block_desc);
	//...
}
	
~~~

##### c.内存操作

​	相关操作函数：

- arena2block:获取对应arena目标索引处的内存块
- block2arena:获取内存块对应的arena
- sys_malloc:在堆中申请size字节内存。先判断用哪个内存池，用户还是内核，然后根据申请的内存大小，分配页框还是内存块。如果>1024字节，直接向上取整分配页框；如果<1024字节，就分配适配的最小内存块，从内存块描述符中的free_list中直接获取内存块。

**值得注意的是， 当描述符的链表中没有空闲的内存块了，才创建arena，且一个arena的所有内存块都是空闲的，将会回收该arena占用的页框。所以在分配内存块之前，还需要判断，是否有空闲的内存块，没有就创建arena，并将arena中的内存池一块一块地添加进描述符的free_list**

~~~c
//return the address of 'index' in the arena
static struct mem_block* arena2block(struct arena* a , uint32_t index){
	return (struct mem_block*)((uint32_t)a + sizeof(struct arena) + index * a->desc->block_size);
}

//return the address of which arena 'b' at
static struct arena* block2arena(struct mem_block* b){
	return (struct arena*)((uint32_t)b & 0xfffff000);
}

//malloc 'size' memory from heap
void* sys_malloc(uint32_t size){
	enum pool_flags PF;
	struct pool* mem_pool;
	uint32_t pool_size;
	struct mem_block_desc* descs;
	struct task_struct* cur_thread = running_thread();

	//judge use which memory pool
	if (cur_thread->pgdir == NULL ){				//kernel memory pool
		PF = PF_KERNEL;
		pool_size = kernel_pool.pool_size;
		mem_pool = &kernel_pool;
		descs = k_block_descs;
	}else {								//user process memory pool
		PF = PF_USER;
		pool_size = user_pool.pool_size;
		mem_pool = &user_pool;
		descs = cur_thread->u_block_desc; 
	}

	//judge if there is enough capacity
	if (!(size > 0 && size < pool_size)){
		return NULL;
	}
	struct arena* a;
	struct mem_block* b;
	lock_acquire(&mem_pool->lock);
	
	//begin malloc
	if (size > 1024){
		uint32_t page_cnt = DIV_ROUND_UP(size + sizeof(struct arena) , PG_SIZE);

		a = malloc_page(PF,page_cnt);

		if (a != NULL){
			memset(a, 0 , page_cnt * PG_SIZE);		//clear the malloc memory
			a->desc = NULL;
			a->cnt = page_cnt;
			a->large = true;
			lock_release(&mem_pool->lock);
			//+1 is mean skip a size of struct arena(meta message)
			return (void*)(a + 1);				
		}else {
			lock_release(&mem_pool->lock);
			return NULL;
		}

	}else {								

		//get the kind of block index
		uint8_t index;
		for (index = 0 ; index < DESC_CNT;  index++){
			if (size <= descs[index].block_size){
				break;
			}
		}
		
		// if the 'free_list' already have no available 'mem_block', now create new arena
		if (list_empty(&descs[index].free_list)){
			a = malloc_page(PF , 1);
			if (a == NULL){
				lock_release(&mem_pool->lock);
				return NULL;
			}
			memset(a , 0 , PG_SIZE);

			a->desc = &descs[index];
			a->large = false;
			a->cnt = descs[index].block_per_arena;
			
			uint32_t block_index;

			enum intr_status old_status = intr_disable();

			//begin to split the memory of arena
			//and add to the 'free_list' of memory block descriptor
			for (block_index = 0 ;block_index < descs[index].block_per_arena;block_index ++){
				b = arena2block(a, block_index);
				ASSERT(!elem_find(&a->desc->free_list , &b->free_elem));
				list_append( &a->desc->free_list , &b->free_elem);
			}
			intr_set_status(old_status);
		}
		
		//begin to malloc memory block
		b = (struct mem_block*)(list_pop(&(descs[index].free_list)) );
		memset(b , 0 , descs[index].block_size);

		a = block2arena(b);
		a->cnt --;
		lock_release(&mem_pool->lock);
		return (void*)b;
	}

}
~~~

### 2.实现内存回收

##### a.实现页框级别内存回收

​	内存回收与内存申请的过程相反，回顾内存申请过程：

- vaddr_get:分配虚拟内存，操作对应虚拟内存池位图，置1
- pallc:分配物理地址，操作对应物理内存池位图，置1
- page_table_add:页表中完成虚拟地址到物理地址的映射

​	内存回收过程：

- pfree:回收物理内存，操作物理内存池位图，置0
- page_table_pte_remove:删除虚拟内存的映射，通过将pte的p位置0，通过invlpg指令更新快表TLB(也可以通过重新加载cr3)
- vaddr_remove:回收虚拟内存，操作虚拟内存池位图，置0

~~~c

//free physic memory to physic memory pool
void pfree(uint32_t pg_phy_addr){
	struct pool* mem_pool;
	uint32_t index = 0 ;
	if (pg_phy_addr >= user_pool.phy_addr_start) {				//user memory pool
		mem_pool = &user_pool;
		index = (pg_phy_addr - user_pool.phy_addr_start ) / PG_SIZE;
	}else {									//kernel memory pool
		mem_pool = &kernel_pool;
		index = (pg_phy_addr - kernel_pool.phy_addr_start) / PG_SIZE;
	}
	bitmap_set(&mem_pool->pool_bitmap , index , 0 );
}

//remove the map of 'vaddr', just to remove the pte of 'vaddr'
static void page_table_pte_remove(uint32_t vaddr){
	uint32_t* pte = pte_ptr(vaddr);
	*pte &= PG_P_0 ;
	asm volatile ("invlpg %0": : "m" (vaddr) : "memory");
}

//free the 'pg_cnt' consecutive pages virtual memory from virtual memory pool
static void vaddr_remove(enum pool_flags pf , void* _vaddr , uint32_t pg_cnt){
	uint32_t bit_start_index = 0 , vaddr = (uint32_t)_vaddr , count = 0 ;
	if (pf == PF_KERNEL){				//kernel virtual memory pool
		bit_start_index = (vaddr - kernel_vaddr.vaddr_start ) / PG_SIZE;
		while(count < pg_cnt) {
			bitmap_set(&kernel_vaddr.vaddr_bitmap , bit_start_index+ count++, 0);
		}
	}else {						//user virtual memory pool
		struct task_struct * cur_thread = running_thread();
		bit_start_index =( vaddr - cur_thread->userprog_vaddr.vaddr_start) / PG_SIZE;
		while (count < pg_cnt ) {
			bitmap_set(&cur_thread->userprog_vaddr.vaddr_bitmap,bit_start_index + count++ , 0 );
		}
	}
}



//free 'pg_cnt' page memory
/***********************   mfree_page 	    ******************
 * 	1.vaddr_remove :		free virtual memory into virtual memory pool
 * 	2.pfree: 			free physic memory into  physic memory pool
 * 	3.page_table_pte_remove: 	remvoe the map
 ******************************************************************/
void mfree_page(enum  pool_flags pf , void* _vaddr , uint32_t pg_cnt){
	//struct pool* mem_pool;
	uint32_t phy_addr ;
	uint32_t vaddr = (uint32_t)_vaddr, count = 0;

	ASSERT(pg_cnt > 0 && vaddr % PG_SIZE == 0 );

	phy_addr = addr_v2p(vaddr); 		//get the physic memory by the virtual address;
	
	//make sure the target page is exist at the out of low 1mb , 1kb pdt and 1kb pt
	ASSERT( (phy_addr % PG_SIZE ) == 0 && phy_addr >= 0x102000);

	//judge use which memory pool
	if (phy_addr >= user_pool.phy_addr_start){			//in user memory pool
		vaddr -= PG_SIZE;	
		while(count < pg_cnt){
			vaddr += PG_SIZE;
			phy_addr = addr_v2p(vaddr);
			ASSERT((phy_addr % PG_SIZE ) == 0 && phy_addr >= user_pool.phy_addr_start);
			pfree(phy_addr);
			page_table_pte_remove(vaddr);
			count++;
		}
	}else {
		vaddr -= PG_SIZE;
		while(count < pg_cnt){
			vaddr += PG_SIZE;
			phy_addr = addr_v2p(vaddr);
			ASSERT((phy_addr % PG_SIZE ) == 0 &&	\
			phy_addr >= kernel_pool.phy_addr_start &&	\
			phy_addr < user_pool.phy_addr_start);
			pfree(phy_addr);
			page_table_pte_remove(vaddr);
			count++;
		}
	}
	//clear the target bit of virtual memory's bitmap
	vaddr_remove(pf , _vaddr , pg_cnt);
}


~~~

##### b.实现任意字节内存回收

​	针对内存>1024，直接使用free_page回收页框

​	针对内存<1024, 将内存块回收进对应规格内存块描述符的空闲块链表，如果该arena的内存块都是空闲，就回收arena占用的页框。

~~~c
//free memory 'ptr'
void sys_free(void* ptr){
	ASSERT( ptr != NULL);
	if (ptr != NULL){
		enum pool_flags pf;
		struct pool* mem_pool;

		//judge thread of process
		if (running_thread()->pgdir == NULL){
			ASSERT((uint32_t)ptr >= K_HEAP_START);
			pf = PF_KERNEL;
			mem_pool = &kernel_pool;
		}else {
			pf = PF_USER;
			mem_pool = &user_pool;
		}

		lock_acquire(&mem_pool->lock);
		struct mem_block* b = ptr;
		struct arena* a = block2arena(b);

		ASSERT(a->large == 1 || a->large == 0);
		if (a->desc == NULL && a->large == true ){
			mfree_page(pf , a , a->cnt);
		}else {
			//recycle the memory block into free_list
			list_append(&a->desc->free_list , &b->free_elem);

			//judge the all block in arena is it all free,
			//if all free that free the whole arena
			if (++a->cnt == a->desc->block_per_arena){
				uint32_t index;
				for (index = 0 ; index < a->desc->block_per_arena ; index++){
					struct mem_block* b = arena2block(a , index);
					ASSERT(elem_find(&a->desc->free_list ,  &b->free_elem));
					list_remove(&b->free_elem);
				}
				mfree_page( pf , a , 1);
			}
		}
		lock_release(&mem_pool->lock);
	}
}


~~~

### 3.总结

​	至此loong-OS的内存管理基本完成。

