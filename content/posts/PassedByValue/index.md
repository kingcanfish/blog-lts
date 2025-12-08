---
title: 值类型, 引用类型, 值传递, 引用传递一锅炖

date: 2020-11-02T14:02:00+08:00

tag: [传参 , 数据类型]

categories: go
description: 值类型, 引用类型, 值传递, 引用传递一锅炖
---



~~先画饼~~

> 面试官 : 请你说一下`值类型` ` 引用类型` `值传递` `引用传递` 四个概念吧
>
> 你 :  小伙子你不讲武德,这好吗 ? 这不好!
>
> 面试官 : 你还有什么想问我的吗?



之前我也是偶然看到这几个概念,以为自己挺了解,然后发现把这四个词放在一起就蒙了....

所以抽空理一理



## 值类型和引用类型

+ 值类型的变量直接持有数据 数据存放在栈上 引用类型的变量持有的数据的引用, 数据本身会放在堆上, 而在栈上会放一个指向这个数据 (储存在堆上) 的指针 
+ 定义值类型和引用类型时变量时, 值类型会默认分配内存(当然值为 `0`, 或者 `false` 之流), 而声明引用类型则不一样,也就是说声明引用类型之后只在堆上建立了一个引用,但是并没有分配内存, 此时给引用类型赋值就会 panic 
+ 那么有小朋友就要问了, 有人说引用类型的初始值是 `nil` 那和你这个说的不一样啊 ,其实 `nil` 也没有错 ,`nil` 的原因呢就是 ,声明了一个引用类型变量 ,这个引用类型变量指向栈上的某个地址,而这个地址上的值应该指向堆上的某个地方,但是没有分配空间, 所以为 `nil`

```go
func main() {
	var i int
	i = 10
	fmt.Println(i)
	return
}
//➜ go run "/home/guo/code/c/main.go"
//10

func main() {
	var m map[int]int
	m[1] = 1
	fmt.Println(m[1])
	return
}
//➜ go run "/home/guo/code/c/main.go"
//panic: assignment to entry in nil map

//goroutine 1 [running]:
//main.main()
//      /home/guo/code/c/main.go:9 +0x45
//exit status 2


```

+ 当你把一个变量赋给新变量的时候呢, 值类型是创建一个新的副本, 并把它给新变量；引用类型呢 就是吧对于数据本身的指向赋值给新变量, 两个变量还是指向的同一块数据, 以代码为例子

  ```go
  func main() {
  	var m int
  	m = 1
  	fmt.Println("m初始值", m)
  	fmt.Println("m地址", &m)
  	a := m
  	fmt.Println("a初始值", a)
  	fmt.Println("a地址", &a)
  	a = 2
  	fmt.Println("m最终值", m)
  	fmt.Println("m地址", &m)
  	fmt.Println("a最终值", a)
  	fmt.Println("a地址", &a)
  	return
  }
  //➜ go run "/home/guo/code/c/main.go"
  //m初始值 1
  //m地址 0xc000014110
  //a初始值 1
  //a地址 0xc000014118
  //m最终值 1
  //m地址 0xc000014110
  //a最终值 2
  //a地址 0xc000014118
  
  
  func main() {
  	m := map[int]int{}
  	m[1] = 1
  	fmt.Println("m[1]初始值", m[1])
  	fmt.Println("m地址", &m)
  	a := m
  	fmt.Println("a初始值", a[1])
  	fmt.Println("a地址", &a)
  	a[1] = 2
  	fmt.Println("m最终值", m[1])
  	fmt.Println("m地址", &m)
  	fmt.Println("a最终值", a[1])
  	fmt.Println("a地址", &a)
  	return
  }
  //➜go run "/home/guo/code/c/main.go"
  //m[1]初始值 1
  //m地址 &map[1:1]
  //a初始值 1
  //a地址 &map[1:1]
  //m最终值 2
  //m地址 &map[1:2]
  //a最终值 2
  //a地址 &map[1:2]
  ```

+ 引用类型可以使用 `make` `new` 分配堆上的内存
+ 如果你要问我说 `指针变量` 是一种什么类型 我的回答是 `无可奉告`

