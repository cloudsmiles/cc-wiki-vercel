# 是否使用过 Redis 集群，集群的原理是什么

Redis Sentinal 着眼于高可用，在 master 宕机时会自动将 slave 提升
为 master，继续提供服务。

Redis Cluster 着眼于扩展性，在单个 Redis 内存不足时，使用 Cluster
进行分片存储。