---
title: 【转】GO：sync.Mutex 的实现与演进
date: 2020-12-09T14:31:00+08:00
tag: [Go, mutex, 锁]
categories: GO
cover: https://pic.guoxy.top/img/mutex.jpg
description: GO：sync.Mutex 的实现与演进
---

> 本文转自：https://www.jianshu.com/p/ce1553cc5b4f

前几天某个群里问，`sync.Mutex` 是否有自旋逻辑，抽时间看了下源码。不得了，小小的 `Mutex` 居然进化了三个版本，从这也可以看到 go 社区一直在积极的优化与演进

1. 最朴素的实现互斥锁，拿到锁返回，拿不到就将当前 goroutine 休眠

2. 增加了自旋 spinlock 的逻辑，也就是说大部份 Mutex 锁住时间如果很短，那么自旋可以减小无谓的 runtime 调度。推荐看官方 spin [commit](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fcommit%2Fedcad8639a902741dc49f77d000ed62b0cc6956f)

3. 进化成了公平锁，老版本中当前抢锁中的 goroutine 大概率比休眠的优先拿到锁，会产生 latency 长尾。新版本中超过一定时间没拿到锁，这个优先级会反转，尽可能减小长尾。推荐大家看 [#issue 13086](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fissues%2F13086)，这里面反映了问题，另外看 [commit](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fcommit%2F0556e26273f704db73df9e7c4c3d2e8434dec7be), 里面有很详细的测试数据，值得学习

那么具体怎么实现呢？分别以 1.3, 1.7, 1.12 三个版本源码为例

### Mutex 结构体及常用变量

```go
type Mutex struct {
    state int32
    sema  uint32
}
// 1.3 与 1.7 老的实现共用的常量
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexWaiterShift = iota
)
// 1.12 公平锁使用的常量
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota
    starvationThresholdNs = 1e6
)
```

从中可以看到，Mutex 有两个变量：

1. state 4 字节 int, 其中低几位用于做标记，高位地址空间用于计数，表示有多少个 goroutine 正在等待而处于休眠中。
2. sema 是一个互斥的信号量，初始默认值是 0，用于将 goroutine park 休眠或是唤醒。sema acquire 时如果 sema 大于 0，那么减一返回，否则休眠等待。sema release 将 sema 加一，然后唤醒等待队列的第一个 goroutine

默认直接使用 sync.Mutex 或是嵌入到结构体中，state 零值代表未上锁，sema 零值也是有意义的，参考下面源码加锁与解锁逻辑，稍想下就会明白的。另外参考大胡子 dave 的[关于零值的文章](https://links.jianshu.com/go?to=https%3A%2F%2Fdave.cheney.net%2F2013%2F01%2F19%2Fwhat-is-the-zero-value-and-why-is-it-useful)

### 朴素互斥锁

朴素是什么意思呢？就是能用，粗糙...

#### 上锁

```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex. 快速上锁，当前 state 为 0，说明没人锁。CAS 上锁后直接返回
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if raceenabled {
            raceAcquire(unsafe.Pointer(m))
        }
        return
    }

    awoke := false // 被唤醒标记，如果是被别的 goroutine 唤醒的那么后面会置 true
    for {
        old := m.state // 老的 m.state 值
        new := old | mutexLocked // 新值要置 mutexLocked 位为 1
        if old&mutexLocked != 0 { // 如果 old mutexLocked 位不为 0，那说明有人己经锁上了，那么将 state 变量的 waiter 计数部分 +1
            new = old + 1<<mutexWaiterShift
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case. 如果走到这里 awoke 为 true, 说明是被唤醒的，那么清除这个 mutexWoken 位，置为 0
            new &^= mutexWoken
        }
        // CAS 更新，如果 m.state 不等于 old，说明有人也在抢锁，那么 for 循环发起新的一轮竞争。
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&mutexLocked == 0 { // 如果 old mutexLocked 位为 1，说明当前 CAS 是为了更新 waiter 计数。如果为 0，说明是抢锁成功，那么直接 break 退出。
                break
            }
            runtime_Semacquire(&m.sema) // 此时如果 sema <= 0 那么阻塞在这里等待唤醒，也就是 park 住。走到这里都是要休眠了。
            awoke = true  // 有人释放了锁，然后当前 goroutine 被 runtime 唤醒了，设置 awoke true
        }
    }

    if raceenabled {
        raceAcquire(unsafe.Pointer(m))
    }
}
```

上锁逻辑其实也不难，这里面更改计数都是用 CAS

1. fast path 快速上锁，如果当前 state == 0, 肯定是没人上锁，也没人等待，CAS 更新后直接退出好了
2. 当前如果有人锁住了，那么更新 m.state 值的 waiter 计数部份，然后 `runtime_Semacquire` 将自己休眠，等待被唤醒
3. `runtime_Semacquire` 函数返回说明锁释放了，有人将自己唤醒了，那么设置  awoke，大循环发起新的一轮竞争。
4. 新的竞争到最后，cas 更新了 new 值，此时 old 值 mutexLocked 位肯定为 0，获取锁成功，break 退出即可。

#### 解锁

```go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
    if raceenabled {
        _ = m.state
        raceRelease(unsafe.Pointer(m))
    }

    // Fast path: drop lock bit. 快速将 state 的 mutexLocked 位清 0，然后 new 返回更新后的值，注意此 add 完成后，很有可能新的 goroutine 抢锁，并上锁成功
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 { // 如果释放了一个己经释放的锁，直接 panic
        panic("sync: unlock of unlocked mutex")
    }

    old := new
    for {// 如果 state 变量的 waiter 计数为 0 说明没人等待锁，直接 return 就好，同时如果 old 值的 mutexLocked|mutexWoken 任一置 1，说明要么有人己经抢上了锁，要么说明己经有被唤醒的 goroutine 去抢锁了，没必要去做通知操作
        // If there are no waiters or a goroutine has already
        // been woken or grabbed the lock, no need to wake anyone.
        if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
            return
        }
        // Grab the right to wake someone. 将 waiter 计数位减一，并设置 awoken 位
        new = (old - 1<<mutexWaiterShift) | mutexWoken
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            runtime_Semrelease(&m.sema) // cas 成功后，再做 sema release 操作，唤醒休眠的 goroutine
            return
        }
        old = m.state
    }
}
```

解锁逻辑也不难，注意一个 goroutine 可以释放别的 goroutine 上的锁

1. 原子操作, m.state - mutexLocked, 如果之后 (new+mutexLocked)&mutexLocked 说明释放了一个没上锁的 Mutex，直接 panic
2. 接下来为什么是  for 循环呢？原因在于，第一步原子操作后，很可能有第三方刚好获得锁了，那么 for 里面的 CAS 肯定会失败
3. 快速判断，如果 waiter 计数为 0，说明没有休眠的 goroutine，不用唤醒。如果 old&(mutexLocked|mutexWoken) != 0 说明要么有人获得了锁，要么己经有 woken 的 goroutine 了，也不用去唤醒。注意这里，mutexLocked 是 for 循环再次判断时才有的， old 值是循环底部重新又获取得
4. 然后 CAS 更新成 new 值，设置 woken 标记位，并将等待 waiter 计数减一。最后 `runtime_Semrelease` 真正的唤醒等待 goroutine

#### 朴素锁的问题

因获取 sema 休眠的 goroutine 会以一个 FIFO 的链表形式保存，如果唤醒时可以优先拿到锁。但是看代码的逻辑，处于休眠中的 goroutine 优先级低于当前活跃的。`Unlock` 解锁的顺间，最新活跃的 goroutine 是会抢到锁的。另外有时锁时间很短，如果没有自旋 spin 的逻辑，所有 goroutine 都要休眠 park, 徒增 runtime 调度的开销。

### 自旋 spin 的优化

后来优化时增加了 spin 逻辑，自旋只存在 `Lock` 阶段，代码以 go 1.7 为例



```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }

    awoke := false
    iter := 0
    for {
        old := m.state
        new := old | mutexLocked
        if old&mutexLocked != 0 { // 如果当前己经锁了，那么判断是否可以自旋
            if runtime_canSpin(iter) {
                // Active spinning makes sense.
                // Try to set mutexWoken flag to inform Unlock
                // to not wake other blocked goroutines.
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true
                }
                runtime_doSpin()
                iter++
                continue
            }
            new = old + 1<<mutexWaiterShift
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                panic("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&mutexLocked == 0 {
                break
            }
            runtime_Semacquire(&m.sema)
            awoke = true
            iter = 0
        }
    }

    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

可以看到，for 循环开始增加了 spin 判断逻辑。

1. 如果 runtime 判断允许自旋，那么走 if 逻辑，否则走原有的 Lock 逻辑
2. 如果当前 m.state 未设置 woken 标记，并且等待 waiter 计数大于 0，说明有人在等待，那么 CAS 更新 m.state 置位 mutexWoken
3. 执行 `runtime_doSpin` 逻辑，同时 iter++ 表示自旋次数



```go
const (
    mutex_unlocked = 0
    mutex_locked   = 1
    mutex_sleeping = 2

    active_spin     = 4
    active_spin_cnt = 30
    passive_spin    = 1
)
// Active spinning for sync.Mutex.
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
    // sync.Mutex is cooperative, so we are conservative with spinning.
    // Spin only few times and only if running on a multicore machine and
    // GOMAXPROCS>1 and there is at least one other running P and local runq is empty.
    // As opposed to runtime mutex we don't do passive spinning here,
    // because there can be work on global runq on on other Ps.
    if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
        return false
    }
    if p := getg().m.p.ptr(); !runqempty(p) {
        return false
    }
    return true
}
```

判断 `runtime_canSpin` 是否允许自旋逻辑也简单，也比较严格

1. iter 不大小最大的 active_spin 次数，默认是 4
2. 当前机器是多核，并且 GOMAXPROCS > 1，这个很好理解，并发为 1 自旋也没意义
3. 最后一个就是当前 P 的本地 runq 队列为空，如果有待运行的 G，那么也不允许自旋

```go
func sync_runtime_doSpin() {
    procyield(active_spin_cnt)
}