![](https://guoxy-static.oss-cn-shanghai.aliyuncs.com/values_reference_types.png)



## 引用传递和值传递



*值类型传递一定是值传递, 引用类型传递不一定是引用传递*

**Golang 中只用值传递一种传递类型**

小伙子,只要记住上面这两句话,就错不了

### 迷惑人心的Map

+ 按照上面来说, 按照 **值传递 **的特性，我们毫无疑问的猜想：函数外两次输出的结果应该是相同的，同时地址应该不同, 然而.......
+ 事实真的是这样吗?

+ 看 `demo` 

```go
package main

import "fmt"

func test_map(m map[string]string){
	fmt.Printf("inner: %v, %p\n",m, m)
	m["a"]="11"
	fmt.Printf("inner: %v, %p\n",m, m)
}

func main() {

	m := map[string]string{
		"a":"1",
		"b":"2",
		"c":"3",
	}
	
	fmt.Printf("outer: %v, %p\n",m, m)
	test_map(m)
	fmt.Printf("outer: %v, %p\n",m, m)
}

//output
//outer: map[a:1 b:2 c:3], 0x442260
//inner: map[a:1 b:2 c:3], 0x442260
//inner: map[a:11 b:2 c:3], 0x442260
//outer: map[b:2 c:3 a:11], 0x442260
```



+ **事实却正是相反**
+ 两个示例代码的结果竟然截然相反，如果上述的内容让你产生了疑惑，并且你希望彻底的了解这过程中发生了什么~

```go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
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
```

如源码所示, `make()`  函数调用会最终调用到 `makemap()` 函数, 上面源码返回的是一个指针 ,也就是说, 变量 m 底层的话实际上也是由指针实现的 ,所以虽然 是按值传递, 但是拷贝的不是 map 本身, 而是 map的指针,  指针拷贝一份还是指向那个 `map` 呀~

### Channel 也是这样吗

+ 直接先看代码

```go
package main

import "fmt"


func test_chan2(ch chan string){
	fmt.Printf("inner: %v, %v\n",ch, len(ch))
	ch<-"b"
	fmt.Printf("inner: %v, %v\n",ch, len(ch))
}

func main() {
	ch := make(chan string, 10)
	ch<- "a"
	
	fmt.Printf("outer: %v, %v\n",ch, len(ch))
	test_chan2(ch)
	fmt.Printf("outer: %v, %v\n",ch, len(ch))
}
//output
//outer: 0x436100, 1
//inner: 0x436100, 1
//inner: 0x436100, 2
//outer: 0x436100, 2
```

+ 看源码

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```

+ 和 `map` 一样 返回的也是一个指针, 这里就不多说了

### 切片永远的神

+ 踩过坑的小伙伴都知道 , `slice` 确确实实是按值传递的 也不是像上面两个兄弟一样传递指针

+ 例子:

  ```go
  package main
  
  import "fmt"
  
  func main() {
  	
  	sl := []string{
  		"a",
  		"b",
  		"c",
  	}
  	
  	fmt.Printf("%v, %p\n",sl, sl)
  	test_slice(sl)
  	fmt.Printf("%v, %p\n",sl, sl)
  }
  
  
  func test_slice(sl []string){
  	fmt.Printf("%v, %p\n",sl, sl)
  	sl[0] = "aa"
  	//sl = append(sl, "d")
  	fmt.Printf("%v, %p\n",sl, sl)
  }
  //output
  //[a b c], 0x442260
  //[a b c], 0x442260
  //[aa b c], 0x442260
  //[aa b c], 0x442260
  ```

  

+ 看 源 码

  ```go
  func makeslice(et *_type, len, cap int) unsafe.Pointer {
  	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
  	if overflow || mem > maxAlloc || len < 0 || len > cap {
  		// NOTE: Produce a 'len out of range' error instead of a
  		// 'cap out of range' error when someone does make([]T, bignumber).
  		// 'cap out of range' is true too, but since the cap is only being
  		// supplied implicitly, saying len is clearer.
  		// See golang.org/issue/4085.
  		mem, overflow := math.MulUintptr(et.size, uintptr(len))
  		if overflow || mem > maxAlloc || len < 0 {
  			panicmakeslicelen()
  		}
  		panicmakeslicecap()
  	}
  
  	return mallocgc(mem, et, true)
  }
  ```

+ 这里为什么返回的是一个指针呢?

> 目前的 [`runtime.makeslice`](https://github.com/golang/go/blob/440f7d64048cd94cba669e16fe92137ce6b84073/src/runtime/slice.go#L34-L50) 会返回指向底层数组的指针，之前版本的 Go 语言中，数组指针、长度和容量会被合成一个 `slice` 结构并返回，但是从 [cmd/compile: move slice construction to callers of makeslice](https://github.com/golang/go/commit/020a18c545bf49ffc087ca93cd238195d8dcc411#diff-d9238ca551e72b3a80da9e0da10586a4) 这次提交之后，构建结构体 `SliceHeader` 的工作就都交给 [`runtime.makeslice`](https://github.com/golang/go/blob/440f7d64048cd94cba669e16fe92137ce6b84073/src/runtime/slice.go#L34-L50) 的调用方处理了，这些调用方会在编译期间构建切片结构体
>
> ```go
> func typecheck1(n *Node, top int) (res *Node) {
> 	switch n.Op {
> 	...
> 	case OSLICEHEADER:
> 	switch 
> 		t := n.Type
> 		n.Left = typecheck(n.Left, ctxExpr)
> 		l := typecheck(n.List.First(), ctxExpr)
> 		c := typecheck(n.List.Second(), ctxExpr)
> 		l = defaultlit(l, types.Types[TINT])
> 		c = defaultlit(c, types.Types[TINT])
> 
> 		n.List.SetFirst(l)
> 		n.List.SetSecond(c)
> 	...
> 	}
> }
> ```
>
> `OSLICEHEADER` 操作会创建我们在上面介绍过的结构体 `SliceHeader`，其中包含数组指针、切片长度和容量，它也是切片在运行时的表示：
>
> ```go
> type SliceHeader struct {
> 	Data uintptr
> 	Len  int
> 	Cap  int
> }
> ```

+ 所以这就是为什么传入切片能够修改外层数据的原因 , 因为数据存在底层的数组里 , 而`slice` 中则是存储指向数组的指针
+ 当 切片不发生扩容的时候, 拷贝的切片 和 函数外的切片指向同一个数组, 所以保持相同
+ 当发生扩容时, 会指向新的数组地址, 所以这时候两个切片的值就会不同啦~
+ 还有一件事:  **既然是按值传递的, 为什么函数内和函数外切片地址是一样的呢**?

> Pointer()函数中，对于Slice类型的数据，返回的一直是指向第一个元素的地址，所以我们通过 `fmt.Printf()` 中%p来打印Slice的地址，其实打印的结果是内部存储数组元素的首地址



以上内容指针对 `Go` 语言, 其他语言或许也有相同的地方,但是也会有很大的不同, ~~特别是C++~~



>参考: 
>
>golang的参数引用: https://juejin.im/post/6844903762079776775
>
>go语言设计与实现: https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/