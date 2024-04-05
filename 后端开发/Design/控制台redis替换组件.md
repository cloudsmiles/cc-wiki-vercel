
## 为什么要进行替换

### go-redis vs redigo

| 特性 | https://github.com/redis/go-redis | https://github.com/gomodule/redigo |
| --- | --- | --- |
| GitHub stars | 18.5k+ | 9.7k+ |
| 类型安全 | ✔️ | ❌ |
| 连接池 | 自动 | 手动 |
| 自定义命令 | ✔️ | ✔️ |
| https://redis.uptrace.dev/guide/go-redis-pubsub.html | ✔️ | ❌ |
| https://redis.uptrace.dev/guide/go-redis-sentinel.html | ✔️ | 使用插件 |
| https://redis.uptrace.dev/guide/go-redis-cluster.html | ✔️ | Using a plugin |
| https://redis.uptrace.dev/guide/ring.html | ✔️ | ❌ |
| https://redis.uptrace.dev/guide/go-redis-monitoring.html | ✔️ | ❌ |

两者主要区别在go-redis为每个redis命令提供安全类型的api，但是redigo使用类似打印方式的api：

```
// go-redis
timeout := time.Second
_, err := rdb.Set(ctx, "key", "value", timeout).Result()

// redigo
_, err := conn.Do("SET", "key", "value", "EX", 1)

```

此外，go-redis自动使用连接池，但redigo需要显示管理连接：

```
// go-redis implicitly uses connection pooling
_, err := rdb.Set(...).Result()

// redigo requires explicit connection management
conn := pool.Get()_, err := conn.Do(...)
conn.Close()

```

### 控制台现状

控制台封装的redis组件，实际上是包装redigo的Do方法，让动态类型变成固定的类型，提供给外层使用，这种方式，不易于以后的扩展，而且很容易有一堆重复的方法，只是入参和出参类型有差别

为了摆脱这种维护方式，可以直接替换go-redis组件，控制台只维护组件的生命周期

## 如何替换

### 组件的初始化

组件的初始化与当前方式相同，使用单例模式，服务启动时初始化，同时规范化目前的redis配置，使用相同格式，但是用标签来区分不同的实例

```
// before
# Redis 本地
backendRedisAddr = "r-uf6f0hryp8u9x4s4fz.redis.rds.aliyuncs.com:6379" backendRedisPassword = "dxCobRvHM2XjjLLcF8Ue"
backendRedisIndex = 1

# Redis 账号 老控制台redis
accountRedisAddr = "r-uf632a1e33126f94.redis.rds.aliyuncs.com:6379" accountRedisPassword = "0gdy6Eyq2RIN"
accountRedisIndex = 0

// after
[redis_console]
redisAddr = "r-uf6f0hryp8u9x4s4fz.redis.rds.aliyuncs.com:6379" redisPassword = "dxCobRvHM2XjjLLcF8Ue"
redisIndex = 1

[redis_old_console]
redisAddr = "r-uf632a1e33126f94.redis.rds.aliyuncs.com:6379"
redisPassword = "0gdy6Eyq2RIN"
redisIndex = 0

```

### 函数替换的可行性

普通函数的替换

> 不封装函数，外部直接调用处理
> 

```
// before
func (c *Client) SetStringNX(key string, value string) (reply int, err error) {
    return redis.Int(c.GetPool().Get().Do("SETNX", key, value))
}

func (c *Client) GetHashM(key string) (reply map[string]string, err error) {
    ss, err := redis.Strings(c.GetPool().Get().Do("HGETALL", key))
    if err != nil {
        return nil, err
    }

    reply = make(map[string]string)
    for i := 0; i < len(ss); i += 2 {
        reply[ss[i]] = ss[i+1]
    }

    return reply, nil
}

func (c *Client) DelKey(key string) (reply int, err error) {
    return redis.Int(c.GetPool().Get().Do("DEL", key))
}

func (c *Client) ExpireKey(key string, seconds int64) (reply int, err error) {
    return redis.Int(c.GetPool().Get().Do("EXPIRE", key, seconds))
}

// after
reply, err := rdb.SetNx(key, value, 0).Result()

reply, err := rdb.HGetAll(key).Result()

reply, err := rdb.Del(key).Result()

reply, err := rdb.Expire(key, time.Second*seconds).Result()

```

多个redis操作的函数替换

> 封装函数，与以前逻辑保持一致
> 

```
// before
func (c *Client) SetNxWithExpire(key, value string, expire int) (res int64, err error) {
  conn := c.GetPool().Get() defer conn.Close()
  reply, err := conn.Do("SETNX", key, value)
  if err != nil {
    return 0, err
  }
  if _, err := conn.Do("EXPIRE", key, expire); err != nil {
    return 0, err
  }
  return reply.(int64), nil
}

// after
func SetNxWithExpire(key, value string, expire int) (res int64, err error) {
  isSet, err := rdb.SetNX(key, key, 0).Result()
  if err != nil {
    return 0, err
  }
  err = rdb.Expire(key, expire*ime.Second).Err()
  if err != nil {
    return 0, err
  }
  if isSet {
    return 1, nil
  }
  return 0, nil
}

```

运行lua脚本

```
// before
func (c *Client) EvalLuaScript(script string, keys []string, args ...interface{}) (reply interface{}, err error) {
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
}

// after
var script = redis.NewScript(lua)
reply, err := script.Run(rdb, keys, args).Result()

```

结果集映射

> 指定的struct需要含有redis:""标签，这与旧方式兼容
> 

```
// before
func (c *Client) GetAllHashStruct(key string, target interface{}) (err error) {
  conn := c.GetPool().Get()
  defer conn.Close()
  ss, err := redis.Values(conn.Do("HGETALL", key))
  if err != nil {
    return err
  }
  if len(ss) == 0 {
    return ErrKeyNotFound
  }
  return redis.ScanStruct(ss, target)
}

// after
err := rdb.HGetAll(key).Scan(target)

```

错误处理

为了兼容旧的错误码返回，需要封装相同的方法

```
// before
func (c *Client) SMembersInt64s(key string) (reply []int64, err error) {
    if len(key) == 0 {
        return nil, errKeyIsBlank
    }

    conn := c.GetPool().Get()
    defer conn.Close()
    ss, err := redis.Int64s(conn.Do("SMEMBERS", key))
    if err != nil {
        return nil, err
    }
    return ss, nil
}

// after
func (c *Client) SMembersInt64s(key string) (reply []int64, err error) {
    if len(key) == 0 {
        return nil, errKeyIsBlank
    }
    ss, err := rdb.SMembers(key).Result()
    if err != nil {
        return nil, err
    }
    for _, v := range ss {
        reply = append(reply, zstring.StringToInt64(v)
    }
    return reply, nil
}

```

## 可能存在的问题

go-redis新的版本，需要传递context，控制台使用时需要传递大量的context.Background()参数