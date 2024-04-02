# panic和recover

- `panic` 能够改变程序的控制流，调用 `panic` 后会立刻停止执行当前函数的剩余代码，并在当前 Goroutine 中递归执行调用方的 `defer`；
- `recover` 可以中止 `panic` 造成的程序崩溃。它是一个只能在 `defer` 中发挥作用的函数，在其他作用域中调用不会发挥作用；

## 现象

我们先通过几个例子了解一下使用 `panic` 和 `recover` 关键字时遇到的现象，部分现象也与上一节分析的 `defer` 关键字有关：

- `panic` 只会触发当前 Goroutine 的 `defer`；
- `recover` 只有在 `defer` 中调用才会生效；
- `panic` 允许在 `defer` 中嵌套多次调用；

### 跨协程失效

panic只会触发当前Goroutine的延迟函数调用。

前面我们曾经介绍过 `defer`关键字对应的 `[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)`会将延迟调用函数与调用方所在 Goroutine 进行关联。所以当程序发生崩溃时只会调用当前 Goroutine 的延迟调用函数也是非常合理的。

如下图所示，一个Goroutine在panic时，也不应该执行其他Goroutine的延迟函数

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/panic和recover/Untitled.png)

### 失效的崩溃恢复

在主程序中调用recover试图中止程序崩溃，但是没有正常退出，原因是recover只有在发生panic之后调用才会生效。所以要在defer中使用recover关键字

### 嵌套崩溃

Go 语言中的 `panic`是可以多次嵌套调用的。多次调用 `panic`也不会影响 `defer` 函数的正常执行，所以使用 `defer` 进行收尾工作一般来说都是安全的。

## 数据结构

`panic` 关键字在 Go 语言的源代码是由数据结构 `[runtime._panic](https://draveness.me/golang/tree/runtime._panic)`表示的。每当我们调用 `panic`都会创建一个如下所示的数据结构存储相关信息:

```
type _panic struct {
	argp      unsafe.Pointer
	arg       interface{}
	link      *_panic
	recovered bool
	aborted   bool
	pc        uintptr
	sp        unsafe.Pointer
	goexit    bool
}
```

1. `argp` 是指向 `defer` 调用时参数的指针；
2. `arg` 是调用 `panic` 时传入的参数；
3. `link` 指向了更早调用的 `[runtime._panic](https://draveness.me/golang/tree/runtime._panic)` 结构；
4. `recovered` 表示当前 `[runtime._panic](https://draveness.me/golang/tree/runtime._panic)` 是否被 `recover` 恢复；
5. `aborted` 表示当前的 `panic` 是否被强行终止；

根据link字段可以看出panic函数可以调用多次，形成_panic链表

## 程序崩溃

这里先介绍分析 `panic` 函数是终止程序的实现原理。编译器会将关键字 `panic` 转换成 `[runtime.gopanic](https://draveness.me/golang/tree/runtime.gopanic)`，该函数的执行过程包含以下几个步骤：

1. 创建新的 `[runtime._panic](https://draveness.me/golang/tree/runtime._panic)` 并添加到所在 Goroutine 的 `_panic` 链表的最前面；
2. 在循环中不断从当前 Goroutine 的 `_defer` 中链表获取 `[runtime._defer](https://draveness.me/golang/tree/runtime._defer)` 并调用 `[runtime.reflectcall](https://draveness.me/golang/tree/runtime.reflectcall)` 运行延迟调用函数；
3. 调用 `[runtime.fatalpanic](https://draveness.me/golang/tree/runtime.fatalpanic)` 中止整个程序；

```
func gopanic(e interface{}) {
	gp := getg()
	...
	var p _panic
	p.arg = e
	p.link = gp._panic
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	for {
		d := gp._defer
		if d == nil {
			break
		}

		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))

		d._panic = nil
		d.fn = nil
		gp._defer = d.link

		freedefer(d)
		if p.recovered {
			...
		}
	}

	fatalpanic(gp._panic)
	*(*int)(nil) = 0
}
```

## 崩溃恢复

到这里我们已经掌握了 `panic`退出程序的过程，接下来将分析 `defer` 中的 `recover`
 是如何中止程序崩溃的。编译器会将关键字 `recover`转换成 `[runtime.gorecover](https://draveness.me/golang/tree/runtime.gorecover)`：

```
func gorecover(argp uintptr) interface{} {
	gp := getg()
	p := gp._panic
	if p != nil && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}
```

该函数的实现很简单，如果当前 Goroutine 没有调用 `panic`，那么该函数会直接返回 `nil`，这也是崩溃恢复在非 `defer`中调用会失效的原因。在正常情况下，它会修改 `[runtime._panic](https://draveness.me/golang/tree/runtime._panic)` 的 `recovered` 字段，`[runtime.gorecover](https://draveness.me/golang/tree/runtime.gorecover)`函数中并不包含恢复程序的逻辑，程序的恢复是由 `[runtime.gopanic](https://draveness.me/golang/tree/runtime.gopanic)`函数负责的：

```
func gopanic(e interface{}) {
	...

	for {
		// 执行延迟调用函数，可能会设置 p.recovered = true
		...

		pc := d.pc
		sp := unsafe.Pointer(d.sp)

		...
		if p.recovered {
			gp._panic = p.link
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}
			if gp._panic == nil {
				gp.sig = 0
			}
			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			mcall(recovery)
			throw("recovery failed")
		}
	}
	...
}
```

`[runtime.recovery](https://draveness.me/golang/tree/runtime.recovery)`在调度过程中会将函数的返回值设置成 1。从 `[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)`
 的注释中我们会发现，当 `[runtime.deferproc](https://draveness.me/golang/tree/runtime.deferproc)` 函数的返回值是 1 时，编译器生成的代码会直接跳转到调用方函数返回之前并执行 `[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)。`

跳转到 `[runtime.deferreturn](https://draveness.me/golang/tree/runtime.deferreturn)`函数之后，程序就已经从 `panic`中恢复了并执行正常的逻辑，而 `[runtime.gorecover](https://draveness.me/golang/tree/runtime.gorecover)`函数也能从 `[runtime._panic](https://draveness.me/golang/tree/runtime._panic)`结构中取出了调用 `panic`时传入的 `arg`参数并返回给调用方。

### recover和panic结对判断

其中 `p.argp`在 `gopanic`中赋值为即将调用的 defer 函数的参数列表起始地址：

```
p.argp = unsafe.Pointer(getargp(0))
reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
```

而 `gorecover`的参数 `argp`则为 `recover`的上一层函数的参数列表起始地址：

```
case ORECOVER:
    n = mkcall("gorecover", n.Type, init, nod(OADDR, nodfp, nil))
```

**所以，`argp == uintptr(p.argp)`意味着 `recover`与对应的 `panic`正好相隔 1 个 在 `gopanic`中调用的 defer 函数。**