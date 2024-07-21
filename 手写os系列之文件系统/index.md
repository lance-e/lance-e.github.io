# 手写OS系列之文件系统


<!--more-->

# 手写OS系列之文件系统!!!

> 文件系统是运行在操作系统的软件模块，是操作系统提供的一套管理磁盘文件读写的方法和数据组织，存储方式。因此文件系统=数据结构+算法。

### 1.硬盘

#### 1.相关概念

- 盘片：类似光盘的一个圆盘，布满磁性介质
- 扇区：硬盘读写的基本单位(簇或块是操作系统读写数据的单位)
- 磁道：盘片上的同心圆
- 磁头：读取磁性介质上数据的部件(一个盘片包含两个盘面，上面和下面，也就对应两个磁头)
- 柱面：不同盘面上的编号相同的磁道从上而下组成的圆柱体的回转面
- 分区：多个编号连续的柱面组成(物理上表现为某段范围内的所有柱面组成的通心环)
- 分区表：固定大小固定位置的数据结构，用来描述所有分区的信息。大小为64byte，表项为16byte，总共只能容纳4个分区。(分区表必须存在于MBR/EBR，前446byte为硬盘的参数和引导程序，然后才是64byte的分区表，最后2byte的魔数)
- 扩展分区：抽象，不具实体的分区，是支持任意数量的分区。为了区分概念，其他分区称为主分区。将该分区再划分出逻辑分区，才能像其他主分区一样使用。
- EBR：由于分区表大小固定，为了向上兼容，所以视MBR中的分区表中的扩展分区为总扩展分区，将它划分为多个子扩展分区，每个子扩展分区“逻辑上”相当于硬盘，因此每个子扩展分区可以有一个分区表，扩展分区采用链式结构将所有子扩展分区的分区表串起来。其分区表表项就包含逻辑分区的信息和描述下一个子扩展分区的地址。EBR存在于子扩展分区的第一个扇区，包含该子扩展分区的分区表，第一个表项是逻辑分区的信息，第二个表项描述下一个子扩展分区地址，第三第四表项未使用。

#### 2.硬盘驱动程序

**仅展示后续常用的函数，内部实现的一些函数请直接看仓库源码**

##### 读写硬盘

ide_read：从硬盘指定位置读取指定数量的扇区的数据到缓冲区中

- a.选择指定硬盘
- b.写入每次操作的扇区起始地址和扇区数
- c.向命令寄存器写入读命令
- d.判断硬盘现在是否可读
- d.从扇区中读数据

~~~c
//read 'sec_cnt' sector from hard disk into buffer
void ide_read(struct disk* hd , uint32_t lba , void* buf, uint32_t  sec_cnt){
	ASSERT(lba <= max_lba);
	ASSERT(sec_cnt > 0 );
	lock_acquire( &hd->my_channel->lock);

	// 1.select hard disk
	select_disk(hd);

	uint32_t secs_op;						//the number of sector every operate
	uint32_t secs_done = 0;						//the number of sector have done
	while(secs_done < sec_cnt){
		if ((secs_done + 256 ) <= sec_cnt){
			secs_op = 256 ;
		}else {
			secs_op = sec_cnt - secs_done;
		}

		// 2.write the number of sector and the start sector number
		select_sector(hd , lba + secs_done, secs_op);

		// 3.write the command into 'reg_cmd' register
		cmd_out(hd->my_channel , CMD_READ_SECTOR);


		/***************** the time to block itself  *************************
		 *  only can block itself when hard disk are working.
		 *  now the hard disk begin busy , 
		 *  block himself , wait for the interrupt handler to wake up itself
		 */
		sema_down(&hd->my_channel->disk_done);
		
		 /*  ******************************************************************/

		// 4.judge the status of hard disk whether readable or not
		if (!busy_wait(hd)){
			char error[64];
			sprintf(error , "%s read sector %d failed !!!!!\n",hd->name , lba);
			PANIC(error);
		}

		//5.read data into buf
		read_from_sector(hd , (void*)((uint32_t)buf + secs_done * 512) , secs_op);
		secs_done += secs_op;
	}
	lock_release(&hd->my_channel->lock);
}


~~~

ide_write：把缓冲区中指定数量扇区大小的数据写入到硬盘指定位置

- a.选择指定硬盘
- b.写入每次操作的扇区起始地址和扇区数
- c.向命令寄存器写入写命令
- d.判断硬盘现在是否可写
- d.向扇区中写数据

~~~c
//write 'sec_cnt' sector from buffer into hard disk
void ide_write(struct disk* hd , uint32_t lba , void* buf , uint32_t sec_cnt){
	ASSERT(lba <= max_lba);
	ASSERT(sec_cnt > 0);
	lock_acquire(&hd->my_channel->lock);

	// 1.select hard disk
	select_disk(hd);
	uint32_t secs_op;
	uint32_t secs_done = 0 ;
	while(secs_done < sec_cnt){
		if ((secs_done + 256 ) <= sec_cnt){
			secs_op = 256;
		}else {
			secs_op = sec_cnt - secs_done;
		}

		// 2.write the number of sector and the start sector number
		select_sector(hd , lba + secs_done , secs_op);

		// 3.write the command into 'reg_cmd' register
		cmd_out(hd->my_channel , CMD_WRITE_SECTOR);

		// 4.judge the status of hard disk whether readable or not
		if (!busy_wait(hd)){
			char error[64];
			sprintf(error , "%s write sector %d failed !!!!!\n",hd->name , lba);
			PANIC(error);
		}

		// 5.read data into buf
		write_to_sector(hd , (void*)((uint32_t)buf + secs_done * 512) , secs_op);
		
		//block itself when hard disk response
		sema_down(&hd->my_channel->disk_done);
		secs_done +=secs_op;
	}
	lock_release(&hd->my_channel->lock);
}

~~~

##### 扫描分区

partition_scan：扫描所有分区，对子扩展分区通过递归遍历逻辑分区

~~~c
//scan the all partition at 'ext_lba' sector in hard disk
static void partition_scan(struct disk* hd , uint32_t ext_lba){
	struct boot_sector* bs = sys_malloc(sizeof(struct boot_sector));
	ide_read(hd , ext_lba , bs , 1);	
	uint8_t index = 0 ;
	struct partition_table_entry* p = bs->partition_table;
	//traversal 4 partition
	while (index++ < 4 ){
		if (p->fs_type == 0x5){						//extension partition
			if (ext_lba_base != 0 ){				//sub extension partition
				partition_scan(hd , p->start_lba + ext_lba_base);
			}else {
				ext_lba_base = p->start_lba; 
				partition_scan(hd , p->start_lba);
			}
		}else if (p->fs_type != 0 ){
			if (ext_lba == 0){
				hd->prim_parts[p_no].start_lba = ext_lba + p->start_lba;
				hd->prim_parts[p_no].sec_cnt = p->sec_cnt;
				hd->prim_parts[p_no].my_disk = hd;
				list_append(&partition_list , &hd->prim_parts[p_no].part_tag);
				sprintf(hd->prim_parts[p_no].name , "%s%d" , hd->name , p_no + 1);
				p_no++;
				ASSERT(p_no < 4 );
			}else {
				hd->logic_parts[l_no].start_lba = ext_lba + p->start_lba;
				hd->logic_parts[l_no].sec_cnt = p->sec_cnt;
				hd->logic_parts[l_no].my_disk = hd;
				list_append(&partition_list , &hd->logic_parts[l_no].part_tag);
				sprintf(hd->logic_parts[l_no].name , "%s%d" , hd->name , l_no + 5);
				l_no++;
				if (l_no >= 8 )		//8 is our regulation
					return;
			}
		}

		p++;
	}
	sys_free(bs);
}

~~~

### 2.文件系统

#### 1.相关概念

- 块/簇：硬盘是低速设备，为了避免频繁访问硬盘，往往等到数据积累到一定量才一次性访问硬盘，一定的数据就是块(在我们的OS中，为了简单，一个块=一个扇区)
- inode(index node):为了硬盘的高利用率，文件系统采用了索引结构，文件中的块可以分散到不连续的零散空间。inode中存放文件系统为文件的所有块建立的索引表，就是块地址数组，每个数组元素都是块地址，同时inode中还包括inode号和文件相关信息，属于FCB的一种。**(为了防止文件使用过多块而导致inode过大，inode中一般前12个元素是直接块，也就是块的地址。当文件超过12个块后，inode的第13个元素存放的就不是块的地址，而是另外一个inode的地址，该inode继续存放文件超出的块，这些块叫做间接块，另外的这个inode为一级间接块表，同样的，当再次超出后，第14个元素将会存放二级间接块表第地址，以此类推)**
- inode_table：为了集中管理所有文件的inode，通过inode编号在inode_table索引到对应的inode
- 目录和文件：都是用inode来表示，不同点在于数据块中的数据不同，文件的数据块中是文件的数据，目录的数据块中是目录项
- 目录项：记录文件名，文件inode编号和文件类型，作用为将文件名和inode编号粘合，表示该inode指向的数据块的文件类型
- 超级块：保存文件系统元信息的元信息，固定在分区的第二个扇区**(整个硬盘为MBR+四个分区，每个分区中OBR+超级块+块位图+inode位图+inode表+根目录+空闲数据块...)**
- 文件描述符：用于描述文件的操作。每个进程都有单独且完全相同的一套文件描述符，所以每个进程的PCB中都有一个文件描述符数组，但是避免占用PCB过多，文件描述符表中存放**文件表**的**下标**(文件表是位于内存中的全局文件结构表，通过表中的文件结构中的inode编号，继续通过索引inode表获取inode)

#### 2.相关概念的数据结构

##### a.硬盘和分区：

~~~c
// the struct of partition
struct partition{
	uint32_t start_lba;				//the start sector
	uint32_t sec_cnt;				//the number of sector
	struct disk* my_disk;				//the disk of which this partition belong to
	struct list_elem part_tag;			//the tag of list
	char name[8];					//the name of partition
	struct super_block* sb;				//the super block
	struct bitmap block_bitmap;			//block bitmap
	struct bitmap inode_bitmap;			//inode bitmap
	struct list open_inode;				//the inode list of this partition
};

//the struct of hard disk
struct disk{
	char name[8];					//the name of hard disk
	struct ide_channel* my_channel;			//the channel if which this disk belong to
	uint8_t dev_no;					//this hard disk is main 0 or slave 1
	struct partition prim_parts[4];			//only 4 primary partition
	struct partition logic_parts[8];		//we set 8 logic partition (but limitless actually)
};
~~~



##### b.超级块：

- 超级块魔数
- 分区全部扇区
- 分区全部inode
- 分区起始扇区地址
- 块位图起始扇区地址
- 块位图本身占用扇区数
- inode位图起始扇区地址
- inode位图本身占用扇区数
- inode_table起始扇区地址
- inode_table本身占用扇区数
- 数据区第一个扇区号
- 根目录所在inode编号
- 目录项大小
- 后面的就是凑数的，凑够512一扇区大小

~~~c
//super block
struct super_block {
	uint32_t magic;							//magic number , mean different fs

	uint32_t sec_cnt;						//the all sectors in this partition
	uint32_t inode_cnt;						//the all inode in this partition
	uint32_t part_lba_base;						//the start lba address of in this partition

	uint32_t block_bitmap_lba;					//the start lba address of block bitmap 
	uint32_t block_bitmap_sects;					//the number of sectors 

