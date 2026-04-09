# Mutex

## 基本原语

Go 语言在 `[sync](https://golang.org/pkg/sync/)`
 包中提供了用于同步的一些基本原语，包括常见的 `[sync.Mutex](https://draveness.me/golang/tree/sync.Mutex)`、`[sync.RWMutex](https://draveness.me/golang/tree/sync.RWMutex)`、`[sync.WaitGroup](https://draveness.me/golang/tree/sync.WaitGroup)`、`[sync.Once](https://draveness.me/golang/tree/sync.Once)`和 `[sync.Cond](https://draveness.me/golang/tree/sync.Cond)`：

![Untitled](RPG从零开始/Program%20Language/Go/高级/并发编程/Mutex/Untitled.png)

这些基本原语提供了较为基础的同步功能，但是它们是一种相对原始的同步机制，在多数情况下，我们都应该使用抽象层级更高的 Channel 实现同步。

## Mutex

### 历史与发展

- 简单的互斥量cas 操作，flag 为1 代表加锁。

问题：请求锁的goroutine会排队等待获取互斥锁。虽然这貌似很公平，但是从性能上来看，却不是最优的。因为如果我们能够把锁交给正在占用CPU时间片的goroutine的话，那就不需要做上下文的切换，在高并发的情况下，可能会有更好的性能。

- 加入唤醒标志

如果想要获取锁的goroutine没有机会获取到锁，就会进行休眠，但是在锁释放唤醒之后，它并不能像先前一样直接获取到锁，还是要和正在请求锁的goroutine进行竞争。这会给后来请求锁的goroutine一个机会，也让CPU中正在执行的goroutine有更多的机会获取到锁，在一定程度上提高了程序的性能。

**这次的改动主要就是，新来的goroutine也有机会先获取到锁**，甚至一个goroutine可能连续获取到锁，打破了先来先得的逻辑。但是，代码复杂度也显而易见。

- 自旋