TEXT runtime·procyield(SB),NOSPLIT,$0-0
    MOVL    cycles+0(FP), AX
again:
    PAUSE
    SUBL    $1, AX
    JNZ again
    RET
```

自旋代码涉及汇编了，在 amd64 平台调用 `PAUSE`，循环 active_spin_cnt 30 次。

代码以 go1.12 为例，可以看到注释关于公平锁的实现初衷和逻辑。越是基础组件更新越严格，背后肯定有相关测试数据。

1. Mutex 两种工作模式，normal 正常模式，starvation 饥饿模式。normal 情况下锁的逻辑与老版相似，休眠的 goroutine 以 FIFO 链表形式保存在 sudog 中，被唤醒的 goroutine 与新到来活跃的 goroutine 竞解，但是很可能会失败。如果一个 goroutine 等待超过 1ms，那么 Mutex 进入饥饿模式
2. 饥饿模式下，解锁后，锁直接交给 waiter FIFO 链表的第一个，新来的活跃 goroutine 不参与竞争，并放到 FIFO 队尾
3. 如果当前获得锁的 goroutine 是 FIFO 队尾，或是等待时长小于 1ms，那么退出饥饿模式
4. normal 模式下性能是比较好的，但是 starvation 模式能减小长尾 latency

#### 公平锁上锁逻辑



```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex. 快速上锁逻辑
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }

    var waitStartTime int64 // waitStartTime 用于判断是否需要进入饥饿模式
    starving := false // 饥饿标记
    awoke := false // 是否被唤醒
    iter := 0 // spin 循环次数
    old := m.state
    for {
        // Don't spin in starvation mode, ownership is handed off to waiters
        // so we won't be able to acquire the mutex anyway. 饥饿模式下不进行自旋，直接进入阻塞队列
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // Active spinning makes sense.
            // Try to set mutexWoken flag to inform Unlock
            // to not wake other blocked goroutines.
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        new := old
        // Don't try to acquire starving mutex, new arriving goroutines must queue.
        if old&mutexStarving == 0 { // 只有此时不是饥饿模式时，才设置 mutexLocked，也就是说饥饿模式下的活跃 goroutine 直接排队去
            new |= mutexLocked
        }
        if old&(mutexLocked|mutexStarving) != 0 { // 处于己经上锁或是饥饿时，waiter 计数 + 1
            new += 1 << mutexWaiterShift
        }
        // The current goroutine switches mutex to starvation mode.
        // But if the mutex is currently unlocked, don't do the switch.
        // Unlock expects that starving mutex has waiters, which will not
        // be true in this case. 如果当前处于饥饿模式下，并且己经上锁了，mutexStarving 置 1，接下来 CAS 会用到
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        if awoke { // 如果当前 goroutine 是被唤醒的，然后清 mutexWoken 位
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&(mutexLocked|mutexStarving) == 0 { // 如果 old 没有上锁并且也不是饥饿模式，上锁成功直接退出
                break // locked the mutex with CAS
            }
            // If we were already waiting before, queue at the front of the queue.
            queueLifo := waitStartTime != 0 // 第一次 queueLifo 肯定是 false
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime() 
            }
            runtime_SemacquireMutex(&m.sema, queueLifo) // park 在这里，如果 queueLifo 为真，那么扔到队头，也就是 LIFO
      // 走到这里，说明被其它 goroutine 唤醒了，继续抢锁时先判断是否需要进入 starving
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs // 超过 1ms 就进入饥饿模式
            old = m.state
            if old&mutexStarving != 0 { // 如果原来就是饥饿模式的话，走 if 逻辑
                // If this goroutine was woken and mutex is in starvation mode,
                // ownership was handed off to us but mutex is in somewhat
                // inconsistent state: mutexLocked is not set and we are still
                // accounted as waiter. Fix that.
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
        // 此时饥饿模式下被唤醒，那么一定能上锁成功。因为 Unlock 保证饥饿模式下只唤醒 park 状态的 goroutine
                delta := int32(mutexLocked - 1<<mutexWaiterShift) // waiter 计数 -1
                if !starving || old>>mutexWaiterShift == 1 { // 如果是饥饿模式下并且自己是最后一个 waiter ，那么清除 mutexStarving 标记
                    // Exit starvation mode.
                    // Critical to do it here and consider wait time.
                    // Starvation mode is so inefficient, that two goroutines
                    // can go lock-step infinitely once they switch mutex
                    // to starvation mode.
                    delta -= mutexStarving
                }
                atomic.AddInt32(&m.state, delta) // 更新，抢锁成功后退出
                break
            }
            awoke = true // 走到这里，不是饥饿模式，重新发起抢锁竞争
            iter = 0
        } else {
            old = m.state // CAS 失败，重新发起竞争
        }
    }

    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