	uint32_t inode_bitmap_lba;					//the start lba address of inode bitmap
	uint32_t inode_bitmap_sects;					//the number of sectors

	uint32_t inode_table_lba;					//the start lba address of inode table
	uint32_t inode_table_sects;					//the number of sectors

	uint32_t data_start_lba;					//the start lba address of first sector in data area
	uint32_t root_inode_no;						//the inode number of root 
	uint32_t dir_entry_size;					//size of directory

	uint8_t pad[460];						//gather enough 512 byte (1 sector)
} __attribute__ ((packed));
~~~

##### c.inode：

- inode编号
- inode大小->文件或目录大小
- inode的打开次数
- 写文件标识，不能并行写文件
- i_sectors:数据块索引表，存放inode的所有数据块地址以及间接块表地址
- inode_tag用于与双向链表配合使用

~~~c
//inode struct
struct inode{
	uint32_t i_no;							//the inode number
	uint32_t i_size;						//size of file or directory

	uint32_t i_open_cnts;						//the times of opened
	bool write_deny;						//can't write file concurrence(test this before process write )
	uint32_t i_sectors[13];						//0~11 is direct block pointer,12 is indirect block pointer
	struct list_elem inode_tag;				
};

~~~

##### d.目录：

- inode指针，且必须是打开的inode
- 目录中目录项索引的偏移量
- 目录的数据缓存

~~~c
//directory struct (unused in hard disk , just use in memory)
struct dir{
	struct inode* inode;
	uint32_t dir_pos;						//the offset of pos (the offset of dir_entry index)
	uint32_t dir_buf[512];						//data cathe in diretory
};
~~~

##### e.目录项：

- 文件名或目录名
- inode编号
- 文件类型

~~~c
//directory entry struct 
struct dir_entry{
	char filename[MAX_FILE_NAME_LEN];				//file or directory name
	uint32_t i_no;							//inode number
	enum file_types f_type;						//file type
};

~~~

##### f.文件结构：

- 文件读写指针偏移量
- 文件标识
- 文件inode指针

~~~c
//file struct
struct file{
	uint32_t fd_pos;		//the operate position of file
	uint32_t fd_flag;
	struct inode* fd_inode;
};
~~~



#### 3.创建文件系统

创建文件系统就是高级格式化分区：

- 根据分区大小，计算文件系统各元信息需要的扇区数及位置
- 内存中创建超级块，将上一步计算的元信息写入超级块
- 将超级块写入磁盘
- 将元信息(块位图+inode位图+inode表)写入磁盘各自位置
- 将根目录写入磁盘

~~~c
//format partition , create file system
static void partition_format( struct partition* part){
	uint32_t boot_sector_sects = 1;
	uint32_t super_block_sects = 1;
	uint32_t inode_bitmap_sects = DIV_ROUND_UP(MAX_FILES_PER_PART , BITS_PER_SECTOR);
	uint32_t inode_table_sects = DIV_ROUND_UP( 	\
		       	(sizeof(struct inode) * MAX_FILES_PER_PART) , SECTOR_SIZE );
	uint32_t used_sectors = boot_sector_sects +super_block_sects + inode_bitmap_sects + inode_table_sects;
	uint32_t free_sectors = part->sec_cnt - used_sectors;

	/********************** handle block bitmap sector*******************************/
	uint32_t block_bitmap_sects;
	block_bitmap_sects = DIV_ROUND_UP(free_sectors , BITS_PER_SECTOR);
	uint32_t block_bitmap_bit_len = free_sectors - block_bitmap_sects;	//the bitmap contain itself
	block_bitmap_sects = DIV_ROUND_UP(block_bitmap_bit_len , BITS_PER_SECTOR);
	/********************************************************************************/

	//initial super block
	struct super_block sb;
	sb.magic = 0x12345678; 						//random
	sb.sec_cnt = part->sec_cnt;
	sb.inode_cnt =MAX_FILES_PER_PART;
	sb.part_lba_base = part->start_lba;

	sb.block_bitmap_lba = sb.part_lba_base + 2;			//skip the obr and super_block
	sb.block_bitmap_sects = block_bitmap_sects;

	sb.inode_bitmap_lba = sb.block_bitmap_lba + sb.block_bitmap_sects;
	sb.inode_bitmap_sects = inode_bitmap_sects;

	sb.inode_table_lba = sb.inode_bitmap_lba + sb.inode_bitmap_sects;
	sb.inode_table_sects = inode_table_sects;

	sb.data_start_lba = sb.inode_table_lba + sb.inode_table_sects;
	sb.root_inode_no = 0;
	sb.dir_entry_size  = sizeof(struct dir_entry);

	printk("%s info:\n", part->name);
	printk(" magic:0x%x\n  part_lba_base:0x%x\n	\
			all_sectors:0x%x\n	\
			inode_cnt:0x%x\n	\
			block_bitmap_lba:0x%x\n	\
			block_bitmap_sectors:0x%x\n	\
			inode_bitmap_lba:0x%x\n	\
			inode_bitmap_sectors:0x%x\n	\
			inode_table_lba:0x%x\n	\
			inode_table_sectors:0x%x\n	\
			data_start_lba:0x%x\n"	\
			,sb.magic, sb.part_lba_base, sb.sec_cnt , sb.inode_cnt , 	\
			sb.block_bitmap_lba, sb.block_bitmap_sects , sb.inode_bitmap_lba,	\
			sb.inode_bitmap_sects, sb.inode_table_lba , sb.inode_table_sects,	\
			sb.data_start_lba);
	

	struct disk* hd = part->my_disk;

	/******************** write super block into the 1 sector of this partition ************/

	ide_write(hd , part->start_lba + 1 ,&sb ,1);

	printk("  super_block_lba:0x%x\n" , part->start_lba + 1);


	//get the large size of meta data , as the buffer size
	uint32_t buf_size = (sb.block_bitmap_sects >= sb.inode_bitmap_sects ? sb.block_bitmap_sects : sb.inode_bitmap_sects);
	buf_size = (buf_size >= sb.inode_table_sects ? buf_size : sb.inode_table_sects) * SECTOR_SIZE;
	uint8_t* buf = (uint8_t*)sys_malloc(buf_size);

	/****************** initial block bitmap , and write into the start lba address *******/

	buf[0] |= 0x01;							//the 0 block is offer to root
	uint32_t block_bitmap_last_byte = block_bitmap_bit_len / 8;
	uint32_t block_bitmap_last_bit = block_bitmap_bit_len % 8;
	
	//other unrelated byte in the last sector less than a sector size
	uint32_t last_size = SECTOR_SIZE - (block_bitmap_last_byte %  SECTOR_SIZE);	
	
	//initial the other unrelated bit to 1 (avoid use these corresponding resources)
	memset(&buf[block_bitmap_last_byte] , 0xff , last_size);
	uint8_t index = 0;
	while(index <= block_bitmap_last_bit){
		buf[block_bitmap_last_byte] &= ~(1 << index++);
	}
	
	ide_write(hd , sb.block_bitmap_lba , buf , sb.block_bitmap_sects);


	/******************** initial inode bitmap , and write into the start lba address********/

	//clear buffer
	memset(buf , 0 , buf_size);
	buf[0] |= 0x01;
	//inode bitmap sectors = 1, don't have the rest of last sectors,so don't need to handle
	
	ide_write(hd , sb.inode_bitmap_lba , buf , sb.inode_bitmap_sects);

	/******************** initial inode table , and write into the start lab address*********/

	//clear buffer
	memset(buf , 0 , buf_size);
	buf[0] |= 0x01;
	struct inode* i = (struct inode*)buf;
	i->i_size = sb.dir_entry_size * 2;	// "." and ".." 
	i->i_no = 0;				//root occupied  inode 0
	i->i_sectors[0] = sb.data_start_lba;

	ide_write(hd , sb.inode_table_lba , buf , sb.inode_table_sects);

	/*********************** write "." and ".." into root directory *************************/

	memset(buf ,0 , buf_size);
	struct dir_entry* p_de = (struct dir_entry*) buf;

	//initial current directory :"."
	memcpy(p_de->filename, ".", 1);	
	p_de->i_no = 0;
	p_de->f_type = FT_DIRECTORY;
	p_de++;
	
	//initial father direcotry of current direcotry :".."
	memcpy(p_de->filename , ".." , 2);
	p_de->i_no = 0 ;
	p_de->f_type = FT_DIRECTORY;

	//sb.data_start_lba had allocate to root directory , there are dir_entry of root directory
	ide_write(hd ,sb.data_start_lba , buf , 1);

	printk("  root_dir_lba:0x%x\n" , sb.data_start_lba);
	printk("%s format done\n" ,part->name);

	sys_free(buf);
}

~~~

#### 4.挂载分区

​	本质就是把该分区的元信息从硬盘读出来，加载到内存中，这样硬盘资源的变化都用内存中元信息来跟踪。如果有写操作，及时将内存中的元数据同步写入到硬盘以持久化

~~~c
struct partition* cur_part;		//default operation partition 
					
//get the partition named "part_name" from partition_list , and assign to "cur_part"
static bool mount_partition(struct list_elem* pelem , int arg){
	char* part_name = (char*) arg;
	struct partition* part = elem2entry(struct partition , part_tag , pelem);
	if (!(strcmp(part->name , part_name))){			//0 :mean equal
		cur_part = part;
		struct disk* hd = cur_part->my_disk;

		/*********** read the super_block from hard disk into memory **************/

		//sb_buf: storage the super_block read from hard disk
		struct super_block* sb_buf = (struct super_block*)sys_malloc(SECTOR_SIZE);

		//sb: in memory
		cur_part->sb = (struct super_block*)sys_malloc(SECTOR_SIZE);
		if (cur_part->sb == NULL){
			PANIC("alloc memory failed!");
		}

		//read super_block from hard disk
		memset(sb_buf , 0  , SECTOR_SIZE);
		ide_read(hd , cur_part->start_lba + 1 , sb_buf , 1);

		//copy the information of sb_buf into 'sb'
		//(copy can filterate the unused message)
		memcpy(cur_part->sb , sb_buf , sizeof(struct super_block));	

		/*********** read the block bitmap from hard disk into memory **************/

		cur_part->block_bitmap.bits = (uint8_t*)sys_malloc(sb_buf->block_bitmap_sects * SECTOR_SIZE);
		if (cur_part->block_bitmap.bits == NULL){
			PANIC("alloc memory failed!");
		}
		cur_part->block_bitmap.btmp_bytes_len = sb_buf->block_bitmap_sects * SECTOR_SIZE; 

		//read block block from hard disk
		ide_read(hd , sb_buf->block_bitmap_lba , cur_part->block_bitmap.bits , sb_buf->block_bitmap_sects);

		/********** read the inode bitmap from hard disk into memory **************/

		cur_part->inode_bitmap.bits = (uint8_t*)sys_malloc(sb_buf->inode_bitmap_sects * SECTOR_SIZE);
		if (cur_part->inode_bitmap.bits == NULL){
			PANIC("alloc memory failed!");
		}
		cur_part->inode_bitmap.btmp_bytes_len = sb_buf->inode_bitmap_sects * SECTOR_SIZE;

		//read inode bitmap from hard disk
		ide_read(hd , sb_buf->inode_bitmap_lba , cur_part->inode_bitmap.bits , sb_buf->inode_bitmap_sects);

		/****************/

		list_init(&cur_part->open_inode);
		printk("mount %s done!\n" , part->name);
		
		return true;					//just for "list_traversal": ture that stop

	}
	return false;
}
~~~

