# redis组件替换设计方案

## 为什么要进行替换

### go-redis vs redigo

两者的比较图：

| Feature | https://github.com/redis/go-redis | https://github.com/gomodule/redigo |
| --- | --- | --- |
| GitHub stars | 18.5k+ | 9.7k+ |
| Type-safe | ✔️ | ❌ |
| Connection pooling | Automatic | Manual |
| Custom commands | ✔️ | ✔️ |
| https://redis.uptrace.dev/guide/go-redis-pubsub.html | ✔️ | ❌ |
| https://redis.uptrace.dev/guide/go-redis-sentinel.html | ✔️ | Using a plugin |
| https://redis.uptrace.dev/guide/go-redis-cluster.html | ✔️ | Using a plugin |
| https://redis.uptrace.dev/guide/ring.html | ✔️ | ❌ |
| https://redis.uptrace.dev/guide/go-redis-monitoring.html | ✔️ | ❌ |

[redigo](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fgithub.com%2Fgomodule%2Fredigo&source=article&objectId=2056319):

- 由于输入是万能类型所以必须记住每个命令的参数和返回值情况, 使用起来非常的不友好，
- 参数类型是万能类型导致在编译阶段无法检查参数类型,
- 每个命令都需要花时间记录使用方法，参数个数等，使用成本高；

[go-redis](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fgithub.com%2Fgo-redis%2Fredis&source=article&objectId=2056319):

- 细化了每个redis每个命令的功能, 我们只需记住命令，具体的用法直接查看接口的申请就可以了，使用成本低；
- 其次它对数据类型按照redis底层的类型进行统一，编译时就可以帮助检查参数类型
- 并且它的响应统一采用 Result 的接口返回，确保了返回参数类型的正确性，对用户更加友好；

> go-redis已经被收录到redis官方库，说明它的稳定性和应用范围是得到肯定的
> 

### 控制台现状

控制台封装的redis组件，实际上是包装redigo的Do方法，让动态类型变成固定的类型，提供给外层使用，这种方式，不仅不易于以后的扩展，而且很容易有重复功能的方法，只是入参和出参类型不太相同。

为了摆脱这种维护方式，可以直接替换go-redis组件，控制台只维护组件的生命周期

## 如何替换

### 组件的初始化

组件的初始化与当前方式相同，使用单例模式，服务启动时初始化，同时规范化目前的redis配置，使用相同格式，但是用标签来区分不同的实例。

| 原来 | 变更后 |
| --- | --- |
| # Redis 本地
backendRedisAddr = "r-uf6f0hryp8u9x4s4fz.redis.rds.aliyuncs.com:6379"
backendRedisPassword = "dxCobRvHM2XjjLLcF8Ue"
backendRedisIndex = 1
# Redis 账号 老控制台redis
accountRedisAddr = "r-uf632a1e33126f94.redis.rds.aliyuncs.com:6379"
accountRedisPassword = "0gdy6Eyq2RIN"
accountRedisIndex = 0 | [redis_console]
backendRedisAddr = "r-uf6f0hryp8u9x4s4fz.redis.rds.aliyuncs.com:6379"
backendRedisPassword = "dxCobRvHM2XjjLLcF8Ue"
backendRedisIndex = 1

[redis_old_console]
accountRedisAddr = "r-uf632a1e33126f94.redis.rds.aliyuncs.com:6379"
accountRedisPassword = "0gdy6Eyq2RIN"
accountRedisIndex = 0 |

### 函数替换的可行性

普通函数的替换

> 不封装函数，外部直接调用处理
> 

| 原来 | 变更后 |
| --- | --- |
| func (c *Client) SetStringNX(key string, value string) (reply int, err error) {
	return redis.Int(c.GetPool().Get().Do("SETNX", key, value))
} | isSet, err := rdb.SetNx(key, value, 0).Result() |
| func (c *Client) GetHashM(key string) (reply map[string]string, err error) {
	ss, err := redis.Strings(c.GetPool().Get().Do("HGETALL", key))
	if err != nil {
		return nil, err
	}

	reply = make(map[string]string)
	for i := 0; i < len(ss); i += 2 {
		reply[ss[i]] = ss[i+1]
	}

	return reply, nil
} | reply, err := rdb.HGetAll(key).Result() |
| func (c *Client) DelKey(key string) (reply int, err error) {
return redis.Int(c.GetPool().Get().Do("DEL", key))
} | reply, err := rdb.Del(key).Result() |
| func (c *Client) ExpireKey(key string, seconds int64) (reply int, err error) {
return redis.Int(c.GetPool().Get().Do("EXPIRE", key, seconds))
} | isSet, err := rdb.Expire(key, time.Second*seconds).Result() |

多个redis操作的函数替换

> 封装函数，与以前逻辑保持一致
> 

| 原来 | 变更后 |
| --- | --- |
| func (c *Client) SetNxWithExpire(key, value string, expire int) (res int64, err error) {
conn := c.GetPool().Get()
defer conn.Close()
reply, err := http://conn.do/("SETNX", key, value)
if err != nil {
return 0, err
}
if _, err := http://conn.do/("EXPIRE", key, expire); err != nil {
return 0, err
}
return reply.(int64), nil
} | func SetNxWithExpire(key, value string, expire int) (res int64, err error) {
isSet, err := rdb.SetNX(key, "123", 12time.Second).Result()
if err != nil {
return 0, err
}
err = rdb.Expire(key, 12time.Second).Err()
if err != nil {
return 0, err
}
if isSet {
return 1, nil
}
return 0, nil
} |
| func (c *Client) IncrWithExpire(key string, second int) (reply *int, err error) {
	id, err := c.Incr(key)
	if err != nil {
		return nil, err
	}
	err = c.Expire(key, second)
	if err != nil {
		return nil, err
	}
	return id, nil
} | func IncrWithExpire(key string, second int) (reply int64, err error) {
id, err := rdb.Incr(key).Result()
if err != nil {
return 0, err
}
err = rdb.Expire(key, second*time.Second).Err()
if err != nil {
return 0, err
}
return id, nil
} |

运行lua脚本

| 原来 | 变更后 |
| --- | --- |
| func (c *Client) EvalLuaScript(script string, keys []string, args ...interface{}) (reply interface{}, err error) {
	conn := c.GetPool().Get()
	defer conn.Close()
	rScript := redis.NewScript(len(keys), script)

	var keysAndArgs []interface{}
	for _, key := range keys {
		keysAndArgs = append(keysAndArgs, key)
	}

	for _, arg := range args {
		keysAndArgs = append(keysAndArgs, arg)
	}

	return rScript.Do(conn, keysAndArgs...)
} | var script = redis.NewScript(lua)
reply, err := script.Run(rdb, keys, args).Result() |

结果集映射

| 原来 | 变更后 |
| --- | --- |
| func (c *Client) GetAllHashStruct(key string, target interface{}) (err error) {
defer handleError(&err, key)
conn := c.GetPool().Get()
defer conn.Close()
ss, err := redis.Values(http://conn.do/("HGETALL", key))
if err != nil {
return err
}
if len(ss) == 0 {
return ErrKeyNotFound
}
return redis.ScanStruct(ss, target)
} | func GetAllHashStruct(key string, target interface{}) error {
return rdb.HGetAll(key).Scan(target)
} |

## 可能存在的问题

go-redis新的版本，需要传递context，控制台使用时需要传递大量的context.Background()参数