整体来讲，公平锁上锁逻辑复杂了不少，边界点要考滤的比较多

1. 同样的 fast path 快速上锁逻辑，原来 m.state 为 0，锁就完事了
2. 进入 for 循环，也要走自旋逻辑，但是多了一个判断，如果当前处于饥饿模式禁止自旋，根据实现原理，此时活跃的 goroutine 要直接进入 park 的队列
3. 自旋后面的代码有四种情况：饥饿抢锁成功，饥饿抢锁失败，正常抢锁成功，正常抢锁失败。上锁失败的最后都要 waiter 计数加一后，更新 CAS
4. 如果 CAS 失败，那么重新发起竞争就好
5. 如果 CAS 成功，此时要判断处于何种情况，如果 old 没上锁也处于 normal 模式，抢锁成功退出
6. 如果 CAS 成功，但是己经有人上锁了，那么要根据 queueLifo 来判断是扔到 park 队首还是队尾，此时当前 goroutine park 在这里，等待被唤醒
7. `runtime_SemacquireMutex` 被唤醒了有两种情况，判断是否要进入饥饿模式，如果老的 old 就是饥饿的，那么自己一定是唯一被唤醒，一定能抢到锁的，waiter 减一，如果自己是最后一个 waiter 或是饥饿时间小于 starvationThresholdNs 那么清除 mutexStarving 标记位后退出
8. 如果老的不是饥饿模式，那么 awoke 置 true，重新竞争

