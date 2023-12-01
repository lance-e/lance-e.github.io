# Go的部分基本数据类型的底层实现


<!--more-->

# 部分基本数据类型的底层实现

##### 基本类型的字节数

1.int大小根据系统字长

2.指针的大小也是系统字长

### 一.空结构体

1.空结构体的地址均相同（zerobase）（不被包含在其他结构体中）

2.空结构体主要是为了**节约内存**，内存占用为**0**

​		a.结合map：只想要key不要value时，可以将value类型设置为空结构体，实现hashset

​		b.结合channel：只想让channel发一个信号，而不想携带任何信息，可以发一个空结构体，当作纯信号

### 二.字符串

runtime包对string底层实现：

~~~go
type stringStruct struct {
	str unsafe.Pointer	//指向底层字节数组的指针
	len int				//字节长度
}
~~~

​	reflect包下的StringHeader对string相似底层实现（已被弃用）

~~~go
type StringHeader struct {
	Data uintptr
	Len  int
}
~~~

1.*字符串本质是个结构体*

2.Data指针指向底层的Byte数组

3.len表示Byte数组长度，而不是字符长度

##### 字符串特点

1.字符串是一个不可改变的字节序列，不可修改

2.字符串支持切片操作，不同位置的切片底层访问的是同一块内存数据

3.由于只读的特性，相同字符串面值常量通常对应同一个字符串常量

##### 字符串的访问

1.对字符串使用len方法得到的是字节数而不是字符数

2.对字符串直接使用下标访问，得到的是**字节**

3.字符串被for range遍历时，被解码成rune类型的**字符**

4.UTF-8 编码算法位于runtime/utf-8.go

##### Go中的字符编码

1.所有字符使用Unicode字符集

2.使用UTF-8编码

###### Unicode

1.一种统一的字符集

2.囊括了159种文字的144679个字符

3.14万个字符至少需要3个字节表示

4.英文字母均排在前128个

###### UTF-8

1.Unicode的一种**变长**格式

2.128个US-ASCII字符只需要一个字节编码

3.西方常用字符需要两个字节

4.其他字符需要3个字节，极少需要4个字节

### 四.切片

*切片的本质是一个结构体*

runtime包对slice的底层实现：

~~~go
type slice struct {
	array unsafe.Pointer	//指向底层数组
	len   int 				//切片引用的底层数组的那部分的长度
	cap   int 				//底层数组的长度就是切片的容量
}
~~~

##### 切片的创建

1.根据数组创建（使用下标）

2.字面量：编译时插入创建数组的代码

示例：

go代码

~~~go
s := []int{3, 2, 1}
~~~

汇编代码（截取）

~~~asm
        0x000e 00014 (E:/golang/GOPATH/src/GoProgram/main.go:6) LEAQ    type:[3]int(SB), AX
        0x0015 00021 (E:/golang/GOPATH/src/GoProgram/main.go:6) PCDATA  $1, $0
        0x0015 00021 (E:/golang/GOPATH/src/GoProgram/main.go:6) CALL    runtime.newobject(SB)
        0x001a 00026 (E:/golang/GOPATH/src/GoProgram/main.go:6) MOVQ    $3, (AX)
        0x0021 00033 (E:/golang/GOPATH/src/GoProgram/main.go:6) MOVQ    $2, 8(AX)
        0x0029 00041 (E:/golang/GOPATH/src/GoProgram/main.go:6) MOVQ    $1, 16(AX)

~~~

3.make：运行时创建数组

示例

用户代码：

~~~go
s := make([]int)
~~~

runtime包下的对make创建切片实现：

~~~go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.Size_, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.Size_, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
~~~

##### 切片的追加

1.不扩容时，只调整len（编译器负责）

2.扩容时，编译时转为调用 runtime.growslice(),使用二倍长的底层数组替代原来的底层数组，废弃原底层数组

3.期望容量大于当前容量的两倍，就会使用期望容量

4.如果当前切片的长度小于1024，将容量翻倍

5.如果当前切片的长度大于1024，每次增加25%

6.切片扩容时，**并发不安全**，注意切片并发要加锁

### 五.map

*map的底层实现是哈希表(散列表)，map的本质是指针，指向一个hmap结构体（A header for a Go map.）*

*hmap中的buckets字段指向bmap结构体（ A bucket for a Go map.），bucket（哈希桶）是map的存储结构*

##### HashMap 的基本方案：

1.开放寻址法

2.拉链法

##### Go 的map

go的map底层结构----->runtime.hmap:

~~~go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
~~~

buckets 字段(哈希桶)是一个指针，指向一个数组：一个由很多bmap组成的数组

bmap的结构------>runtime.bmap:

~~~go
// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
~~~

bmap包含的字段：

1.tophash：长度为8的数组，高八位，为了快速遍历

2.data：key和value，一个桶中可以放八个键值对，八个key放一个数组中，八个value放一个数组中

3.overflow：溢出bucket的地址（指向下一个bmap）

*key，value，overflow都不会显示出来，而是通过maptype计算偏移量获取。因为key，value的数据类型不确定，只有编译的时候才会将八个key和八个value放入哈希桶*

extra字段指向mapextra结构体：

mapextra结构------->runtime.mapextra:

~~~go
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
~~~

nextOverflow字段指向下一个可用的溢出桶

