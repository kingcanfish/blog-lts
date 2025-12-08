---

title: Go和Redis中的Hash

date: 2020-05-30T21:43:24+08:00

tag: [Go, Redis, Hash, 底层原理]

categories: GO
cover: https://static.guoxy.top/img/image-20200617115140687.png
descrcript: Go和Redis中的Hash
---



之前看  *Redis设计与实现*  的时候看了一些字典的底层实现是基于hash表的，然后这几天看到Golang 的map 时也是基于hash表的，~~实际上大部分开发语言的map数据类型基本上都是基于hash表的~~ 所以就打算写一篇比较垃圾的博客~~集百家之长~~把这两个东西的底层实现放在一块讲了，加深下自己的映像。（由于水平不够，如果有错误还请一起探讨指出）

## Hash

[散列表](https://baike.baidu.com/item/散列表/10027933)（Hash table，也叫哈希表），是根据关键码值(Key value)而直接进行访问的[数据结构](https://baike.baidu.com/item/数据结构/1450)。也就是说，它通过把关键码值映射到表中一个位置来访问记录，以加快查找的速度。这个映射函数叫做[散列函数](https://baike.baidu.com/item/散列函数/2366288)，存放记录的[数组](https://baike.baidu.com/item/数组/3794097)叫做[散列表](https://baike.baidu.com/item/散列表/10027933)。

给定表M，存在函数f(key)，对任意给定的关键字值key，代入函数后若能得到包含该关键字的记录在表中的地址，则称表M为哈希(Hash）表，函数f(key)为哈希(Hash) 函数。

> 摘自百度百科

## Redis 中的 Hash  

### 结构图

> 图源 *Redis 设计与实现*

![image-20200603091130859](https://static.guoxy.top/img/image-20200603091130859.png)

### Hash 表  

`Redis` 中的hash表由以下结构定义:

```c
typedef struct dictht {
    dictEntry **table; //哈希桶数组
    unsigned long size; //哈希表的大小,就是 哈希桶数组长度
    unsigned long sizemask; //哈希表大小掩码，数值上为size-1，用于计算索引值
    unsigned long used; //哈希表已有的节点数量
}
```

+ table 为一个 数组，每个数组里都存放着一个一组指向 `dict.h/dictEntry`  结构的指针，然后每一个 `dicEntry`  结构 保存着 一个键值对，这个结构 也称为`HashBucket` ，哈希桶
+ used 表示 这个 hash表中现在已有的键值对的数量

### Hash 表节点（哈希桶）

 `dictEntry`  结构定义:

```c
typedef struct dictEntry {
    void *key; //键
    unoin{
        void *val; // 值
        unit64_t u64;
        int64_t s64;
    } v;
    struct dictEntry *next;
} dictEntry;
```

+ `next` 属性是指向另一个 哈希表节点的指针，该指针可以将多个 hash 键值对连在一起，用来解决哈希冲突的问题。

### 字典

`Redis` 中的字典由`dict.h/dict` 结构表示，也就是我们 `Redis` 用到的数据结构的最外层了

```c
typedef struct dict {
    dictType *type; //某类型的特定函数
    void *privdata; //私有数据
    dictht ht[2]; //这里包含了两个Hash 表
    int rehashidx; // rehash索引，用来表示当前rehash的状态
}
```

~~好累啊，下次再更新~~我回来了

+ `ht`属性为包含两个项的数组，数组中的每个项都是一个 `dictht`  的哈希表，一般情况下 `redis` 只会使用第一个hash表，第二个hash表时用来rehash的
+ `hashidx`用来记录rehash的状态
  + 没有进行rehash的时候，值为-1
+ type  属性是一个指向dictType 类型的指针, 这个结构里面包含了一些用于操作特定类型键值对的函数，比如：
  + 计算哈希值的函数（hashFunction）
  + 复制键的函数
  + 复制值的函数
  + 对比键的函数
  + 销毁键的函数
  + 销毁值的函数

### 哈希算法

当添加新键值对进redis 时，redis 首先要根据键计算出哈希值和索引值，再根据索引值将包含键值对的hash表节点（hash桶）放到哈希表数组（哈希桶数组）的指定索引上面

计算路径：

```c
hash = dict -> type -> hashFunction
```

```c
index = hash & dict -> ht[x].sizemask
```

+ 通过每一种 type 的hash函数计算出hash值，然后和哈希表中的掩码进行按位与运算得到index值
+ 使用的算法为 [MurmurHash2](https://zh.wikipedia.org/wiki/Murmur哈希)

### 哈希冲突

+ redis 使用链地址法 ( separate chaining ) 来解决哈希冲突，每个哈希桶都有一个next指针来构成单链表
+ 为了考虑速度，redis 总是将新节点添加到链表头的位置(因为没有指向链表尾的指针，所以如果放在链表尾需要遍历整个链表)

### Rehash

#### 为什么要rehash？

> 随着操作的不断的执行，哈希表保存的键值对会相应增加或者减少,如果一个索引上链接这N多键值对而其他索引上有没有键值对~~那和遍历一个链表有什么区别~~ 
>
> 所以为了让哈希表的负载因子维持在一个合理的范围内，就需要对hash表进行相应的扩容和收缩
>
> *rehash是指重新计算键的hash值和index值*
>
> *负载因子=ht[0].used /  ht[0].size*



#### rehash 步骤

0. + redis 没有执行`BGSAVE` 或者 `BGREWRITEAOF` 命令时，负载因子大于 1 开始扩展
   + 执行`BGSAVE` 或者 `BGEWRITEAOF` 时，负载因子大于 5 开始拓展
   + 负载因子小于 0.1 时开始收缩

1. 创建适当空间用于 ht[1] ，空间大小取决于是要扩展收缩以及现在ht[0] 中存在的键值对数量 (`dict->ht[0]->used`)
   + **拓展**: 创建新 hashtable 的大小为 第一个大于等于两倍的现在已存在的键值对数量的2的n次方
   + ~~说起来有点绕，我自己都不知道在说什么~~
   + 假如现在存了7个键值对，那么就是要扩展到大于2*7=14 个的 hashtable
   + 2^0 =1 不够，2^1不够，2^2不够，2^3不够，2^4 够了够了谢谢妈妈妈妈真好！就它了！所以新创建的表大小为16~
   + **收缩**：新hashtable的大小为第一个大于等于现在已存在的键值对数量的2的n次方
   + 一样举例，比如现在hashtable 大小为8，但是只存了三个数据，就缩小到4
2. 将 ht[0] 上的键值对 rehash 到 ht[1] 上，具体怎么rehash 请查看后面的 **渐进式rehash** 
3. 迁移完成后，释放 ht[0] , 将 ht[1] 设为 ht[0] ， 同时创建新的 ht[1] 准备下一次 rehash

#### 渐进式 rehash

顾名思义，就是慢慢慢慢 rehash， 并不是一次性做完

1. `rehashidx`  设置为 0 ,表示 rehash 工作正式开始
2. 当对某个键做出增删查改的操作时，redis 都会将该键索引上的键值对 rehash 到 ht[1],同时rehashidx 加1
3. 当 rehash 全部完成后，rehashidx 重新设置为 -1 

*在rehash 过程中，查找会先在 ht[0] 上查找，如果没有则到 ht[1] 上查找；而添加 则只会在 ht[1] 上添加。保证 ht[0] 只增不减*



## Go 中的  Hash

###  结构图

<img src="https://static.guoxy.top/img/go-map.png" style="zoom: 50%;" />

### hash 表

Go 中的 hash 表在 `/src/runtime/map.go` 中定义

```go
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
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
```

Go 的源码注释一直是很完善的，想必看注释也看得八九不离十了

+ `count`  属性 就是 `len(map)` 调用得到的值，map 中存在的键值对的数量
+ ` flag  `属性暂时还不清楚
+ `B` 属性用来标记hash 桶的数量，是2的指数，也就是 B 为3 的时候 hash bucket 的数量为8
+ ` noverflow`  属性是 overflow 的 bucket 近似数
+ ` hash0`  属性是哈的种子，它能为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时作为参数传入
+ `oldbuckets` 是哈希在扩容时用于保存之前 `buckets` 的字段，它的大小是当前 `buckets` 的一半

### hash 桶

```go
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
	// Followed by an overflow  pointer.
}
```

看似这个结构里只有一个属性，其实在编译期间会对 该结构进行重建，重建得到的 struct 如下

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

![image-20200606221605785](https://static.guoxy.top/img/image-20200606221605785.png)

+ 上面猛男色对应的就是 `topbits` / `tophash`
+ 青色对应的是 `keys`
+ 黄色对应的是 `values` 
+ 基佬色：每个桶只能装8个键值对，当溢出时，就需要链接 一个溢出桶，你懂的

### Key 的寻找过程

+ 当一个 key 甩过来让你找，当然是用 hash 函数计算 hash 值辣，那么就能得到一个 64 位的 hash 值（32 位CPU得到的是32位，是时候退出群聊了）
+ 假设 `hmap.B`  为 5， 那么一共就有 2^5 也就是 32 个 hash 桶
+ 经过计算后得到下面的hash值

```
 10010111  000011110110110010001111001010100010010110010101010  01010
```

+ 我们对后 `B`  位 ( 这里是5 )进行对 `B`  取余操作 ( 实现时是用按位操作，取余开销太大，这里说取余是为了方便理解 )，得到值为 10，所以要找的值在10 号桶里
+ 进入 10 号桶，再取 hash 值的高 8 位，就是前面 `bmap` 中定义的 `tophash`了，然后对比，桶里面 `tophash` 在哪个位置那么 键 和 值 就在哪个位置
+ 如果 8 个`tophash` 都没有的话，进入溢出桶继续寻找，如果走遍溢出桶都没有，那就是真没有

+ *word is cheap show me the picture*

<img src="https://static.guoxy.top/img/goMapFindKey.png" style="zoom: 33%;" />

### rehash 

+ 同样这里有一个装载因子的概念，Go源码给的定义是这样的：

```
loadFactor := count / (2^B)
```

+ 扩容条件：
  + 装载因子超过阈值，源码里定义的阈值是 6.5
  + overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15

#### 增量扩容

当满足第一个扩容条件时触发增量扩容

+ 创建一组新桶，桶的数量是旧桶数量的两倍
+ 将原来的桶 设置到 `oldbuckets` 上，再将新桶设置到 `buckets` 上，同理 `overflow` 也设置到 `oldoverflow` 上 
+  渐进式 hash：go 只在 插入、更新、删除 操作上会发生搬迁工作，而且每次最多搬迁两个 `bucket` 
+  搬迁时 怎么确定 落入哪个新桶？
  + 由之前的知识我们知道落入哪个桶时由 hash 值的后 B 位决定的
  + 比如 扩容前B是 ⑤，所以由后 5 位 决定，现在扩容成了 ⑥ ， 所以就在加一位，由 6 位决定
  + 这样能保证 旧桶的能够落到两个新桶内 
+ 当搬迁完毕时 `oldbuckets` 和 `oldoverflow` 设为 `nil`

*所以当每次进行插入，更新，删除的操作时，会判断 `oldbuckets` 状态 来确定是否在扩容中*

#### 等量扩容

当满足第二个条件时进行等量扩容

*可能是由于大量插入再大量删除键值对造成的，会导致键值对在桶（以及 `overflow` ）中过于分散，就像一座城由很多房子，很多是空的，只有几栋房子有人家，我们就很难找到他们，那只要我们将他们集中在某个小区中，我们就能够快速的找到他们 ~*

当扩容完毕后，所有的键值对都集中到了相对集中的 `bucket` 上，`overflow` 上的键值对就拜拜辣

## 总结

###  相同点

+ `hashtable` -> `hashbucket` -> `key/value`

+ 都使用拉链法来解决 hash 冲突，但是也有不同的是 `Redis` 是链接 键值对，`Go` 链接溢出桶
+ rehash 时都采用渐进式 rehash

### 不同点

+ `Redis` 一个桶只能存一个键值对； `Go` 一个桶存可以存 8 个键值对

+ 确定位置时 `Redis` 使用的是 hash 值 和 表掩码为与 得到 索引值 确定位置； ` Go ` 用 hash 值得 高8位和低 B 位确定位置



## 参考

>*《Redis 设计与实现》*
>
>  *[cch123/golang-notes](https://github.com/cch123/golang-notes/blob/master/channel.md)*
>
>  *[Go 设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)*