#### 5.文件相关操作

##### a.创建或打开文件：

- 先搜索路径，查找该文件是否存在
- 根据flags，创建(file_create)或者打开文件(file_open)

~~~c
//open or create file . success return file descriptor , fail return -1
int32_t sys_open(const char* pathname ,uint8_t flags){
	if (pathname[strlen(pathname) - 1] == '/' ){
		printk("can't open a directory %s\n", pathname);
		return -1;
	}

	ASSERT(flags <= 7);
	int32_t fd = -1;

	struct path_search_record searched_record ; 
	memset(&searched_record ,  0 , sizeof(struct path_search_record));
	
	int32_t path_depth = path_depth_cnt((char*) pathname);
	
	
	//determine if the file exist
	int inode_no =  search_file(pathname , &searched_record);
	bool found = inode_no == -1 ? false : true;

	if (searched_record.file_type == FT_DIRECTORY){
		printk("can't open a directory with open() , use opendir() to instead\n");
		dir_close(searched_record.parent_dir);
		return -1;
	}

	int32_t path_searched_depth = path_depth_cnt(searched_record.searched_path);
	if (path_searched_depth != path_depth){
		printk("cannot access %s: Not a directory , subpath %s is't exist\n",	\
				pathname , searched_record.searched_path);
		dir_close(searched_record.parent_dir);
		return -1;
	}

	if (!found && !(flags & O_CREAT)) {
		printk("in path %s , file %s isn't exist \n" , pathname , 	\
				(strrchr(searched_record.searched_path, '/') + 1));
		dir_close(searched_record.parent_dir);
		return -1;
	}else if (found && flags & O_CREAT){
		printk("%s has already exist!\n" , pathname);
		dir_close(searched_record.parent_dir);
		return -1;
	}

	switch(flags & O_CREAT){
		case O_CREAT:
			printk("creating file\n");
			fd = file_create(searched_record.parent_dir , (strrchr(pathname, '/' ) + 1) , flags);
			dir_close(searched_record.parent_dir);
			break;
		default:
			//other case :O_RDONLY , O_WRONLY ,O_RDWR
			fd = file_open(inode_no , flags);
			
	}

	//thie fd is the pcb's fd_table index , not the index of global file_table
	return fd;
}	
	
~~~

​	file_create:

- 创建inode：通过inode_bitmap_alloc，inode_init函数
- 在全局文件结构表中添加该文件，最后会再添加到PCB的fd表中，并返回fd
- 在父目录中安装目录项：通过create_dir_entry创建目录项，sync_dir_entry安装
- 同步内存中的元信息到硬盘：inode_sync,bitmap_sync

~~~c
//create file
int32_t file_create(struct dir* parent_dir , char* filename , uint8_t flag){
	void* io_buf = sys_malloc(1024);
	if (io_buf == NULL){
		printk("in file_create: sys_malloc for io_buf failed\n");
		return -1;
	}

	uint8_t rollback_step = 0;			//used for roll back
	
	//allocate for new file
	int32_t inode_no =  inode_bitmap_alloc(cur_part);
	if (inode_no == -1){
		printk("in file_create: inode_bitmap_alloc for inode_no failed\n");
		return -1;
	}

	//this node will used in "file_table" , so must allocate in heap memory
	struct task_struct* cur = running_thread();
	uint32_t* pg_dir_bak = cur->pgdir;
	cur->pgdir =  NULL;

	struct inode* new_file_inode = (struct inode*)sys_malloc(sizeof(struct inode));
	if (new_file_inode == NULL){
		printk("in file_create: sys_malloc for new_file_inode failed\n");
		rollback_step = 1;
		goto rollback;
	}
	cur->pgdir = pg_dir_bak;

	//initial
	inode_init(inode_no,new_file_inode);

	int fd_idx = get_free_slot_in_global();
	if (fd_idx == -1){
		printk("exceed max open file\n");
		rollback_step = 2;
		goto rollback;
	}

	file_table[fd_idx].fd_inode = new_file_inode;
	file_table[fd_idx].fd_pos = 0 ;
	file_table[fd_idx].fd_flag = flag;
	file_table[fd_idx].fd_inode->write_deny = false;

	//add this new dir_entry to parent direcotry
	struct dir_entry new_dir_entry ;
	memset(&new_dir_entry , 0 , sizeof(struct dir_entry));
	create_dir_entry(filename , inode_no , FT_REGULAR, &new_dir_entry);


	//!!!synchronous into hard disk
	
	// a. install new directory entry into parent directory
	if (!sync_dir_entry(parent_dir , &new_dir_entry , io_buf)){
		printk("sync dir_entry to disk failed \n");
		rollback_step = 3;
		goto rollback;
	}
	
	// b. sync file inode
	memset(io_buf , 0 , 1024);
	inode_sync(cur_part , parent_dir->inode , io_buf);
	// c. sync parent directory inode
	memset(io_buf , 0 , 1024);	
	inode_sync(cur_part, new_file_inode , io_buf);
	// d. sync inode bitmap
	bitmap_sync(cur_part , inode_no , INODE_BITMAP);

	// e. add new open file into open_inode list
	list_push(&cur_part->open_inode, &new_file_inode->inode_tag);
	new_file_inode->i_open_cnts = 1;

	sys_free(io_buf);
	return  pcb_fd_install(fd_idx);

	//roll back
rollback:
	switch(rollback_step){
		case 3:
			memset(&file_table[fd_idx] , 0 , sizeof(struct file));
		case 2:
			sys_free(new_file_inode);
			cur->pgdir = pg_dir_bak;
		case 1:
			bitmap_set(&cur_part->inode_bitmap , inode_no , 0 );
			break;
	}
	sys_free(io_buf);
	return -1;
}
~~~

​	file_open:

- 通过添加到file_table 和PCB中的文件描述符表
- 同时使用inode_open，打开inode

~~~c
//open file
int32_t file_open(uint32_t inode_no , uint8_t flag){
	int32_t fd_index = get_free_slot_in_global();
	if (fd_index == -1){
		printk("exceed max open files\n");
		return -1;
	}
	file_table[fd_index].fd_inode = inode_open(cur_part , inode_no);
	file_table[fd_index].fd_pos = 0;
	file_table[fd_index].fd_flag = flag;
	bool* write_deny = &file_table[fd_index].fd_inode->write_deny;

	if (flag & O_WRONLY || flag & O_RDWR ){
		enum intr_status old_status = intr_disable();
		if (!(*write_deny)){		
			*write_deny = true;
			intr_set_status(old_status);
		}else {		//the file is being written by someone else
			intr_set_status(old_status);
			printk("file can't be write now , try again later\n");
			return -1;
		}
	}
	//if read or create , don't need care "write_deny"
	return pcb_fd_install(fd_index);

}
~~~

##### b.关闭文件：

- 先获取file_table的下标，然后调用file_close关闭文件

~~~c
//close file
int32_t  sys_close(int32_t fd){
	int32_t ret = -1;
	if (fd > 2 ){
		uint32_t global_fd = fd_local2global(fd);
		ret = file_close(&file_table[global_fd]);
		running_thread()->fd_table[fd] = -1;
	}
	return ret;
}
~~~

​	file_close:

- 通过inode_close关闭inode

~~~c
//close file
int32_t file_close(struct file* file){
	if (file == NULL){
		return -1;
	}
	file->fd_inode->write_deny = false;
	inode_close(file->fd_inode);
	file->fd_inode = NULL;
	return 0;
}
~~~

##### c.写入文件：

- 如果是写入标准输出，就直接打印
- 如果是写入文件，就调用file_write

~~~c
//write "count" of  data from "buf" into file
int32_t sys_write(int32_t fd ,const void* buf , uint32_t count){
	if (fd < 0 ){
		printk("sys_write: fd error\n");		
		return -1;
	}
	if (fd == stdout_no){
		char tmp_buf[1024] = {0};
		memcpy(tmp_buf , (char*)buf , count);
		console_put_str(tmp_buf);
		return count;
	}
	uint32_t global_fd = fd_local2global(fd);
	struct file* wr_file = &file_table[global_fd];
	if (wr_file->fd_flag & O_WRONLY || wr_file->fd_flag & O_RDWR){
		uint32_t bytes_written = file_write(wr_file , buf ,count);
		return bytes_written;
	}else{
		console_put_str("sys_write: not allowed to write file without flag O_WRONLY or O_RDWR\n");
		return -1;
	}

}
~~~



​	file_write:

- 收集所有数据块地址到all_blocks,同时原先文件的数据块不够的话需要申请数据块，并同步到硬盘
- 循环写入数据块，通过ide_write

