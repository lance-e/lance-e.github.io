# 手写OS系列之内存管理


<!--more-->

# 手写OS系列之内存管理

### 1实现位图

​	位图，bitmap，用于管理资源。就是使用字节中的位来映射其他单位大小的资源，一一对应的关系。

##### 1. lib/kernel/bitmap.h:

​	头文件定义了位图的结构，包含一个位图的字节长度和字节指针

~~~c
#ifndef __LIB_KERNEL_BITMAP_H
#define __LIB_KERNEL_BITMAP_H
#include "global.h"
#define BITMAP_MASK 1 
struct bitmap{
	uint32_t btmp_bytes_len;				// the number of byte in this bitmap
	uint8_t* bits;						// the pointer of bitmap(unit is byte)
};

void bitmap_init(struct bitmap* btmp);
bool bitmap_scan_test(struct bitmap* btmp , uint32_t bit_idx);
int bitmap_scan(struct bitmap* btmp , uint32_t cnt);
void bitmap_set(struct bitmap* btmp , uint32_t bit_idx, int8_t value);

#endif
~~~

##### 2.lib/kernel/bitmap.c:

- bitmap_init：初始化位图，使用memset将bitmap的所有字节置0(未使用)
- bitmap_scan_test：判断该位是否使用，使用return1，未使用return0
- bitmap_scan：**连续**申请cnt个位，申请成功返回起始bit下标，失败返回-1
- bitmap_set：将指定位设置为指定值(1或0)

~~~c
#include "bitmap.h"
#include "stdint.h"
#include "string.h"
#include "print.h"
#include "interrupt.h"
#include "debug.h"

// initial bitmap
void bitmap_init(struct bitmap* btmp){
	memset(btmp->bits, 0 , btmp->btmp_bytes_len);
}

// judge the index in bitmap is it equal to 1 ,if == 1 return true; else return false
bool bitmap_scan_test(struct bitmap* btmp , uint32_t bit_idx){
	// index /8  to index the array 
	uint32_t byte_idx = bit_idx /  8;
	// index %8 to index the bit in the array
	uint32_t bit_odd = bit_idx  % 8 ;
	
	return btmp->bits[byte_idx] & (BITMAP_MASK << bit_odd);
}

//apply for  'cnt' bit  in bitmap ,if success return the index of begin bit ,else return -1 
int bitmap_scan(struct bitmap* btmp , uint32_t cnt){

	// get the index of byte 
	uint32_t idx_byte = 0 ;
	while ((0xff == btmp->bits[idx_byte]) && (idx_byte < btmp->btmp_bytes_len)){
		idx_byte++;
	}
	ASSERT(idx_byte < btmp->btmp_bytes_len);
	if (idx_byte == btmp->btmp_bytes_len){
		return -1;
	}

	//get the index of bit
	int idx_bit = 0 ;
	while ((uint8_t)(BITMAP_MASK << idx_bit) & btmp->bits[idx_byte]){
		idx_bit++;
	}

	// start of conserve bit
	int bit_idx_start = idx_byte * 8 + idx_bit;
	if (cnt == 1 ){
		return bit_idx_start;
	}

	//remaining bit
	uint32_t bit_left = (btmp->btmp_bytes_len * 8 - bit_idx_start);
	//current bit is remain,so count + 1 , and point to next
	uint32_t bit_next = bit_idx_start + 1 ;
	uint32_t count = 1;

	//calculate whether have 'cnt' remain bit,if have ,return the start index 
	bit_idx_start = -1 ;
	while(bit_left-- > 0 ) {
		if (!(bitmap_scan_test(btmp,bit_next))){
			// the bit of 'bit_next' is not use , so continue calculate
			count ++ ;
		}else {
			// the bit of 'bit_next' is use ,so restart
			count = 0 ;
		}
		if (count == cnt){
			bit_idx_start = bit_next - cnt + 1 ;
			break;
		}
		bit_next++;
	}
	
	return bit_idx_start;
}

