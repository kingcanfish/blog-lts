---
title: Go的channel 的小学习

date: 2020-06-11T14:31:00+08:00

tag: [Go, channel, 底层原理]

categories: GO
cover: https://static.guoxy.top/img/image-20200611233011241.png

description: channel 的小学习
---

## channel

GO 语言的 channel  是一个比较神奇的东西，也可以说在 go 协程通信里面是不可或缺的一部分，现在就整理一下channel 相关的知识，下面的内容并非所有都是原创，只是简单的说一下可能比较重要的地方  具体的用法我也不多说，网上的教程很多

源码如下，位置在  ` src/runtime/chan.go`

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}

type waitq struct {
	first *sudog
	last  *sudog
}
```

+ `qcount`  是 chan 中已有元素的数量， `dataqsiz` 循环队列的 节点数
+ `buf` 是一个 循环队列，缓冲区
+ `sendx` 为写入数据的索引位，之后的数据写在环形队列的这个索引位置上；
+ `recvx` 读取数据的索引位下一次读取数据就从这个位置开始读取
+ `elemtype` chan 元素类型， 一个chan 只可以有一种类型
+ `recvq` 读取的 goroutine ( 源码里成为G，我觉得大概是一个意思) 的阻塞队列; `sendq` 是写入 chan 的 goroutine 的阻塞队列
+  `lock` 互斥锁，不允许并发读写,仅允许一个goroutine 进行操作，也就是说串行的
+  `elemsize` 代表类型大小，方便在缓冲队列中定位元素的位置



## 向 channel 写数据

简单过程如下：

1. 如果等待接收队列recvq不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从recvq取出G,并把数据写入，最后把该G唤醒，结束发送过程
2. 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程
3. 如果缓冲区中没有空余位置，将待发送数据写入 G，将当前G加入 `sendq`，进入睡眠，等待被读 goroutine 唤醒



<img src="https://static.guoxy.top/img/chan-03-send_data.png"  />

## 向 Channel 读数据

1.  如果等待发送队列 `sendq` 不为空，且没有缓冲区，直接从 `sendq` 中取出G，把G中数据读出，最后把G唤醒，结束读取
2. 如果等待发送队列 `sendq` 不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把发送队列第一个G的数据写入缓冲区尾部，把G唤醒，结束读取过程
3. 如果等待发送队列 `sendq` 为空，但是缓冲区中有数据，则从缓冲区取出数据，结束读取过程
4. 如果等待发送队列 `sendq` 为空，且缓冲区没数据 ，将当前 `goroutine` 加入 `recvq`，进入睡眠，等待被写 `goroutine` 唤醒



![](https://static.guoxy.top/img/chan-04-recieve_data.png)

## 举个例子

拿 餐厅的厨师，窗口， 服务员为例子

+ 厨师就是向 chan 写入的
+ 窗口就是缓冲队列
+ 服务员就是向 chan 读取的



写入数据：

+ 厨师准备上菜，看看窗口外有没有服务员，如果有服务员，就叫醒服务员拿菜，然后把菜给他
+ 没有服务员的话就看看窗口上菜有没有放满，如果没放满，就把菜放在窗口上，然后溜了
+ 如果窗口满了或者没有窗口，那就在窗口旁边睡觉，等待服务员把我叫醒拿菜

读取数据：

+ 先看看有没有睡觉的厨师，如果有，再看看窗口有没有菜，如果也有，那就从窗口拿走一个菜，然后叫醒厨师可以把菜放下了
+ 如果窗口放不了菜（无缓存队列） 直接叫醒厨师把他手上的菜拿过来
+ 如果没有睡觉的厨师，窗口有菜拿的话直接从窗口拿菜，拿完之后就溜了溜了
+ 如果没有睡觉的厨师窗口也没有菜，那就在窗口旁边睡觉等待厨师送菜过来



## 关闭 channel

关闭channel时会把`recvq` 中的G全部唤醒，本该写入G的数据位置为nil。把 `sendq` 中的G全部唤醒，但这些 G 会 panic。

除此之外，panic出现的常见场景还有： 1. 关闭值为nil的channel 2. 关闭已经被关闭的channel 3. 向已经关闭的channel写数据







 