~~~c
//write "count" data of buffer into file
int32_t file_write(struct file* file , const void* buf , uint32_t count){
	if ((file->fd_inode->i_size + count ) > (BLOCK_SIZE * 140 )){
		//512 * 140 = 71680 , only support 140 block
		printk("exceed max file_size 71680 bytes , write file failed\n");		
		return -1;
	}
	uint8_t* io_buf = sys_malloc(512);
	if (io_buf == NULL){
		printk("file_write: sys_malloc for io_buf failed\n");
		return -1;
	}

	//used for record all block address of file
	uint32_t* all_blocks = (uint32_t*)sys_malloc(BLOCK_SIZE + 48);
	if (all_blocks == NULL){
		printk("file_write: sys_malloc for all_blocks failed\n");
		return -1;
	}

	int32_t block_lba = -1;			//block address
	uint32_t block_bitmap_idx = 0;		//the index of block_bitmap (used in block_bitmap_sync)
	int32_t indirect_block_table;		//first level indirect block table address
	uint32_t block_idx;			//block index

	const uint8_t* src = buf;		//data of will write
	uint32_t bytes_written = 0;		//record the written data size
	uint32_t size_left = count;		//record the not written data size
	uint32_t sec_idx ;			//sector index	
	uint32_t sec_lba ;			//sector address
	uint32_t sec_off_bytes;			//sector byte offset
	uint32_t sec_left_bytes;		//sector left byte
	uint32_t chunk_size;			//the size of data per write 
	
	//judge is this file first time to write (mean no block , need to allocate a  block)
	if (file->fd_inode->i_sectors[0] == 0 ){
		block_lba = block_bitmap_alloc(cur_part);
		if (block_lba == -1){
			printk("file_write: block_bitmap_alloc for block_lba failed\n");
			return -1;
		}
		file->fd_inode->i_sectors[0] = block_lba;

		//sync block bitmap in disk
		block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
		ASSERT(block_bitmap_idx != 0 );
		bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);
	}

	//the number of block already occupied before write
	uint32_t file_has_used_blocks = file->fd_inode->i_size / BLOCK_SIZE + 1;
	//the number of blocks to be occupied after write 
	uint32_t file_will_used_blocks = (file->fd_inode->i_size + count ) / BLOCK_SIZE + 1;

	ASSERT( file_will_used_blocks < 140);
	//the increment
	uint32_t add_blocks = file_will_used_blocks -  file_has_used_blocks;

	//-------------  collect all block address into "all_blocks" ---------------

	if (add_blocks == 0 ){		//no increment
		if (file_will_used_blocks <= 12){
			block_idx = file_has_used_blocks - 1 ;
			all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];
		}else {
			//if had used the indirect block before write , need read the indirect block first
			ASSERT(file->fd_inode->i_sectors[12] != 0);
			indirect_block_table  =  file->fd_inode->i_sectors[12];
			ide_read(cur_part->my_disk , indirect_block_table , all_blocks+12 , 1);
		}
		
	}else {		//have increment
		if (file_will_used_blocks <= 12){ 
			//1.get sector with remaining space for continued use , write into "all_blocks"
			block_idx = file_has_used_blocks - 1;
			ASSERT(file->fd_inode->i_sectors[block_idx] != 0 );
			all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];

			//2.allocate sectors will use in future , write into "all_blocks"
			block_idx = file_has_used_blocks;
			while (block_idx < file_will_used_blocks){
				block_lba = block_bitmap_alloc(cur_part);
				if (block_lba == -1){
					printk("file_write: block_bitmap_alloc for situation 1 failed\n");
					return -1;
				}
				ASSERT(file->fd_inode->i_sectors[block_idx] == 0);
				file->fd_inode->i_sectors[block_idx] = all_blocks[block_idx] = block_lba;

				//sync block bitmap in disk
				block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
				ASSERT(block_bitmap_idx != 0 );
				bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);

				block_idx++;
			}
		}else if (file_will_used_blocks > 12 && file_has_used_blocks <= 12){
			//1.get sector with remaining space for continued use , write into "all_blocks"
			block_idx = file_has_used_blocks - 1;
			all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];

			//2.create the first level indirect block table
			block_lba = block_bitmap_alloc(cur_part);
			if (block_lba == -1){
				printk("file_write: block_bitmap_alloc for situation 2 failed\n");
				return -1;
			}
			ASSERT(file->fd_inode->i_sectors[12] == 0 );
			file->fd_inode->i_sectors[12] = indirect_block_table = block_lba;

			//3.allocate sectors will use in future , write into "all_blocks"
			block_idx = file_has_used_blocks;
			while (block_idx < file_will_used_blocks){
				block_lba = block_bitmap_alloc(cur_part);
				if (block_lba == -1){
					printk("file_write: block_bitmap_alloc for situation 2 failed\n");
					return -1;
				}
				if (block_idx < 12){
					ASSERT(file->fd_inode->i_sectors[block_idx] == 0);
					file->fd_inode->i_sectors[block_idx] = all_blocks[block_idx] = block_lba;
				}else {
					//indirect block just write into all_blocks, all ok that sync
					all_blocks[block_idx] = block_lba;
				}

				//sync block bitmap in disk
				block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
				bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);

				block_idx++;
			}
			//sync all indirect block in disk
			ide_write(cur_part->my_disk , indirect_block_table , all_blocks+12 , 1);
		}else if (file_has_used_blocks > 12){
			ASSERT(file->fd_inode->i_sectors[12] != 0);
			//1.get the address 
			indirect_block_table = file->fd_inode->i_sectors[12];
			//2.get the all indirect block into "all_blocks"
			ide_read(cur_part->my_disk , indirect_block_table , all_blocks+12 , 1);
			//3.allocate sectors will use in future , write into "all_blocks"
			block_idx = file_has_used_blocks;
			while (block_idx < file_will_used_blocks){
				block_lba = block_bitmap_alloc(cur_part);
				if (block_lba == -1){
					printk("file_write: block_bitmap_alloc for situation 3 failed\n");
					return -1;
				}
				all_blocks[block_idx] = block_lba;

				//sync block bitmap in disk
				block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
				bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);

				block_idx++;
			}
			//sync all indirect block in disk
			ide_write(cur_part->my_disk , indirect_block_table , all_blocks+12 , 1);
		}
	}

	//------------------------- begin write data --------------------------------

	bool first_write_block = true;		//show is it has remaining space
	file->fd_pos = file->fd_inode->i_size -1;	

	while (bytes_written < count){
		memset(io_buf , 0 , BLOCK_SIZE);
		sec_idx = file->fd_inode->i_size / BLOCK_SIZE;
		sec_lba = all_blocks[sec_idx];
		sec_off_bytes = file->fd_inode->i_size % BLOCK_SIZE;
		sec_left_bytes = BLOCK_SIZE - sec_off_bytes;

		//judge the size of data in this writing
		chunk_size = size_left > sec_left_bytes ? sec_left_bytes : size_left; 
		
		if (first_write_block){
			ide_read(cur_part->my_disk , sec_lba , io_buf , 1);
			first_write_block = false;
		}

		//combinate old data and new data
		memcpy(io_buf + sec_off_bytes , (uint8_t*)src , chunk_size);
		ide_write(cur_part->my_disk , sec_lba , io_buf , 1);
		//printk("file write at lba 0x%x\n" , sec_lba);
		src += chunk_size;	//point to next data
		file->fd_inode->i_size += chunk_size; 	//update the size of file
		file->fd_pos += chunk_size;
		bytes_written += chunk_size;
		size_left -= chunk_size;
	}
	
	//synchronous inode in disk

	//buffer may need 2 sector , but "io_buf" just 512 byte , so allocate a new buffer
	uint8_t* sync_buf = sys_malloc(1024);
	if (sync_buf == NULL){
		printk("file_write: sys_malloc for sync_buf failed\n");
		return -1;
	}else {
		inode_sync(cur_part , file->fd_inode , sync_buf);
		sys_free(sync_buf);
	}
	sys_free(all_blocks);
	sys_free(io_buf);
	return bytes_written;
}

~~~

##### d.读取文件：

- 获取全局文件结构表索引，然后调用file_read，读入缓冲区

~~~c
//read "count" of data from file into "buf"
int32_t sys_read(int32_t fd ,void* buf , uint32_t count){
	if (fd < 0){
		printk("sys_read: fd error\n");
		return -1;
	}
	ASSERT(buf != NULL);
	uint32_t global_fd = fd_local2global(fd);
	return file_read(&file_table[global_fd] , buf , count);

}
~~~

​	file_read:

- 收集所有数据块地址到all_blocks
- 循环读取数据块，通过ide_read

~~~c
//read file
int32_t file_read(struct file* file , void* buf , uint32_t count){
	uint8_t* buf_dst = (uint8_t*)buf;
	uint32_t size = count , size_left = size;

	//if count more than the left size of file , so use the left size as the new targete count
	if ((file->fd_pos + count ) > file->fd_inode->i_size){
		size = file->fd_inode->i_size - file->fd_pos;
		size_left = size;
		if (size == 0 ){		//file is full
			return -1;
		}
	}
	
	uint8_t* io_buf = sys_malloc(512);
	if (io_buf == NULL){
		printk("file_read: sys_malloc for io_buf failed\n");
		return -1;
	}

	//used for record all block address of file
	uint32_t* all_blocks = (uint32_t*)sys_malloc(BLOCK_SIZE + 48);
	if (all_blocks == NULL){
		printk("file_read: sys_malloc for all_blocks failed\n");
		return -1;
	} 

	uint32_t block_read_start_idx = file->fd_pos / BLOCK_SIZE;
	uint32_t block_read_end_idx = (file->fd_pos + size) / BLOCK_SIZE;
	uint32_t read_block = block_read_end_idx - block_read_start_idx;
	
	ASSERT(block_read_start_idx < 139 && block_read_end_idx < 139);

	int32_t indirect_block_table;		//first level indirect block table address
	uint32_t block_idx;			//block index
	
	//-------------  collect all block address into "all_blocks" ---------------
	
	if (read_block == 0 ){
		ASSERT(block_read_start_idx == block_read_end_idx);
		if (block_read_end_idx < 12){
			block_idx = block_read_start_idx;
			all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];
		}else{		//need to read all indirect block 
			indirect_block_table = file->fd_inode->i_sectors[12];
			ide_read(cur_part->my_disk , indirect_block_table , all_blocks+12 , 1);
		}
	}else {
		if (block_read_end_idx < 12){
			block_idx = block_read_start_idx;
			while(block_idx <= block_read_end_idx){
				all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];
				block_idx++;
			}
		}else if (block_read_start_idx < 12 && block_read_end_idx > 12){
			block_idx = block_read_start_idx;
			while(block_idx <= 12 ){
				all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];
				block_idx++;
			}
			ASSERT(file->fd_inode->i_sectors[12] != 0);
			//read all indirect block
			indirect_block_table = file->fd_inode->i_sectors[12];
			ide_read(cur_part->my_disk , indirect_block_table , all_blocks+12 , 1);
		}else {
			ASSERT(file->fd_inode->i_sectors[12] != 0);
			//read all indirect block
			indirect_block_table = file->fd_inode->i_sectors[12];
			ide_read(cur_part->my_disk , indirect_block_table , all_blocks+12 , 1);

		}

	}

	//------------------------- begin read data --------------------------------
	uint32_t sec_idx ;			//sector index	
	uint32_t sec_lba ;			//sector address
	uint32_t sec_off_bytes;			//sector byte offset
	uint32_t sec_left_bytes;		//sector left byte
	uint32_t chunk_size;			//the size of data per write 
	uint32_t bytes_read  = 0 ;		//record the read data size

	while (bytes_read < size){

		memset(io_buf , 0 , BLOCK_SIZE);

		sec_idx = file->fd_pos / BLOCK_SIZE;		//different from write ,here use fd_pos
		sec_lba = all_blocks[sec_idx];
		sec_off_bytes = file->fd_pos  % BLOCK_SIZE;
		sec_left_bytes = BLOCK_SIZE - sec_off_bytes;

		//judge the size of data in this writing
		chunk_size = size_left > sec_left_bytes ? sec_left_bytes : size_left; 
		
		ide_read(cur_part->my_disk , sec_lba , io_buf , 1);

		//combinate old data and new data
		memcpy(buf_dst , io_buf + sec_off_bytes , chunk_size);
	
		buf_dst  += chunk_size;	//point to next data
		file->fd_pos += chunk_size;
		bytes_read += chunk_size;
		size_left -= chunk_size;
	}
	
	
	sys_free(all_blocks);
	sys_free(io_buf);
	return bytes_read;	
}
~~~

##### e.文件读写指针重定位：

- 通过不同的whence，设置指针位置

~~~c
//reset the offset pointer for file read and write operate
int32_t sys_lseek(int32_t fd , int32_t offset , uint8_t whence){
	if (fd < 0 ){
		printk("sys_lseek: fd error\n");
		return -1;
	}
	ASSERT(whence > 0 && whence <4 );
	uint32_t global_fd = fd_local2global(fd);
	struct file* file = &file_table[global_fd];
	int32_t new_pos = 0 ;
	int32_t file_size = (int32_t)file->fd_inode->i_size;
	switch (whence){
		case SEEK_SET:
			new_pos = offset;
			break;
		case SEEK_CUR:
			new_pos = (int32_t)file->fd_pos + offset;
			break;
		case SEEK_END:			//in this case , offset must less than 0
			new_pos = file_size + offset;
	}
	if (new_pos < 0 || new_pos > (file_size + 1)){
		return -1;	
	}
	file->fd_pos = new_pos;
	return file->fd_pos;
}
~~~

~~~c
//offset of file read and write position
enum whence{
	SEEK_SET = 1,
	SEEK_CUR ,
	SEEK_END
};
~~~

##### f.删除文件：

