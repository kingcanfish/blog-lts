---
title: Go 语言的 sync.Map 浅析
date: 2020-12-02T10:46:00
tag: [map, sync, go]
categories: go
cover: https://pic.guoxy.top/img/dvoe.svg
description: Go 语言的 sync.Map 浅析
---



## Start



今天讲一下 `Go` 语言中的 `sync.Map` 包 , 那么 `sync.Map` 和内建的 `map` 有什么区别呢， `map` 是线程不安全的 ，并发读的情况下还好 ，要是并发写的话 会报`pannic` 

```go
func main() {

	m := map[string]int{}

	for i := 0; i < 10; i++ {
		go func() {
			m["s"] = 1
		}()
	}
	return
}
//➜  go-inside go run "/home/guo/code/goproj/src/github.com/kingcanfish/go-inside/main.go"
//fatal error: concurrent map writes
```

那么问题来了，如果有多线程或者多协程并发需求的话应该怎么办呢？

我们可以在 `map` 结构体外层包装一个带锁的结构，比如

```go
type currentMap struct {
	mu sync.RWMutex
	m map[interface{}] interface{}
}

cm := currentMap{m:make(map[interface{}]interface{})}

//read
cm.RLock()
value := cm.m["somekey"]
cm.RUnlock()

//write
cm.Lock()
cm.m["comekey"] = "some"
cm.Unlock()
```

因为互斥锁效率有点低，对于 `map`  这种读多写少的 数据结构 ，我们能最大限度的提高读写效率， 所以我们使用读写锁  

但是数据量大的话， 大并发情况下只使用一把锁会使客户端争用严重，影响性能， 熟悉 Java 同学都知道 Java 中的 `ConcurrentHashMap` 并不是全局使用一个读写锁的， `Java` 的实现方法是内部一个区间使用一把锁 ， GitHub 上用 `Go` 版本的实现 

> https://github.com/orcaman/concurrent-map

## sync.Map 正文

Golang 在版本 1.9 之后引入 `sync.Map`  用于并发情况下使用 `Map` , 下面分析一下 `sync.Map`  那不到400行的源码  ~~但是我还是看了好几天啊淦~~

**以下基于`Go v1.15.5`** 最近几个版本对 `sync.Map` 有做改动， 对比自己的版本要注意区别~~后面的题外话会说到~~

### 结构

```go
type Map struct {
	mu Mutex

	// read contains the portion of the map's contents that are safe for
	// concurrent access (with or without mu held).
	//
	// The read field itself is always safe to load, but must only be stored with
	// mu held.
	//
	// Entries stored in read may be updated concurrently without mu, but updating
	// a previously-expunged entry requires that the entry be copied to the dirty
	// map and unexpunged with mu held.
	read atomic.Value // readOnly

	// dirty contains the portion of the map's contents that require mu to be
	// held. To ensure that the dirty map can be promoted to the read map quickly,
	// it also includes all of the non-expunged entries in the read map.
	//
	// Expunged entries are not stored in the dirty map. An expunged entry in the
	// clean map must be unexpunged and added to the dirty map before a new value
	// can be stored to it.
	//
	// If the dirty map is nil, the next write to the map will initialize it by
	// making a shallow copy of the clean map, omitting stale entries.
	dirty map[interface{}]*entry

	// misses counts the number of loads since the read map was last updated that
	// needed to lock mu to determine whether the key was present.
	//
	// Once enough misses have occurred to cover the cost of copying the dirty
	// map, the dirty map will be promoted to the read map (in the unamended
	// state) and the next store to the map will make a new dirty copy.
	misses int
}

type readOnly struct {
	m       map[interface{}]*entry
	amended bool // true if the dirty map contains some key not in m.
}

// expunged is an arbitrary pointer that marks entries which have been deleted
// from the dirty map.
var expunged = unsafe.Pointer(new(interface{}))

// An entry is a slot in the map corresponding to a particular key.
type entry struct {
	// p points to the interface{} value stored for the entry.
	//
	// If p == nil, the entry has been deleted and m.dirty == nil.
	//
	// If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
	// is missing from m.dirty.
	//
	// Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty
	// != nil, in m.dirty[key].
	//
	// An entry can be deleted by atomic replacement with nil: when m.dirty is
	// next created, it will atomically replace nil with expunged and leave
	// m.dirty[key] unset.
	//
	// An entry's associated value can be updated by atomic replacement, provided
	// p != expunged. If p == expunged, an entry's associated value can be updated
	// only after first setting m.dirty[key] = e so that lookups using the dirty
	// map find the entry.
	p unsafe.Pointer // *interface{}
}
```

