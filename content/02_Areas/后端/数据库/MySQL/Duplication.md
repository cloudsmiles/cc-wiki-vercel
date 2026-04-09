# Duplication

## 主从同步

嘻嘻，先复习一下主从复制原理吧，如图：

[https://mmbiz.qpic.cn/mmbiz_png/sMmr4XOCBzHb5ZvMdLOvjicVicD6zLAPaBwh3oiaibRLvVAXicia6QFy8ACcZ22ia8QUzPQmpFMQ9RTgWy7Q2nZiaDaS7g/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/sMmr4XOCBzHb5ZvMdLOvjicVicD6zLAPaBwh3oiaibRLvVAXicia6QFy8ACcZ22ia8QUzPQmpFMQ9RTgWy7Q2nZiaDaS7g/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

主从复制分了五个步骤进行：

- 步骤一：主库的更新事件(update、insert、delete)被写到binlog
- 步骤二：从库发起连接，连接到主库。
- 步骤三：此时主库创建一个binlog dump thread，把binlog的内容发送到从库。
- 步骤四：从库启动之后，创建一个I/O线程，读取主库传过来的binlog内容并写入到relay log
- 步骤五：还会创建一个SQL线程，从relay log里面读取内容，从ExecMasterLog_Pos位置开始执行读取到的更新事件，将更新内容写入到slave的db

## 主从同步延迟的原因

**1、主节点binLog数据未及时同步**

**2、从节点I/O线程未及时将数据写入RelayLog**

**3、从节点SQL线程未及时将RelayLog写入数据库**

针对这三个方面，我们逐一分析可能导致主从延迟的原因。

### **主节点binLog数据未及时同步**

针对[主库binLog](https://www.zhihu.com/search?q=%E4%B8%BB%E5%BA%93binLog&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2688404743%7D)数据没及时同步造成主从延迟的情况，可能性主要有如下几个：

1、**主库存在高并发**，某一时刻，大量的写请求到主库上，意味着binLog需要进行频繁的写入及同步，此时就可能导致主节点binlog dump线程、从节点I/O线程没法及时同步，从而造成主从延迟。

2、**网络IO存在问题**，某一时刻如果网络IO挂掉了，数据没法发送到从库，那么此时也会导致从库的数据无法更新，造成主从延迟。

3、**执行大事务**，一旦执行大事务，主库必须等到事务执行完成之后才能写入binLog，进而也会影响后续的binLog的同步及写入，造成主从延迟。

### **I/O线程未及时写RelayLog**

针对从节点I/O线程未及时写入RelayLog情况，其主要的可能性是

1、**存在高并发**，I/O线程未能及时将数据刷入到relayLog中。但I/O线程通常为磁盘读写，效率较快，一般不会成为主要的问题。

2、**机器性能较差**，导致relayLog同步速度较慢。

### **SQL线程未及时写数据库**

针对这最后一种情况，产生主从延迟的原因可能更多一些：

1、**relayLog随机重放**。在`主库中写binlog`及`从库中写relayLog`都是顺序进行的磁盘读写，效率较高。但是SQL数据对relayLog进行数据重放的时候是随机写盘的，执行效率相对较慢，从而出现主从延迟。

2、**锁等待**，从库除了同步以外还会需要支持正常的业务查询操作，而如果当前需要修改的数据被访问了，那么此时SQL线程就会先进行等待，直到锁被释放以后，再获取锁进行修改。而这一段时间，就有可能导致主从延迟。

3、**高并发**，高并发情况下产生的DML数量超过了SQL Thread所能处理的速度时，那么此时就会产生主从延迟。

4、**出现慢SQL**，慢SQL会导致relayLog较久才能写入到数据库中，进而造成主从延迟。

## 主从同步延迟的解决办法

### **强一致性方案**

最简单粗暴的方案，其实就是针对一些实时性要求比较高的操作，通过代码指定的方式，强行让[查询语句](https://www.zhihu.com/search?q=%E6%9F%A5%E8%AF%A2%E8%AF%AD%E5%8F%A5&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2688404743%7D)走主库进行查询。这样可以保证数据的绝对准确，带来的问题是会加大主库的并发数量，增加宕机的风险。

### **并行复制**

这种方案主要针对**高并发情况下从库SQL单线程出现瓶颈**的时候使用，将SQL线程转化成多个线程来进行重放，加快对DML数据的处理速度，从而减缓主从延迟。mySql自5.7版本后就已经支持并行复制了。可以在从服务上设置 `slave_parallel_workers`为一个大于0的数，然后把`slave_parallel_type`参数设置为`LOGICAL_CLOCK`即可。

### **降低并发**

针对过高的并发，其实会导致同步过程的三个阶段都会出现问题，为此，在接口设计时，要充分考虑接口的请求量，并适当采用如`令牌桶、漏桶`算法等进行限流。或采用`分布式锁`等对关键数据进行加锁，控制并发量，减少主从延迟对业务的影响。

### **增加NOSQL层**

通过在从库前增加NOSQL层来缓解，主库写入SQL的时候，按照一定策略将关键数据放入到缓存中，查询时优先查询缓存中的数据。可以在一定程度上减少主从延迟所带来的问题。

参考

[https://www.cnblogs.com/xuxubaobao/p/10839979.html](https://www.cnblogs.com/xuxubaobao/p/10839979.html)

[https://www.zhihu.com/question/20025096/answer/2688404743?utm_id=0](https://www.zhihu.com/question/20025096/answer/2688404743?utm_id=0)

数据分片 问题的解决