// set the 'bit_idx' bit in the bitmap to 'value'
void bitmap_set(struct bitmap* btmp , uint32_t bit_idx , int8_t value){
	ASSERT((value == 0) || (value == 1));
	uint32_t byte_idx = bit_idx / 8 ;
	uint32_t bit_odd  = bit_idx % 8 ;
	if (value){
		btmp->bits[byte_idx] |= (BITMAP_MASK << bit_odd);
	}else{
		btmp->bits[byte_idx] &= (BITMAP_MASK << bit_odd);
	}
}
~~~



### 2.内存管理系统

#### 1.内存池规划：

##### a.物理内存池规划：

​	因为内核和用户进程都是运行在物理内存中，为了内核的正常运行，必须要给内核预留足够的内存，所以我们将物理内存划分成两部分，也就是分成俩个内存池，一个内核物理内存池，一个用户物理内存池。为了方便实现，这里我们将两个物理内存池大小设置为**相同**大小，即各占一半。

​	kernel/memory.c:物理内存池定义，同时定义内核物理内存池和用户物理内存池实例。

~~~c
struct pool{
	struct bitmap pool_bitmap;			//to manage the physic memory
	uint32_t phy_addr_start;			//the start of physic memory to manage
	uint32_t pool_size;				//the size of pool
};
struct pool kernel_pool,user_pool;
~~~

##### b.虚拟内存池规划：

​	在32位环境下，虚拟内存空间可以达到4GB，由于分页机制下，每个用户进程都有自己的4GB虚拟内存空间，各个用户进程以及内核的虚拟地址可以相同(当然，物理地址肯定不相同，这是分页机制完成的)，同时虚拟内存通常是连续的，但是物理内存可以连续也可以不连续。每个用户进程都应该有自己的虚拟内存池，用来映射对应的物理内存池。内核毫无疑问也是必须要有自己的虚拟内存池，即**一个任务一个虚拟内存池**。（注：这里我们先实现内核虚拟内存池，其他的将在以后实现用户进程时补充）。

​	kernel/memory.h:虚拟内存池定义

~~~c
//virtual address pool
struct virtual_addr{
	struct bitmap vaddr_bitmap;
	uint32_t vaddr_start;
};
~~~

​	kernel/memory.c:定义了内存虚拟内存池实例

~~~c
struct virtual_addr kernel_vaddr;
~~~

#### 2.位图的位置：

这里需要拓展一下PCB的知识：

- 任何进程都必须包含一个PCB，PCB必须完整，单独的占用一个物理页，也就是4kb。
- PCB的最低处是包含线程或进程的信息，如pid，线程状态等，最高处是线程或进程在0特权级下使用的栈。

在loader中我们设置内核使用的栈顶是0xc009f000(对应物理地址0x9f000)，因此可以推断0xc009e000是将来内核主线程的pcb。

​	一个页框大小为4kb，一个大小为页框的位图可表示128mb(4k X 4k X 8)，所以位图位置安排在0xc009a000，这样刚好是四个页框，可以表示512mb(其实一个页框就够了)，这样位图就位于低端1mb中(0x100000)之下了(可以不用位图管理自己了)。

kernel/memory.c:位图的地址

~~~c
#define MEM_BITMAP_BASE 0xc009a000 
~~~

#### 3.初始化物理内存池：

​	在最初的loader.S中我们将低端1mb之上的0x100000~0x101ff用作了页目录表和页表，所以我们的虚拟内存的0xc0100000~0xc0101ff不映射这部分物理地址，其实就是将物理地址可以开始映射的部分又提高到了0x101ff。

​	256：一页页目录表，第0和第768个页目录项指向同一个页表算作一页，第769~1022项共254项，总共254+1+1=256

~~~c
	uint32_t page_table_size = PG_SIZE * 256;

	uint32_t used_mem = page_table_size + 0x100000; 
~~~

kernel/memory.c:

​	mem_pool_init计算未使用的所有内存，并将三个内存池的位图初始化(置0)

​	内核物理内存池，用户物理内存池，内核虚拟内存池，依次从0xc009a000(物理内存0x9a000)定义。

​	

~~~c
#define K_HEAP_START 0xc0100000

//initial memory pool
static void mem_pool_init(uint32_t all_mem){
	put_str("  mem_pool_init start\n");

	uint32_t page_table_size = PG_SIZE * 256;

	uint32_t used_mem = page_table_size + 0x100000; 

	uint32_t free_mem = all_mem - used_mem;
	uint16_t all_free_pages = free_mem / PG_SIZE;

	uint16_t kernel_free_pages = all_free_pages /2 ;
	uint16_t user_free_pages = all_free_pages - kernel_free_pages;

	// a byte can express 8 pages
	uint32_t kbm_length = kernel_free_pages / 8 ;	
	uint32_t ubm_length = user_free_pages / 8;

	//kernel pool start address
	uint32_t kp_start = used_mem;
	//user pool start address
	uint32_t up_start = kp_start + kernel_free_pages * PG_SIZE;

	//initial
	kernel_pool.phy_addr_start = kp_start;
	kernel_pool.pool_size = kernel_free_pages * PG_SIZE;
	kernel_pool.pool_bitmap.btmp_bytes_len = kbm_length;
	kernel_pool.pool_bitmap.bits = (void*) MEM_BITMAP_BASE;				//0x9a000
	user_pool.phy_addr_start = up_start;
	user_pool.pool_size = user_free_pages * PG_SIZE;
	user_pool.pool_bitmap.btmp_bytes_len  =ubm_length;
	user_pool.pool_bitmap.bits = (void*)(MEM_BITMAP_BASE + kbm_length);		//follow on kernel pool bitmap

	/*******************print the memory pool information*****************/
	put_str("     kernel_pool_bitmap_start:");
	put_int((int)kernel_pool.pool_bitmap.bits);
	put_str("  kernel_pool_phy_address_start:");
	put_int(kernel_pool.phy_addr_start);
	put_str("\n");

	put_str("     user_pool_bitmap_start:");
	put_int((int)user_pool.pool_bitmap.bits);
	put_str("  user_pool_phy_address_start:");
	put_int(user_pool.phy_addr_start);
	put_str("\n");

	//set bitmap to 0
	bitmap_init(&kernel_pool.pool_bitmap);
	bitmap_init(&user_pool.pool_bitmap);

	/***************** initial the kernel virtual memory pool************/
	//initial the bitmap of kernel virtual address
	//used for maintain the kernel virtual address , so the size is equal to kernel pool 
	kernel_vaddr.vaddr_bitmap.btmp_bytes_len = kbm_length;
	kernel_vaddr.vaddr_bitmap.bits = (void*)(MEM_BITMAP_BASE + kbm_length + ubm_length);


	kernel_vaddr.vaddr_start = K_HEAP_START;
	bitmap_init(&kernel_vaddr.vaddr_bitmap);

	
	put_str("  mem_pool_init done \n");

}


void mem_init(){
	put_str("mem_init start\n");
	uint32_t mem_bytes_total = (* (uint32_t*)(0xb00));				//set at loader.S 
	mem_pool_init(mem_bytes_total);
	put_str("mem_init done\n");
}
~~~

#### 4.分配页内存(重头戏>_<)：

##### 1.内存池标记以及页表项或页目录项属性：

kernel/memory.h:

~~~c
//judge use which pool
enum pool_flags{
	PF_KERNEL =1 ,
	PF_USER = 2
};

#define PG_P_1 1 
#define PG_P_0 0 
#define PG_RW_R 0 
#define PG_RW_W 2 
#define PG_US_S 0
#define PG_US_U 4
~~~

kernel/memory.c:

##### 1.获取pde和pte的索引：

​	开启分页机制后，高10位索引pde，中间10位索引pte。**注意是索引**

