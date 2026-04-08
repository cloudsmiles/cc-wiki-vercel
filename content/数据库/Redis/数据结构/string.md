# string

SDS 是 Redis 自己构建的一种字符串类型，其底层数据结构是一个结构体，包含以下几个字段：

- `len`：表示字符串的长度。
- `free`：表示 SDS 分配的内存中未被使用的字节数。
- `buf`：表示实际存储字符串数据的数组。

SDS 的实现原理如下：

1. 初始时分配一个较小的空间，并将 `len` 字段设置为 0，`free` 字段设置为剩余空间的大小。
2. 当需要修改字符串时，如果剩余空间不足，则会自动扩展空间，扩展后 `free` 字段的大小会增加。
3. 当字符串长度小于等于 `free` 字段的大小时，SDS 不会重新分配内存，而是直接修改 `buf` 中的值；当字符串长度大于 `free` 字段的大小时，SDS 会重新分配内存并拷贝数据。
4. 当字符串长度缩短时，为了节省内存，SDS 不会立即释放多余的空间，而是将多余的空间保留下来，以备后续的扩展使用。

示例图如下：

![Untitled](RPG从零开始/Database/Redis/数据结构/string/Untitled.png)

## SDS与C字符串的区别

C语言使用长度为N+1的字符数组来表示长度为N的字符串，并且字符串的最后一个元素是空字符\0。Redis采用SDS相对于C字符串有如下几个优势：

- 常数复杂度获取字符串长度
- 杜绝缓冲区溢出
- 减少修改字符串时带来的内存重分配次数
- 二进制安全

## raw和embstr编码的SDS区别

长度大于39字节的字符串，编码类型为raw，底层数据结构是简单动态字符串(SDS)。

比如当我们执行set story "Long, long, long ago there lived a king ..."(长度大于39)之后，Redis就会创建一个raw编码的String对象。数据结构如下：

![Untitled](RPG从零开始/Database/Redis/数据结构/string/Untitled%201.png)

长度小于等于39个字节的字符串，编码类型为embstr，底层数据结构则是embstr编码SDS。

embstr编码是专门用来保存短字符串的，它和raw编码最大的不同在于：raw编码会调用两次内存分配分别创建redisObject结构和sdshdr结构，而embstr编码则是只调用一次内存分配，在一块连续的空间上同时包含redisObject结构和sdshdr`结构。

![Untitled](RPG从零开始/Database/Redis/数据结构/string/Untitled%202.png)

## Redis字符串结构特点

- O(1) 时间复杂度获取：字符串长度，已用长度，未用长度。
- 可用于保存字节数组，支持安全的二进制数据存储。
- 内部实现空间预分配机制，降低内存再分配次数。
- 惰性删除机制，字符串缩减后的空间不释放，作为预分配空间保留。

参考

[https://zhuanlan.zhihu.com/p/531323771](https://zhuanlan.zhihu.com/p/531323771)