- 先通过search_file查找文件，判断是否存在
- 当文件存在且未被使用，调用delete_dir_entry删除父目录中该文件的目录项,调用inode_release释放inode相关资源

~~~c
//delete file (not directory)
int32_t sys_unlink(const char* pathname){
	//1. to inspect is it exist 
	struct path_search_record search_record;
	memset(&search_record , 0 , sizeof(struct path_search_record));
	int inode_no = search_file(pathname , &search_record);
	ASSERT(inode_no != 0 );
	if (inode_no == -1){
		printk("file %s not found\n" , pathname);
		dir_close(search_record.parent_dir);
		return -1;
	}else if (search_record.file_type == FT_DIRECTORY){
		printk("can't delete directory with sys_unlink() , you should use sys_rmdir() to instead\n");
		dir_close(search_record.parent_dir);
		return -1;
	}

	//2. to inspect is it using
	uint32_t file_idx = 0 ;
	while (file_idx < MAX_FILE_OPEN){
		if (file_table[file_idx].fd_inode != NULL && file_table[file_idx].fd_inode->i_no == (uint32_t)inode_no){
			break;
		}
		file_idx++;
	}

	//mean this file are using 
	if (file_idx < MAX_FILE_OPEN){
		printk("file %s is in use , not allow to delete\n", pathname);
		dir_close(search_record.parent_dir);
		return -1;
	}

	ASSERT(file_idx == MAX_FILE_OPEN);
	void* io_buf = sys_malloc(SECTOR_SIZE + SECTOR_SIZE);
	struct dir* parent_dir = search_record.parent_dir;
	//delete
	delete_dir_entry(cur_part , parent_dir , inode_no , io_buf);
	inode_release(cur_part , inode_no);
	sys_free(io_buf);
	dir_close(search_record.parent_dir);
	return 0;
}
~~~

​	inode_release:

- 回收数据块：全部先收集到all_blocks中，然后逐个回收，将块位图的位置0，并同步到硬盘
- 回收inode：inode位图的位置0，并同步到硬盘

~~~c
//release the inode's block and itself
void inode_release(struct partition* part , uint32_t inode_no ){
	struct inode* inode_to_del = inode_open(part , inode_no);
	ASSERT(inode_to_del->i_no == inode_no);

	//------------ 1. release all used block ---------------------
	
	uint8_t block_idx = 0 , block_cnt = 12;
	uint32_t block_bitmap_idx;
	uint32_t all_blocks[140] = {0};
	//1. first to collect direct block
	while(block_idx < 12){
		all_blocks[block_idx] = inode_to_del->i_sectors[block_idx];
		block_idx++;
	}
	//2. second to collect all indirect block and release first level indirect block table's block
	if (inode_to_del->i_sectors[12] != 0){
		//collect all indirect block
		ide_read(part->my_disk , inode_to_del->i_sectors[12] , all_blocks+12 , 1);
		block_cnt = 140;
		//release indirect block table's block
		block_bitmap_idx = inode_to_del->i_sectors[12] - part->sb->data_start_lba;
		bitmap_set(&part->block_bitmap , block_bitmap_idx , 0);
		bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);
	}
	//3. third to release all block (have cllect to "all_blocks")
	block_idx = 0 ;
	while (block_idx < block_cnt){
		if (all_blocks[block_idx] != 0 ){
			block_bitmap_idx = 0 ;			//use for next ASSERT
			block_bitmap_idx = all_blocks[block_idx] - part->sb->data_start_lba;
			ASSERT(block_bitmap_idx > 0);
			bitmap_set(&part->block_bitmap , block_bitmap_idx , 0);
			bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);
		}
		block_idx++;
	}

	//--------------- 2. release used inode ----------------------
	bitmap_set(&part->inode_bitmap , inode_no , 0 );
	bitmap_sync(cur_part , inode_no , INODE_BITMAP);

	/*****/// the inode_delect just used to debug , it is unnecessary 
	void* io_buf = sys_malloc(1024);
	inode_delete(part , inode_no , io_buf);
	sys_free(io_buf);
	/******/
	inode_close(inode_to_del);
}
~~~

​	delete_dir_entry:

- 收集所有块地址
- 遍历每一个块
- 在块中找目标目录项
- 找到了目录项，开始删除:
  - 当该目录项所在目录只有该目录项，且不是第一个目录(包含"."和"..",不能删)，就也把该目录(块)回收了:回收块位图的位，将块地址从inode的块地址表去掉
  - 只清空该目录项，同步到硬盘中

~~~c
//delete directory entry which inode_no is "inode_no" from parent directory "pdir"
bool delete_dir_entry(struct partition* part , struct dir* pdir , uint32_t inode_no ,void* io_buf){
	struct inode* dir_inode = pdir->inode;	
	uint32_t block_idx = 0 ;
	uint32_t all_blocks[140] = {0};
	
	//-------------  collect all block address into "all_blocks" ---------------
	
	while (block_idx < 12){
		all_blocks[block_idx] = pdir->inode->i_sectors[block_idx];
		block_idx++;
	}

	if (pdir->inode->i_sectors[12]){
		ide_read(part->my_disk , pdir->inode->i_sectors[12] , all_blocks+12 , 1);
	}

	uint32_t dir_entry_size = part->sb->dir_entry_size;
	uint32_t dir_entrys_per_sec = SECTOR_SIZE / dir_entry_size;
	struct dir_entry* dir_e = (struct dir_entry*)io_buf;
	struct dir_entry* dir_entry_found = NULL;
	uint32_t dir_entry_idx , dir_entry_cnt ;	//idx used to traversal , cnt used to judge is it need release  block
	bool is_dir_first_block = false;		// first block in a directory
	
	//------------  traversal all block , to get directory entry --------------
	block_idx = 0 ;
	while(block_idx < 140){
		is_dir_first_block = false;
		if (all_blocks[block_idx] == 0){
			block_idx++;
			continue;
		}
		dir_entry_idx = dir_entry_cnt = 0 ;
		//read sector
		memset(io_buf , 0 , SECTOR_SIZE);
		ide_read(part->my_disk , all_blocks[block_idx] , io_buf , 1);
		//traversal all directory entry in a sector
		while (dir_entry_idx < dir_entrys_per_sec){
			if ((dir_e + dir_entry_idx)->f_type != FT_UNKNOWN){
				if (!strcmp((dir_e + dir_entry_idx)->filename , ".")){
					is_dir_first_block = true;
				}else if (strcmp((dir_e + dir_entry_idx)->filename , "." ) && strcmp((dir_e + dir_entry_idx)->filename , ".." )){
					dir_entry_cnt++;	//count the number of dir_entry
					if ((dir_e + dir_entry_idx)->i_no == inode_no){
						ASSERT(dir_entry_found == NULL); //inode_no only one
						dir_entry_found = dir_e + dir_entry_idx;
					}
				}
			}
			dir_entry_idx++;
		}

		//if not found target directory entry , traversal next block
		if (dir_entry_found == NULL){
			block_idx ++;
			continue;
		}

		//if found target directory entry
		ASSERT(dir_entry_cnt >= 1);

		if (dir_entry_cnt == 1 && !is_dir_first_block){ 	//clear dir_e and release block
			// 1. delete in block bitmap
			uint32_t block_bitmap_idx = all_blocks[block_idx] - part->sb->data_start_lba;
			bitmap_set(&part->block_bitmap , block_bitmap_idx , 0 );
			bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);

			// 2.delete in inode's i_sectors 
			if (block_idx < 12){
				all_blocks[block_idx] = 0;
			}else {
				uint32_t indirect_block_idx = 12 ;
				uint32_t indirect_block_cnt = 0 ; // if only a indirect block ,will release the indirect block table's block
				while(indirect_block_idx < 140){
					if (all_blocks[indirect_block_idx] != 0){
						indirect_block_cnt ++;
					}
				}
				ASSERT(indirect_block_cnt >= 1);
				if (indirect_block_cnt > 1){
					all_blocks[indirect_block_idx]  = 0 ;
					ide_write(part->my_disk , dir_inode->i_sectors[12] , all_blocks+12 , 1);
				}else { 
					all_blocks[indirect_block_idx]  = 0 ;
					ide_write(part->my_disk , dir_inode->i_sectors[12] , all_blocks+12 , 1);
					uint32_t block_bitmap_idx = all_blocks[12] - part->sb->data_start_lba;
					bitmap_set(&part->block_bitmap , block_bitmap_idx , 0 );
					bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);
					dir_inode->i_sectors[12] = 0;
				}
			}
		}else { //only clear directory entry
			memset(dir_entry_found , 0 , dir_entry_size);
			ide_write(part->my_disk , all_blocks[block_idx] , io_buf , 1);
		}

		//update inode and sync in disk
		ASSERT(dir_inode->i_size >= dir_entry_size);
		dir_inode->i_size -= dir_entry_size;
		memset(io_buf , 0 , SECTOR_SIZE * 2);
		inode_sync(cur_part , dir_inode , io_buf);

		return true;

	}
	//not found target direcotry entry in all sector
	return false;
}
~~~

### 6.目录相关操作

##### a.创建目录：

- 确定目标目录不存在
- 创建inode：inode_bitmap_alloc , inode_init
- 新目录中分配块给该目录的目录项：block_bitmap_allc,并同步硬盘
- 新目录中添加"."和".."(必须)
- 将该目录添加到其父目录:create_dir_entry创建父目录的目录项 , sync_dir_entry同步到父目录
- 同步资源变动到硬盘

