# select

`select` 是操作系统中的系统调用，我们经常会使用 `select`、`poll` 和 `epoll` 等函数构建 I/O 多路复用模型提升程序的性能。Go 语言的 `select` 与操作系统中的 `select` 比较相似，本节会介绍 Go 语言 `select` 关键字常见的现象、数据结构以及实现原理。

C 语言的 `select` 系统调用可以同时监听多个文件描述符的可读或者可写的状态，Go 语言中的 `select` 也能够让 Goroutine 同时等待多个 Channel 可读或者可写，在多个文件或者 Channel状态改变之前，`select` 会一直阻塞当前线程或者 Goroutine。

select与switch类似，但是与switch不同，select的case表达式必须都是channel的收发操作

## 现象

当我们在 Go 语言中使用 `select` 控制结构时，会遇到两个有趣的现象：

1. `select` 能在 Channel 上进行非阻塞的收发操作；
2. `select` 在遇到多个 Channel 同时响应时，会随机执行一种情况；

## 数据结构

`select`在 Go 语言的源代码中不存在对应的结构体，但是我们使用 `[runtime.scase](https://draveness.me/golang/tree/runtime.scase)` 结构体表示 `select`控制结构中的 `case`：

```
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}
```

因为非默认的 `case`中都与 Channel 的发送和接收有关，所以 `[runtime.scase](https://draveness.me/golang/tree/runtime.scase)`结构体中也包含一个 `[runtime.hchan](https://draveness.me/golang/tree/runtime.hchan)` 类型的字段存储 `case`中使用的 Channel。

## 实现原理

`select`语句在编译期间会被转换成 `OSELECT`节点。每个 `OSELECT` 节点都会持有一组 `OCASE`
 节点，如果 `OCASE`的执行条件是空，那就意味着这是一个 `default`节点。

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/select/Untitled.png)

上图展示的就是 `select`语句在编译期间的结构，每一个 `OCASE`既包含执行条件也包含满足条件后执行的代码。

编译器在中间代码生成期间会根据 `select` 中 `case` 的不同对控制语句进行优化，这一过程都发生在 `[cmd/compile/internal/gc.walkselectcases](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkselectcases)` 函数中，我们在这里会分四种情况介绍处理的过程和结果：

1. `select` 不存在任何的 `case`；
2. `select` 只存在一个 `case`；
3. `select` 存在两个 `case`，其中一个 `case` 是 `default`；
4. `select` 存在多个 `case`；

### 直接阻塞

当select不存在任何case时，在代码中的流程是：

```
func walkselectcases(cases *Nodes) []*Node {
	n := cases.Len()

	if n == 0 {
		return []*Node{mkcall("block", nil, nil)}
	}
	...
}
```

以上代码将类似 `select {}`的语句转换成调用 `[runtime.block](https://draveness.me/golang/tree/runtime.block)`函数，会直接阻塞当前Goroutine，导致无法被唤醒状态

### 单一管道

如果当前的 `select`条件只包含一个 `case`，那么编译器会将 `select` 改写成 `if`条件语句。下面对比了改写前后的代码：

```
// 改写前
select {
case v, ok <-ch: // case ch <- v
    ...
}

// 改写后
if ch == nil {
    block()
}
v, ok := <-ch // case ch <- v
...
```

`[cmd/compile/internal/gc.walkselectcases](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkselectcases)`在处理单操作 `select`语句时，会根据 Channel 的收发情况生成不同的语句。当 `case`中的 Channel 是空指针时，会直接挂起当前 Goroutine 并陷入永久休眠。

### 非阻塞操作

当 `select`中仅包含两个 `case`，并且其中一个是 `default`时，Go 语言的编译器就会认为这是一次非阻塞的收发操作。`[cmd/compile/internal/gc.walkselectcases](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkselectcases)` 会对这种情况单独处理。

不过在正式优化之前，该函数会将 `case`中的所有 Channel 都转换成指向 Channel 的地址，我们会分别介绍非阻塞发送和非阻塞接收时，编译器进行的不同优化。

**发送**

首先是 Channel 的发送过程，当 `case`中表达式的类型是 `OSEND`时，编译器会使用条件语句和 `[runtime.selectnbsend](https://draveness.me/golang/tree/runtime.selectnbsend)`函数改写代码：

```
select {
case ch <- i:
    ...
default:
    ...
}

if selectnbsend(ch, i) {
    ... // return chansend(c, elem, false, getcallerpc())
} else {
    ...
}
```

这段代码中最重要的就是 `[runtime.selectnbsend](https://draveness.me/golang/tree/runtime.selectnbsend)`，它为我们提供了向 Channel 非阻塞地发送数据的能力。由于我们向 `[runtime.chansend](https://draveness.me/golang/tree/runtime.chansend)`函数传入了非阻塞，所以在不存在接收方或者缓冲区空间不足时，当前 Goroutine 都不会阻塞而是会直接返回。