~~~c
#define PDE_IDX(addr) ((addr & 0xffc00000) >> 22)	//get the hight 10 bit
#define PTE_IDX(addr) ((addr & 0x003ff000) >> 12)	//get the middle 10 bit
~~~

##### 2.申请虚拟内存：

​	这里仅实现了向内核申请，用户进程部分以后补充

~~~c
//apply for 'pg_cnt' virtual page in the 'pf' virtual memory pool
//success :return the begin of address , failed: return NULL
static void* vaddr_get(enum pool_flags pf,uint32_t pg_cnt){
	int vaddr_start =  0 , bit_idx_start = -1;
	uint32_t cnt = 0;
	if (pf == PF_KERNEL){
		bit_idx_start = bitmap_scan(&kernel_vaddr.vaddr_bitmap,pg_cnt);
		if (bit_idx_start == -1 ){
			return NULL;
		}
		while (cnt < pg_cnt){
			bitmap_set(&kernel_vaddr.vaddr_bitmap,bit_idx_start+cnt++,1);
		}
		vaddr_start = kernel_vaddr.vaddr_start + bit_idx_start * PG_SIZE;
	}else {
		// will add in future !!!!!!!!!!!!
	}
	return (void*) vaddr_start;
}
~~~

##### 3.获取指定虚拟内存的页目录项和页表项的地址：

​	超级超级绕的要来了x_x：

​	*先说正常情况：cpu处理32位的虚拟地址，先将高10位作为pde索引，找到对应页表地址，在使用中间10位，索引pte，找到物理页地址，最后用后12位直接用作偏移地址，加上刚才的物理页地址，得到最终物理地址。*

​	***最初我们设计页目录表时，设计最后一个页目录项是该页目录表的地址！*** 

###### 获取pte指针：

​	因为最后一个页目录项是页目录表的地址，所以我们用第1023项，最后一个页目录项，作为目标虚拟地址的**高10位**，此时地址就为`0xffc00000`。cpu处理这里完成后以为找到了页表的地址，实际上是页目录表的地址。



​	然后cpu将使用虚拟地址的中间十位来索引pte，我们需要的是现在cpu来索引pde(因为刚才获取的是页目录表地址)，所以这时就需要参数vaddr的高十位来凑目标虚拟地址的**中间10位**，此时地址就为` 0xffc00000 +( (vaddr & 0xffc00000 )   >> 10 ) `。cpu这里完成后以为获取了物理页的地址，实际上是页表的地址。



​	最后我们需要让cpu索引pte，获取到pte(**注意这里并没有寻址pte中的物理页地址，而是pte本身，所以需要偏移来获取**)，这是cpu会使用低12位来偏移求得目标地址，所以我们使用参数vaddr的pte部分来求偏移，因为是偏移，pte只是索引且pte大小为4byte，所以我们需要手动乘四来凑目标虚拟地址的**最后10位**，此时地址构成为`0xffc00000 + ((vaddr & 0xffc00000 ) >> 10 ) + PTE_IDX(vaddr) * 4)`

###### 获取pde指针：

​	同上思路一致，这里需要套两层娃，也就是高10位和中间10位均代表最后一个页目录项。

~~~c
//get the PTE pointer of vaddr;!!!!!
uint32_t* pte_ptr(uint32_t vaddr){
       uint32_t* pte = (uint32_t*) (0xffc00000 + ((vaddr & 0xffc00000) >> 10) +PTE_IDX(vaddr) * 4 );
       return pte;
}

//get the PDE pointer of vaddr;!!!!!
uint32_t* pde_ptr(uint32_t vaddr){
	uint32_t* pde = (uint32_t*)((0xfffff000) + PDE_IDX(vaddr) * 4);
	return pde;
}
~~~

##### 4.申请物理页：

kernel/memory.c:好理解

