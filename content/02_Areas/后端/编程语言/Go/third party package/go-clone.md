# go-clone

## **背景**

这个库是 [github.com/huandu/go-clone](https://link.zhihu.com/?target=https%3A//github.com/huandu/go-clone)，主要用途是对任意的 Go 结构进行深拷贝，创造一个内容完全相同的副本，得到的值可通过 [reflect.DeepEqual](https://link.zhihu.com/?target=https%3A//pkg.go.dev/reflect%3Ftab%3Ddoc%23DeepEqual) 检查。

这个功能看起来挺常用的，不过很奇怪在 Go 世界里面可用的实现却很少，在动手实现之前我调查了几个类似的库或者可用来做深拷贝：

- [encoding/gob](https://link.zhihu.com/?target=https%3A//pkg.go.dev/encoding/gob%3Ftab%3Ddoc) 或 [encoding/json](https://link.zhihu.com/?target=https%3A//pkg.go.dev/encoding/json%3Ftab%3Ddoc)：先将数据结构进行编码（[gob.Encoder](https://link.zhihu.com/?target=https%3A//pkg.go.dev/encoding/gob%3Ftab%3Ddoc%23Encoder) 或 [json.Marshal](https://link.zhihu.com/?target=https%3A//pkg.go.dev/encoding/json%3Ftab%3Ddoc%23Marshal)），得到 `[]byte`之后再解码（[gob.Decoder](https://link.zhihu.com/?target=https%3A//pkg.go.dev/encoding/gob%3Ftab%3Ddoc%23Decoder) 或 [json.Unmarshal](https://link.zhihu.com/?target=https%3A//pkg.go.dev/encoding/json%3Ftab%3Ddoc%23Unmarshal)）。这种做法的好处是简单粗暴，基本上能够应对大部分的需求，缺点则是性能极低，且有各种限制，比如无法处理递归指针、无法处理 `interface` 类型数据，特别是 JSON，会丢失缺少大部分数据类型甚至精度。
- [github.com/jinzhu/copier](https://link.zhihu.com/?target=https%3A//github.com/jinzhu/copier) 或 ：这两个库都实现了基本的 `struct` 拷贝能力，不过缺乏递归指针的处理，也不能作为通用的深拷贝来使用。
    
    [github.com/ulule/deepcopier](https://link.zhihu.com/?target=https%3A//github.com/ulule/deepcopier)
    

## **实现思路**

要实现深拷贝函数 `Clone(v interface{}) interface{}`，其基本思路很简单：

- 首先通过函数 `val := reflect.ValueOf(v)` 拿到 `v` 的反射值；
- 根据 `val.Kind()` 区分各种类型，主要分两种：一种是 scala 类型，即数值类型，包括各种整型、浮点、虚数、字符串等，直接返回原值即可；一种是复杂类型，每种类型用对应的反射方法来创建，包括 `reflect.New`/`reflect.MakeMap`/`reflect.MakeSlice` / `reflect.MakeChan` 等方法；
- 通过各种反射方法来将新申请 `val.Set*` 方法将新值设置到新申请的变量里面。

这里面比较麻烦的是处理 `struct`，为了深拷贝结构，必须首先通过 `val.NumField()` 得到 struct field 个数，然后用循环不断的将 `val.Field(i)` 的值拷贝到新申请的变量对应字段里面去，这里递归调用深拷贝方法即可。