##### map的初始化

1.make创建：
汇编代码（截取）：可以看到底层调用了runtime.makemap()

~~~asm
        0x0020 00032 (E:/golang/GOPATH/src/GoProgram/main.go:8) CALL    runtime.makemap(SB)
~~~

runtime.makemap

~~~go
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
~~~

2.字面量

​		a.元素少于25个时，转化为简单赋值

​		b.元素多于25个时，转化为循环赋值

##### map的访问：

计算tophash

##### map的扩容

1.map 溢出桶太多时会导致严重的性能下降

2.runtime.mapassign()可能会触发扩容的情况：

​			a.装载因子超过 6.5（平均每个槽6.5个key）

​			b.使用了太多溢出桶（溢出桶超过了普通桶）

##### map的扩容的类型

1.等量扩容：数据不多但是溢出桶太多了

2.翻倍扩容：数据太多了

###### 扩容步骤一：

1.创建一组新桶

2.oldbuckets指向原有的桶数组

3.bukets指向新的桶数组

4.map标记为扩容状态

###### 扩容步骤二：

1.将所有的数据从旧桶驱逐到新桶

2.采用渐进式驱逐

3.每次操作一个旧桶时，将旧桶数据驱逐到新桶

4.读取时不进行驱逐，只判断读取新桶还是旧桶

###### 扩容步骤三:

1.所有的旧桶驱逐完成后

2.oldbuckets回收

##### map 的并发问题

1.map的读写有并发问题

2.A 协程在桶中读数据时，B 协程驱逐了这个桶

3.A 协程会读到错误的数据或者找不到数据

##### 并发问题的解决方案

1.给map加锁（mutax）（并发性能会被影响）

2.使用**sync.Map**

##### sync.Map

~~~go
type Map struct {
	mu Mutex						//互斥锁
	read atomic.Pointer[readOnly]	//存储readOnly结构体，其中包含m 和 amended字段
	dirty map[any]*entry			//原生的map
	misses int						//统计有多少次读取read没有命中
}
type readOnly struct {
	m       map[any]*entry			//原生的map
	amended bool 					// true if the dirty map contains some key not in m.
}
type entry struct {
	p atomic.Pointer[any]			
}
~~~

sync.Map实现的接口：

~~~go
type mapInterface interface {
	Load(any) (any, bool)
	Store(key, value any)
	LoadOrStore(key, value any) (actual any, loaded bool)
	LoadAndDelete(key any) (value any, loaded bool)
	Delete(any)
	Swap(key, value any) (previous any, loaded bool)
	CompareAndSwap(key, old, new any) (swapped bool)
	CompareAndDelete(key, old any) (deleted bool)
	Range(func(key, value any) (shouldContinue bool))
}
~~~

![image-20231107113301275](C:\Users\龙旭\AppData\Roaming\Typora\typora-user-images\image-20231107113301275.png)

场景：

1.正常读写

2.追加

3.追加后读写

4.dirty提升

5.正常删除

6.追加后删除

总结：

a.map在扩容时会有并发问题

b.sync.Map使用了两个map，分离了扩容问题

c.不会引发扩容问题的操作（查，改）使用read map

d.可能引发扩容的操作（新增）使用dirty map

### 六.接口

##### Go隐式接口特点

1.只要实现了接口的全部方法，就是自动实现接口

2.可以在不修改代码的情况下抽象出新的接口

（更加方便系统的扩展和重构）

##### 接口**值**的底层表示

接口数据使用runtime.iface表示：

~~~go
type iface struct {
	tab  *itab				//指向itab结构体，itab结构体中记录了接口类型信息和实现的方法
	data unsafe.Pointer		//指向底层真实数据
}
~~~

~~~go
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
~~~

##### 类型断言

1.类型断言是一个使用在接口**值**上的操作

2.可以将接口**值**转换为其他类型值（实现或者兼容接口)

3.可以配合switch进行类型判断

##### 结构体和结构体指针实现接口

![image-20231107172212881](C:\Users\龙旭\AppData\Roaming\Typora\typora-user-images\image-20231107172212881.png)

可查看plan9汇编

~~~go
go build -gcflags -S main.go
~~~

在使用结构体实现接口时，编译器会使用结构体指针再实现一遍接口，所以使用结构体和结构体指针进行初始化变量时，都不会报错。

但使用结构体指针实现接口时，编译器不会再用结构体再来实现接口，所以使用结构体初始化变量时会不通过

##### 空接口

1.runtime.eface结构体

~~~go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
~~~

2.空接口底层不是普通接口

3.空接口值可以承载任何数据

##### 空接口的用途

1.作为任意类型的函数入参

2.函数调用时，会新生成一个新的接口，再传参

### nil,空结构体，空接口区别

##### nil：

是六种类型的零值

每种类型的nil是不同的，无法比较

~~~go
// nil is a predeclared identifier representing the zero value for a
// pointer, channel, func, interface, map, or slice type.
var nil Type // Type must be a pointer, channel, func, interface, map, or slice type
~~~

##### 空结构体

1.Go中的非常特殊的类型

2.空结构体的值不是nil

3.空结构体的指针也不是nil，但是都相同（zerobase）

##### 空接口

1.空接口不一定是nil接口，（数据是nil，类型不是nil，就不是nil接口)

2.两个属性都是nil才是nil接口