如果新来的goroutine或者是被唤醒的goroutine首次获取不到锁，它们就会通过自旋（spin，通过循环不断尝试，spin的逻辑是在[runtime实现](https://github.com/golang/go/blob/846dce9d05f19a1f53465e62a304dea21b99f910/src/runtime/proc.go#L5580)的）的方式，尝试检查锁是否被释放。在尝试一定的自旋次数后，再执行原来的逻辑。

对于临界区代码执行非常短的场景来说，这是一个非常好的优化。**因为临界区的代码耗时很短，锁很快就能释放，而抢夺锁的goroutine不用通过休眠唤醒方式等待调度**，直接spin几次，可能就获得了锁。

- 饥饿模式和正常模式

新来的Goroutine也参与竞争，有可能每次都会被新来的goroutine抢到获取锁的机会，在极端情况下，等中的goroutine可能会一直获取不到锁，这就是**饥饿问题**

正常模式拥有更好的性能，因为即使有等待抢锁的waiter，goroutine也可以连续多次获取到锁。

饥饿模式是对公平性和性能的一种平衡，它避免了某些goroutine长时间的等待锁。在饥饿模式下，优先对待的是那些一直在等待的waiter。

### 数据结构

Go 语言的 `[sync.Mutex](https://draveness.me/golang/tree/sync.Mutex)` 由两个字段 `state` 和 `sema` 组成。其中 `state` 表示当前互斥锁的状态，而 `sema` 是用于控制锁状态的信号量。总共占8字节

```
type Mutex struct {
	state int32
	sema  uint32
}
```

### 状态

互斥锁的状态比较复杂，如下图所示，最低三位分别表示 `mutexLocked`、`mutexWoken` 和 `mutexStarving`，剩下的位置用来表示当前有多少个 Goroutine 在等待互斥锁的释放：

![Untitled](RPG从零开始/Program%20Language/Go/高级/并发编程/Mutex/Untitled%201.png)

**互斥锁的状态**

在默认情况下，互斥锁的所有状态位都是 0，`int32` 中的不同位分别表示了不同的状态：

- `mutexLocked` — 表示互斥锁的锁定状态，1代表已加锁；
- `mutexWoken` — 表示是否唤醒，1代表已唤醒；
- `waitersCount` — 当前互斥锁上等待的 Goroutine 个数；
- `mutexStarving` — 表示是否处于饥饿状态，1代表是；

### 正常模式和饥饿模式

正常模式下，waiter都是进入先入先出队列，被唤醒的waiter并不会直接持有锁，而是要和新来的goroutine进行竞争。新来的goroutine有先天的优势，它们正在CPU中运行，可能它们的数量还不少，所以，在高并发情况下，被唤醒的waiter可能比较悲剧地获取不到锁，这时，它会被插入到队列的前面。如果waiter获取不到锁的时间超过阈值1毫秒，那么，这个Mutex就进入到了饥饿模式。

在饥饿模式下，Mutex的拥有者将直接把锁交给队列最前面的waiter。新来的goroutine不会尝试获取锁，即使看起来锁没有被持有，它也不会去抢，也不会spin，它会乖乖地加入到等待队列的尾部。

如果拥有Mutex的waiter发现下面两种情况的其中之一，它就会把这个Mutex转换成正常模式:

1. 此waiter已经是队列中的最后一个waiter了，没有其它的等待锁的goroutine了；
2. 此waiter的等待时间小于1毫秒。

### 加锁和解锁

![Untitled](RPG从零开始/Program%20Language/Go/高级/并发编程/Mutex/Untitled%202.png)

**加锁流程图**

先直接用cas原语加锁，如果m.state等于0，那么直接将m.state 置为mutexLocked，代表加锁，如果成功，m.state为0x0000 0001，代表此时上锁成功。如果没成功就走lockSlow。

```
func (m *Mutex) Lock() {
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	m.lockSlow()
}
```

lockSlow

这里将它分成几个部分介绍获取锁的过程：

1. 判断当前 Goroutine 能否进入自旋；
2. 通过自旋等待互斥锁的释放；
3. 计算互斥锁的最新状态
    1. 处于正常模式，会获取锁
    2. 锁被占用 或者 处于饥饿模式下，新增一个等待者
    3. 当前 `goroutine` 已经进入饥饿了，且锁还没有释放，为了让当前goroutine能够获取锁，需要把 `Mutex` 的状态改为饥饿状态，进入饥饿模式
    4. 如果当前 `goroutine` 是被唤醒的，清除系统唤醒标记
4. 更新互斥锁的状态
    1. 之前是非上锁的正常状态，设置成功说明本次`抢锁成功`
    ，可以返回去操作临界区了
    2. 进入等待队列，如果返回的状态为饥饿模式，说明是直接从队列的第一个抽取，可以直接获取锁（而且根据条件判断，切换成正常模式）；
    3. 如果返回的状态为正常模式，说明是被唤醒的goroutine，可以再去获取锁（会判断当前goroutine是否为饥饿，看是否需要走饥饿模式）
5. 更新失败，继续进入循环

**通过自旋方式获取锁：**

```
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
```

**进入自旋的条件：**

1. 互斥锁只有在普通模式才能进入自旋；
2. `[runtime.sync_runtime_canSpin](https://draveness.me/golang/tree/runtime.sync_runtime_canSpin)` 需要返回 `true`：
    1. 运行在多 CPU 的机器上；
    2. 当前 Goroutine 为了获取该锁进入自旋的次数小于四次；
    3. 当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空；

实际上的自旋是执行30次的PAUSE指令，该指令只会占用 CPU 并消耗 CPU 时间

处理了自旋相关的特殊逻辑之后，**互斥锁会根据上下文计算当前互斥锁最新的状态**。几个不同的条件分别会更新 `state` 字段中存储的不同信息 — `mutexLocked`、`mutexStarving`、`mutexWoke`和 `mutexWaiterShift`：

```
		new := old
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			new &^= mutexWoken
		}
```

如果没有通过 CAS 获得锁，会调用 `[runtime.sync_runtime_SemacquireMutex](https://draveness.me/golang/tree/runtime.sync_runtime_SemacquireMutex)`通过信号量保证资源不会被两个 Goroutine 获取。`[runtime.sync_runtime_SemacquireMutex](https://draveness.me/golang/tree/runtime.sync_runtime_SemacquireMutex)` 会在方法中不断尝试获取锁并陷入休眠等待信号量的释放，一旦当前 Goroutine 可以获取信号量，它就会立刻返回

- 在正常模式下，这段代码会设置唤醒和饥饿标记、重置迭代次数并重新执行获取锁的循环；
- 在饥饿模式下，当前 Goroutine 会获得互斥锁，如果等待队列中只存在当前 Goroutine或者等待时间小于1ms，互斥锁还会从饥饿模式中退出；

```
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // 通过 CAS 函数获取了锁
			}
			...
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving　
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}
}
```

互斥锁的解锁过程 `[sync.Mutex.Unlock](https://draveness.me/golang/tree/sync.Mutex.Unlock)` 与加锁过程相比就很简单，该过程会先使用 `[sync/atomic.AddInt32](https://draveness.me/golang/tree/sync/atomic.AddInt32)` 函数快速解锁，这时会发生下面的两种情况：

- 如果该函数返回的新状态等于 0，当前 Goroutine 就成功解锁了互斥锁；
- 如果该函数返回的新状态不等于 0，说明除了锁状态，其他字段还不是为0，比如有等待的goroutine，这段代码会调用 `[sync.Mutex.unlockSlow](https://draveness.me/golang/tree/sync.Mutex.unlockSlow)` 开始慢速解锁：

```
func (m *Mutex) Unlock() {
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		m.unlockSlow(new)
	}
}
```

unlockSlow:

- 在正常模式下，上述代码会使用如下所示的处理过程：
    - 如果互斥锁不存在等待者或者互斥锁的 `mutexLocked`、`mutexStarving`、`mutexWoken` 状态不都为 0，那么当前方法可以直接返回，不需要唤醒其他等待者；
    - 如果互斥锁存在等待者，会通过 `[sync.runtime_Semrelease](https://draveness.me/golang/tree/sync.runtime_Semrelease)` 唤醒等待者并移交锁的所有权；
- 在饥饿模式下，上述代码会直接调用 `[sync.runtime_Semrelease](https://draveness.me/golang/tree/sync.runtime_Semrelease)` 将当前锁交给下一个正在尝试获取锁的等待者，等待者被唤醒后会得到锁，在这时互斥锁还不会退出饥饿状态；

```
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 { // 正常模式
		old := new
		for {
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else { // 饥饿模式
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```