#### 公平锁解锁逻辑



```go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
    if race.Enabled {
        _ = m.state
        race.Release(unsafe.Pointer(m))
    }

    // Fast path: drop lock bit. 和原有逻辑一样，先减去 mutexLocked，并判断是否解锁了未上锁的 Mutex, 直接 panic
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    if new&mutexStarving == 0 { // 查看 mutexStarving 标记位，如果 0 走老逻辑，否则走 starvation 分支
        old := new
        for {
            // If there are no waiters or a goroutine has already
            // been woken or grabbed the lock, no need to wake anyone.
            // In starvation mode ownership is directly handed off from unlocking
            // goroutine to the next waiter. We are not part of this chain,
            // since we did not observe mutexStarving when we unlocked the mutex above.
            // So get off the way.
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // Grab the right to wake someone.
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false)
                return
            }
            old = m.state
        }
    } else {
        // Starving mode: handoff mutex ownership to the next waiter.
        // Note: mutexLocked is not set, the waiter will set it after wakeup.
        // But mutex is still considered locked if mutexStarving is set,
        // so new coming goroutines won't acquire it.
        runtime_Semrelease(&m.sema, true) // 直接 runtime_Semrelease 唤醒等待的 goroutine
    }
}
```

1. 原子操作，将 m.state 减去 mutexLocked，然后判断是否释放了未上锁的 Mutex，直接 panic
2. 根据 m.state 的 mutexStarving 判断当前处于何种模式，0 走 normal 分支，1 走 starvation 分支
3. starvation 模式下，直接 `runtime_Semrelease` 做信号量 UP 操作，唤醒 FIFO 队列中的第一个 goroutine
4. noarmal 模式类似原有逻辑，唯一不同的是多了一个  mutexStarving 位判断逻辑



> 推荐阅读《Go设计与实现》6.2同步原语和锁
>
> https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/