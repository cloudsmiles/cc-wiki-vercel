# order by

## **一个使用order by 的简单例子**

假设用一张员工表，表结构如下：

```
CREATE TABLE `staff` (
`id` BIGINT ( 11 ) AUTO_INCREMENT COMMENT '主键id',
`id_card` VARCHAR ( 20 ) NOT NULL COMMENT '身份证号码',
`name` VARCHAR ( 64 ) NOT NULL COMMENT '姓名',
`age` INT ( 4 ) NOT NULL COMMENT '年龄',
`city` VARCHAR ( 64 ) NOT NULL COMMENT '城市',
PRIMARY KEY ( `id`),
INDEX idx_city ( `city` )
) ENGINE = INNODB COMMENT '员工表';
```

我们现在有这么一个需求：**查询前10个，来自深圳员工的姓名、年龄、城市，并且按照年龄小到大排序**。对应的 SQL 语句就可以这么写：

```
select name,age,city from staff where city = '深圳' order by age limit 10;

```

这条语句的逻辑很清楚，但是它的**底层执行流程**是怎样的呢？

## **order by 工作原理**

### **explain 执行计划**

我们先用**Explain**关键字查看一下执行计划

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65mxYnh0cWkEVG4n2zt20icHtt6pA5eXz0kQt0yL3Xmre2MEKZadZVAyg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65mxYnh0cWkEVG4n2zt20icHtt6pA5eXz0kQt0yL3Xmre2MEKZadZVAyg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- 执行计划的**key**这个字段，表示使用到索引idx_city
- Extra 这个字段的 **Using index condition** 表示索引条件
- Extra 这个字段的 **Using filesort**表示用到排序

我们可以发现，这条SQL使用到了索引，并且也用到排序。那么它是**怎么排序**的呢？

### **全字段排序**

MySQL 会给每个查询线程分配一块小**内存**，用于**排序**的，称为 **sort_buffer**。什么时候把字段放进去排序呢，其实是通过`idx_city`索引找到对应的数据，才把数据放进去啦。

加上**order by**之后，整体的执行流程就是：

1. MySQL 为对应的线程初始化**sort_buffer**，放入需要查询的name、age、city字段；
2. 从**索引树idx_city**， 找到第一个满足 city='深圳’条件的主键 id，也就是图中的id=9；
3. 到**主键 id 索引树**拿到id=9的这一行数据， 取name、age、city三个字段的值，存到sort_buffer；
4. 从**索引树idx_city** 拿到下一个记录的主键 id，即图中的id=13；
5. 重复步骤 3、4 直到**city的值不等于深圳**为止；
6. 前面5步已经查找到了所有**city为深圳**的数据，在 sort_buffer中，将所有数据根据age进行排序；
7. 按照排序结果取前10行返回给客户端。

执行示意图如下：

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65Gx3PbkcEKH5GOKWnpicr6sVJ1HSEiaJo6xqeUV1X5WRxdFRbfz1DEOow/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65Gx3PbkcEKH5GOKWnpicr6sVJ1HSEiaJo6xqeUV1X5WRxdFRbfz1DEOow/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

将查询所需的字段全部读取到sort_buffer中，就是**全字段排序**
。这里面，有些小伙伴可能会有个疑问,把查询的所有字段都放到sort_buffer，而sort_buffer是一块内存来的，如果数据量太大，sort_buffer放不下怎么办呢？

### **磁盘临时文件辅助排序**

实际上，sort_buffer的大小是由一个参数控制的：**sort_buffer_size**。如果要排序的数据小于sort_buffer_size，排序在**sort_buffer** 内存中完成，如果要排序的数据大于sort_buffer_size，则**借助磁盘文件来进行排序**

如何确定是否使用了磁盘文件来进行排序呢？可以使用以下这几个命令

```
## 打开optimizer_trace，开启统计
set optimizer_trace = "enabled=on";
## 执行SQL语句
select name,age,city from staff where city = '深圳' order by age limit 10;
## 查询输出的统计信息
select * from information_schema.optimizer_trace
```

可以从 **number_of_tmp_files** 中看出，是否使用了临时文件。

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65RZ9YcPAmsEnj3YxNBuVbnp2x0ncg9qzBYlpYWsuUd2eAUEkWDJlaxg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65RZ9YcPAmsEnj3YxNBuVbnp2x0ncg9qzBYlpYWsuUd2eAUEkWDJlaxg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**number_of_tmp_files** 表示使用来排序的磁盘临时文件数。如果number_of_tmp_files>0，则表示使用了磁盘文件来进行排序。

使用了磁盘临时文件，整个排序过程又是怎样的呢？

