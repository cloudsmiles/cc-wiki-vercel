# defer

## 现象

我们在 Go 语言中使用 `defer` 时会遇到两个常见问题，这里会介绍具体的场景并分析这两个现象背后的设计原理：

- `defer` 关键字的调用时机以及多次调用 `defer` 时执行顺序是如何确定的；
- `defer` 关键字使用传值的方式传递参数时会进行预计算，导致不符合预期的结果；
- defer与return的运行顺序

### 作用域

向 `defer` 关键字传入的函数会在函数返回之前运行。假设我们在 `for`循环中多次调用 `defer`关键字：

```
func main() {
	for i := 0; i < 5; i++ {
		defer fmt.Println(i)
	}
}

$ go run main.go
4
3
2
1
0
```

运行上述代码会**倒序执行传入 `defer`关键字的所有表达式**。

defer不是在退出代码块的作用域时执行的，它只会在当前函数和方法返回之前被调用。

### 预计算参数

Go 语言中所有的函数调用都是传值的，虽然 `defer`是关键字，但是也继承了这个特性。假设我们想要计算`main`函数运行的时间，可能会写出以下的代码：

```
func main() {
	startedAt := time.Now()
	defer fmt.Println(time.Since(startedAt))

	time.Sleep(time.Second)
}

$ go run main.go
0s
```

原因是调用defer关键字的时候，会立刻拷贝函数中引用的外部参数，所以 `time.Since(startedAt)`的结果不是在 `main` 函数退出之前计算的，而是在 `defer`关键字调用时计算的

想要解决这个问题的方法非常简单，我们只需要向 `defer` 关键字传入匿名函数：

```
func main() {
	startedAt := time.Now()
	defer func() { fmt.Println(time.Since(startedAt)) }()

	time.Sleep(time.Second)
}

$ go run main.go
1s
```

虽然调用 `defer`关键字时也使用值传递，但是因为拷贝的是函数指针，所以 `time.Since(startedAt)`会在 `main`函数返回前调用并打印出符合预期的结果。

### return的处理

return返回时，实际上并不是原子操作，而是以下的操作：

- 设置返回参数
- 调用ret指令

**defer的处理时机，是在设置返回参数之后，调用ret指令之前**

## 数据结构

在介绍 `defer` 函数的执行过程与实现原理之前，我们首先来了解一下 `defer`关键字在 Go 语言源代码中对应的数据结构：

```
type _defer struct {
	siz       int32
	started   bool
	openDefer bool
	sp        uintptr
	pc        uintptr
	fn        *funcval
	_panic    *_panic
	link      *_defer
}
```

`[runtime._defer](https://draveness.me/golang/tree/runtime._defer)`结构体是延迟调用链表上的一个元素，所有的结构体都会通过 `link`字段串联成链表。

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/defer/Untitled.png)

我们简单介绍一下 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)` 结构体中的几个字段：

- `siz` 是参数和结果的内存大小；
- `sp` 和 `pc` 分别代表栈指针和调用方的程序计数器；
- `fn` 是 `defer` 关键字中传入的函数；
- `_panic` 是触发延迟调用的结构体，可能为空；
- `openDefer` 表示当前 `defer` 是否经过开放编码的优化；

## 执行机制

处理defer关键字，会根据条件的不同，使用三种不同的机制处理：

```
func (s *state) stmt(n *Node) {
	...
	switch n.Op {
	case ODEFER:
		if s.hasOpenDefers {
			s.openDeferRecord(n.Left) // 开放编码
		} else {
			d := callDefer // 堆分配
			if n.Esc == EscNever {
				d = callDeferStack // 栈分配
			}
			s.callResult(n.Left, d)
		}
	}
}
```

## 堆上分配

堆上分配的 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)`结构体是默认的兜底方案，当该方案被启用时，编译器会调用 `[cmd/compile/internal/gc.state.callResult](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.callResult)`和 `[cmd/compile/internal/gc.state.call](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call)`，这表示 `defer`在编译器看来也是函数调用。

`[cmd/compile/internal/gc.state.call](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call)` 会负责为所有函数和方法调用生成中间代码，它的工作包括以下内容：