**接收**

由于从 Channel 中接收数据可能会返回一个或者两个值，所以接收数据的情况会比发送稍显复杂，不过改写的套路是差不多的：

```
// 改写前
select {
case v <- ch: // case v, ok <- ch:
    ......
default:
    ......
}

// 改写后
if selectnbrecv(&v, ch) { // if selectnbrecv2(&v, &ok, ch) {
    ... // selected, _ = chanrecv(c, elem, false)
			  // selected, *received = chanrecv(c, elem, false)
} else {
    ...
}
```

返回值数量不同会导致使用函数的不同，两个用于非阻塞接收消息的函数 `[runtime.selectnbrecv](https://draveness.me/golang/tree/runtime.selectnbrecv)`
 和 `[runtime.selectnbrecv2](https://draveness.me/golang/tree/runtime.selectnbrecv2)` 只是对 `[runtime.chanrecv](https://draveness.me/golang/tree/runtime.chanrecv)`返回值的处理稍有不同与 `[runtime.chansend](https://draveness.me/golang/tree/runtime.chansend)`
 一样，`[runtime.chanrecv](https://draveness.me/golang/tree/runtime.chanrecv)`也提供了一个 `block`参数用于控制这次接收是否阻塞。

### 常规流程

在默认的情况下，编译器会使用如下的流程处理 `select` 语句：

1. 将所有的 `case` 转换成包含 Channel 以及类型等信息的 `[runtime.scase](https://draveness.me/golang/tree/runtime.scase)` 结构体；
2. 调用运行时函数 `[runtime.selectgo](https://draveness.me/golang/tree/runtime.selectgo)` 从多个准备就绪的 Channel 中选择一个可执行的 `[runtime.scase](https://draveness.me/golang/tree/runtime.scase)` 结构体；
3. 通过 `for` 循环生成一组 `if` 语句，在语句中判断自己是不是被选中的 `case`；

一个包含三个 `case` 的正常 `select` 语句其实会被展开成如下所示的逻辑，我们可以看到其中处理的三个部分：

```
selv := [3]scase{}
order := [6]uint16
for i, cas := range cases {
    c := scase{}
    c.kind = ...
    c.elem = ...
    c.c = ...
}
chosen, revcOK := selectgo(selv, order, 3)
if chosen == 0 {
    ...
    break
}
if chosen == 1 {
    ...
    break
}
if chosen == 2 {
    ...
    break
}
```

runtime.selectGo的运行过程：

1. 执行一些必要的初始化操作并确定 `case` 的处理顺序；
2. 在循环中根据 `case`的类型做出不同的处理

**初始化**

`[runtime.selectgo](https://draveness.me/golang/tree/runtime.selectgo)`函数首先会进行执行必要的初始化操作并决定处理 `case`的两个顺序 — 轮询顺序 `pollOrder`和加锁顺序 `lockOrder`：

```
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	ncases := nsends + nrecvs
	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]

	norder := 0
	for i := range scases {
		cas := &scases[i]
	}

	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	pollorder = pollorder[:norder]
	lockorder = lockorder[:norder]

	// 根据 Channel 的地址排序确定加锁顺序
	...
	sellock(scases, lockorder)
	...
}
```

轮询顺序 `pollOrder` 和加锁顺序 `lockOrder` 分别是通过以下的方式确认的：

- 轮询顺序：通过 `[runtime.fastrandn](https://draveness.me/golang/tree/runtime.fastrandn)` 函数引入随机性；
- 加锁顺序：按照 Channel 的地址排序后确定加锁顺序；

随机的轮询顺序可以避免 Channel 的饥饿问题，保证公平性；而根据 Channel 的地址顺序确定加锁顺序能够避免死锁的发生。这段代码最后调用的 `[runtime.sellock](https://draveness.me/golang/tree/runtime.sellock)` 会按照之前生成的加锁顺序锁定 `select` 语句中包含所有的 Channel。

**循环**

当我们为 `select` 语句锁定了所有 Channel 之后就会进入 `[runtime.selectgo](https://draveness.me/golang/tree/runtime.selectgo)` 函数的主循环，它会分三个阶段查找或者等待某个 Channel 准备就绪：

1. 查找是否已经存在准备就绪的 Channel，即可以执行收发操作；
2. 将当前 Goroutine 加入 Channel 对应的收发队列上并等待其他 Goroutine 的唤醒；
3. 当前 Goroutine 被唤醒之后找到满足条件的 Channel 并进行处理；

`[runtime.selectgo](https://draveness.me/golang/tree/runtime.selectgo)` 函数会根据不同情况通过 `goto` 语句跳转到函数内部的不同标签执行相应的逻辑，其中包括：

