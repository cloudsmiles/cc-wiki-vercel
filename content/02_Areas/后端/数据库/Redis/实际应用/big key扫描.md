# big key扫描

## 问题

有时候会因为业务人员操作不当，在 Redis 实例中形成了很大的对象，比如一个很大的 hash 或一个很大的 zset，都是可能出现的。这样的对象给 Redis 的集群数据迁移带来了很大的问题，因为在集群环境下，如果某个 key 太大，会导致数据迁移卡顿。另外在内存分配上，如果一个 key 太大，那么当它需要扩容时，会一次性申请更大一块内存，这也会导致卡顿。如果这个 key 被删除，内存会被一次性回收，卡顿现象也会再次产生。

在平时的业务开发时，要尽量避免大 key 的产生。

如果观察到 redis的内存大起大落，这极有可能是因为大 key 导致的，这时候就需要定位出具体是哪个 key，要进一步定位出具体的业务来源，然后再改进相关业务代码设计。

## 如何定位大 key？

为了避免给线上 Redis 带来卡顿，就要用到 scan 指令，对于扫描出来的第一个key，使用 type 指令获得 key 的类型，然后使用相应数据结构的 size 或 len 方法来得到它的大小，对于每一种类型，将大小排名的前若干名作为扫描结果展示出来。

上这样的过程需要编写脚本，比较繁琐，不过 Redis 官方已经在 cli 指令中提供了这样的扫描功能，可以直接使用：

```bash
$ redis-cli -h 127.0.0.1  -p 6379 --bigkeys
 
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).
 
[00.00%] Biggest string found so far 'key9484' with 4 bytes
 
-------- summary -------
 
Sampled 10008 keys in the keyspace!
Total key length in bytes is 68962 (avg len 6.89)
 
Biggest string found 'key9484' has 4 bytes
 
10008 strings with 38898 bytes (100.00% of keys, avg size 3.89)
0 lists with 0 items (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
```

如果担心这个指令会大幅提升 Redis 的 ops 导致线上报警，还可以增加一个休眠参数：

```bash
$ redis-cli -h 127.0.0.1  -p 6379 --bigkeys -i 0.1
 
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).
 
[00.00%] Biggest string found so far 'key9484' with 4 bytes
 
-------- summary -------
 
Sampled 10008 keys in the keyspace!
Total key length in bytes is 68962 (avg len 6.89)
 
Biggest string found 'key9484' has 4 bytes
 
10008 strings with 38898 bytes (100.00% of keys, avg size 3.89)
0 lists with 0 items (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
```

上面的指令每隔 100 条 scan 指令就会休眠 0.1s，ops 就不会剧烈提升，但是扫描的时间会变长。

reference:

[https://jinguoxing.github.io/redis/2018/09/04/redis-scan/](https://jinguoxing.github.io/redis/2018/09/04/redis-scan/)