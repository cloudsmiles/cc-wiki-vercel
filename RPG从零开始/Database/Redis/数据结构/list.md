# list

列表对象的编码可以是linkedlist或者ziplist，对应的底层数据结构是链表和压缩列表。

默认情况下，当列表对象保存的所有字符串元素的长度都小于64字节，且元素个数小于512个时，列表对象采用的是ziplist编码，否则使用linkedlist编码。可以通过配置文件修改该上限值。

## 链表

提供了高效的节点重排能力以及顺序性的节点访问方式。在Redis中，每个链表节点使用listNode结构表示：

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode
```

多个listNode通过prev和next指针组成双端链表，如下图所示：

![Untitled](RPG从零开始/Database/Redis/数据结构/list/Untitled.png)

为了操作起来比较方便，Redis使用了list结构持有链表。list结构为链表提供了表头指针head、表尾指针tail，以及链表长度计数器len，而dup、free和match成员则是实现多态链表所需类型的特定函数。

```c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表包含的节点数量
    unsigned long len;
    // 节点复制函数
    void *(*dup)(void *ptr);
    // 节点释放函数
    void (*free)(void *ptr);
    // 节点对比函数
    int (*match)(void *ptr, void *key);
} list;
```

![Untitled](RPG从零开始/Database/Redis/数据结构/list/Untitled%201.png)

## Redis链表实现的特点：

- 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O(n)。
- 无环：表头节点的prev指针和表尾    节点的next指针都指向NULL，对链表的访问以NULL为终点。
- 带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O(1)。
- 带链表长度计数器：程序使用list结构的len属性来对list持有的节点进行计数，程序获取链表中节点数量的复杂度为O(1)。
- 多态：链表节点使用void*指针来保存节点值，可以保存各种不同类型的值。

## 压缩链表

压缩列表。redis的列表键和哈希键的底层实现之一。此数据结构是为了节约内存而开发的。和各种语言的数组类似，它是由连续的内存块组成的，这样一来，由于内存是连续的，就减少了很多内存碎片和指针的内存占用，进而节约了内存。

![Untitled](RPG从零开始/Database/Redis/数据结构/list/Untitled%202.png)

entry的结构是这样的：

![Untitled](RPG从零开始/Database/Redis/数据结构/list/Untitled%203.png)

参考

[https://zhuanlan.zhihu.com/p/531323771](https://zhuanlan.zhihu.com/p/531323771)