~~~c
//create directory
int32_t sys_mkdir(const char* pathname){
	uint8_t rollback_step = 0 ;
	void* io_buf = sys_malloc(SECTOR_SIZE * 2);
	if (io_buf == NULL){
		printk("sys_mkdir: sys_malloc for io_buf failed\n");
		return -1;
	}

	//------------------ 1.to find is it exist -------------------------
	
	struct path_search_record searched_record;
	memset(&searched_record , 0 , sizeof(struct path_search_record));
	int inode_no = -1;
	inode_no = search_file(pathname , &searched_record);
	if (inode_no != -1){
		printk("sys_mkdir: file or directory %s exist\n", pathname);
		rollback_step = 1;
		goto rollback;
	}else {
		//not found , but still to judge the subpath is it exist
		uint32_t pathname_depth = path_depth_cnt((char*)pathname);
		uint32_t path_searched_depth = path_depth_cnt(searched_record.searched_path);
		if (pathname_depth != path_searched_depth){
			printk("sys_mkdir: can't access %s : Not a directory ,subpath %s is't exist\n",pathname , searched_record.searched_path);
			rollback_step = 1;
			goto rollback;
		}
	}

	struct dir* parent_dir = searched_record.parent_dir;
	char* dir_name = strrchr(searched_record.searched_path , '/') + 1;

	//------------------  2.create inode for new directory ---------------
	
	inode_no = inode_bitmap_alloc(cur_part);
	if (inode_no == -1){
		printk("sys_mkdir: inode_bitmap_alloc for inode_no failed\n");
		rollback_step = 1;
		goto rollback;
	}
	struct inode new_dir_inode;
	inode_init(inode_no , &new_dir_inode);  //initial
	

	//-------  3.allocate a new block for direcotry entry to storage "." ,".." -----------
	uint32_t block_bitmap_idx = 0 ;
	int32_t block_lba = -1;
	block_lba = block_bitmap_alloc(cur_part);
	if (block_lba == -1){
		printk("sys_mkdir: block_bitmap_alloc for block_lba failed\n");
		rollback_step = 2;
		goto rollback;
	}
	new_dir_inode.i_sectors[0] = block_lba; 	//first directory entry
	//sync in disk
	block_bitmap_idx = block_lba -  cur_part->sb->data_start_lba;
	bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);

	//-------- 4.write "." and ".." into first direcotry entry ----------
	memset(io_buf , 0 , SECTOR_SIZE * 2);
	struct dir_entry* p_de = (struct dir_entry*)io_buf;

	memcpy(p_de->filename , "." , 1);
	p_de->i_no = inode_no;
	p_de->f_type = FT_DIRECTORY;
	p_de++;

	memcpy(p_de->filename , "..", 2);
	p_de->i_no = parent_dir->inode->i_no;
	p_de->f_type = FT_DIRECTORY;

	ide_write(cur_part->my_disk , new_dir_inode.i_sectors[0] , io_buf , 1);
	new_dir_inode.i_size = 2 * cur_part->sb->dir_entry_size;

	//-----  5.add direcotry entry of new directory in it's parent direcotry --------
	struct dir_entry new_dir_entry;
	memset(&new_dir_entry , 0 , sizeof(struct dir_entry));
	create_dir_entry(dir_name , inode_no , FT_DIRECTORY , &new_dir_entry);
	memset(io_buf , 0 , SECTOR_SIZE * 2);
	//sync dir_entry in disk
	if (!sync_dir_entry(parent_dir , &new_dir_entry , io_buf)){
		printk("sys_mkdir: sync_dir_entry for new_dir_entry failed\n");
		rollback_step = 2;
		goto rollback;
	}


	//-------------- 6.synchronous all new resours in disk -----------------
	//sync parent dir inode
	memset(io_buf , 0, SECTOR_SIZE * 2);
	inode_sync(cur_part , parent_dir->inode , io_buf);
	//sync new dir inode
	memset(io_buf , 0, SECTOR_SIZE * 2);
	inode_sync(cur_part , &new_dir_inode , io_buf);
	//sync inode block 
	bitmap_sync(cur_part , inode_no , INODE_BITMAP);

	sys_free(io_buf);
	dir_close(searched_record.parent_dir);
	return 0;

rollback:
	switch (rollback_step){
		case 2:
			//rollback the allocate inode 
			bitmap_set(&cur_part->inode_bitmap , inode_no , 0 );

		case 1:
			dir_close(searched_record.parent_dir);
			break;
	}
	sys_free(io_buf);
	return -1;
}
~~~

##### b.打开和关闭目录：

- 分别调用dir_open和dir_close

~~~c
//open directory
struct dir* sys_opendir(const char* name){
	ASSERT(strlen(name) < MAX_PATH_LEN);
	if (name[0]== '/' && (name[1] == 0 || name[1] == '.')){
		return &root_dir;
	}

	//to judge is it exist
	struct path_search_record searched_record;
	struct dir* ret = NULL;
	memset(&searched_record , 0 , sizeof(struct path_search_record));
	int inode_no = -1;
	inode_no = search_file(name , &searched_record);
	if (inode_no == -1){
		printk("In %s , sub path %s not exist \n",name , searched_record.searched_path); 
	}else{
		if (searched_record.file_type == FT_REGULAR){
			printk("%s is regular file\n" , name);
		}else if (searched_record.file_type == FT_DIRECTORY){
			ret = dir_open(cur_part , inode_no);
		}
	}
	dir_close(searched_record.parent_dir);
	return ret;
}

//close directory
int32_t sys_closedir(struct dir* dir){
	int32_t ret = -1;
	if (dir != NULL){
		dir_close(dir);
		ret = 0 ;
	}
	return ret;
}
~~~

##### c.读取一个目录项：

- 调用dir_read

~~~c
//read a directory entry at direcotry
struct dir_entry* sys_readdir(struct dir* dir){
	ASSERT(dir != NULL);
	return dir_read(dir);
}
~~~

​	dir_read:

- 收集所有块地址
- 遍历所有块地址
- 遍历一个块上的扇区
- 每次只返回一个目录项，该函数由主调函数循环调用，**通过dir_pos，来定位每次返回的地方**

~~~c
//read directory: read a directory entry
struct dir_entry* dir_read(struct dir* dir){
	struct dir_entry* dir_e = (struct dir_entry*)dir->dir_buf;	
	struct inode* dir_inode = dir->inode;
	uint32_t all_blocks[140] = {0};
	uint32_t block_cnt = 12;
	uint32_t block_idx = 0;
	uint32_t dir_entry_idx = 0 ;

	//get all blocks 
	while(block_idx < 12){
		all_blocks[block_idx] = dir_inode->i_sectors[block_idx];
		block_idx++;
	}
	if(dir_inode->i_sectors[12] != 0){
		ide_read(cur_part->my_disk , dir_inode->i_sectors[12] , all_blocks+12 , 1);
		block_cnt = 140;
	}
	block_idx = 0 ;
	
	uint32_t cur_dir_entry_pos = 0 ;	//record current offset of directory entry
	uint32_t dir_entry_size = cur_part->sb->dir_entry_size;
	uint32_t dir_entrys_per_sec = SECTOR_SIZE / dir_entry_size;
	
	//traversal all blocks
	while (dir->dir_pos < dir_inode->i_size){
		if (dir->dir_pos >= dir_inode->i_size){
			return NULL;
		}
		if (all_blocks[block_idx] == 0 ){
			block_idx++;
			continue;
		}
		memset(dir_e , 0 , SECTOR_SIZE);
		ide_read(cur_part->my_disk , all_blocks[block_idx]  , dir_e , 1);
		dir_entry_idx = 0 ;
		//traversal all directory entry in this sector
		while(dir_entry_idx < dir_entrys_per_sec){
			if((dir_e + dir_entry_idx)->f_type){
				if (cur_dir_entry_pos < dir->dir_pos){
					cur_dir_entry_pos += dir_entry_size;
					dir_entry_idx ++;
					continue;
				}
				ASSERT(cur_dir_entry_pos == dir->dir_pos);
				dir->dir_pos += dir_entry_size;
				return dir_e + dir_entry_idx;
			}
			dir_entry_idx++;
		}
		block_idx++;
	}
	return NULL;
}
~~~

##### d.删除目录：

- 先找目标目录
- 然后调用dir_remove删除

~~~c
//delete empty directory
int32_t sys_rmdir(const char* pathname){
	//first to judge is this path exist
	struct path_search_record searched_record;
	memset(&searched_record , 0 , sizeof(struct path_search_record));
	int inode_no = search_file(pathname , &searched_record);
	ASSERT(inode_no != 0 );
	int32_t ret = -1 ;
	if (inode_no == -1){
		printk("In %s , sub path %s not exist\n", pathname , searched_record.searched_path);
	}else{
		if (searched_record.file_type == FT_REGULAR){
			printk("%s is a regular file\n", pathname);
		}else {
			struct dir* dir = dir_open(cur_part , inode_no);
			if (!dir_is_empty(dir)){
				printk("directory %s is not empty, it is not allowed to delete a nonempty direcotry\n" , pathname);
			}else {
				if (!dir_remove(searched_record.parent_dir , dir)){
					ret = 0 ;
				}
			}
			dir_close(dir);
		}
	}
	dir_close(searched_record.parent_dir);
	return ret;
}
~~~

​	dir_remove:

- 与删除文件的实现类似，删除父目录中的目录项，回收目录的inode

~~~c
//remove directory from parent directory
int32_t dir_remove(struct dir* parent_dir , struct dir* child_dir){
	struct inode* child_dir_inode = child_dir->inode;	
	uint32_t block_idx =  1;
	while (block_idx < 13){ 	//only have a block at sector[0] , other must 0 
		ASSERT(child_dir->inode->i_sectors[block_idx] == 0);
		block_idx++;
	}
	void* io_buf = sys_malloc(SECTOR_SIZE * 2);
	if (io_buf == NULL){
		printk("dir_remove: sys_malloc for io_buf failed\n");
		return -1;
	}
	//delete directory entry
	delete_dir_entry(cur_part , parent_dir , child_dir_inode->i_no , io_buf);
	//release inode
	inode_release(cur_part , child_dir_inode->i_no);
	sys_free(io_buf);
	return 0;
}
~~~

##### e.获取和改变工作目录：

获取工作路径：

- 获取工作路径inode：从PCB中取
- 从下往上逐层找父目录，直到根目录。通过循环调用get_parent_dir_inode_nr 和get_child_dir_name

改变工作路径：

- 先查找指定路径是否存在
- 直接改变PCB中的当前路径

~~~c
//get current working directory
char* sys_getcwd(char* buf , uint32_t size){
	//if user process offer a NULL to buf , syscall must malloc for buf
	ASSERT(buf != NULL);
	void* io_buf = sys_malloc(SECTOR_SIZE);
	if (io_buf == NULL){
		return NULL;
	}

	struct task_struct* cur_thread  = running_thread();
	int32_t parent_inode_nr = 0 ;
	int32_t child_inode_nr = cur_thread->cwd_inode_nr;
	ASSERT(child_inode_nr >= 0 && child_inode_nr < 4096);
	
	if (child_inode_nr == 0){
		buf[0] = '/';
		buf[1] = 0;
		return buf;
	}
	memset(buf , 0 ,size);
	char full_path_reverse[MAX_PATH_LEN] = {0}; 	//full path buffer
	
	//from low to high : to find the final parent directory 
	while((child_inode_nr)){
		parent_inode_nr = get_parent_dir_inode_nr(child_inode_nr , io_buf);
		if (get_child_dir_name(parent_inode_nr , child_inode_nr , full_path_reverse, io_buf)== -1){
			sys_free(io_buf);
			return NULL;
		}
		child_inode_nr = parent_inode_nr;
	}
	ASSERT(strlen(full_path_reverse) <= size);

	//now get full path , but the order is reverse
	//---- begin remake full path
	char* last_slash;
	while((last_slash = strrchr(full_path_reverse , '/'))){
		uint16_t len =strlen(buf);
		strcpy(buf + len , last_slash);
		*last_slash = 0 ;
	}
	sys_free(io_buf);
	return buf;
}


//change current working directory to target path
int32_t sys_chdir(const char* path){
	int32_t ret = -1;
	struct path_search_record searched_record;
	memset(&searched_record , 0 , sizeof(struct path_search_record));
	int inode_no = search_file(path , &searched_record);
	if (inode_no != -1 ){
		if (searched_record.file_type == FT_DIRECTORY){
			running_thread()->cwd_inode_nr = inode_no;
			ret = 0;
		}else {
			printk("sys_chdir: %s is regular file or other!\n" , path);
		}
	}
	dir_close(searched_record.parent_dir);
	return ret;
}


~~~

get_parent_dir_inode_nr：获取父目录的inode编号

get_child_dir_name：在指定父目录中找指定子目录项的名字

~~~c
//get parent directory's inode number
static uint32_t get_parent_dir_inode_nr(uint32_t child_inode_nr , void* io_buf){
	struct inode* child_inode = inode_open(cur_part , child_inode_nr );
	uint32_t block_lba = child_inode->i_sectors[0];
	ASSERT(block_lba >= cur_part->sb->data_start_lba );
	inode_close(child_inode);
	ide_read(cur_part->my_disk , block_lba , io_buf , 1);
	struct dir_entry* dir_e = (struct dir_entry*)io_buf; 
	ASSERT(dir_e[1].i_no < 4096 && dir_e[1].f_type == FT_DIRECTORY);
	return dir_e[1].i_no;
}