1. 获取需要执行的函数名、闭包指针、代码指针和函数调用的接收方；
2. 获取栈地址并将函数或者方法的参数写入栈中；
3. 使用 `[cmd/compile/internal/gc.state.newValue1A](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.newValue1A)` 以及相关函数生成函数调用的中间代码；
4. 如果当前调用的函数是 `defer`，那么会单独生成相关的结束代码块；
5. 获取函数的返回值地址并结束当前调用；

`defer` 关键字在运行期间会调用 `[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)`，这个函数接收了参数的大小和闭包所在的地址两个参数。

而编译器还会在调用defer的函数末尾插入`[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)`的函数调用。

所以也就是这两个runtime的函数，是defer关键字运行时机制的入口，分别承担不同的工作：

- `[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)` 负责创建新的延迟调用；
- `[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)` 负责在函数调用结束时执行所有的延迟调用；

### 创建延迟调用

`[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)`会为 `defer`创建一个新的 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)` 结构体、设置它的函数指针 `fn`、程序计数器 `pc` 和栈指针 `sp`并将相关的参数拷贝到相邻的内存空间中：

```
func deferproc(siz int32, fn *funcval) {
	sp := getcallersp()
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	callerpc := getcallerpc()

	d := newdefer(siz)
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
	d.fn = fn
	d.pc = callerpc
	d.sp = sp
	switch siz {
	case 0:
	case sys.PtrSize:
		*(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
	default:
		memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
	}

	return0()
}
```

`[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)` 中 `[runtime.newdefer](https://draveness.me/golang/tree/runtime.newdefer)` 的作用是想尽办法获得 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)` 结构体，这里包含三种路径：

1. 从调度器的延迟调用缓存池 `sched.deferpool` 中取出结构体并将该结构体追加到当前 Goroutine 的缓存池中；
2. 从 Goroutine 的延迟调用缓存池 `pp.deferpool` 中取出结构体；
3. 通过 `[runtime.mallocgc](https://draveness.me/golang/tree/runtime.mallocgc)` 在堆上创建一个新的结构体；

```
func newdefer(siz int32) *_defer {
	var d *_defer
	sc := deferclass(uintptr(siz))
	gp := getg()
	if sc < uintptr(len(p{}.deferpool)) {
		pp := gp.m.p.ptr()
		if len(pp.deferpool[sc]) == 0 && sched.deferpool[sc] != nil {
			for len(pp.deferpool[sc]) < cap(pp.deferpool[sc])/2 && sched.deferpool[sc] != nil {
				d := sched.deferpool[sc]
				sched.deferpool[sc] = d.link
				pp.deferpool[sc] = append(pp.deferpool[sc], d)
			}
		}
		if n := len(pp.deferpool[sc]); n > 0 {
			d = pp.deferpool[sc][n-1]
			pp.deferpool[sc][n-1] = nil
			pp.deferpool[sc] = pp.deferpool[sc][:n-1]
		}
	}
	if d == nil {
		total := roundupsize(totaldefersize(uintptr(siz)))
		d = (*_defer)(mallocgc(total, deferType, true))
	}
	d.siz = siz
	d.link = gp._defer
	gp._defer = d
	return d
}
```

无论使用哪种方式，只要获取到 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)`结构体，**它都会被追加到所在 Goroutine `_defer`链表的最前面。**

`defer` 关键字的插入顺序是从后向前的，而 `defer`关键字执行是从前向后的，这也是为什么后调用的 `defer`会优先执行。

### 执行延迟调用

`[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)`会从 Goroutine 的 `_defer`链表中取出最前面的 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)`并调用 `[runtime.jmpdefer](https://draveness.me/golang/tree/runtime.jmpdefer)`传入需要执行的函数和参数：

```
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	...

	switch d.siz {
	case 0:
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	gp._defer = d.link
	freedefer(d)
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}

```

`[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)`会多次判断当前 Goroutine 的 `_defer`链表中是否有未执行的结构体，该函数只有在所有延迟函数都执行后才会返回。

## 栈上分配

在默认情况下，我们可以看到 Go 语言中 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)` 结构体都会在堆上分配，如果我们能够将部分结构体分配到栈上就可以节约内存分配带来的额外开销。

Go 语言团队在 1.13 中对 `defer` 关键字进行了优化，当该关键字在函数体中最多执行一次时，编译期间的 `[cmd/compile/internal/gc.state.call](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call)`会将结构体分配到栈上并调用 `[runtime.deferprocStack](https://draveness.me/golang/tree/runtime.deferprocStack)`：

```
func (s *state) call(n *Node, k callKind) *ssa.Value {
	...
	var call *ssa.Value
	if k == callDeferStack {
		// 在栈上创建 _defer 结构体
		t := deferstruct(stksize)
		...

		ACArgs = append(ACArgs, ssa.Param{Type: types.Types[TUINTPTR], Offset: int32(Ctxt.FixedFrameSize())})
		aux := ssa.StaticAuxCall(deferprocStack, ACArgs, ACResults) // 调用 deferprocStack
		arg0 := s.constOffPtrSP(types.Types[TUINTPTR], Ctxt.FixedFrameSize())
		s.store(types.Types[TUINTPTR], arg0, addr)
		call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, aux, s.mem())
		call.AuxInt = stksize
	} else {
		...
	}
	s.vars[&memVar] = call
	...
}
```

因为在编译期间我们已经创建了 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)`结构体，所以在运行期间 `[runtime.deferprocStack](https://draveness.me/golang/tree/runtime.deferprocStack)`只需要设置一些未在编译期间初始化的字段，就可以将栈上的 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)`追加到函数的链表上：

```
func deferprocStack(d *_defer) {
	gp := getg()
	d.started = false
	d.heap = false // 栈上分配的 _defer
	d.openDefer = false
	d.sp = getcallersp()
	d.pc = getcallerpc()
	d.framepc = 0
	d.varp = 0
	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.fd)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
	*(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
}
```

除了分配位置的不同，栈上分配和堆上分配的 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)`并没有本质的不同，而该方法可以适用于绝大多数的场景，与堆上分配的 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)`相比，该方法可以将 `defer`关键字的额外开销降低 ~30%。

