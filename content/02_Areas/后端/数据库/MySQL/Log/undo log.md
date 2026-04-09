# undo log

undo log 和 redo log 也是引擎层的 log 文件，undo log 提供了回滚和多个行版本控制（MVCC），在数据库修改操作时，不仅记录了 redo log，还记录了 undo log，如果因为某些原因导致事务执行失败回滚了，可以借助 undo log 进行回滚。

## undo log的作用

虽然 undo log 和 redo log 都是InnoDB 特有的，但 undo log 记录的是 逻辑日志，redo log 记录的是物理日志。对记录做变更操作时不仅会产生 redo 记录，也会产生 undo 记录（insert,update,delete），undo log 日志用于存放数据被修改前的值，比如 `update T set c=c+1 where ID=2;` 这条 SQL，undo log 中记录的是 c 在 +1 前的值，如果这个 update 出现异常需要回滚，可以使用 undo log 实现回滚，保证事务一致性。

而多版本并发控制（MVCC） ，也用到了 undo log ，当读取的某一行被其他事务锁定时，它可以从 undo log 中获取该行记录以前的数据是什么，从而提供该行版本信息，让用户实现非锁定一致性读取。

## **undo log的存储机制**

undo log的存储由InnoDB存储引擎实现，数据保存在InnoDB的数据文件中。在InnoDB存储引擎中，undo log是采用分段(segment)的方式进行存储的。rollback segment称为回滚段，每个回滚段中有1024个undo log segment。在MySQL5.5之前，只支持1个rollback segment，也就是只能记录1024个undo操作。在MySQL5.5之后，可以支持128个rollback segment，分别从resg slot0 - resg slot127，每一个resg slot，也就是每一个回滚段，内部由1024个undo segment 组成，即总共可以记录128 * 1024个undo操作。

下面以一张图来说明undo log日志里面到底存了哪些信息？

![https://img-blog.csdnimg.cn/img_convert/5bb83db5911858e862facb7f8869f8c4.png](https://img-blog.csdnimg.cn/img_convert/5bb83db5911858e862facb7f8869f8c4.png)

如上图，可以看到，undo log日志里面不仅存放着数据更新前的记录，还记录着RowID、事务ID、回滚指针。其中事务ID每次递增，回滚指针第一次如果是insert语句的话，回滚指针为NULL，第二次update之后的undo log的回滚指针就会指向刚刚那一条undo log日志，依次类推，就会形成一条undo log的回滚链，方便找到该条记录的历史版本。

## **undo log的工作原理**

在更新数据之前，MySQL会提前生成undo log日志，当事务提交的时候，并不会立即删除undo log，因为后面可能需要进行回滚操作，要执行回滚（rollback）操作时，从缓存中读取数据。undo log日志的删除是通过通过后台purge线程进行回收处理的。

同样，通过一张图来理解undo log的工作原理。

![https://img-blog.csdnimg.cn/img_convert/e0adbdf1c448a617ea22b9f2aaaf682c.png](https://img-blog.csdnimg.cn/img_convert/e0adbdf1c448a617ea22b9f2aaaf682c.png)

如上图：

1、事务A执行update操作，此时事务还没提交，会将数据进行备份到对应的undo buffer，然后由undo buffer持久化到磁盘中的undo log文件中，此时undo log保存了未提交之前的操作日志，接着将操作的数据，也就是Teacher表的数据持久保存到InnoDB的数据文件IBD。

2、此时事务B进行查询操作，直接从undo buffer缓存中进行读取，这时事务A还没提交事务，如果要回滚（rollback）事务，是不读磁盘的，先直接从undo buffer缓存读取。

用undo log实现原子性和持久化的事务的简化过程：

假设有A、B两个数据，值分别为1,2。

- A. 事务开始
- B. 记录A=1到undo log中
- C. 修改A=3
- D. 记录B=2到undo log中
- E. 修改B=4
- F. 将undo log写到磁盘 -------undo log持久化
- G. 将数据写到磁盘 -------数据持久化
- H. 事务提交 -------提交事务

之所以能同时保证原子性和持久化，是因为以下特点：

1. 更新数据前记录undo log。
2. 为了保证持久性，必须将数据在事务提交前写到磁盘，只要事务成功提交，数据必然已经持久化到磁盘。
3. undo log必须先于数据持久化到磁盘。如果在G,H之间发生系统崩溃，undo log是完整的，可以用来回滚。
4. 如果在A - F之间发生系统崩溃，因为数据没有持久化到磁盘，所以磁盘上的数据还是保持在事务开始前的状态。

缺陷：每个事务提交前将数据和undo log写入磁盘，这样会导致大量的磁盘IO，因此性能较差。 如果能够将数据缓存一段时间，就能减少IO提高性能，但是这样就会失去事务的持久性。

参考

[https://blog.csdn.net/glenshappy/article/details/127708967](https://blog.csdn.net/glenshappy/article/details/127708967)