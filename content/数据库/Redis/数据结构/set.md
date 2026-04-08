# set

集合对象的编码可以是intset或者hashtable。

当集合对象保存的元素都是整数，并且个数不超过512个时，使用intset编码，否则使用hashtable编码。

## set-intset

整数集合(intset)是Redis用于保存整数值的集合抽象数据结构，它可以保存类型为int16_t、int32_t或者int64_t的整数值，并且保证集合中的数据不会重复。Redis使用intset结构表示一个整数集合。

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```

contents数组是整数集合的底层实现：整数集合的每个元素都是contents数组的一个数组项，各个项在数组中按值大小从小到大有序排列，并且数组中不包含重复项。虽然contents属性声明为int8_t类型的数组，但实际上，contents数组不保存任何int8_t类型的值，数组中真正保存的值类型取决于encoding。如果encoding属性值为INTSET_ENC_INT16，那么contents数组就是int16_t类型的数组，以此类推。

当新插入元素的类型比整数集合现有类型元素的类型大时，整数集合必须先升级，然后才能将新元素添加进来。这个过程分以下三步进行。

1. 根据新元素类型，扩展整数集合底层数组空间大小。
2. 将底层数组现有所有元素都转换为与新元素相同的类型，并且维持底层数组的有序性。
3. 将新元素添加到底层数组里面。

举个栗子，当我们执行SADD numbers 1 3 5向集合对象插入数据时，该集合对象在内存的结构如下：

![Untitled](RPG从零开始/Database/Redis/数据结构/set/Untitled.png)

## set-hashtable

hashtable编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象对应一个集合元素，字典的值都是NULL。当我们执行SADD fruits "apple" "banana" "cherry"向集合对象插入数据时，该集合对象在内存的结构如下：

![Untitled](RPG从零开始/Database/Redis/数据结构/set/Untitled%201.png)

参考

[https://zhuanlan.zhihu.com/p/531323771](https://zhuanlan.zhihu.com/p/531323771)