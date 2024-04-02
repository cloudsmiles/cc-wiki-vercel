# MySQL遇到过死锁问题吗，你是如何解决的？

我排查死锁的一般步骤是酱紫的：

- 查看死锁日志show engine innodb status;
- 找出死锁Sql
- 分析sql加锁情况
- 模拟死锁案发
- 分析死锁日志
- 分析死锁结果

可以看我这两篇文章哈：

- [手把手教你分析Mysql死锁问题](http://mp.weixin.qq.com/s?__biz=MzIwOTE2MzU4NA==&mid=2247483983&idx=1&sn=88bad91b6058c5fc40d346c0d71428c5&chksm=97794660a00ecf762978d9ebbc390d61de619ba22701f33720e3b95e5edf137e56a18ce5be46&scene=21#wechat_redirect)
- [Mysql死锁如何排查：insert on duplicate死锁一次排查分析过程](http://mp.weixin.qq.com/s?__biz=MzIwOTE2MzU4NA==&mid=2247483716&idx=1&sn=5bc2f65b14e7912cfbbba23016cb033d&chksm=9779456ba00ecc7dd840aea135008c46f88ec6da2586c09f77b85a5c4e20287f9ee8cfb31295&scene=21#wechat_redirect)