//find child directory entry's name . from "p_inode_nr" to find "c_inode_nr"
static int get_child_dir_name(uint32_t p_inode_nr , uint32_t c_inode_nr , char* path , void* io_buf){
	struct inode* parent_dir_inode = inode_open(cur_part , p_inode_nr);
	uint8_t block_idx = 0 ;
	uint32_t all_blocks[140] = {0};
	uint32_t block_cnt = 12;
	
	//get all blocks

	while(block_idx < 12){
		all_blocks[block_idx] = parent_dir_inode->i_sectors[block_idx];
		block_idx++;
	}
	if (parent_dir_inode->i_sectors[12]){
		ide_read(cur_part->my_disk , parent_dir_inode->i_sectors[12] , all_blocks + 12, 1);
		block_cnt = 140;
	}
	inode_close(parent_dir_inode);

	struct dir_entry* dir_e = (struct dir_entry*)io_buf;
	uint32_t dir_entry_size = cur_part->sb->dir_entry_size;
	uint32_t dir_entrys_per_sec = SECTOR_SIZE / dir_entry_size;
	block_idx = 0;

	//traversal all block
	while(block_idx < block_cnt){
		if (all_blocks[block_idx]){
			ide_read(cur_part->my_disk , all_blocks[block_idx] , io_buf , 1);
			uint8_t dir_e_idx = 0 ;
			while (dir_e_idx < dir_entrys_per_sec){
				if ((dir_e + dir_e_idx)->i_no == c_inode_nr){
					strcat(path , "/");
					strcat(path , (dir_e+dir_e_idx)->filename);
					return 0;
				}
				dir_e_idx++;
			}
		}
		block_idx++;
	}
	return -1;
}
~~~

##### f.获取文件/目录属性：

- 通过查找到的文件或目录，给返回的文件状态结构赋值，然后返回该状态

~~~c
//get status of file
int32_t sys_stat(const char* path , struct stat* buf){
	if (!strcmp(path , "/" ) || !strcmp(path , "/.") || !strcmp(path , "/..")){
		buf->st_filetype = FT_DIRECTORY;
		buf->st_ino = 0 ;
		buf->st_size = root_dir.inode->i_size;
		return 0 ;
	}

	int32_t ret = -1;
	struct path_search_record searched_record;
	memset(&searched_record , 0 , 1);
	int inode_no = search_file(path , &searched_record);
	if (inode_no != -1){
		struct inode* obj_inode = inode_open(cur_part , inode_no);
		buf->st_size = obj_inode->i_size;
		inode_close(obj_inode);
		buf->st_filetype = searched_record.file_type;
		buf->st_ino = inode_no;
		ret = 0 ;
	}else {
		printk("sys_stat: can't search %s\n" , path);
	}
	return ret;
}
~~~

文件属性结构：

~~~c
//attribute of file
struct stat{
	uint32_t st_ino;					//inode number
	uint32_t st_size;					//size
	enum file_types st_filetype;				//file type
};
~~~



#### 7.基础函数

##### inode:

- inode_locate:获取inode的位置信息，存在inode_position中
- inode_sync:同步inode数据
- inode_open:打开inode，也就是在内核空间中分配内存(内核：为了被所有任务共享)
- inode_close:关闭inode，也就是在内核空间中释放内存
- inode_init:初始化inode

~~~c
//storage the position of inode
struct inode_position{
	bool two_sec;			//is it cross sector
	uint32_t sec_lba;		//sector number where the inode are
	uint32_t off_size;		//byte offset in this sector
};


//get the sector number and byte offset of inode
static void inode_locate(struct partition* part , uint32_t inode_no , struct inode_position* inode_pos){
	ASSERT(inode_no < 4096);	
	uint32_t inode_table_lba = part->sb->inode_table_lba;

	uint32_t inode_size = sizeof(struct inode);
	uint32_t off_size = inode_no * inode_size; 		//byte offset
	uint32_t off_sec = off_size  / 512;			//sector offset
	uint32_t off_size_in_sec = off_size % 512;		//offset in sector
	
	//judge whether cross two sector
	uint32_t left_in_sector = 512 - off_size_in_sec;
	if (left_in_sector > inode_size){
		inode_pos->two_sec =  false;
	}else {
		inode_pos->two_sec =  true;
	}
	inode_pos->sec_lba = inode_table_lba + off_sec;
	inode_pos->off_size = off_size_in_sec;
}

//write inode into partition
void inode_sync(struct partition* part , struct inode* inode , void* io_buf){
	uint8_t inode_no = inode->i_no;	
	struct inode_position inode_pos;
	inode_locate(part, inode_no , &inode_pos);
	ASSERT( inode_pos.sec_lba <= (part->start_lba + part->sec_cnt));

	//clear the struct member of inode : inode_tag , i_open_cnts is unused in hard disk 
	struct inode pure_inode;
	memcpy(&pure_inode , inode , sizeof(struct inode));	
	pure_inode.i_open_cnts = 0 ;
	pure_inode.write_deny = false;
	pure_inode.inode_tag.prev = pure_inode.inode_tag.next = NULL;

	char* inode_buf = (char*)io_buf;
	if (inode_pos.two_sec){
		//operate in sector , so read the before sector , spell the new data and write 
		ide_read(part->my_disk , inode_pos.sec_lba , inode_buf ,2);
		//spell
		memcpy((inode_buf + inode_pos.off_size) ,&pure_inode , sizeof(struct inode));
		//write the new spelled data
		ide_write(part->my_disk , inode_pos.sec_lba , inode_buf , 2);
	}else {
		//operate in sector , so read the before sector , spell the new data and write 
		ide_read(part->my_disk , inode_pos.sec_lba ,inode_buf , 1);
		//spell
		memcpy((inode_buf + inode_pos.off_size) ,&pure_inode , sizeof(struct inode));
		//write the new spelled data
		ide_write(part->my_disk , inode_pos.sec_lba , inode_buf , 1);
	}
}

//open inode by inode_no
struct inode* inode_open(struct partition* part , uint32_t inode_no){
	//first to find in alread opened inode list  
	struct list_elem* elem = part->open_inode.head.next;
	struct inode* inode_found;
	while (elem != &part->open_inode.tail){
		inode_found = elem2entry(struct inode , inode_tag , elem);
		if (inode_found->i_no == inode_no){
			inode_found->i_open_cnts ++;
			return inode_found;
		}
		elem = elem->next;
	}

	//not find in opened list , so read from hard disk and append to opened list
	struct inode_position inode_pos;
	inode_locate(part , inode_no , &inode_pos);

	//in order to make the new inode shared by all process.So add in kernel space
	//( make user process's page table = NULL , that malloc will create at kernel space
	struct task_struct* cur = running_thread();
	uint32_t* cur_pagedir_bak = cur->pgdir;
	cur->pgdir = NULL;
	inode_found = (struct inode*)sys_malloc(sizeof(struct inode));
	cur->pgdir= cur_pagedir_bak;

	char* inode_buf;
	if (inode_pos.two_sec){
		inode_buf = (char*) sys_malloc(1024);
		ide_read(part->my_disk , inode_pos.sec_lba , inode_buf , 2);
	}else {
		inode_buf = (char*) sys_malloc(512);
		ide_read(part->my_disk , inode_pos.sec_lba , inode_buf , 1);
	}

	memcpy(inode_found , inode_buf + inode_pos.off_size , sizeof(struct inode));
	list_push(&part->open_inode , &inode_found->inode_tag );
	inode_found->i_open_cnts = 1;
	
	sys_free(inode_buf);
	return inode_found;
}
	

//close inode or reduce the number of inode had opened 
void inode_close(struct inode* inode){
	enum intr_status old_status= intr_disable();
	if (--inode->i_open_cnts == 0 ){
		list_remove(&inode->inode_tag);
		struct task_struct* cur = running_thread();
		uint32_t* cur_pgdir_bak = cur->pgdir;
		cur->pgdir = NULL;
		sys_free(inode);
		cur->pgdir = cur_pgdir_bak;
	}
	intr_set_status(old_status);
}

//initial the target inode
void inode_init(uint32_t inode_no , struct inode* new_inode){
	new_inode->i_no = inode_no;
	new_inode->i_size = 0;
	new_inode->i_open_cnts = 0;
	new_inode->write_deny = false;
	uint8_t index = 0;
	while(index < 13){
		new_inode->i_sectors[index] = 0 ;
		index++;
	}
}
~~~

##### file:

- get_free_slot_in_global：在全局文件结构表中找到空位
- pcb_fd_install：将全局文件结构表索引安装到PCB的文件描述符表中
- inode_bitmap_alloc：分配一个inode位
- block_bitmap_alloc：分配一个block位
- bitmap_sync：同步硬盘的位图数据

~~~c
//file table
struct file file_table[MAX_FILE_OPEN];

//get a free slot in "file_table"
int32_t get_free_slot_in_global(void){
	uint32_t index =3 ;
	while (index < MAX_FILE_OPEN){
		if (file_table[index].fd_inode== NULL){
			break;
		}
		index ++;
	}
	if (index == MAX_FILE_OPEN){
		printk("exceed max open files\n");
		return -1;
	}
	return index;
}
		
//install the fd into thread/process 's fd_table
int32_t pcb_fd_install(int32_t global_fd_index){
	struct task_struct* cur = running_thread();
	uint8_t local_fd_index =  3 ;		//skip the stdin stdout stderr
	while (local_fd_index < MAX_FILES_OPEN_PER_PROC){
		if (cur->fd_table[local_fd_index] == -1){
			cur->fd_table[local_fd_index] = global_fd_index;
			break;
		}
		local_fd_index++;
	}
	if (local_fd_index == MAX_FILES_OPEN_PER_PROC){
		printk("exceed max open files_per_proc\n");
		return -1;
	}
	return local_fd_index;
}


//allocate inode bitmap
int32_t inode_bitmap_alloc(struct partition* part){
	int32_t index = bitmap_scan(&part->inode_bitmap, 1);
	if (index == -1){
		return -1;
	}
	bitmap_set(&part->inode_bitmap , index , 1);
	return index;
}

//allocate  data block  (we set block == sector == 512byte)
int32_t block_bitmap_alloc(struct partition* part){
	int32_t index = bitmap_scan(&part->block_bitmap , 1);
	if (index == -1){
		return -1;
	}
	bitmap_set(&part->block_bitmap , index , 1);
	return (part->sb->data_start_lba + index);
}

//synchronous the 512 byte in memory where 'bit_idx' at  into hard disk
void bitmap_sync(struct partition* part , uint32_t bit_idx, uint8_t btmp){
	uint32_t off_sec =  bit_idx / 4096;		//sector offset(8 * 512 )
	uint32_t off_size = off_sec * BLOCK_SIZE;	//byte offset
	uint32_t sec_lba ;
	uint8_t* bitmap_off;
	switch(btmp){
		case INODE_BITMAP:
			sec_lba = part->sb->inode_bitmap_lba + off_sec;
			bitmap_off = part->inode_bitmap.bits + off_size;
			break;
		case BLOCK_BITMAP:
			sec_lba = part->sb->block_bitmap_lba + off_sec;
			bitmap_off = part->block_bitmap.bits + off_size;
			break;
	}
	ide_write(part->my_disk , sec_lba , bitmap_off , 1);
}
~~~

