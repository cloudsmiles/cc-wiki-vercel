# redo log

**前言**

说到`MySQL`，有两块日志一定绕不开，一个是`InnoDB`存储引擎的`redo log`（重做日志），另一个是`MySQL Servce`层的 `binlog`（归档日志）。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54CRTlmxBCoPUkSTVPkwmYrUZFMp2cEMibD02jLjLibRFl5BQZXbaHbhUg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54CRTlmxBCoPUkSTVPkwmYrUZFMp2cEMibD02jLjLibRFl5BQZXbaHbhUg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

只要是数据更新操作，就一定会涉及它们，今天就来聊聊`redo log`（重做日志）。

# **redo log**

`redo log`（重做日志）是`InnoDB`存储引擎独有的，它让`MySQL`拥有了崩溃恢复能力。

比如`MySQL`实例挂了或宕机了，重启时，`InnoDB`存储引擎会使用`redo log`恢复数据，保证数据的持久性与完整性。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54bBA4hk3gw55HvxibrWwaj8Ms6mhmAL5RWEfk5YKiaEz4H45DUaWCYepw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54bBA4hk3gw55HvxibrWwaj8Ms6mhmAL5RWEfk5YKiaEz4H45DUaWCYepw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

然后会把“在某个数据页上做了什么修改”记录到重做日志缓存（`redo log buffer`）里，接着刷盘到`redo log`文件里。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54zUDHSoo2miaeyicIo2SGBY0FicnkbWeicrTlQH0LenmpScjibL35u61KVoQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54zUDHSoo2miaeyicIo2SGBY0FicnkbWeicrTlQH0LenmpScjibL35u61KVoQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

理想情况，事务一提交就会进行刷盘操作，但实际上，刷盘的时机是根据策略来进行的。

> 小贴士：每条redo记录由“表空间号+数据页号+偏移量+修改数据长度+具体修改的数据”组成
> 

## **刷盘时机**

`InnoDB`存储引擎为`redo log`的刷盘策略提供了`innodb_flush_log_at_trx_commit`参数，它支持三种策略

- **设置为0的时候，表示每次事务提交时不进行刷盘操作**
- **设置为1的时候，表示每次事务提交时都将进行刷盘操作（默认值）**
- **设置为2的时候，表示每次事务提交时都只把redo log buffer内容写入page cache**

另外`InnoDB`存储引擎有一个后台线程，每隔`1`秒，就会把`redo log buffer`中的内容写到文件系统缓存（`page cache`），然后调用`fsync`刷盘。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54Ad70tZojSrwI8YOGP7ibboticxTic0pmOk6FClqx08AA75BictzAdJDD7g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54Ad70tZojSrwI8YOGP7ibboticxTic0pmOk6FClqx08AA75BictzAdJDD7g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

也就是说，一个没有提交事务的`redo log`记录，也可能会刷盘。

为什么呢？

因为在事务执行过程`redo log`记录是会写入`redo log buffer`中，这些`redo log`记录会被后台线程刷盘。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54v2N1so73Jm9TKRrmQCyA3dxNmMgwJhCiaNYrKyXBxv5ydMQm9GRIhUg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54v2N1so73Jm9TKRrmQCyA3dxNmMgwJhCiaNYrKyXBxv5ydMQm9GRIhUg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

除了后台线程每秒`1`次的轮询操作，还有一种情况，当`redo log buffer`占用的空间即将达到`innodb_log_buffer_size`一半的时候，后台线程会主动刷盘。

下面是不同刷盘策略的流程图

### **innodb_flush_log_at_trx_commit=0**

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54oWOszyDsmLmIt7hhyicaia7PMUL5kMr1rUQ8AhA2QqaFJfucySByb5ag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54oWOszyDsmLmIt7hhyicaia7PMUL5kMr1rUQ8AhA2QqaFJfucySByb5ag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

为`0`时，如果`MySQL`挂了或宕机可能会有`1`秒数据的丢失。

### **innodb_flush_log_at_trx_commit=1**

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54APdmn5HovTEfk1qS4Z8jX9rGFQqqpAibibfRuR6K3VmxWk7CoUBe8QbQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54APdmn5HovTEfk1qS4Z8jX9rGFQqqpAibibfRuR6K3VmxWk7CoUBe8QbQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

为`1`时， 只要事务提交成功，`redo log`记录就 一定在硬盘里，不会有任何数据丢失。

如果事务执行期间`MySQL`挂了或宕机，这部分日志丢了，但是事务并没有提交，所以日志丢了也不会有损失。

### **innodb_flush_log_at_trx_commit=2**

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54IlvrwXTNgOcv8aCIicNXzhicdOKqicpibJOLLhOqmicBHWoTayWm7TfEYAQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nzoia78ia1wnynufibsPx05L54IlvrwXTNgOcv8aCIicNXzhicdOKqicpibJOLLhOqmicBHWoTayWm7TfEYAQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

为`2`时， 只要事务提交成功，`redo log buffer`中的内容只写入文件系统缓存（`page cache`）。

如果仅仅只是`MySQL`挂了不会有任何数据丢失，但是宕机可能会有`1`秒数据的丢失。

# **小结**

相信大家都知道`redo log`的作用和它的刷盘时机、存储形式。

现在我们来思考一问题，只要每次把修改后的数据页直接刷盘不就好了，还有`redo log`什么事。

它们不都是刷盘么？差别在哪里？

```
1 Byte = 8bit
1 KB = 1024 Byte
1 MB = 1024 KB
1 GB = 1024 MB
1 TB = 1024 GB

```

实际上，数据页大小是`16KB`，刷盘比较耗时，可能就修改了数据页里的几`Byte`数据，有必要把完整的数据页刷盘吗？

而且数据页刷盘是随机写，因为一个数据页对应的位置可能在硬盘文件的随机位置，所以性能是很差。

如果是写`redo log`，一行记录可能就占几十`Byte`，只包含表空间号、数据页号、磁盘文件偏移 量、更新值，再加上是顺序写，所以刷盘速度很快。

所以用`redo log`形式记录修改内容，性能会远远超过刷数据页的方式，这也让数据库的并发能力更强。

参考

[https://mp.weixin.qq.com/s?__biz=MzAwMDg2OTAxNg==&mid=2652054485&idx=1&sn=cd6bead326dc5f5d8cf6af16893e9676&scene=21#wechat_redirect](https://mp.weixin.qq.com/s?__biz=MzAwMDg2OTAxNg==&mid=2652054699&idx=1&sn=018017d9f3a61ca284970bbf65ea5138&chksm=8105d35cb6725a4a97664bd0b3f2b5c08862e2467a0b0dbf413fd706502bc2fe9367f30d9636&scene=178&cur_album_id=1952926902587834371#rd)

[https://juejin.cn/post/6987557227074846733](https://juejin.cn/post/6987557227074846733)

[https://blog.csdn.net/hbhe0316/article/details/122701149](https://blog.csdn.net/hbhe0316/article/details/122701149)