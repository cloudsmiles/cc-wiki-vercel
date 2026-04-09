# change buffer

# **Change Buffer是什么**

`MySQL`在启动成功后，会向内存申请一块内存空间，这块内存空间称为`Buffer Pool`。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel49k57d4QEBsiaCYb5iapbklf578lkesibuRVUjewv0wu7FLzvedkOjqkXA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel49k57d4QEBsiaCYb5iapbklf578lkesibuRVUjewv0wu7FLzvedkOjqkXA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

`Buffer Pool`内维护了很多内容，比如**缓存页、各种链表、redo log buff、change buffer等等**。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4QCgutCSH5DkKJyTpAh3eYTUZA7qIgsKtdjnibnLGGagLZn2V50mFfGA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4QCgutCSH5DkKJyTpAh3eYTUZA7qIgsKtdjnibnLGGagLZn2V50mFfGA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

回到正题，`change buffer`是用来干嘛的？

当索引字段内容发生更新时（update、insert、delete），要更新对应的**索引页**，如果**索引页**在Buffer Pool里命中的话，就直接更新**缓存页**。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4lcKiaQyaWUzWGlw0s5RlPGusmcJre6icbqZuibYFnAIGNraV2Bgcicpb8g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4lcKiaQyaWUzWGlw0s5RlPGusmcJre6icbqZuibYFnAIGNraV2Bgcicpb8g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

否则，InnoDB会将这些更新操作缓存在change buffer中，这样就无需从硬盘读入**索引页**。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4aC1S9vvMnV3UibWj0kzpguK33JzCnPkK8CYOPzQQqS5gnVVA3p4Jib2Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4aC1S9vvMnV3UibWj0kzpguK33JzCnPkK8CYOPzQQqS5gnVVA3p4Jib2Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

下次查询索引页时，会将索引页读入`Buffer Pool`，然后将`change buffer`中的操作应用到对应的缓存页，得到最新结果，这个过程称为`merge`，通过这种方式就能保证数据逻辑的正确性。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4PaAHia9apuDYYxY2B1c4sMsXn749HSZ1navGNAtMpy9gTIVIcTYqibOw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4PaAHia9apuDYYxY2B1c4sMsXn749HSZ1navGNAtMpy9gTIVIcTYqibOw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## **持久化**

看到这里小伙伴有疑问了，`change buffer`在内存中，如果万一`MySql`实例挂了或宕机了，这次的更新操作不全丢了吗？

其实不用担心，`InnoDB`对这块有相应的持久化方案，会有后台线程定期把`change buffer`持久化到硬盘的系统表空间（`ibdata1`）。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4Qa9mhbHPvRnpjBczCuvnRctXdc2nflRV8QaIv9R8MYxFWgMe129QdA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4Qa9mhbHPvRnpjBczCuvnRctXdc2nflRV8QaIv9R8MYxFWgMe129QdA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

并且每次`change buffer`记录的内容，会写入到`redo log buff`中，由后台线程定期将`redo log buff`持久化到硬盘的`redolog`日志。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4Pb1HtFJF2MWc9yP4g2hE3R5TknQW620krsymWNSH9ylxzvicOD4Ykyg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4Pb1HtFJF2MWc9yP4g2hE3R5TknQW620krsymWNSH9ylxzvicOD4Ykyg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

最后`MySql`重启，可以通过`ibdata1`或`redolog`恢复`change buffer`，恢复的过程，分为下面几种情况

1. `change buffer`的数据刷盘到`ibdata`，直接根据`ibdata`恢复
2. `change buffer`的数据未刷盘，`redolog`里记录了`change buffer`的内容
    - `change buffer`写入`redo log`，`redo log`虽做了刷盘但未`commit`,`binlog`未刷盘,这部分数据丢失
    - `change buffer`写入`redolog`，`redolog`虽做了刷盘但未`commit`,`binlog`已刷盘,先从`binlog`恢复`redolog`,再从`redolog`恢复`change buffe`
    - `change buffer`写入`redolog`，`redolog`和`binlog`都已刷盘，直接从`redolog`里恢复。

# **如何使用Change Buffer**

看到这里，相信大家对`change buffer`有了基本的认识。

现在可以展开讲讲`change buffer`的使用限制。

是的，你没听错，`change buffer`不能随随便便用。

一般我们可以把常用索引分类为下面几种

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4unzofgIoiaVhkIicqnLxhwNTmvnZricVdlcTkicuxG33msS4Gjpjw5jcgA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4unzofgIoiaVhkIicqnLxhwNTmvnZricVdlcTkicuxG33msS4Gjpjw5jcgA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其中**聚簇索引**和**唯一索引**是无法使用`change buffer`，因为它们具备**唯一性**。

[https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4nz7gaJm51tJ75Af1UVtQZW36A6xCib5QTsTDrskYkPoUcpSiahBPXMag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8nxsZWew5X4phJgzuCTfTel4nz7gaJm51tJ75Af1UVtQZW36A6xCib5QTsTDrskYkPoUcpSiahBPXMag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

当更新**唯一索引**字段的内容时，需要把相应的索引页加载进`Buffer Pool`，验证唯一性约束，此时都已经读入到`Buffer Pool`了，那直接更新会更快，没必要使用`change buffer`。

参考

[https://mp.weixin.qq.com/s?__biz=MzAwMDg2OTAxNg==&mid=2652055800&idx=1&sn=b3d9b5c9ced407850129e61da164db34&chksm=8105df0fb6725619b5b483dd6330fc252cf4d62062b8269295b087a384e462c4d234f267fa67&scene=178&cur_album_id=1952926902587834371#rd](https://mp.weixin.qq.com/s?__biz=MzAwMDg2OTAxNg==&mid=2652055800&idx=1&sn=b3d9b5c9ced407850129e61da164db34&chksm=8105df0fb6725619b5b483dd6330fc252cf4d62062b8269295b087a384e462c4d234f267fa67&scene=178&cur_album_id=1952926902587834371#rd)