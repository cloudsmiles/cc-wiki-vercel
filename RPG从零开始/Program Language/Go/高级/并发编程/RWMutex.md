# RWMutex

读写互斥锁 `[sync.RWMutex](https://draveness.me/golang/tree/sync.RWMutex)`是细粒度的互斥锁，它不限制资源的并发读，但是读写、写写操作无法并行执行。

## 数据结构

`[sync.RWMutex](https://draveness.me/golang/tree/sync.RWMutex)` 中总共包含以下 5 个字段：

```
type RWMutex struct {
	w           Mutex
	writerSem   uint32
	readerSem   uint32
	readerCount int32
	readerWait  int32
}
```

- `w` — 复用互斥锁提供的能力；
- `writerSem` 和 `readerSem` — 分别用于写等待读和读等待写：
- `readerCount` 存储了当前正在执行的读操作数量；
- `readerWait` 表示当写操作被阻塞时等待的读操作个数；

我们会依次分析获取写锁和读锁的实现原理，其中：

- 写操作使用 `[sync.RWMutex.Lock](https://draveness.me/golang/tree/sync.RWMutex.Lock)` 和 `[sync.RWMutex.Unlock](https://draveness.me/golang/tree/sync.RWMutex.Unlock)` 方法；
- 读操作使用 `[sync.RWMutex.RLock](https://draveness.me/golang/tree/sync.RWMutex.RLock)` 和 `[sync.RWMutex.RUnlock](https://draveness.me/golang/tree/sync.RWMutex.RUnlock)` 方法；

## 写锁

当资源的使用者想要获取写锁时，需要调用 `[sync.RWMutex.Lock](https://draveness.me/golang/tree/sync.RWMutex.Lock)` 方法：

```
func (rw *RWMutex) Lock() {
	rw.w.Lock()
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
}
```

1. 调用结构体持有的 `[sync.Mutex](https://draveness.me/golang/tree/sync.Mutex)` 结构体的 `[sync.Mutex.Lock](https://draveness.me/golang/tree/sync.Mutex.Lock)` 阻塞后续的写操作；
    - 因为互斥锁已经被获取，其他 Goroutine 在获取写锁时会进入自旋或者休眠；
2. 调用 `[sync/atomic.AddInt32](https://draveness.me/golang/tree/sync/atomic.AddInt32)` 函数阻塞后续的读操作：
3. 如果仍然有其他 Goroutine 持有互斥锁的读锁，该 Goroutine 会调用 `[runtime.sync_runtime_SemacquireMutex](https://draveness.me/golang/tree/runtime.sync_runtime_SemacquireMutex)` 进入休眠状态等待所有读锁所有者执行结束后释放 `writerSem` 信号量将当前协程唤醒；

写锁的释放会调用 `[sync.RWMutex.Unlock](https://draveness.me/golang/tree/sync.RWMutex.Unlock)`：

```
func (rw *RWMutex) Unlock() {
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		throw("sync: Unlock of unlocked RWMutex")
	}
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	rw.w.Unlock()
}
```

与加锁的过程正好相反，写锁的释放分以下几个执行：

1. 调用 `[sync/atomic.AddInt32](https://draveness.me/golang/tree/sync/atomic.AddInt32)` 函数将 `readerCount` 变回正数，释放读锁；
2. 通过 for 循环释放所有因为获取读锁而陷入等待的 Goroutine：
3. 调用 `[sync.Mutex.Unlock](https://draveness.me/golang/tree/sync.Mutex.Unlock)` 释放写锁；

## 读锁

读锁的加锁方法 `[sync.RWMutex.RLock](https://draveness.me/golang/tree/sync.RWMutex.RLock)`很简单，该方法会通过 `[sync/atomic.AddInt32](https://draveness.me/golang/tree/sync/atomic.AddInt32)`将 `readerCount`加一：

```
func (rw *RWMutex) RLock() {
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}
```

1. 如果该方法返回负数 — 其他 Goroutine 获得了写锁，当前 Goroutine 就会调用 `[runtime.sync_runtime_SemacquireMutex](https://draveness.me/golang/tree/runtime.sync_runtime_SemacquireMutex)` 陷入休眠等待锁的释放；
2. 如果该方法的结果为非负数 — 没有 Goroutine 获得写锁，当前方法会成功返回；

当 Goroutine 想要释放读锁时，会调用如下所示的 `[sync.RWMutex.RUnlock](https://draveness.me/golang/tree/sync.RWMutex.RUnlock)` 方法：

```
func (rw *RWMutex) RUnlock() {
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		rw.rUnlockSlow(r)
	}
}
```

该方法会先减少正在读资源的 `readerCount` 整数，根据 `[sync/atomic.AddInt32](https://draveness.me/golang/tree/sync/atomic.AddInt32)` 的返回值不同会分别进行处理：

- 如果返回值大于等于零 — 读锁直接解锁成功；
- 如果返回值小于零 — 有一个正在执行的写操作，在这时会调用`[sync.RWMutex.rUnlockSlow](https://draveness.me/golang/tree/sync.RWMutex.rUnlockSlow)` 方法；

```
func (rw *RWMutex) rUnlockSlow(r int32) {
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		throw("sync: RUnlock of unlocked RWMutex")
	}
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```

`[sync.RWMutex.rUnlockSlow](https://draveness.me/golang/tree/sync.RWMutex.rUnlockSlow)`会减少获取锁的写操作等待的读操作数 `readerWait`并在所有读操作都被释放之后触发写操作的信号量 `writerSem`，该信号量被触发时，调度器就会唤醒尝试获取写锁的 Goroutine。

## 特点

- writer互斥，如果有reader，等待reader完成，唤醒writer
- reader不互斥，如果有writer，等待writer完成，唤醒reader
- 由于writer和reader都会进入等待的过程，并且相互唤醒，所以reader和writer的竞争机会相同