##### dir：

- open_root_dir：打开根目录
- dir_open：打开目录，调用inode_open
- search_dir_entry：在父目录下搜索指定名字的目录项
- dir_close：关闭目录，调用inode_close
- create_dir_entry：创建目录项
- sync_dir_entry：将目录项写入父目录中

~~~c
struct dir root_dir;				//root direcotory

//open root directory
void open_root_dir(struct partition* part){
	root_dir.inode = inode_open(part , part->sb->root_inode_no);
	root_dir.dir_pos = 0;
}


//open dir by "inode_no"
struct dir* dir_open(struct partition* part , uint32_t inode_no){
	struct dir* pdir = (struct dir*)sys_malloc(sizeof(struct dir));
	pdir->inode = inode_open(part ,inode_no);
	pdir->dir_pos =  0;
	return pdir;
}


//search the "name" in direcotory "pdir" at partition "part" , and save at dir_entry "dir_e"
bool search_dir_entry(struct partition* part , struct dir* pdir , const char* name , struct dir_entry* dir_e){
	uint32_t block_cnt = 140;		//12 + 128
	uint32_t* all_blocks = (uint32_t*)sys_malloc(48 + 512);		//storage all block  pointer
	if (all_blocks == 0 ){
		printk("search_dir_entry : sysmalloc for all_blocks failed\n");
		return false;
	}

	uint32_t block_idx =  0 ;
	while (block_idx < 12){
		all_blocks[block_idx] = pdir->inode->i_sectors[block_idx];
		block_idx ++;
	}

	block_idx = 0 ;

	if (pdir->inode->i_sectors[12] != 0){			//indirect block exists
		ide_read(part->my_disk , pdir->inode->i_sectors[12] , all_blocks+12 , 1);
	}

	//now all_block had storage all sectors used in this file / directory
	
	uint8_t* buf = (uint8_t *) sys_malloc(SECTOR_SIZE);
	struct dir_entry* p_de = (struct dir_entry*)buf;
	uint32_t dir_entry_size = part->sb->dir_entry_size;	
	uint32_t dir_entry_cnt = SECTOR_SIZE / dir_entry_size;	//the number of dir_entry in a sector

	//begin to search in all blocks
	while(block_idx < block_cnt){
		if (all_blocks[block_idx] == 0){		//empty
			block_idx++;
			continue;
		}
		//not empty , read this sector
		ide_read(part->my_disk , all_blocks[block_idx] , buf , 1);

		uint32_t dir_entry_idx = 0;
		//traversal all the directory entry in this sector
		while(dir_entry_idx < dir_entry_cnt){
			if (!strcmp(p_de->filename , name)){
				memcpy(dir_e , p_de , dir_entry_size);
				sys_free(buf);
				sys_free(all_blocks);
				return true;
			}
			dir_entry_idx++;
			p_de++;
		}	
		block_idx++;
		p_de = (struct dir_entry*)buf;
		memset(buf , 0 , SECTOR_SIZE);
	}
	sys_free(buf);
	sys_free(all_blocks);
	return false;
}
	

//close dir
void dir_close(struct dir* dir){
	if (dir == &root_dir){
		return ;
	}
	inode_close(dir->inode);
	sys_free(dir);
}

//initial directory entry in memory
void create_dir_entry(char* filename , uint32_t inode_no ,uint8_t file_type, struct dir_entry* p_de){
	ASSERT(strlen(filename) <= MAX_FILE_NAME_LEN);
	memcpy(p_de->filename , filename , strlen(filename));
	p_de->i_no = inode_no;
	p_de->f_type = file_type;
}

//write directory entry into parent directory
bool sync_dir_entry(struct dir* parent_dir, struct dir_entry* p_de ,void* io_buf){
	struct inode* dir_inode = parent_dir->inode;
	uint32_t dir_size = dir_inode->i_size;
	uint32_t dir_entry_size  = cur_part->sb->dir_entry_size;

	ASSERT(dir_size % dir_entry_size == 0 );

	uint32_t dir_entrys_per_sec = (512 / dir_entry_size);

	int32_t block_lba = -1; 		//record block lba address

	uint8_t block_idx = 0;
	uint32_t all_blocks[140] = {0};		// storage 12 direct block and 128 indirect block 

	//12 direct block save in "all_blocks"
	while(block_idx < 12){
		all_blocks[block_idx] = dir_inode->i_sectors[block_idx];
		block_idx++;
	}

	//dir_e : use to tarversal the directory entry in sector
	struct dir_entry* dir_e = (struct dir_entry*)io_buf;

	int32_t block_bitmap_idx = -1;

	//begin to traversal all block to find free space 
	block_idx = 0;
	while (block_idx < 140){
		block_bitmap_idx = -1;
		if (all_blocks[block_idx]==0){
			block_lba  =  block_bitmap_alloc(cur_part);
			if (block_lba == -1){
				printk("alloc block bitmap for sync_dir_entry failed\n");
				return false;
			}
			//sync after allocate block
			block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
			ASSERT(block_bitmap_idx != -1);
			bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);
			
			block_bitmap_idx = - 1;
			if (block_idx < 12){		//direct block
				dir_inode->i_sectors[block_idx] = all_blocks[block_idx] = block_lba;	
			}else if(block_idx == 12){	//not allocate first level indirect block table (12 is mean the 0 th indirect block)
				dir_inode->i_sectors[12]  = block_lba;	//give to block table  
				
				//allocate a new block for indirect block
				block_lba = -1;
				block_lba = block_bitmap_alloc(cur_part);
				if (block_lba == -1){	//allocate failed , restore before data
					block_bitmap_idx = dir_inode->i_sectors[12] - 	\
							   cur_part->sb->data_start_lba;
					bitmap_set(&cur_part->block_bitmap , block_bitmap_idx , 0 );
					dir_inode->i_sectors[12] = 0 ;
					printk("alloc block bitmap for sync_dir_entry failed \n");
					return false;
				}
				//sync after allocate block
				block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
				ASSERT(block_bitmap_idx != -1);
				bitmap_sync(cur_part , block_bitmap_idx , BLOCK_BITMAP);

				all_blocks[block_idx] = block_lba;
				//write the address of 0th indirect block into indirect block table
				ide_write(cur_part->my_disk , dir_inode->i_sectors[12] , all_blocks + 12 , 1);

			}else {
				all_blocks[block_idx] = block_lba;
				//write the address of indirect block into indirect block table
				ide_write(cur_part->my_disk , dir_inode->i_sectors[12] , all_blocks + 12, 1);
			}

			//write the new directory entry "p_de" into the new allocate block
			memset(io_buf , 0 , 512);	//clear
			memcpy(io_buf , p_de , dir_entry_size);
			ide_write(cur_part->my_disk,   all_blocks[block_idx] , io_buf , 1);
			dir_inode->i_size += dir_entry_size;
			return true;

		}
		//block already exist ,read into memory ,and find empty directory entry in it
		ide_read(cur_part->my_disk , all_blocks[block_idx] , io_buf , 1);
		
		uint8_t dir_entry_idx = 0 ;
		while(dir_entry_idx < dir_entrys_per_sec){
			if((dir_e + dir_entry_idx)->f_type == FT_UNKNOWN){
				memcpy(dir_e + dir_entry_idx , p_de, dir_entry_size);
				ide_write(cur_part->my_disk , all_blocks[block_idx] , io_buf , 1);
				dir_inode->i_size += dir_entry_size;
				return true;
			}				
			dir_entry_idx++;
		}
		block_idx++;
	}
	printk("directory is fall!\n");
	return false;
}
~~~

##### 路径搜索：

- path_parse：解析最上层的路径，存储到name_store，返回去掉最上层的路径
- path_depth_cnt：返回路径深度，通过循环调用path_parse
- search_file：搜索指定文件，返回其inode编号，searched_record用来备份各层父目录inode编号

~~~c
//parse the path :
//eg: "///a/b" , name_store : a , return pathname: /b. 
static char* path_parse(char* pathname , char* name_store){
	if (pathname[0] == '/' ){
		while(*(++pathname) == '/');		//skip lisk this: "////a/b"
	}
	while(*pathname != '/' && *pathname!= 0 ){
		*name_store++ = *pathname++;
	}
	if (pathname[0] == 0){
		return NULL;
	}
	return pathname;
}

//return the depth of path
int32_t path_depth_cnt(char* pathname){
	ASSERT(pathname != NULL);
	char* p = pathname;
	char name[MAX_FILE_NAME_LEN];

	uint32_t depth = 0 ;
	p = path_parse(p , name);
	while (name[0]){
		depth++;
		memset(name , 0 , MAX_FILE_NAME_LEN);
		if (p){					//p != NULL, continue
			p = path_parse( p, name);
		}
	}
	return depth;
}


//search file
static int search_file(const char* pathname , struct path_search_record* searched_record){
	//if search root dir , than direct return
	if (!strcmp(pathname , "/" ) || !strcmp(pathname ,"/.") || !strcmp(pathname , "/..")){
		searched_record->searched_path[0] = 0;			
		searched_record->parent_dir =  &root_dir;
		searched_record->file_type = FT_DIRECTORY;
		return 0 ;
	}
	uint32_t path_len = strlen(pathname);
	ASSERT(pathname[0] == '/' && path_len > 1 && path_len < MAX_PATH_LEN);

	char* sub_path = (char*) pathname;
	struct dir* parent_dir = &root_dir;
	struct dir_entry dir_e;
	char name[MAX_PATH_LEN] = {0};			//record parsed name after path_parse
	
	searched_record->parent_dir = parent_dir;
	searched_record->file_type = FT_UNKNOWN;
	uint32_t parent_inode_no = 0 ;			//inode number of parent directory
								
	sub_path = path_parse(sub_path, name);
	while (name[0]){
		ASSERT(strlen(searched_record->searched_path) < 512);

		strcat(searched_record->searched_path , "/");
		strcat(searched_record->searched_path , name);
	
		if (search_dir_entry(cur_part , parent_dir , name ,&dir_e)){

			memset(name , 0 , MAX_FILE_NAME_LEN);

			if (sub_path){
				sub_path = path_parse(sub_path , name);
			}
			if (dir_e.f_type == FT_DIRECTORY){		//directory
				parent_inode_no = parent_dir->inode->i_no;
				dir_close(parent_dir);	
				parent_dir = dir_open(cur_part , dir_e.i_no);
				searched_record->parent_dir = parent_dir;
				continue;
			}else if (dir_e.f_type == FT_REGULAR){		//file
				searched_record->file_type = FT_REGULAR;
				return dir_e.i_no;
			}
		}else {	 //not found
			//don't close the parent_dir , will used in create
			return -1;
		}
	}


	//only find directory, so return it's parent directory
	dir_close(searched_record->parent_dir);
	searched_record->parent_dir = dir_open(cur_part , parent_inode_no);
	searched_record->file_type = FT_DIRECTORY;

	return dir_e.i_no;
}
~~~

~~~c
//record the parent directory
struct path_search_record{
	char searched_path[MAX_PATH_LEN];			//parent path
	struct dir* parent_dir;					//parent directory
	enum file_types file_type;				//file type
};
~~~

### 8.总结：

​	文件系统完工！！！

​	后续会添加面试题...

