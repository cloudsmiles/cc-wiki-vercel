# binlog与redo log

## 前言

`redo log`（重做日志）让`InnoDB`存储引擎拥有了崩溃恢复能力。

`binlog`（归档日志）保证了`MySQL`集群架构的数据一致性。

虽然它们都属于持久化的保证，但是则重点不同。

在执行更新语句过程，会记录`redo log`与`binlog`两块日志，以基本的事务为单位，`redo log`在事务执行过程中可以不断写入，而`binlog`只有在提交事务时才写入，所以`redo log`与`binlog`的写入时机不一样。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKCF1P0plT6VqFDupQPxG5fKiaSKdE5AksqO64Qnfkb4wox51rVC2HNXA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKCF1P0plT6VqFDupQPxG5fKiaSKdE5AksqO64Qnfkb4wox51rVC2HNXA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 日志不一致的问题

回到正题，`redo log`与`binlog`两份日志之间的逻辑不一致，会出现什么问题？

我们以`update`语句为例，假设`id=2`的记录，字段`c`值是`0`，把字段`c`值更新成`1`，`SQL`语句为`update T set c=1 where id=2`。

假设执行过程中写完`redo log`日志后，`binlog`日志写期间发生了异常，会出现什么情况呢？

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKpP6Ztb44qRNgbPiaCibQPLtZ5vFIxvJXoha1jAbIybsT89x0ayjdbGeQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKpP6Ztb44qRNgbPiaCibQPLtZ5vFIxvJXoha1jAbIybsT89x0ayjdbGeQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

由于`binlog`没写完就异常，这时候`binlog`里面没有对应的修改记录。因此，之后用`binlog`日志恢复数据时，就会少这一次更新，恢复出来的这一行`c`值是`0`，而原库因为`redo log`日志恢复，这一行`c`值是`1`，最终数据不一致。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKYlAQ1IwJXfpbAl2YPSkfFCr8RoicNjwicXh0s8dfiarhg944OdTAY68dg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKYlAQ1IwJXfpbAl2YPSkfFCr8RoicNjwicXh0s8dfiarhg944OdTAY68dg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 两阶段提交

**为了解决两份日志之间的逻辑一致问题，`InnoDB`存储引擎使用两阶段提交方案。**

原理很简单，将`redo log`的写入拆成了两个步骤`prepare`和`commit`，这就是**两阶段提交**。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKObGZHzrNY2hhGsRzHDMwXmuL79fA1bKBz5GfSQL8VvCSEz5sT2Un5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKObGZHzrNY2hhGsRzHDMwXmuL79fA1bKBz5GfSQL8VvCSEz5sT2Un5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

使用**两阶段提交**后，写入`binlog`时发生异常也不会有影响，因为`MySQL`根据`redo log`日志恢复数据时，发现`redo log`还处于`prepare`阶段，并且没有对应`binlog`日志，就会回滚该事务。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TK2ySUBibRXA1Us8Mm6IV5QPEWhytPq2qalyCHOVhwu2eTYAshP4icQ3PA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TK2ySUBibRXA1Us8Mm6IV5QPEWhytPq2qalyCHOVhwu2eTYAshP4icQ3PA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

再看一个场景，`redo log`设置`commit`阶段发生异常，那会不会回滚事务呢？

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKTgM6SIW09SygzLicSicTskPVIAUwV9mmH181fSdV1ofSPXxtK2DM9cbw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nyq7TPySfnaZkZlwBscQ1TKTgM6SIW09SygzLicSicTskPVIAUwV9mmH181fSdV1ofSPXxtK2DM9cbw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

并不会回滚事务，它会执行上图框住的逻辑，虽然`redo log`是处于`prepare`阶段，但是能通过事务`id`找到对应的`binlog`日志，所以`MySQL`认为是完整的，就会提交事务恢复数据。

## redo log与binlog区别

1. redo log 是InnoDB 引擎特有的；而 binlog 是MySQL Server 层实现的
2. redo log 是物理日志，记录的是“在某个数  据页做了什么修改”；而 binlog 是逻辑日志，记录的是语句的原始逻辑。比如 `update T set c=c+1 where ID=2;`这条SQL，redo log 中记录的是 ：`xx页号，xx偏移量的数据修改为xxx；`binlog 中记录的是：`id = 2 这一行的 c 字段 +1`
3. redo log 是循环写的，固定空间会用完；binlog 可以追加写入，一个文件写满了会切换到下一个文件写，并不会覆盖之前的记录
4. 记录内容时间不同，redo log 记录事务发起后的 DML 和 DDL语句；binlog 记录commit 完成后的 DML 语句和 DDL 语句
5. 作用不同，redo log 作为异常宕机或者介质故障后的数据恢复使用；binlog 作为恢复数据使用，主从复制搭建。

参考

[https://juejin.cn/post/6987557227074846733](https://juejin.cn/post/6987557227074846733)

[https://mp.weixin.qq.com/s?__biz=MzAwMDg2OTAxNg==&mid=2652054869&idx=1&sn=b7ca964517c40a7ef990760ff659ac65&chksm=8105d2a2b6725bb44b1e6a29dd92755280cba541c2b5d7f097e8e9f704dd6650617aef46c047&scene=178&cur_album_id=1952926902587834371#rd](https://mp.weixin.qq.com/s?__biz=MzAwMDg2OTAxNg==&mid=2652054869&idx=1&sn=b7ca964517c40a7ef990760ff659ac65&chksm=8105d2a2b6725bb44b1e6a29dd92755280cba541c2b5d7f097e8e9f704dd6650617aef46c047&scene=178&cur_album_id=1952926902587834371#rd)