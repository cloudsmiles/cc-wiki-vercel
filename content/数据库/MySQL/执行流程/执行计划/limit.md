# limit

## **简单回顾MySQL中的 limit 的用法**

假如你的需求是：从第几条开始总共返回多少条。这时候你将会用到MySQL中的limit。

语法：SELECT * FROM table LIMIT [offset,] rows

## MySQL中用 limit 做分页

在系统中需要进行分页操作的时候，我们通常会使用LIMIT加上偏移量的办法实现，同时加上合适的ORDER BY 子句。如果有对应的索引，通常效率会不错，否则，MySQL需要做大量的文件排序操作。
一个非常常见又令人头疼的问题就是，在偏移量非常大的时候，例如可能是LIMIT 10000 ,20这样的查询，这时MySQL需要查询10 020条记录然后只返回最后20条，前面10000条记录都将被拋弃，这样的代价非常高（查询会比较慢，或者某些情况下很慢）。

在上述的执行过程中，造成 limit 大偏移量执行时间变久的原因有：

- 查询所有列导致回表
- `limit a, b`会查询前a+b条数据，然后丢弃前a条数据

综合上述两个原因，MySQL 花费了大量时间在回表上，而其中a次回表的结果又不会出现在结果集中，这才导致查询时间变得越来越长。

## 优化方案

### **覆盖索引**

既然无效的回表是导致查询变慢的主要原因，那么优化方案就主要从减少回表次数方面入手，假设在`limit a, b`中我们首先得到了a+1到a+b条数据的id，然后再进行回表获取其他列数据，那么就减少了a次回表操作，速度肯定会快上不少。

这里就涉及到覆盖索引了，所谓的覆盖索引就是从非主聚簇索引中就能查到的想要数据，而不需要通过回表从主键索引中查询其他列，能够显著提升性能。

基于这样的思路，优化方案就是先查询得到主键id，然后再根据主键id查询其他列数据，优化后的 SQL 以及执行时间如下表。

| 优化后的 SQL | 执行时间 |
| --- | --- |
| select * from user a join (select id from user where sex = 1 limit 100, 10) b on a.id=b.id; | OK, Time: 0.000000s |
| select * from user a join (select id from user where sex = 1 limit 1000, 10) b on a.id=b.id; | OK, Time: 0.00000s |
| select * from user a join (select id from user where sex = 1 limit 10000, 10) b on a.id=b.id; | OK, Time: 0.002000s |
| select * from user a join (select id from user where sex = 1 limit 100000, 10) b on a.id=b.id; | OK, Time: 0.015000s |
| select * from user a join (select id from user where sex = 1 limit 1000000, 10) b on a.id=b.id; | OK, Time: 0.151000s |
| select * from user a join (select id from user where sex = 1 limit 10000000, 10) b on a.id=b.id; | OK, Time: 1.161000s |

果然，执行效率得到了显著提升。

### 条件过滤

当然还有一种有缺陷的方法是基于排序做条件过滤。

比如像上面的示例 user 表，我要使用 limit 分页得到1000001到1000010条数据，可以这样写 SQL：

```
select * from user where sex = 1 and id > (select id from user where sex = 1 limit 1000000, 1) limit 10;
复制代码
```

但是使用这样的方式优化是有条件的：主键id必须是有序的。在有序的条件下，也可以使用比如创建时间等其他字段来代替主键id，但是前提是这个字段是建立了索引的。

总之，使用条件过滤的方式来优化 limit 是有诸多限制的，一般还是推荐使用覆盖索引的方式来优化。

参考

[https://juejin.cn/post/7094807113364406309](https://juejin.cn/post/7094807113364406309)

[https://blog.csdn.net/qq_32649581/article/details/123737215](https://blog.csdn.net/qq_32649581/article/details/123737215)