## 开放编码

Go 语言在 1.14 中通过开放编码（Open Coded）实现 `defer` 关键字，该设计使用代码内联优化 `defer` 关键的额外开销并引入函数数据 `funcdata`管理 `panic`的调用[3](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#fn:3)，该优化可以将 `defer`
 的调用开销从 1.13 版本的 ~35ns 降低至 ~6ns 左右

然而开放编码作为一种优化 `defer` 关键字的方法，它不是在所有的场景下都会开启的，开放编码只会在满足以下的条件时启用：

1. 函数的 `defer` 数量少于或者等于 8 个；
2. 函数的 `defer` 关键字不能在循环中执行；
3. 函数的 `return` 语句与 `defer` 语句的乘积小于或者等于 15 个；

### 启用优化

Go 语言会在编译期间就确定是否启用开放编码，在编译器生成中间代码之前，我们会使用 `[cmd/compile/internal/gc.walkstmt](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt)`修改已经生成的抽象语法树，设置函数体上的 `OpenCodedDeferDisallowed`属性：

```
const maxOpenDefers = 8

func walkstmt(n *Node) *Node {
	switch n.Op {
	case ODEFER:
		Curfn.Func.SetHasDefer(true)
		Curfn.Func.numDefers++
		if Curfn.Func.numDefers > maxOpenDefers {
			Curfn.Func.SetOpenCodedDeferDisallowed(true)
		}
		if n.Esc != EscNever {
			Curfn.Func.SetOpenCodedDeferDisallowed(true)
		}
		fallthrough
	...
	}
}
```

从代码中可以看到，如果defer多于8个，或者处于for循环，将会禁用开放编码优化

在 SSA 中间代码生成阶段的 `[cmd/compile/internal/gc.buildssa](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa)`中，我们也能够看到启用开放编码优化的其他条件，也就是返回语句的数量与 `defer`数量的乘积需要小于 15。

### 延迟记录

延迟比特和延迟记录是使用开放编码实现 `defer`的两个最重要结构，一旦决定使用开放编码，`[cmd/compile/internal/gc.buildssa](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa)`会在编译期间在栈上初始化大小为 8 个比特的 `deferBits`变量：

```
func buildssa(fn *Node, worker int) *ssa.Func {
	...
	if s.hasOpenDefers {
		deferBitsTemp := tempAt(src.NoXPos, s.curfn, types.Types[TUINT8]) // 初始化延迟比特
		s.deferBitsTemp = deferBitsTemp
		startDeferBits := s.entryNewValue0(ssa.OpConst8, types.Types[TUINT8])
		s.vars[&deferBitsVar] = startDeferBits
		s.deferBitsAddr = s.addr(deferBitsTemp)
		s.store(types.Types[TUINT8], s.deferBitsAddr, startDeferBits)
		s.vars[&memVar] = s.newValue1Apos(ssa.OpVarLive, types.TypeMem, deferBitsTemp, s.mem(), false)
	}
}
```

延迟比特中的每一个比特位都表示该位对应的 `defer` 关键字是否需要被执行，如下图所示，其中 8 个比特的倒数第二个比特在函数返回前被设置成了 1，那么该比特位对应的函数会在函数返回前执行：

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/defer/Untitled%201.png)

