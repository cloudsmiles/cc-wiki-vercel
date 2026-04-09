# slow log

与其他任意存储系统例如mysql，mongodb可以查看慢日志一样，redis也可以，即通过命令slowlog。

## 用法

**SLOWLOG subcommand [argument]**

subcommand主要有：

- get，用法：slowlog get [argument]，获取argument参数指定数量的慢日志。
- len，用法：slowlog len，总慢日志数量。
- reset，用法：slowlog reset，清空慢日志。

## 示例

```bash
!redis-cli slowlog get 5
1) 1) (integer) 2
   2) (integer) 1537786953
   3) (integer) 17980
   4) 1) "scan"
      2) "0"
      3) "match"
      4) "key99*"
      5) "count"
      6) "1000"
   5) "127.0.0.1:50129"
   6) ""
2) 1) (integer) 1
   2) (integer) 1537785886
   3) (integer) 39537
   4) 1) "keys"
      2) "*"
   5) "127.0.0.1:49701"
   6) ""
3) 1) (integer) 0
   2) (integer) 1537681701
   3) (integer) 18276
   4) 1) "ZADD"
      2) "page_rank"
      3) "10"
      4) "google.com"
   5) "127.0.0.1:52334"
   6) ""
```

命令耗时超过多少才会保存到slowlog中，可以通过命令config set slowlog-log-slower-than 2000配置并且不需要重启redis。

注意：单位是微妙，2000微妙即2毫秒。

reference:

[https://jinguoxing.github.io/redis/2018/09/04/redis-scan/](https://jinguoxing.github.io/redis/2018/09/04/redis-scan/)