- `bufrecv`：可以从缓冲区读取数据；
- `bufsend`：可以向缓冲区写入数据；
- `recv`：可以从休眠的发送方获取数据；
- `send`：可以向休眠的接收方发送数据；
- `rclose`：可以从关闭的 Channel 读取 EOF；
- `sclose`：向关闭的 Channel 发送数据；
- `retc`：结束调用并返回；

第一阶段

会根据pollOrder顺序，循环遍历scase，查找已经准备就绪的scase，在这个阶段，我们会根据 `case`
 的四种类型分别处理：

1. 当 `case` 不包含 Channel 时；
    - 这种 `case` 会被跳过；
2. 当 `case` 会从 Channel 中接收数据时；
    - 如果当前 Channel 的 `sendq` 上有等待的 Goroutine，就会跳到 `recv` 标签并从缓冲区读取数据后将等待 Goroutine 中的数据放入到缓冲区中相同的位置；
    - 如果当前 Channel 的缓冲区不为空，就会跳到 `bufrecv` 标签处从缓冲区获取数据；
    - 如果当前 Channel 已经被关闭，就会跳到 `rclose` 做一些清除的收尾工作；
3. 当 `case` 会向 Channel 发送数据时；
    - 如果当前 Channel 已经被关，闭就会直接跳到 `sclose` 标签，触发 `panic` 尝试中止程序；
    - 如果当前 Channel 的 `recvq` 上有等待的 Goroutine，就会跳到 `send` 标签向 Channel 发送数据；
    - 如果当前 Channel 的缓冲区存在空闲位置，就会将待发送的数据存入缓冲区；
4. 当 `select` 语句中包含 `default` 时；
    - 表示前面的所有 `case` 都没有被执行，这里会解锁所有 Channel 并返回，意味着当前 `select` 结构中的收发都是非阻塞的；

第二阶段

按照需要将当前 Goroutine 加入到 Channel 的 `sendq`或者 `recvq`队列中。

根据lockOrder遍历scase，将Goroutine对应的runtime.sudog结构加入队列，这些结构体都会被串成链表附着在 Goroutine 上。在入队之后会调用 `[runtime.gopark](https://draveness.me/golang/tree/runtime.gopark)`挂起当前 Goroutine 等待调度器的唤醒。

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/select/Untitled%201.png)

第三阶段

等到 `select`中的一些 Channel 准备就绪之后，当前 Goroutine 就会被调度器唤醒。这时会继续执行 `[runtime.selectgo](https://draveness.me/golang/tree/runtime.selectgo)`函数的第三部分，从 `[runtime.sudog](https://draveness.me/golang/tree/runtime.sudog)`中读取数据：

```
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	...
	sg = (*sudog)(gp.param)
	gp.param = nil

	casi = -1
	cas = nil
	sglist = gp.waiting
	for _, casei := range lockorder {
		k = &scases[casei]
		if sg == sglist {
			casi = int(casei)
			cas = k
		} else {
			c = k.c
			if int(casei) < nsends {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	c = cas.c
	goto retc
	...
}
```

再次遍历scase，我们会先获取当前 Goroutine 接收到的参数 `sudog`结构，我们会依次对比所有 `case`对应的 `sudog`结构找到被唤醒的 `case`，获取该 `case`对应的索引并返回。

由于当前的 `select`结构找到了一个 `case`执行，那么剩下 `case` 中没有被用到的 `sudog`就会被忽略并且释放掉。为了不影响 Channel 的正常使用，我们还是需要将这些废弃的 `sudog`从 Channel 中出队。

## 总结

我们简单总结一下 `select`结构的执行过程与实现原理，首先在编译期间，Go 语言会对 `select`
 语句进行优化，它会根据 `select`中 `case`的不同选择不同的优化路径。

在编译器已经对 `select`语句进行优化之后，通常情况下，Go 语言会在运行时执行编译期间展开的 `[runtime.selectgo](https://draveness.me/golang/tree/runtime.selectgo)`函数，该函数会按照以下的流程执行：

1. 随机生成一个遍历的轮询顺序 `pollOrder` 并根据 Channel 地址生成锁定顺序 `lockOrder`；
2. 根据 `pollOrder` 遍历所有的 `case` 查看是否有可以立刻处理的 Channel；
    1. 如果存在，直接获取 `case` 对应的索引并返回；
    2. 如果不存在，创建 `[runtime.sudog](https://draveness.me/golang/tree/runtime.sudog)` 结构体，将当前 Goroutine 加入到所有相关 Channel 的收发队列，并调用 `[runtime.gopark](https://draveness.me/golang/tree/runtime.gopark)` 挂起当前 Goroutine 等待调度器的唤醒；
3. 当调度器唤醒当前 Goroutine 时，会再次按照 `lockOrder` 遍历所有的 `case`，从中查找需要被处理的 `[runtime.sudog](https://draveness.me/golang/tree/runtime.sudog)` 对应的索引；