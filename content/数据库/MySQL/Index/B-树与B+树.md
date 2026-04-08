# B-树与B+树

### **B-树 简介**

B-树，也称为B树，是一种平衡的多叉树（可以对比一下平衡二叉查找树），它比较适用于对外查找。看下这几个概念哈：

> ❝
> 
> - 阶数：一个节点最多有多少个孩子节点。（一般用字母m表示）
> - 关键字：节点上的数值就是关键字
> - 度：一个节点拥有的子节点的数量。

一颗m阶的B-树，有以下特征：

> ❝
> 
> - 根结点至少有两个子女；
> - 每个非根节点所包含的关键字个数 j 满足：⌈m/2⌉ - 1 <= j <= m - 1.(⌈⌉表示向上取整)
> - 有k个关键字(关键字按递增次序排列)的非叶结点恰好有k+1个孩子。
> - 所有的叶子结点都位于同一层。

一棵简单的B-树如下：

[https://mmbiz.qpic.cn/mmbiz_png/sMmr4XOCBzEK1LibhR8zQXiaxCiaaVYkDLZEiaQiaI8Bb8NFb7Wp0AlM8SqVAjJJOWN5Lreg0csHTibpJTYbovysia2sA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/sMmr4XOCBzEK1LibhR8zQXiaxCiaaVYkDLZEiaQiaI8Bb8NFb7Wp0AlM8SqVAjJJOWN5Lreg0csHTibpJTYbovysia2sA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### **B+ 树简介**

B+树是B-树的变体，也是一颗多路搜索树。一棵m阶的B+树主要有这些特点：

> ❝
> 
> - 每个结点至多有m个子女;
> - 非根节点关键值个数范围：[[m 2]] <= k <= m-1
> - 相邻叶子节点是通过指针连起来的，并且是关键字大小排序的。

一颗3阶的B+树如下：

[https://mmbiz.qpic.cn/mmbiz_png/sMmr4XOCBzEK1LibhR8zQXiaxCiaaVYkDLZWuloYEthOeMFqC3BHVHbiaF4iaJG3ptBCY5R6VDjRvlrXhvb55uKU0qg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/sMmr4XOCBzEK1LibhR8zQXiaxCiaaVYkDLZWuloYEthOeMFqC3BHVHbiaF4iaJG3ptBCY5R6VDjRvlrXhvb55uKU0qg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

B+树和B-树的主要区别如下：

- B-树内部节点是保存数据的;而B+树内部节点是不保存数据的，只作索引作用，它的叶子节点才保存数据。
- B+树相邻的叶子节点之间是通过链表指针连起来的，B-树却不是。
- 查找过程中，B-树在找到具体的数值以后就结束，而B+树则需要通过索引找到叶子结点中的数据才结束
- B-树中任何一个关键字出现且只出现在一个结点中，而B+树可以出现多次。

参考

[https://mp.weixin.qq.com/s/3ySO_VDOy6Z4orOON9eWzA](https://mp.weixin.qq.com/s/3ySO_VDOy6Z4orOON9eWzA)