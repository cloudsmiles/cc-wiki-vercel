# for和range

## 现象

循环永动机 —- for…range运行次数有限

神奇的指针 —- 指向最后一个元素

遍历清空数组 — 编译优化，`[runtime.memclrNoHeapPointers](https://draveness.me/golang/tree/runtime.memclrNoHeapPointers)`

随机遍历 —- map遍历元素随机

## 经典循环

Go 语言中的经典循环在编译器看来是一个 `OFOR` 类型的节点，这个节点由以下四个部分组成：

1. 初始化循环的 `Ninit`；
2. 循环的继续条件 `Left`；
3. 循环体结束时执行的 `Right`；
4. 循环体 `NBody`：

```
for Ninit; Left; Right {
    NBody
}
```

在生成SSA中间代码过程，会切分成不同的块结构，与我们理解的for循环结构一致

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/for和range/Untitled.png)

## 范围循环

与简单的经典循环相比，范围循环在 Go 语言中更常见、实现也更复杂。这种循环同时使用 `for`
 和 `range`两个关键字，编译器会在编译期间将所有 for-range 循环变成经典循环。从编译器的视角来看，就是将 `ORANGE`类型的节点转换成 `OFOR`节点:

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/for和range/Untitled%201.png)

### 数组和切片

对于数组和切片来说，Go 语言有三种不同的遍历方式，这三种不同的遍历方式分别对应着代码中的不同条件，它们会在 `[cmd/compile/internal/gc.walkrange](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkrange)`函数中转换成不同的控制逻辑，我们会分成几种情况分析该函数的逻辑：

1. 分析遍历数组和切片清空元素的情况；
2. 分析使用 `for range a {}` 遍历数组和切片，不关心索引和数据的情况；
3. 分析使用 `for i := range a {}` 遍历数组和切片，只关心索引的情况；（在body增加临时变量）
4. 分析使用 `for i, elem := range a {}` 遍历数组和切片，关心索引和数据的情况；（在body增加临时变量）

```
ha := a
hv1 := 0
hn := len(ha)
v1 := hv1
v2 := nil
for ; hv1 < hn; hv1++ {
    tmp := ha[hv1]
    v1, v2 = hv1, tmp
    ...
}
```

对于所有的 range 循环，Go 语言都会在编译期将原切片或者数组赋值给一个新变量 ha，在赋值的过程中就发生了拷贝，而我们又通过 `len`关键字预先获取了切片的长度，所以在循环中追加新的元素也不会改变循环执行的次数，这也就解释了循环永动机一节提到的现象。

而遇到这种同时遍历索引和元素的 range 循环时，Go 语言会额外创建一个新的 `v2`变量存储切片中的元素，**循环中使用的这个变量 v2 会在每一次迭代被重新赋值而覆盖，赋值时也会触发拷贝。**所以获取这个变量的地址是一样的，不应该直接获取 range 返回的变量地址 `&v2`，而应该使用 `&a[index]`这种形式

## 哈希表

在遍历哈希表时，编译器会使用 `[runtime.mapiterinit](https://draveness.me/golang/tree/runtime.mapiterinit)`和 `[runtime.mapiternext](https://draveness.me/golang/tree/runtime.mapiternext)`两个运行时函数重写原始的 for-range 循环：

```
ha := a
hit := hiter(n.Type)
th := hit.Type
mapiterinit(typename(t), ha, &hit)
for ; hit.key != nil; mapiternext(&hit) {
    key := *hit.key
    val := *hit.val
}
```

使用 `[runtime.mapiterinit](https://draveness.me/golang/tree/runtime.mapiterinit)` 函数初始化遍历开始的元素：

```
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	it.t = t
	it.h = h
	it.B = h.B
	it.buckets = h.buckets

	r := uintptr(fastrand())
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))
	it.bucket = it.startBucket
	mapiternext(it)
}

```

该函数会初始化 `[runtime.hiter](https://draveness.me/golang/tree/runtime.hiter)`结构体中的字段，并通过 `[runtime.fastrand](https://draveness.me/golang/tree/runtime.fastrand)`生成一个随机数帮助我们随机选择一个遍历桶的起始位置。Go 团队在设计哈希表的遍历时就不想让使用者依赖固定的遍历顺序，所以引入了随机数保证遍历的随机性。

使用`[runtime.mapiternext](https://draveness.me/golang/tree/runtime.mapiternext)`函数遍历哈希表

代码的核心逻辑是：

1. 在待遍历的桶为空时，选择需要遍历的新桶；
2. 不存在待遍历的桶时。返回 `(nil, nil)` 键值对并中止遍历；
3. 选择符合要求的桶，开始遍历桶中的元素
4. 遍历完正常桶之后，通过`[runtime.bmap.overflow](https://draveness.me/golang/tree/runtime.bmap.overflow)`遍历哈希中的溢出桶

![Untitled](RPG从零开始/Program%20Language/Go/高级/常用关键字/for和range/Untitled%202.png)

**哈希表的遍历过程**

简单总结一下哈希表遍历的顺序，首先会选出一个绿色的正常桶开始遍历，随后遍历所有黄色的溢出桶，最后依次按照索引顺序遍历哈希表中其他的桶，直到所有的桶都被遍历完成。

### 字符串

遍历字符串的过程与数组、切片和哈希表非常相似，只是在遍历时会获取字符串中索引对应的字节并将字节转换成 `rune`。**我们在遍历字符串时拿到的值都是 `rune`类型的变量**，`for i, r := range s {}`的结构都会被转换成如下所示的形式：

```
ha := s
for hv1 := 0; hv1 < len(ha); {
    hv1t := hv1
    hv2 := rune(ha[hv1])
    if hv2 < utf8.RuneSelf {
        hv1++
    } else {
        hv2, hv1 = decoderune(ha, hv1)
    }
    v1, v2 = hv1t, hv2
}
```

使用下标访问字符串中的元素时得到的就是字节，但是这段代码会将当前的字节转换成 `rune`
 类型。如果当前的 `rune`是 ASCII 的，那么只会占用一个字节长度，每次循环体运行之后只需要将索引加一，但是如果当前 `rune`占用了多个字节就会使用 `[runtime.decoderune](https://draveness.me/golang/tree/runtime.decoderune)`函数解码

### 通道

使用 range 遍历 Channel 也是比较常见的做法，一个形如 `for v := range ch {}`的语句最终会被转换成如下的格式：

```
ha := a
hv1, hb := <-ha
for ; hb != false; hv1, hb = <-ha {
    v1 := hv1
    hv1 = nil
    ...
}
```

该循环会使用 `<-ch` 从管道中取出等待处理的值，这个操作会调用 `[runtime.chanrecv2](https://draveness.me/golang/tree/runtime.chanrecv2)` 并阻塞当前的协程，当 `[runtime.chanrecv2](https://draveness.me/golang/tree/runtime.chanrecv2)` 返回时会根据布尔值 `hb` 判断当前的值是否存在：

- 如果不存在当前值，意味着当前的管道已经被关闭；
- 如果存在当前值，会为 `v1` 赋值并清除 `hv1` 变量中的数据，然后重新陷入阻塞等待新数据；

## 总结

这一节介绍的两个关键字 `for`和 `range`都是我们在学习和使用 Go 语言中无法绕开的，通过分析和研究它们的底层原理，让我们对实现细节有了更清楚的认识，包括 Go 语言遍历数组和切片时会复用变量、哈希表的随机遍历原理以及底层的一些优化，这都能帮助我们更好地理解和使用 Go 语言。