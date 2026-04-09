# make和new

当我们想要在 Go 语言中初始化一个结构时，可能会用到两个不同的关键字 — `make` 和 `new`。因为它们的功能相似，所以初学者可能会对这两个关键字的作用感到困惑[1](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#fn:1)，但是它们两者能够初始化的变量却有较大的不同。

- `make` 的作用是初始化内置的数据结构，也就是我们在前面提到的切片、哈希表和 Channel；
- `new` 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针；
    
    
    ![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/make和new/Untitled.png)
    

## make

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/make和new/Untitled%201.png)

在编译期间的类型检查阶段，Go 语言会将代表 `make`关键字的 `OMAKE`节点根据参数类型的不同转换成了 `OMAKESLICE`、`OMAKEMAP`和 `OMAKECHAN` 三种不同类型的节点，这些节点会调用不同的运行时函数来初始化相应的数据结构。

## new

编译器会在中间代码生成阶段通过以下两个函数处理该关键字：

1. `[cmd/compile/internal/gc.callnew](https://draveness.me/golang/tree/cmd/compile/internal/gc.callnew)` 会将关键字转换成 `ONEWOBJ` 类型的节点；
    
    [2](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#fn:2)
    
2. `[cmd/compile/internal/gc.state.expr](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.expr)` 会根据申请空间的大小分两种情况处理：
    1. 如果申请的空间为 0，就会返回一个表示空指针的 `zerobase` 变量；
    2. 在遇到其他情况时会将关键字转换成 `[runtime.newobject](https://draveness.me/golang/tree/runtime.newobject)` 函数：
    
    ```
    func callnew(t *types.Type) *Node {
    	...
    	n := nod(ONEWOBJ, typename(t), nil)
    	...
    	return n
    }
    
    func (s *state) expr(n *Node) *ssa.Value {
    	switch n.Op {
    	case ONEWOBJ:
    		if n.Type.Elem().Size() == 0 {
    			return s.newValue1A(ssa.OpAddr, n.Type, zerobaseSym, s.sb)
    		}
    		typ := s.expr(n.Left)
    		vv := s.rtcall(newobject, true, []*types.Type{n.Type}, typ)
    		return vv[0]
    	}
    }
    ```
    

需要注意的是，无论是直接使用 `new`，还是使用 `var`初始化变量，它们在编译器看来都是 `ONEW`
 和 `ODCL`节点。如果变量会逃逸到堆上，这些节点在这一阶段都会被 `[cmd/compile/internal/gc.walkstmt](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt)`转换成通过 `[runtime.newobject](https://draveness.me/golang/tree/runtime.newobject)`函数并在堆上申请内存。

不过这也不是绝对的，如果通过 `var`或者 `new`创建的变量不需要在当前作用域外生存，例如不用作为返回值返回给调用方，那么就不需要初始化在堆上。

`[runtime.newobject](https://draveness.me/golang/tree/runtime.newobject)` 函数会获取传入类型占用空间的大小，调用 `[runtime.mallocgc](https://draveness.me/golang/tree/runtime.mallocgc)` 在堆上申请一片内存空间并返回指向这片内存空间的指针：

```
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}
```