1. 从**主键Id索引树**，拿到需要的数据，并放到**sort_buffer内存**块中。当sort_buffer快要满时，就对sort_buffer中的数据排序，排完后，把数据临时放到磁盘一个小文件中。
2. 继续回到主键 id 索引树取数据，继续放到sort_buffer内存中，排序后，也把这些数据写入到磁盘临时小文件中。
3. 继续循环，直到取出所有满足条件的数据。最后把磁盘的临时排好序的小文件，合并成一个有序的大文件。

**TPS:** 借助磁盘临时小文件排序，实际上使用的是**归并排序**算法。

小伙伴们可能会有个疑问，既然**sort_buffer**放不下，就需要用到临时磁盘文件，这会影响排序效率。那为什么还要把排序不相关的字段（name，city）放到sort_buffer中呢？只放排序相关的age字段，它**不香**吗？可以了解下**rowid 排序**。

### **rowid 排序**

rowid 排序就是，只把查询SQL**需要用于排序的字段和主键id**，放到sort_buffer中。那怎么确定走的是全字段排序还是rowid 排序排序呢？

实际上有个参数控制的。这个参数就是**max_length_for_sort_data**，它表示MySQL用于排序行数据的长度的一个参数，如果单行的长度超过这个值，MySQL 就认为单行太大，就换rowid 排序。我们可以通过命令看下这个参数取值。

```
show variables like 'max_length_for_sort_data';
```

**max_length_for_sort_data** 默认值是1024。因为本文示例中name,age,city长度=64+4+64 =132 < 1024, 所以走的是全字段排序。我们来改下这个参数，改小一点，

```
## 修改排序数据最大单行长度为32
set max_length_for_sort_data = 32;
## 执行查询SQL
select name,age,city from staff where city = '深圳' order by age limit 10;
```

使用rowid 排序的话，整个SQL执行流程又是怎样的呢？

1. MySQL 为对应的线程初始化**sort_buffer**，放入需要排序的age字段，以及主键id；
2. 从**索引树idx_city**， 找到第一个满足 city='深圳’条件的主键 id，也就是图中的id=9；
3. 到**主键 id 索引树**拿到id=9的这一行数据， 取age和主键id的值，存到sort_buffer；
4. 从**索引树idx_city** 拿到下一个记录的主键 id，即图中的id=13；
5. 重复步骤 3、4 直到**city的值不等于深圳**为止；
6. 前面5步已经查找到了所有city为深圳的数据，在 **sort_buffer**中，将所有数据根据age进行排序；
7. 遍历排序结果，取前10行，并按照 id 的值**回到原表**中，取出city、name 和 age 三个字段返回给客户端。

执行示意图如下：

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65KylUcdB0IUG6OGEAK95wnGgkNQO8zr5zibdiawXD08l6REB8qVH3vYCQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65KylUcdB0IUG6OGEAK95wnGgkNQO8zr5zibdiawXD08l6REB8qVH3vYCQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

对比一下**全字段排序**的流程，rowid 排序多了一次**回表**。

我们通过**optimizer_trace**，可以看到是否使用了rowid排序的：

```
## 打开optimizer_trace，开启统计
set optimizer_trace = "enabled=on";
## 执行SQL语句
select name,age,city from staff where city = '深圳' order by age limit 10;
## 查询输出的统计信息
select * from information_schema.optimizer_trace
```

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65fxGiatHhp7EwQc7SxjSEDiczBWibZj9LIU0McPSJkweje3kIniapXHwccg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65fxGiatHhp7EwQc7SxjSEDiczBWibZj9LIU0McPSJkweje3kIniapXHwccg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### **全字段排序与rowid排序对比**

- 全字段排序：sort_buffer内存不够的话，就需要用到磁盘临时文件，造成**磁盘访问**。
- rowid排序：sort_buffer可以放更多数据，但是需要再回到原表去取数据，比全字段排序多一次**回表**。

一般情况下，对于InnoDB存储引擎，会优先使**用全字段**排序。可以发现 **max_length_for_sort_data** 参数设置为1024，这个数比较大的。一般情况下，排序字段不会超过这个值，也就是都会走**全字段**排序。

## **order by的一些优化思路**

我们如何优化order by语句呢？

- 因为数据是无序的，所以就需要排序。如果数据本身是有序的，那就不用排了。而索引数据本身是有序的，我们通过建立**联合索引**，优化order by 语句。
- 我们还可以通过调整**max_length_for_sort_data**等参数优化；

参考

[https://mp.weixin.qq.com/s/h9jWeoyiBGnQLvDrtXqVWw](https://mp.weixin.qq.com/s/h9jWeoyiBGnQLvDrtXqVWw)