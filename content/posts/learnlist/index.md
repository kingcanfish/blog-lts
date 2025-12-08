---
title: 一些学习知识点

date: 2020-10-22T10:32:26+08:00

tag: [ 面试, 知识点, 操作系统, 计算机网络]

categories: 面试
description: 知识点杂记
---



## 操作系统

### LRU

+ LRU（Least recently used，最近最少使用）算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。 新数据插入到链表头部； 每当缓存命中（即缓存数据被访问），则将数据移到链表头部； 当链表满的时候，将链表尾部的数据丢弃。

+ 哈希表+ 双向链表的实现

  ```go
  //LRUCache 缓存
  type LRUCache struct {
  	size     int						// 缓存当前存在的元素数量
  	capacity int						// 缓存容量
  	cache    map[int]*Linkednode		// 哈希表,用于快速访问
  	head     *Linkednode
  	tail     *Linkednode
  }
  
  // Linkednode 数据
  type Linkednode struct {
  	key   int
  	value int
  	pre   *Linkednode
  	next  *Linkednode
  }
  
  func initLinkednode(key, value int) *Linkednode {
  	return &Linkednode{
  		key:   key,
  		value: value,
  	}
  }
  
  //Constructor 构造器
  func Constructor(capacity int) LRUCache {
  	l := LRUCache{
  		size:     0,
  		capacity: capacity,
  		cache:    make(map[int]*Linkednode),
  		head:     initLinkednode(0, 0),
  		tail:     initLinkednode(0, 0),
  	}
  	l.head.next = l.tail
  	l.tail.pre = l.head
  	return l
  
  }
  
  func (this *LRUCache) Get(key int) int {
  	if _, ok := this.cache[key]; !ok {
  		return -1
  	}
  
  	this.moveToHead(this.cache[key])
  	return this.cache[key].value
  
  }
  
  // Put 放入新节点
  func (this *LRUCache) Put(key int, value int) {
  	if _, ok := this.cache[key]; ok {
  		this.cache[key].value = value
  		this.moveToHead(this.cache[key])
  		return
  	}
  	newNode := initLinkednode(key, value)
  	this.cache[key] = newNode
  	this.addToHead(newNode)
  	this.size++
  	if this.size > this.capacity {
  		 node := this.deleteTail()
  		delete(this.cache, node.key)
  		this.size--
  	}
  	return
  }
  
  //addToHead 将这个节点添加到双向链表的头部
  func (this *LRUCache) addToHead(node *Linkednode) {
  	node.pre = this.head
  	node.next = this.head.next
  	this.head.next.pre = node
  	this.head.next = node
  	return
  }
  
  func (l *LRUCache) moveToHead(node *Linkednode) {
      //先删掉这个元素再把它放入头部
  	l.deleteNode(node)
  	l.addToHead(node)
  	return
  }
  
  func (l *LRUCache) deleteTail() *Linkednode {
  	node := l.tail.pre
  	l.deleteNode(node)
  	return node
  
  }
  
  func (l *LRUCache) deleteNode(node *Linkednode) {
  	node.next.pre = node.pre
  	node.pre.next = node.next
  	return
  }
  ```

### select, poll, epoll

1. 区别

### 内存

1. 为什么虚拟内存会远大于物理内存
2. 

# 网络

## http

1. https
   + HTTPS通信的建立机制 
   + 客户端发起 HTTPS 请求，服务端返回证书，客户端对证书进行验证，验证通过后本地生成用于改造对称加密算法的随机数，通过证书中的公钥对随机数进行加密传输到服务端，服务端接收后通过私钥解密得到随机数，之后的数据交互通过对称加密算法进行加解密。
2. post和get的区别 
3. 接口请求头有哪些参数，说五个并解释意思 
4. 

## tcp

1. 滑动窗口
2. 滑窗过小怎么办，提示有个算法
3. TCP有哪些机制可以完成可靠传输 
4. 2次握手行不行?为什么要3次 
5. 为什么要2倍timewait 
6. tcp半连接队列 
7. 

# 数据库

## mysql

1. MySQL 语句执行的过程 
2. MySQL 的缓存会失效吗
3. MySQL 主从同步机制，如果同步失败会怎么样？ 
4. MySQL 事务隔离界别有哪些？哪些情况下分别采取什么样的隔离级别 

## redis

1. 常见数据结构
2. 过期数据删除
   + **懒删除 + 定期删除**  懒删除:访问时判断有没有过期, 过期则删除,定期删除:定期遍历一部分键,过期则删除
3. 内存满了怎么办

# 算法

1. 判断是否环[链表](https://www.nowcoder.com/jump/super-jump/word?word=链表)，要找环位置怎么处理？ 
   + 快慢指针,快指针2步,慢指针1步 , 相遇则有环 , 在相遇的地方再在链表头放一个 指针ptr ,和慢指针一起走,他们一定会在入环的地方相遇

2. 如果一个很大的array中只有一个数出现1次，其他都出现2次。要找这个数。怎么做
   + 将所有数字进行异或,最后得到的值就是只出现一次的数
3. 找出一个很大array中出现次数超过一半的数，怎么做？ 
   + 排序,这个数一定在中间
   + 摩尔投票法: 投票法简单来说就是不同则抵消，占半数以上的数字必然留到最后。
4. 找 一个很大array中出现第k小的数怎么做？
   + topK 问题 建立一个 容量为K的小顶堆,最后一个数就是 topK小 ,也可以说是优先队列->手写堆排序,请

