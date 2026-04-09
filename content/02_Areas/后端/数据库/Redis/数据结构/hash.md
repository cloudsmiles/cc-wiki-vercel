# hash

哈希对象的编码可以是ziplist或者hashtable。

哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节并且保存的键值对数量小于 512 个，使用ziplist 编码；否则使用hashtable；

## hash-ziplist

ziplist底层使用的是压缩列表实现，前面已经详细介绍了压缩列表的实现原理。每当有新的键值对要加入哈希对象时，先把保存了键的节点推入压缩列表表尾，然后再将保存了值的节点推入压缩列表表尾。比如，我们执行如下三条HSET命令：

```c
HSET profile name "tom"
HSET profile age 25
HSET profile career "Programmer"
```

如果此时使用ziplist编码，那么该Hash对象在内存中的结构如下：

![Untitled](RPG从零开始/Database/Redis/数据结构/hash/Untitled.png)

## hash-hashtable

hashtable 编码的哈希对象使用字典dictht作为底层实现。字典是一种保存键值对的数据结构。

```c
typedef struct dictht{
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size-1
    unsigned long sizemask;
    // 该哈希表已有节点数量
    unsigned long used;
} dictht
```

table属性是一个数组，数组中的每个元素都是一个指向dictEntry结构的指针，每个dictEntry结构保存着一个键值对。size属性记录了哈希表的大小，即table数组的大小。used属性记录了哈希表目前已有节点数量。sizemask总是等于size-1，这个值主要用于数组索引。比如下图展示了一个大小为4的空哈希表。

![Untitled](RPG从零开始/Database/Redis/数据结构/hash/Untitled%201.png)

### 哈希表节点

哈希表节点使用dictEntry结构表示，每个dictEntry结构都保存着一个键值对：

```c
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        unit64_t u64;
        nit64_t s64;
    } v;
    // 指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

key属性保存着键值对中的键，而v属性则保存了键值对中的值。值可以是一个指针，一个uint64_t整数或者是int64_t整数。next属性指向了另一个dictEntry节点，在数组桶位相同的情况下，将多个dictEntry节点串联成一个链表，以此来解决键冲突问题。(链地址法)

### 字典

Redis字典由dict结构表示：

```c
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    //rehash索引
    // 当rehash不在进行时，值为-1
    int rehashidx;
}
```

ht是大小为2，且每个元素都指向dictht哈希表。一般情况下，字典只会使用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用。rehashidx记录了rehash的进度，如果目前没有进行rehash，值为-1。

![Untitled](RPG从零开始/Database/Redis/数据结构/hash/Untitled%202.png)

### rehash

为了使hash表的负载因子(ht[0]).used/ht[0]).size)维持在一个合理范围，当哈希表保存的元素过多或者过少时，程序需要对hash表进行相应的扩展和收缩。rehash（重新散列）操作就是用来完成hash表的扩展和收缩的。rehash的步骤如下：

1. 为ht[1]哈希表分配空间，如果是扩展操作，那么ht[1]的大小为第一个大于ht[0].used*2的2^n。比如ht[0].used=5，那么此时ht[1]的大小就为16。(大于10的第一个2^n的值是16)如果是收缩操作，那么ht[1]的大小为第一个大于ht[0].used的2^n。比如ht[0].used=5，那么此时ht[1]的大小就为8。(大于5的第一个2^n的值是8)
2. 将保存在ht[0]中的所有键值对rehash到ht[1]中。
3. 迁移完成之后，释放掉ht[0]，并将现在的ht[1]设置为ht[0]，在ht[1]新创建一个空白哈希表，为下一次rehash做准备。

### 渐进式rehash

前面讲过，扩展或者收缩需要将ht[0]里面的元素全部rehash到ht[1]中，如果ht[0]元素很多，显然一次性rehash成本会很大，从影响到Redis性能。为了解决上述问题，Redis使用了渐进式rehash技术，具体来说就是分多次，渐进式地将ht[0]里面的元素慢慢地rehash到ht[1]中。下面是渐进式rehash的详细步骤：

1. 为ht[1]分配空间。
2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash正式开始。
3. 在rehash进行期间，每次对字典执行添加、删除、查找或者更新时，除了会执行相应的操作之外，还会顺带将ht[0]在rehashidx索引位上的所有键值对rehash到ht[1]中，rehash完成之后，rehashidx值加1。
4. 随着字典操作的不断进行，最终会在某个时刻迁移完成，此时将rehashidx值置为-1，表示rehash结束。

因为在渐进式rehash时，字典会同时使用ht[0]和ht[1]两张表，所以此时对字典的删除、查找和更新操作都可能会在两个哈希表进行。比如，如果要查找某个键时，先在ht[0]中查找，如果没找到，则继续到ht[1]中查找。

参考

[https://zhuanlan.zhihu.com/p/531323771](https://zhuanlan.zhihu.com/p/531323771)