+ `Map`  结构包含了互斥锁 m  , 只读的底层 `map`  read (`atomic.Value`  实际上指向一个接口， 而这个接口又指回了 `readOnly`  这个数据结构，通过 `atomic.Value`  来提供一些原子性的操作， 比如`CAS` --比较并交换)
+ `readOnly`  这个结构包含底层map 和amended 两个结构 ,其中amended 用来表示 有没有存在 dirty 中但是没有存在 `readOnly`  中的元素, 如果有则为真, 否则为假
+ `missed` 表示 `read`  中未命中的次数, 无论后续在 `dirty`  中是否再次命中 ,该值都会加1
+ `uxpunged`  是开始就初始化好的空指针, 如果有键要删除,可以将它标为 `uxpunged`  也就是空指针, 等待特定条件下回收
+ `dirty`  简简单单的底层 `map`
+ `entry`  简简单单指向真正值的指针
+ 详细的说明的话还是要看注释

+ 不如看图

![](https://pic.guoxy.top/img/20201202160643.png )

### 内置方法



#### Load

```go
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		// Avoid reporting a spurious miss if m.dirty got promoted while we were
		// blocked on m.mu. (If further loads of the same key will not miss, it's
		// not worth copying the dirty map for this key.)
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

func (e *entry) load() (value interface{}, ok bool) {
	p := atomic.LoadPointer(&e.p)
	if p == nil || p == expunged {
		return nil, false
	}
	return *(*interface{})(p), true
}
```

**Load** 方法的主要代码就在这里了

+ 首先会从 `read`  获取, 如果没有获取到 那么这时候在再判断 `amended`  字段 ,之前我们说了 `amended`  是一个哨兵位, 用来表示 `dirty`  中是否有 `read` 中没有的数据

+ 所以判断 `!ok && read.amended` 进入 dirty 检查, 注意这里在上锁的时候使用了双重检查锁, 因为在边界上可能发生中断 , 恢复的时候可能 `read`  中已经有值了, 所以需要再进行一次检查 , 单例模式里面有双重检查锁的应用 ~~扯远了~~  

+ 再往下 如果 `!ok` 的话,说明 `amended`  为 `false`  就说明 `dirty` 中肯定没有,直接返回  `false`

+ 再往下就是获得到了 `entry`  调用 `e.load()` ,如果 e 的指针指向的 `nil` 或者 `expunged` ( 之前被删除标记成了` expunged` ) 说明值是不存在的 ,所以返回 `nil`,

+ 否则返回 这个指针指向的值

+ 还有 `m.missLocked()` 函数 如下

  ```go
  func (m *Map) missLocked() {
  	m.misses++
  	if m.misses < len(m.dirty) {
  		return
  	}
  	m.read.Store(readOnly{m: m.dirty})
  	m.dirty = nil
  	m.misses = 0
  }
  ```

  + 当 `read`未命中的时候, 会调用这个函数, 此时 `misses`  自增 `1` , 当miss的次数大于 `dirty` 中元素的个数,的时候,会进行升级, 将 `dirty` 中所有的键值对提升到 `read` 中, 同时计数清 `0` ,  `dirty`  也清空 ~ 

#### Store

```go
func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.
			m.dirty[key] = e
		}
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
		e.storeLocked(&value)
	} else {
		if !read.amended {
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}
func (e *entry) tryStore(i *interface{}) bool {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
			return true
		}
	}
}

func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}
```

以上就是**Store**方法的主要实现

+ 和 `Load` 一样 先从read 里面读取, 如果 `ok` 的话,直接调用 `e.tryStore()`方法 原子地更换 底层的值, 又由于**底层实现是指针** , `read` 里面替换了 `dirty` 里面也会替换, 注意

  + ```go
    if p == expunged {
    	return false
    }
    ```

    如果指向的是 `expunged` 的话,返回 `false`  , 流程往后走

+ 如果`read`里面没有或者 read 里指向的是 expunged 的话上互斥锁,这里又用到了双重检查锁,理由同 Load 部分

+ 如果read 里面是 expunged然后通过 `e.unexpungeLocked()` 函数调用 CAS (比较并交换) 如果 `e.p` 指向的是expunged 的话,替换成nil

  + 疑问:这里为什么要将expunged替换成nil呢?既然都以及被标记为待回收的垃圾了,那直接复制就好了,为什么要这样?
  + 猜测: 和后面 `e.storeLocked(&value)` 的操作有关, 这个操作可能就是要nil 才能满足背后的原子性的store操作(仅为猜测与思考,不一定准确)

+ 如果上一步判断 `read`  没有的话,就在 `dirty` 里面检查是否命中 ,如果命中的话 直接替换

+ 如果dirty里面也没有的话, 继续往下判断 如果 `read.amended` 为false 的话, 说明 `dirty`  现在为 `nil` , 这个键值对为第一个添加到 `dirty` 里面的, 调用`m.dirtyLocked()` 将 `read` 里面的元素迭代添加到 `dirty`  里面, 添加的过程中会判断键值对是不是出于 `unexpunge`  或者 `nil`  状态并进行过滤

+ 如果不是 `dirty` 的第一个元素, 直接创建个新的添加进去就行了

+ **Tips** : 为什么向 `dirty` 添加第一个元素的时候要将 `read`  里面的拷贝到 `dirty` 里?

+ 因为miss过多将 dirty全部提升到read里 ,注意这里时直接将read的指针指向dirty ,所以之前read里的数据就会丢失,所以为了避免 `read`  里的数据丢失,就要将read里面的数据再存到dirty里, 由于这是一个迭代传值的过程,频繁进行这个操作肯定是会影响性能的

#### Delete

```go
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			delete(m.dirty, key)
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
		return e.delete()
	}
	return nil, false
}

// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
	m.LoadAndDelete(key)
}

func (e *entry) delete() (value interface{}, ok bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == nil || p == expunged {
			return nil, false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return *(*interface{})(p), true
		}
	}
}
```



**Delete** 方法看起来比较简单

实际上Delete 方法就是`LoadAndDelete` 方法的套壳,只是没有吧Load出来的数据返回而已

在前几个版本中貌似没有 `LoadAndDelete` 方法,至少1.9.x 版本是没有的,现在只不过是把之前 `Delete` 的逻辑移动到了 `DeleteAndLoad` 方法中而已

+ 首先也是从 `read` 中获取
+ 没有获取到的话再通过` read.amended` 判断 `dirty` 中有没有数据, 有的话通过双重检查锁进入 `dirty` 中获取, 如果在 `dirty` 中获取到了,调用 `delete()` 方法直接删除
+ 注意这里是直接删除,能被 GC 回收,在1.15.1版本是调用 `e.delete()` 方法的,是不会进行垃圾回收的,有内存泄露的风险,直至我现在 1.15.5 版本已经修复~
+ 如果有的话就进入 `e.delete()` 方法将 `e` 指向 `nil`  间接删除, 这里值实体是可以被 GC 回收的, 但是 `entry` 依然是在的,所以这部分是不会被 GC 回收的       

#### Range

```go
// Range calls f sequentially for each key and value present in the map.
// If f returns false, range stops the iteration.
//
// Range does not necessarily correspond to any consistent snapshot of the Map's
// contents: no key will be visited more than once, but if the value for any key
// is stored or deleted concurrently, Range may reflect any mapping for that key
// from any point during the Range call.
//
// Range may be O(N) with the number of elements in the map even if f returns
// false after a constant number of calls.
func (m *Map) Range(f func(key, value interface{}) bool) {
	// We need to be able to iterate over all of the keys that were already
	// present at the start of the call to Range.
	// If read.amended is false, then read.m satisfies that property without
	// requiring us to hold m.mu for a long time.
	read, _ := m.read.Load().(readOnly)
	if read.amended {
		// m.dirty contains keys not in read.m. Fortunately, Range is already O(N)
		// (assuming the caller does not break out early), so a call to Range
		// amortizes an entire copy of the map: we can promote the dirty copy
		// immediately!
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		if read.amended {
			read = readOnly{m: m.dirty}
			m.read.Store(read)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}

	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		if !f(k, v) {
			break
		}
	}
}
```

+ **Range** 方法提供了一个遍历元素的方法,需要传入一个回调函数, 该方法会将遍历出来的键值对传入这个回调函数里,用户自己的逻辑就要放在这个函数里实现

+ 而且什么时候停止遍历有回调函数决定, 当回调函数返回 false 的时候就会停止遍历

+ 注意因为是 Range , 所以在 Range 之前需要将 `dirty` 提升到 `read` 然后对 `read` 实现遍历

+ **example**

  ```go
  var m sync.Map
  
  m.Range(func(k,v interface{})bool{
          fmt.Print(k)
          fmt.Print(":")
          fmt.Print(v)
          fmt.Println()
          return true
   })
  ```

### Other

> 这里有关于 `v1.15.1` 内存泄露的案例 https://gocn.vip/topics/10860
>
---
`sync.Map`  是典型的用空间换时间的做法, 通过一个read做高速缓冲, 极大提高了多线程读的效率,但是由于 `first entry` 的存在,会将 `read` 中数据迭代拷贝回`dirty` 中,这里可能会比较耗时,所以`sync.Map`的使用场景应该更偏向于读多写少的情况

> 参考博文:
>
> https://www.cnblogs.com/qcrao-2018/p/12833787.html
>
> https://tonybai.com/2020/11/10/understand-sync-map-inside-through-examples/