~~~c
//get the physic page from m_pool
static void* palloc(struct pool* m_pool){
	int bit_idx = bitmap_scan(&m_pool->pool_bitmap,1);
	if (bit_idx == -1 ){
		return NULL;
	}
	bitmap_set(&m_pool->pool_bitmap,bit_idx,1);
	uint32_t page_phyaddr = ((bit_idx * PG_SIZE) + m_pool->phy_addr_start);
	return (void*)page_phyaddr;
}
~~~

##### 5.虚拟地址和物理地址的映射：

kernel/memory.c:

​	需要注意的点是，页目录项(\*pde)必须先于页表项(\*pte)存在，也就是说必须要先创建好了页表，才能有对应的物理页存在，不然就会缺页中断。

~~~c
//make the map of _vaddr and _page_phyaddr in page table
static void page_table_add(void* _vaddr, void* _page_phyaddr){
	uint32_t vaddr = (uint32_t) _vaddr,page_phyaddr = (uint32_t) _page_phyaddr;
	uint32_t* pde = pde_ptr(vaddr);
	uint32_t* pte = pte_ptr(vaddr);
	// execute *pte must after pde had create,otherwise wile Page_Fault
	if(*pde & 0x00000001){
		//make sure pte is not exist 
		ASSERT(!(*pte & 0x00000001));
		if (!(*pte & 0x00000001)){
			*pte = (page_phyaddr | PG_US_U | PG_RW_W | PG_P_1);
		}else {
			PANIC("pte repeat");
			*pte = (page_phyaddr | PG_US_U | PG_RW_W | PG_P_1);
		}
	}else {
		//pde wasn't exist ,so create pde first
		uint32_t pde_phyaddr = (uint32_t) palloc(&kernel_pool);
		*pde = (pde_phyaddr | PG_US_U | PG_RW_W | PG_P_1);
		memset((void*)((int)pte & 0xfffff000) , 0 , PG_SIZE);
		
		ASSERT(!(*pte & 0x00000001));
		*pte = (page_phyaddr | PG_US_U | PG_RW_W | PG_P_1);
	}
}

~~~

##### 6.分配指定个页空间

kernel/memory.c:

分配流程：

- 从虚拟内存池中申请虚拟内存(vaddr_get)
- 从物理内存池中申请物理地址(palloc)
- 进行虚拟地址和物理地址的映射(page_table_add)

​	get_kernel_pages 是malloc_page的封装。

~~~c
//malloc 'pg_cnt' page memory
/***********************   malloc_page 	******************
 * 	1.vaddr_get :		apply for virtual memory in virtual memory pool
 * 	2.palloc: 		apply for physic memory in physic memory pool
 * 	3.page_table_add: 	make the map
 ******************************************************************/
void* malloc_page(enum pool_flags pf,uint32_t pg_cnt){
	ASSERT(pg_cnt > 0 && pg_cnt < 3840);		//here set max is 15*1024*1024 /4096 = 3840 page
	void * vaddr_start = vaddr_get(pf,pg_cnt);
	if (vaddr_start == NULL ){
		return NULL;
	}

	uint32_t vaddr = (uint32_t)vaddr_start , cnt = pg_cnt;
	struct pool* mem_pool = pf & PF_KERNEL ? &kernel_pool : &user_pool;

	//physic memory isn't continuous
	while (cnt-- > 0){
		void * page_phyaddr = palloc(mem_pool);
		if (page_phyaddr == NULL){
			//physic memory will roll back , will write in future
			return NULL;
		}
		page_table_add((void*)vaddr,page_phyaddr);
		vaddr += PG_SIZE;
	}
	return vaddr_start;
}

//apply for 'pg_cnt'  page in kernel physic memory pool
void* get_kernel_pages(uint32_t pg_cnt){
	void* vaddr = malloc_page(PF_KERNEL,pg_cnt);
	if (vaddr != NULL){
		memset(vaddr , 0 , pg_cnt * PG_SIZE);
	}
	return vaddr;
}


~~~



### 3.总结



*虚拟空间巧调度，进程安然各运行。*



*物理资源精分配，内存高效无忧愁。*