延迟比特的作用就是标记哪些 `defer`关键字在函数中被执行，这样在函数返回时可以根据对应 `deferBits`的内容确定执行的函数，**而正是因为 `deferBits`的大小仅为 8 比特，所以该优化的启用条件为函数中的 `defer` 关键字少于 8 个。**

很多 `defer` 语句都可以在编译期间判断是否被执行，如果函数中的 `defer`语句都会在编译期间确定，中间代码生成阶段就会直接调用 `[cmd/compile/internal/gc.state.openDeferExit](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.openDeferExit)`在函数返回前生成判断 `deferBits`的代码，也就是上述伪代码中的后半部分。

不过当程序遇到运行时才能判断的条件语句时，我们仍然需要由运行时的 `[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)`
 决定是否执行 `defer`关键字：

```
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	sp := getcallersp()
	if d.openDefer {
		runOpenDeferFrame(gp, d)
		gp._defer = d.link
		freedefer(d)
		return
	}
	...
}
```

该函数为开放编码做了特殊的优化，运行时会调用 `[runtime.runOpenDeferFrame](https://draveness.me/golang/tree/runtime.runOpenDeferFrame)` 执行活跃的开放编码延迟函数，该函数会执行以下的工作：

1. 从 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)` 结构体中读取 `deferBits`、函数 `defer` 数量等信息；
2. 在循环中依次读取函数的地址和参数信息并通过 `deferBits` 判断该函数是否需要被执行；
3. 调用 `[runtime.reflectcallSave](https://draveness.me/golang/tree/runtime.reflectcallSave)` 调用需要执行的 `defer` 函数；

## 总结

`defer` 关键字的实现主要依靠编译器和运行时的协作，我们总结一下本节提到的三种机制：

- 堆上分配 · 1.1 ~ 1.12
    - 编译期将 `defer` 关键字转换成 `[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)` 并在调用 `defer` 关键字的函数返回之前插入 `[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)`；
    - 运行时调用 `[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)` 会将一个新的 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)` 结构体追加到当前 Goroutine 的链表头；
    - 运行时调用 `[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)` 会从 Goroutine 的链表中取出 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)` 结构并依次执行；
- 栈上分配 · 1.13
    - 当该关键字在函数体中最多执行一次时，编译期间的 `[cmd/compile/internal/gc.state.call](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call)` 会将结构体分配到栈上并调用 `[runtime.deferprocStack](https://draveness.me/golang/tree/runtime.deferprocStack)`；
- 开放编码 · 1.14 ~ 现在
    - 编译期间判断 `defer` 关键字、`return` 语句的个数确定是否开启开放编码优化；
    - 通过 `deferBits` 和 `[cmd/compile/internal/gc.openDeferInfo](https://draveness.me/golang/tree/cmd/compile/internal/gc.openDeferInfo)` 存储 `defer` 关键字的相关信息；
    - 如果 `defer` 关键字的执行可以在编译期间确定，会在函数返回前直接插入相应的代码，否则会由运行时的 `[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)` 处理；