# 哈希表map原理

[理解 Golang 哈希表 Map 的原理](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#%E8%AE%BF%E9%97%AE)

[Go 语言 map 的底层实现完整剖析](https://zhuanlan.zhihu.com/p/406751292)

## 哈希碰撞冲突

### 开放寻址法

```jsx
index := hash("author") % array.len
```

如果index冲突，就选择后面的元素写入

开放寻址法中对性能影响最大的是**装载因子**
，它是数组中元素的数量与数组大小的比值。随着装载因子的增加，线性探测的平均用时就会逐渐增加，这会影响哈希表的读写性能。当装载率超过 70% 之后，哈希表的性能就会急剧下降，而一旦装载率达到 100%，整个哈希表就会完全失效，这时查找和插入任意元素的时间复杂度都是 𝑂(𝑛)O(n)
 的，这时需要遍历数组中的全部元素，所以在实现哈希表时一定要关注装载因子的变化。

### 拉链法

数组+链表的组合

```jsx
index := hash("Key6") % array.len
```

先通过index，找到存储的桶，然后

1. 找到键相同的键值对 — 更新键对应的值；
2. 没有找到键相同的键值对 — 在链表的末尾追加新的键值对；

## 数据结构

### Go的map数据结构

```go
// A header for a Go map.
type hmap struct {
    // Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
    // Make sure this stays in sync with the compiler's definition.
    count     int // # live cells == size of map.  Must be first (used by len() builtin)
    flags     uint8
    B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
    noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
    hash0     uint32 // hash seed

    buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
    oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
    nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

    extra *mapextra // optional fields
}
```

hmap存放map的元数据

count：表示map元素数量

flags：代表目前map的状态

B：代表桶数量的log2值，即2^B次方 = 桶数量

hash0：随机哈希种子，目的是降低哈希冲突

buckets：指向当前哈希桶数组的指针

oldbuckets：指向旧桶

nevacuate：桶进行调整时的搬迁进度，小于此地址的 buckets 是已经搬迁完

extra：表示溢出桶

### Go的桶数据结构

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    elems    [8]elemtype
    //pad      uintptr(新的 go 版本已经移除了该字段, 我未具体了解此处的 change detail, 之前设置该字段是为了在 nacl/amd64p32 上的内存对齐)
    overflow uintptr
}
```

topbits存放高八位hash值，keys和elems存放键值，overflow存放溢出桶的指针

## 初始化

### 字面量

目前的现代编程语言基本都支持使用字面量的方式初始化哈希，一般都会使用 `key: value`
 的语法来表示键值对，Go 语言中也不例外：

```
hash := map[string]int{
	"1": 2,
	"3": 4,
	"5": 6,
}
```

当哈希表中的元素数量少于或者等于 25 个时，编译器会将字面量初始化的结构体转换成逐个赋值的方式，将所有的键值对一次加入到哈希表中

一旦哈希表中元素的数量超过了 25 个，编译器会创建两个数组分别存储键和值，这些键值对会通过如下所示的 for 循环加入哈希：

```
hash := make(map[string]int, 26)
vstatk := []string{"1", "2", "3", ... ， "26"}
vstatv := []int{1, 2, 3, ... , 26}
for i := 0; i < len(vstak); i++ {
    hash[vstatk[i]] = vstatv[i]
}
```

**不过无论使用哪种方法，使用字面量初始化的过程都会使用 Go 语言中的关键字 `make`来创建新的哈希并通过最原始的 `[]`语法向哈希追加元素。**

### 运行时

当创建的哈希被分配到栈上并且其容量小于 `BUCKETSIZE = 8`
 时，Go 语言在编译阶段会使用如下方式快速初始化哈希，这也是编译器对小容量的哈希做的优化：

```
var h *hmap
var hv hmap
var bv bmap
h := &hv
b := &bv
h.buckets = b
h.hash0 = fashtrand0()
```

除了上述的优化，其他的make最后调用的都是 `[runtime.makemap](https://draveness.me/golang/tree/runtime.makemap)`：

```
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

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

这个函数会按照下面的步骤执行：

1. 计算哈希占用的内存是否溢出或者超出能分配的最大值；
2. 调用 `[runtime.fastrand](https://draveness.me/golang/tree/runtime.fastrand)` 获取一个随机的哈希种子；
3. 根据传入的 `hint` 计算出需要的最小需要的桶的数量；
4. 使用 `[runtime.makeBucketArray](https://draveness.me/golang/tree/runtime.makeBucketArray)` 创建用于保存桶的数组；

## Go的map解析

1.使用拉链法方式，底层本质是数组+链表

2.map的查找

2.1 key→hashFuction()→hashVal

2.2 取hashVal的低八位，与数组长度求余，得到桶编号，桶编号*桶长度+基础地址，得到对应的哈希桶地址

2.3 取hashVal的高八位，辅助key的查找，如果桶的topbits命中高八位，根据topbits的位置，得到key的位置，如果key相同，则返回，如果不相同，继续往下比较topbits，如果该桶没有，则往溢出桶查找

3.map的插入/更新

3.1 同key的查找

3.2 如果能查找则更新，查找不到，往下增加数据，或者创建溢出桶

3.3 需要判断是否扩容，等扩容完，再写入

4.map的删除

4.1 同key的查找

4.2 清除数据

5.map的扩容

go的装载因子计算：

loadFactor = 元素数量/哈希桶数量（含溢出桶）

5.0 map 的扩容是发生在插入和删除的过程

5.1 扩容条件，装载因子达到6.5以上，或者溢出桶过多，装载因子不大，溢出桶过多说明很多桶的利用率不高，造成碎片，需要重新搬迁

5.2 扩容流程，先是开创新的内存空间，然后再迁移

5.3 方式上，有等量扩容和翻倍扩容，等量扩容是指溢出桶过多，需要重新组织，桶数组的相对位置不变；翻倍扩容需要对元素进行分流，分为Xpart和Ypart，即两个同等容量的桶数组（通过对topbits的前一位进行判断，0和1分成Xpart和Ypart，新的bucket还是取高八位）