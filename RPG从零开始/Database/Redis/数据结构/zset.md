# zset

有序集合的编码可以是ziplist或者skiplist。

当有序集合保存的元素个数小于128个，且所有元素成员长度都小于64字节时，使用ziplist编码，否则，使用skiplist编码。

## zset-ziplist

ziplist编码的有序集合使用压缩列表作为底层实现，每个集合元素使用两个紧挨着一起的两个压缩列表节点表示，第一个节点保存元素的成员(member)，第二个节点保存元素的分值(score)。

压缩列表内的集合元素按照分值从小到大排列。如果我们执行ZADD price 8.5 apple 5.0 banana 6.0 cherry命令，向有序集合插入元素，该有序集合在内存中的结构如下：

![Untitled](RPG从零开始/Database/Redis/数据结构/zset/Untitled.png)

## zset-skiplist

skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表。

```
typedef struct zset {

    zskiplist *zs1;
    dict *dict;
}
```

### 跳跃表

跳跃表(skiplist)是一种有序的数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

Redis的跳跃表由zskiplistNode和zskiplist两个结构定义，zskiplistNode结构表示跳跃表节点，zskiplist保存跳跃表节点相关信息，比如节点的数量，以及指向表头和表尾节点的指针等。

### **跳跃表节点 zskiplistNode**

跳跃表节点zskiplistNode结构定义如下：

```c
typedef struct zskiplistNode {
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```

下图是一个层高为5，包含4个跳跃表节点(1个表头节点和3个数据节点)组成的跳跃表:

![Untitled](RPG从零开始/Database/Redis/数据结构/zset/Untitled%201.png)

1. 每次创建一个新的跳跃表节点的时候，会根据幂次定律(越大的数出现的概率越低)随机生成一个1-32之间的值作为当前节点的"层高"。每层元素都包含2个数据，前进指针和跨度。前进指针:每层都有一个指向表尾方向的前进指针，用于从表头向表尾方向访问节点。跨度:层的跨度用于记录两个节点之间的距离。
2. 后退指针(BW)节点的后退指针用于从表尾向表头方向访问节点，每个节点只有一个后退指针，所以每次只能后退一个节点。
3. 分值和成员节点的分值(score)是一个double类型的浮点数，跳跃表中所有节点都按分值从小到大排列。节点的成员(obj)是一个指针，指向一个字符串对象。在跳跃表中，各个节点保存的成员对象必须是唯一的，但是多个节点的分值确实可以相同。

需要注意的是，表头节点不存储真实数据，并且层高固定为32，从表头节点第一个不为NULL最高层开始，就能实现快速查找。

### 跳跃表 zskiplist

实际上，仅靠多个跳跃表节点就可以组成一个跳跃表，但是Redis使用了zskiplist结构来持有这些节点，这样就能够更方便地对整个跳跃表进行操作。比如快速访问表头和表尾节点，获得跳跃表节点数量等等。zskiplist结构定义如下：

```c
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct skiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 最大层数
    int level;
} zskiplist;
```

### **有序集合对象的skiplist实现**

前面讲过，skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表。

```c
typedef struct zset {
    zskiplist *zs1;
    dict *dict;
}
```

zset结构中的zs1跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素。通过跳跃表，可以对有序集合进行基于score的快速范围查找。zset结构中的dict字典为有序集合创建了从成员到分值的映射，字典的键保存了成员，字典的值保存了分值。通过字典，可以用O(1)复杂度查找给定成员的分值。

假如还是执行ZADD price 8.5 apple 5.0 banana 6.0 cherry命令向zset保存数据，如果采用skiplist编码方式的话，该有序集合在内存中的结构如下：

![Untitled](RPG从零开始/Database/Redis/数据结构/zset/Untitled%202.png)