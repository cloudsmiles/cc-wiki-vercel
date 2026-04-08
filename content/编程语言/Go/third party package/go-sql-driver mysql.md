# go-sql-driver/mysql

**go-sql-driver/mysql** 的核心功能是，遵循 **database/sql 标准库**中预留的接口协议，提供出对应于 mysql 的实现版本，将和 mysql 服务端的数据传输、通信协议，预处理模式、事务操作等内容封装实现在其中.

**go-sql-driver/mysql** 在整个 **database/sql** 运行框架中的定位如下图所示：

![https://pic1.zhimg.com/80/v2-11437af8fb6a4fe12047795c8f792348_1440w.webp](https://pic1.zhimg.com/80/v2-11437af8fb6a4fe12047795c8f792348_1440w.webp)

本期涉及实现的 database/sql 接口协议如下图所示：

![Untitled](RPG从零开始/Program%20Language/Go/third%20party%20package/go-sql-driver%20mysql/Untitled.png)

本期正是沿着各个接口实现类的顺序展开源码走读，对应的分享大纲如下：

![Untitled](RPG从零开始/Program%20Language/Go/third%20party%20package/go-sql-driver%20mysql/Untitled%201.png)

## **1 数据库驱动**

## **1.1 驱动**

首先进入第一个核心模块：数据库驱动. 在 database/sql 标准库中定义的接口协议如下：

```go
type Driver interface {
    // 打开一个新的数据库连接
    Open(name string) (Conn, error)
}
```

在上期分享中也有提到，在使用 mysql driver 时，只需要匿名导入 **go-sql-driver/mysql** 的 lib 包，即可完成 driver 的注册操作. 实现方式如下：

```go
import (
    // 注册 mysql 数据库驱动
    _ "github.com/go-sql-driver/mysql"
)
```

其实现原理在于，在 **go-sql-driver/mysql** 包下会通过 init 方法，在包初始化时就将 mysql driver 实例注册到 database/sql 的驱动 map 之中：

```go
func init() {
    sql.Register("mysql", &MySQLDriver{})
}
```

**go-sql-driver/mysql** 包下实现的驱动类定义位于 driver.go 文件中，对应的代码如下：

```go
// MySQL 版本的数据库驱动
type MySQLDriver struct{}
```

对应实现的 Open 方法用于创建数据库连接，核心步骤包括：

- 解析 dsn，转为配置类实例
- 构造连接器实例
- 通过连接器完成连接创建操作

```go
func (d MySQLDriver) Open(dsn string) (driver.Conn, error) {
    // 解析 dsn
    cfg, err := ParseDSN(dsn)
    if err != nil {
        return nil, err
    }
    // 构造连接器
    c, err := newConnector(cfg)
    if err != nil {
        return nil, err
    }
    // 通过连接器创建连接
    return c.Connect(context.Background())
}
```

## **1.2 连接器**

连接器 Connector 同样是遵循 database/sql 库中定义的接口规范来实现的.

database/sql connector 接口定义：

```go
// database/sql 定义的抽象的连接器接口
type Connector interface {
    // 创建连接
    Connect(context.Context) (Conn, error)
    // 返回数据库驱动实例
    Driver() Driver
}
```

**go-sql-driver/mysql** 实现的连接器类位于 connecto.go 文件中：

```go
type connector struct {
    cfg               *Config // immutable private copy.
    encodedAttributes string  // Encoded connection attributes.
}

func newConnector(cfg *Config) (*connector, error) {
    encodedAttributes := encodeConnectionAttributes(cfg.ConnectionAttributes)
    if len(encodedAttributes) > 250 {
        return nil, fmt.Errorf("connection attributes are longer than 250 bytes: %dbytes (%q)", len(encodedAttributes), cfg.ConnectionAttributes)
    }
    return &connector{
        cfg:               cfg,
        encodedAttributes: encodedAttributes,
    }, nil
}
```

此处着重探讨一下通过 connector 的 connect 方法实现的数据库连接创建流程.

通过 mysql 客户端创建一笔与 mysql 服务端之间的数据库连接的核心交互流程如下：

![https://pic3.zhimg.com/80/v2-c12dd5ef35b5ce049f73d6d41483dd9e_1440w.webp](https://pic3.zhimg.com/80/v2-c12dd5ef35b5ce049f73d6d41483dd9e_1440w.webp)

其中主要包含如下几个核心步骤：

- 创建连接（net.Dialer.DialContext）
- 设置为 tcp 长连接（net.TCPConn.KeepAlive）
- 创建连接缓冲区（mc.buf = newBuffer）
- 设置连接超时配置（mc.buf.timeout = mc.cfg.ReadTimeout；mc.writeTimeout = mc.cfg.WriteTimeout）
- 接收来自服务端的握手请求（mc.readHandshakePacket）
- 向服务端发起鉴权请求（mc.writeHandshakeResponsePacket）
- 处理鉴权结果（mc.handleAuthResult）
- 设置 dsn 中的参数变量（mc.handleParams）

完整的代码以及对应的注释展示如下：

```go
// 创建一笔新的数据库连接
func (c *connector) Connect(ctx context.Context) (driver.Conn, error) {
    var err error
    // 构造 mysql 连接实例
    mc := &mysqlConn{
        maxAllowedPacket: maxPacketSize,
        maxWriteSize:     maxPacketSize - 1,
        closech:          make(chan struct{}),
        cfg:              c.cfg,
        connector:        c,
    }
    mc.parseTime = mc.cfg.ParseTime

    // 根据传输协议类型获取连接构造器
    dialsLock.RLock()
    dial, ok := dials[mc.cfg.Net]
    dialsLock.RUnlock()
    if ok {
        dctx := ctx
        if mc.cfg.Timeout > 0 {
            var cancel context.CancelFunc
            dctx, cancel = context.WithTimeout(ctx, c.cfg.Timeout)
            defer cancel()
        }
        mc.netConn, err = dial(dctx, mc.cfg.Addr)
    } else {
        // 构造 net conn 实例
        nd := net.Dialer{Timeout: mc.cfg.Timeout}
        mc.netConn, err = nd.DialContext(ctx, mc.cfg.Net, mc.cfg.Addr)
    }

    // ...
    // 将 tcp 连接设置为长连接
    if tc, ok := mc.netConn.(*net.TCPConn); ok {
        if err := tc.SetKeepAlive(true); err != nil {
            c.cfg.Logger.Print(err)
        }
    }

    // 启动 watcher，关注 context 状态，即时回收连接资源
    mc.startWatcher()
    if err := mc.watchCancel(ctx); err != nil {
        mc.cleanup()
        return nil, err
    }
    defer mc.finish()

    // 构造连接的数据缓冲区 buffer
    mc.buf = newBuffer(mc.netConn)

    // 设置单次读、写操作的超时时间
    mc.buf.timeout = mc.cfg.ReadTimeout
    mc.writeTimeout = mc.cfg.WriteTimeout

    // 读取来自 mysql 服务端的握手报文
    authData, plugin, err := mc.readHandshakePacket()
    if err != nil {
        mc.cleanup()
        return nil, err
    }

    if plugin == "" {
        plugin = defaultAuthPlugin
    }

    // 获取鉴权加密信息
    authResp, err := mc.auth(authData, plugin)
    // ...
    // 发送握手响应. 携带上数据库、用户名、密码等鉴权信息
    if err = mc.writeHandshakeResponsePacket(authResp, plugin); err != nil {
        mc.cleanup()
        return nil, err
    }

    // 处理鉴权响应结果
    if err = mc.handleAuthResult(authData, plugin); err != nil {
        // ...
        mc.cleanup()
        return nil, err
    }

    // ...
    // 处理 dsn 中的参数
    err = mc.handleParams()
    // ...
    return mc,nil
}
```

## **1.3 配置**

与 mysql 连接配置有关的内容被聚合在 dsn.go 文件定义的 Config 类中，核心字段均已给出注释：

```go
type Config struct {
    User                 string            // 用户名
    Passwd               string            // 密码
    Net                  string            // 网络 tcp 等
    Addr                 string            // ip:port
    DBName               string            // 数据库名
    Params               map[string]string // 连接参数
    ConnectionAttributes string            // Connection Attributes, comma-delimited string of user-defined "key:value" pairs
    Collation            string            // 连接字符集
    Loc                  *time.Location    // Location for time.Time values
    MaxAllowedPacket     int               // Max packet size allowed
    ServerPubKey         string            // Server public key name
    pubKey               *rsa.PublicKey    // Server public key
    TLSConfig            string            // TLS configuration name
    TLS                  *tls.Config       // TLS configuration, its priority is higher than TLSConfig
    Timeout              time.Duration     // Dial timeout
    ReadTimeout          time.Duration     // 读请求超时配置
    WriteTimeout         time.Duration     // 写请求超时配置
    Logger               Logger            // Logger

    AllowAllFiles            bool // 允许使用 LOAD DATA LOCAL INFILE 导入数据库
    AllowCleartextPasswords  bool // 支持明文密码客户端
    AllowFallbackToPlaintext bool // Allows fallback to unencrypted connection if server does not support TLS
    AllowNativePasswords     bool // Allows the native password authentication method
    AllowOldPasswords        bool // 允许使用不安全的旧密码
    CheckConnLiveness        bool // Check connections for liveness before using them
    ClientFoundRows          bool // 返回匹配的行数而非影响的行数
    ColumnsWithAlias         bool // 将表名添加在列名前缀
    InterpolateParams        bool // 将参数占位符插入 sql
    MultiStatements          bool // 允许一条语句执行多笔查询操作
    ParseTime                bool // 格式化时间为 time.Time 格式
    RejectReadOnly           bool // Reject read-only connections
}
```

该文件中两个核心方法：

- **ParseDSN**：完成 dsn 到 config 实例的转换
- **FormatDSN**：完成 config 实例到 dsn 的转换

两个方法的具体细节大家可以深入到 dsn.go 文件中查看，这里不再赘述：

```go
// 解析 dsn，构造 config 实例
func ParseDSN(dsn string) (cfg *Config, err error) {
    // New config with some default values
    cfg = NewConfig()
    // 从 dsn 中解析参数填充到 cfg 中...
    return
}
```

```go
// 从 config 中解析出 dsn
func (cfg *Config) FormatDSN() string {
    // ...
}
```

## **1.4 协议**

![https://pic2.zhimg.com/80/v2-093f71db5840395eef0e3f547c891295_720w.webp](https://pic2.zhimg.com/80/v2-093f71db5840395eef0e3f547c891295_720w.webp)

在 mysql 客户端读取和发送与服务端之间的消息报文时，采用的一套特定的协议规则：

- 每笔消息分为请求头和正文两部分
- **在请求头部分中：**
- **前三个字节对应的是消息正文长度**，共 24 个 bit 位，表示的长度最大值为 2^24 - 1，因此消息最大长度为 16MB-1byte. 如果消息长度大于该阈值，则需要进行分包
- **第四个字节对应为请求的 sequence 序列号**. 一个新的客户端从 0 开始依次递增序列号，每次读消息时，会对序列号进行校验，要求必须必须和本地序号保持一致
- **在正文部分中：**
- **对于客户端接收服务端消息的场景，首个字节标识了这条消息的状态.** 倘若为 0，代表响应成功；倘若为 255，代表有错误发生；其他枚举值含义此处不再赘述
- **对于客户端发送消息到服务端的场景，首个字节标识了这笔请求的类型**. 则首个字节代表的是 sql 指令的类型. 具体类型在本文 2.3 小节中展开介绍

理清了通信协议后，下面走读一下客户端通过 mysqlConn 执行读、写消息的源码流程：

**读流程：**

```go
// 从 conn 中读取来自服务端的消息
func (mc *mysqlConn) readPacket() ([]byte, error) {
    var prevData []byte
    for {
        // 读出头 4 个字节的请求头
        data, err := mc.buf.readNext(4)
        if err != nil {
            if cerr := mc.canceled.Value(); cerr != nil {
                return nil, cerr
            }
            mc.cfg.Logger.Print(err)
            mc.Close()
            return nil, ErrInvalidConn
        }

        // 头三个字节对应为消息长度
        pktLen := int(uint32(data[0]) | uint32(data[1])<<8 | uint32(data[2])<<16)

        // 第 4 个字节为请求序列号，需要检验其一致性
        if data[3] != mc.sequence {
            mc.Close()
            if data[3] > mc.sequence {
                return nil, ErrPktSyncMul
            }
            return nil, ErrPktSync
        }
        // 每次分包序列号都需要递增
        mc.sequence++

        // 消息长度为 0，则直接返回 prevData
        if pktLen == 0 {
            // there was no previous packet
            if prevData == nil {
                mc.cfg.Logger.Print(ErrMalformPkt)
                mc.Close()
                return nil, ErrInvalidConn
            }

            return prevData, nil
        }

        // 读取指定长度的消息
        data, err = mc.buf.readNext(pktLen)
        if err != nil {
            if cerr := mc.canceled.Value(); cerr != nil {
                return nil, cerr
            }
            mc.cfg.Logger.Print(err)
            mc.Close()
            return nil, ErrInvalidConn
        }

        // 未达到包长度上限 1<<24 - 1 字节，则直接返回结果
        if pktLen < maxPacketSize {
            // zero allocations for non-split packets
            if prevData == nil {
                return data, nil
            }

            return append(prevData, data...), nil
        }

        // 倘若达到了包的长度上限，需要进行分包处理
        prevData = append(prevData, data...)
    }
}
```

**写流程：**

```go
// Write packet buffer 'data'
func (mc *mysqlConn) writePacket(data []byte) error {
    // 消息长度
    pktLen := len(data) - 4

    // 消息太长了
    if pktLen > mc.maxAllowedPacket {
        return ErrPktTooLarge
    }

    for {
        // 将消息长度信息存储到前 3 个字节
        var size int
        if pktLen >= maxPacketSize {
            data[0] = 0xff
            data[1] = 0xff
            data[2] = 0xff
            size = maxPacketSize
        } else {
            data[0] = byte(pktLen)
            data[1] = byte(pktLen >> 8)
            data[2] = byte(pktLen >> 16)
            size = pktLen
        }
        // 第 4 个字节存储请求序号
        data[3] = mc.sequence

        // 设置单次写操作的超时时长
        if mc.writeTimeout > 0 {
            if err := mc.netConn.SetWriteDeadline(time.Now().Add(mc.writeTimeout)); err != nil {
                return err
            }
        }

        // 执行写操作
        n, err := mc.netConn.Write(data[:4+size])
        if err == nil && n == 4+size {
            mc.sequence++
            if size != maxPacketSize {
                return nil
            }
            pktLen -= size
            data = data[size:]
            continue
        }

        // ...
    }
}
```

在此处，我们也能够看出来，**针对一笔特定的数据库连接实例，其本身是不支持并发使用的**，其中使用的缓冲区 buffer、序列号 sequence 等状态数据都是未通过互斥锁进行保护的临界资源.

## **2 数据库连接**

本章中，我们探讨 **go-sql-driver/mysql** 库对数据库连接 Conn 的实现. 这里可以称得上是全文最关键的部分，和 mysql 服务端的所有交互流程都是紧密围绕着 Conn 展开的.

## **2.1 连接**

首先回顾一下 database/sql 库中定义的数据库连接接口：

```go
type Conn interface {
    // 预处理 sql，生成 statement
    Prepare(query string) (Stmt, error)

    // 关闭连接
    Close() error

    // 开启事务
    Begin() (Tx, error)
}
```

接下来是 **go-sql-driver/mysql** 库对 Conn 的实现版本:

![Untitled](RPG从零开始/Program%20Language/Go/third%20party%20package/go-sql-driver%20mysql/Untitled%202.png)

值得一提的是，在使用 mysqlConn 的过程中：

- 对于**读流程**：主要通过数据缓冲区 buffer 进行数据的缓存
- 对于**写流程**：直接通过网络连接 netConn 发送数据

mysqlConn 的实现源码位于 connection.go 文件中，代码及注释展示如下

```go
type mysqlConn struct {
    // 缓冲区数据
    buf              buffer
    // 网络连接
    netConn          net.Conn
    rawConn          net.Conn    // underlying connection when netConn is TLS connection.
    result           mysqlResult // sql 执行结果
    cfg              *Config // 配置文件
    connector        *connector // 连接器 
    maxAllowedPacket int
    maxWriteSize     int
    writeTimeout     time.Duration // 单批次写操作超时时间
    flags            clientFlag // 客户端状态标识
    status           statusFlag  // 服务端状态标识
    sequence         uint8 // 客户端请求序号
    parseTime        bool

    watching bool // 是否开启了 watcher 协程
    watcher  chan<- context.Context // watcher 协程监听的 context
    closech  chan struct{} // 控制整个 conn 的生命周期
    finished chan<- struct{} // 标识连接是否已完成
    canceled atomicError // 标识连接是否已取消
    closed   atomicBool  // 标识连接是否已关闭
}
```

mysqlConn 对外可以通过公开方法 Close 实现关闭，对内主要使用 cleanup 方法释放连接资源. 在 cleanup 方法内部会通过一个原子变量 closed 来保证关闭操作不被重复执行.

```go
func (mc *mysqlConn) Close() (err error) {
    // Makes Close idempotent
    if !mc.closed.Load() {
        err = mc.writeCommandPacket(comQuit)
    }

    mc.cleanup()

    return
}

// closed the network connection.
func (mc *mysqlConn) cleanup() {
    if mc.closed.Swap(true) {
        return
    }

    // Makes cleanup idempotent
    close(mc.closech)
    if mc.netConn == nil {
        return
    }
    if err := mc.netConn.Close(); err != nil {
        mc.cfg.Logger.Print(err)
    }
    mc.clearResult()
}
```

## **2.2 缓冲区**

mysqlConn 中内置的数据缓冲区 buffer 类定义如下：

- buf：用于存放数据的字节切片
- nc：从属的连接，通常为 tcp 连接
- idx：当前已读取数据的进度索引
- length：剩余未读取数据的长度
- timeout：单次读操作的超时时长

```go
type buffer struct {
    // 缓冲区中的数据
    buf     []byte 
    // 关联的连接，通常为 tcp
    nc      net.Conn
    // 已读取数据的索引 index
    idx     int
    // 未读取数据的长度
    length  int
    // 数据库连接超时设置
    timeout time.Duration 
    // ...
}
```

```go
// newBuffer allocates and returns a new buffer.
func newBuffer(nc net.Conn) buffer {
    fg := make([]byte, defaultBufSize)
    return buffer{
        buf:  fg,
        nc:   nc,
        // ...
    }
}
```

mysql 客户端从 mysqlConn 缓冲区中读取数据的主流程如下，这部分可以和 1.2 小节介绍的 readPacket 方法进行串联呼应：

![Untitled](RPG从零开始/Program%20Language/Go/third%20party%20package/go-sql-driver%20mysql/Untitled%203.png)

核心的 readNext 方法源码为：

```go
// 从缓冲区读取 need 个字节数据
func (b *buffer) readNext(need int) ([]byte, error) {
    // 倘若剩余数据量不足，则需要调用 fill 方法对 buffer 扩容，且会从 conn 中读取数据填充到 buffer 中
    if b.length < need {
        // refill
        if err := b.fill(need); err != nil {
            return nil, err
        }
    }

    offset := b.idx
    b.idx += need
    b.length -= need
    return b.buf[offset:b.idx], nil
}
```

在 buffer 中剩余数据量不足时，会调用 fill 方法从 conn 中读取数据，往 buffer 中执行填充和扩容操作：

```go
// 从 conn 中读取数据填充到 buffer
func (b *buffer) fill(need int) error {
    n := b.length
    
    // 新的字节切片
    dest := b.dbuf[b.flipcnt&1]

    // 如有必要，进行扩容
    if need > len(dest) {
        // Round up to the next multiple of the default size
        dest = make([]byte, ((need/defaultBufSize)+1)*defaultBufSize)

        // ...
    }

    // ...
    b.buf = dest
    b.idx = 0

    // 从 conn 中读取数据填充到 buffer. 因为可能涉及到分包,因此需要使用 for 循环
    for {
        // 设置读取数据的超时时间
        if b.timeout > 0 {
            if err := b.nc.SetReadDeadline(time.Now().Add(b.timeout)); err != nil {
                return err
            }
        }

       // 从 tcp 连接中读取数据，填充到 buffer 中
        nn, err := b.nc.Read(b.buf[n:])
        n += nn

        switch err {
        // 读取数据未发生错误
        case nil:
            // 未达到指定长度，则需要处理下一个包
            if n < need {
                continue
            }
            // 已达到长度，直接返回            
            b.length = n
            return nil

        // 读完全部数据
        case io.EOF:
            // 读取数据量已达到指定长度，返回结果
            if n >= need {
                b.length = n
                return nil
            }
            // 返回预期之外的 EOF 错误
            return io.ErrUnexpectedEOF

        default:
            return err
        }
    }
}
```

## **2.3 查询**

下面是通过 mysqlConn 执行查询类请求的流程：

![https://pic4.zhimg.com/80/v2-683f9dd8063895c72054f5d39c65af13_720w.webp](https://pic4.zhimg.com/80/v2-683f9dd8063895c72054f5d39c65af13_720w.webp)

对于 query 方法，入参中的 query 字段为 sql 模板，args 字段为用于填充占位符的参数.

**query 方法的出参类型为 textRows，其首先会读取响应报文中第一部分，填充各个列的信息**，**后续内容会保留在内置的 conn 中，通过使用方调用 rows 的 Next 方法时再进行读取操作**.

```go
func (mc *mysqlConn) Query(query string, args []driver.Value) (driver.Rows, error) {
    return mc.query(query, args)
}

func (mc *mysqlConn) query(query string, args []driver.Value) (*textRows, error) {
    handleOk := mc.clearResult()
    // 连接已关闭？
    if mc.closed.Load() {
        mc.cfg.Logger.Print(ErrInvalidConn)
        return nil, driver.ErrBadConn
    }
    
    // 提前执行 sql 中的参数替换
    if len(args) != 0 {
        if !mc.cfg.InterpolateParams {
            return nil, driver.ErrSkip
        }
        // 提前处理，进行 sql 中的参数替换
        prepared, err := mc.interpolateParams(query, args)
        if err != nil {
            return nil, err
        }
        query = prepared
    }
    
    // 将 sql 发送到服务端
    err := mc.writeCommandPacketStr(comQuery, query)
    if err == nil {
        // 读取响应的请求头
        var resLen int
        // 获取到列的个数 resLen
        resLen, err = handleOk.readResultSetHeaderPacket()
        if err == nil {
            // 构造 textRows 实例
            rows := new(textRows)
            rows.mc = mc

            if resLen == 0 {
                rows.rs.done = true

                switch err := rows.NextResultSet(); err {
                case nil, io.EOF:
                    return rows, nil
                default:
                    return nil, err
                }
            }

            // 读取列信息数据填充到 rows 实例中
            rows.rs.columns, err = mc.readColumns(resLen)
            return rows, err
        }
    }
    return nil, mc.markBadConn(err)
}
```

将 sql 指令发往服务端是通过 writeCommandPacketStr 方法实现的. 其中**对应于 data[4] 位置的字节标识了 sql 指令的类型**. 常用的几种类型包括：

```go
const (
    comQuit byte = iota + 1 // 1——退出
    comInitDB // 2——初始化数据库
    comQuery  // 3——非 prepare 模式的查询、操作
    // ...
    comCreateDB  // 5——创建数据库
    comDropDB  // 6——删库跑路
    // ...
    comStmtPrepare  // 22—— prepare statement
    comStmtExecute  // 23—— statement exec
    // ...
    comStmtClose  // 25—— statement close
    // ...
)
```

writeCommandPacketStr 方法源码如下：

```go
func (mc *mysqlConn) writeCommandPacketStr(command byte, arg string) error {
    // 重置序列号 sequence
    mc.sequence = 0

    // 根据消息长度构造字节数组. 倘若此时缓冲区 buffer 仍在使用中，数据未读取干净，会报错
    pktLen := 1 + len(arg)
    data, err := mc.buf.takeBuffer(pktLen + 4)
    if err != nil {
        // cannot take the buffer. Something must be wrong with the connection
        mc.cfg.Logger.Print(err)
        return errBadConnNoWrite
    }

    // 设置 sql 指令类型
    data[4] = command

    // 拷贝 sql 指令
    copy(data[5:], arg)

    // 发送消息
    return mc.writePacket(data)
}
```

获取指定长度的字节切片：

```go
func (b *buffer) takeBuffer(length int) ([]byte, error) {
    if b.length > 0 {
        return nil, ErrBusyBuffer
    }

    // test (cheap) general case first
    if length <= cap(b.buf) {
        return b.buf[:length], nil
    }

    if length < maxPacketSize {
        b.buf = make([]byte, length)
        return b.buf, nil
    }

    // buffer is larger than we want to store.
    return make([]byte, length), nil
}
```

## **2.4 执行**

下面是通过 mysqlConn 执行操作类 sql 的流程，入口方法为 Exec 方法：

![https://pic1.zhimg.com/80/v2-1ab63d3a03206fb23cc2c10d0a4412d4_720w.webp](https://pic1.zhimg.com/80/v2-1ab63d3a03206fb23cc2c10d0a4412d4_720w.webp)

Exec 方法的入参中，query 为 sql 模板，args 为占位符及对应的参数. 对应的源码及注释展示如下：

```go
func (mc *mysqlConn) Exec(query string, args []driver.Value) (driver.Result, error) {
    // 连接已关闭？
    if mc.closed.Load() {        
        return nil, driver.ErrBadConn
    }
    if len(args) != 0 {
        if !mc.cfg.InterpolateParams {
            return nil, driver.ErrSkip
        }
        // 填充参数变量
        prepared, err := mc.interpolateParams(query, args)
        if err != nil {
            return nil, err
        }
        query = prepared
    }

    // 执行 sql 
    err := mc.exec(query)
    if err == nil {
        copied := mc.result
        return &copied, err
    }
    return nil, mc.markBadConn(err)
}
```

一次性读取数据，直到 EOF：

```go
// Reads Packets until EOF-Packet or an Error appears. Returns count of Packets read
func (mc *mysqlConn) readUntilEOF() error {
    for {
        data, err := mc.readPacket()
        if err != nil {
            return err
        }

        switch data[0] {
        case iERR:
            return mc.handleErrorPacket(data)
        case iEOF:
            if len(data) == 5 {
                mc.status = readStatus(data[3:])
            }
            return nil
        }
    }
}
```

## **3 预处理状态**

本章要介绍的内容是 **go-sql-driver/mysql** 中被广泛使用的预处理 prepare 模式.

**prepare 模式的本质是，通过 prepare 操作，将一份 sql 模板提前发往 mysql 服务端. 后续在该 sql 模板下的多笔操作，都只需要将对应的参数发往服务端**，即可实现对模板的复用.

prepare 模式的优势体现在：

- 模板复用：sql 模板一次编译，多次复用，可以节约性能
- **语法安全**： 模板和参数隔离，可以**有效防止 sql 注入的问题**
- 协议优化: **prepare 模式采用 binary protocol**，相比于**传统模式下的 text protocol** 更加节省 io，有更好的传输性能

## **3.1 预处理**

在 database/sql 库中定义的预处理状态 Stmt 接口规范如下：

```go
type Stmt interface {
    // 关闭预处理 statement
    Close() error

    // 查看 statement 中有多少个 args
    NumInput() int

    // 执行操作类请求
    Exec(args []Value) (Result, error)

    // 执行查询类请求
    Query(args []Value) (Rows, error)
}
```

**go-sql-driver/mysql** 库实现的 statment 类如下，对应的代码位于 statement.go 文件中：

```go
type mysqlStmt struct {
    // 关联的 mysql 连接
    mc         *mysqlConn
    // 预处理语句的标识 id
    id         uint32
    // 预处理状态中多少待填充参数
    paramCount int
}
```

prepare statement 是通过调用 mysqlConn 的 prepare 方法开启的，对应流程及源码如下：

![https://pic2.zhimg.com/80/v2-7b43f2a660db9bad53ec4314e2ba704d_720w.webp](https://pic2.zhimg.com/80/v2-7b43f2a660db9bad53ec4314e2ba704d_720w.webp)

```go
// 构造 prepare statemnt
func (mc *mysqlConn) Prepare(query string) (driver.Stmt, error) {
    // 连接已关闭
    if mc.closed.Load() {
        // ...
        return nil, driver.ErrBadConn
    }
    // 将 sql 模板发放 mysql 服务端
    err := mc.writeCommandPacketStr(comStmtPrepare, query)
    if err != nil {
        // ...
        return nil, driver.ErrBadConn
    }

    // 内置 mysql 连接，构造 statement 实例
    stmt := &mysqlStmt{
        mc: mc,
    }

    // 读取 prepare 请求的响应. 在该方法中会读取到该 statement 全局唯一的 id
    columnCount, err := stmt.readPrepareResultPacket()
    if err == nil {
        // 读取填充参数个数
        if stmt.paramCount > 0 {
            if err = mc.readUntilEOF(); err != nil {
                return nil, err
            }
        }

        // 读取列个数
        if columnCount > 0 {
            err = mc.readUntilEOF()
        }
    }

    return stmt, err
}
```

其他公开方法包括：

**关闭 statement**：

```go
// 关闭预处理状态
func (stmt *mysqlStmt) Close() error {
    if stmt.mc == nil || stmt.mc.closed.Load() {
        // ...
        return driver.ErrBadConn
    }

    // 发送 comStmtClose 类型的指令，传输 statement 的 id
    err := stmt.mc.writeCommandPacketUint32(comStmtClose, stmt.id)
    stmt.mc = nil
    return err
}
```

**返回参数个数：**

```go
func (stmt *mysqlStmt) NumInput() int {
    return stmt.paramCount
}
```

## **3.2 查询**

通过 statement 执行查询类请求时，只需传入参数即可，对应的流程如下：

![https://pic4.zhimg.com/80/v2-8676dfaf3b1a5b7fcd25220474b7638f_720w.webp](https://pic4.zhimg.com/80/v2-8676dfaf3b1a5b7fcd25220474b7638f_720w.webp)

源码及注释展示如下：

```go
func (stmt *mysqlStmt) Query(args []driver.Value) (driver.Rows, error) {
    return stmt.query(args)
}
```

```go
// 执行查询 sql 请求
func (stmt *mysqlStmt) query(args []driver.Value) (*binaryRows, error) {
    // mysql 连接关闭
    if stmt.mc.closed.Load() {
        // ...
        return nil, driver.ErrBadConn
    }
    // 发送执行语句
    err := stmt.writeExecutePacket(args)
    if err != nil {
        return nil, stmt.mc.markBadConn(err)
    }

    mc := stmt.mc
    // Read Result
    handleOk := stmt.mc.clearResult()
    // 读取响应，获取到的结果列的长度
    resLen, err := handleOk.readResultSetHeaderPacket()
    if err != nil {
        return nil, err
    }

    rows := new(binaryRows)

    // 读取列
    if resLen > 0 {
        rows.mc = mc
        rows.rs.columns, err = mc.readColumns(resLen)
    } else {
        rows.rs.done = true

        switch err := rows.NextResultSet(); err {
        case nil, io.EOF:
            return rows, nil
        default:
            return nil, err
        }
    }

    return rows, err
}
```

通过 statement 将参数发往服务端的流程是通过 writeExecutePacket 方法实现的，对应源码及注释如下：

![https://pic4.zhimg.com/80/v2-dbb403ec17fa9d978130029cc7237d2b_720w.webp](https://pic4.zhimg.com/80/v2-dbb403ec17fa9d978130029cc7237d2b_720w.webp)

源码及注释展示如下：

```go
func (stmt *mysqlStmt) Exec(args []driver.Value) (driver.Result, error) {
    // mysql 连接已关闭
    if stmt.mc.closed.Load() {
        // ...
        return nil, driver.ErrBadConn
    }
    // 发送 sql 到服务端执行
    err := stmt.writeExecutePacket(args)
    if err != nil {
        return nil, stmt.mc.markBadConn(err)
    }

    mc := stmt.mc
    handleOk := stmt.mc.clearResult()

    // 读取响应的列长度
    resLen, err := handleOk.readResultSetHeaderPacket()
    if err != nil {
        return nil, err
    }

    if resLen > 0 {
        // 读取列信息数据，填充到 conn buffer 中
        if err = mc.readUntilEOF(); err != nil {
            return nil, err
        }

        // 读取行数据，填充到 conn buffer 中
        if err := mc.readUntilEOF(); err != nil {
            return nil, err
        }
    }

    // 丢弃后续多余的内容
    if err := handleOk.discardResults(); err != nil {
        return nil, err
    }

    copied := mc.result
    return &copied, nil
}
```

## **4 事务**

接下来是 **go-sql-driver/mysql** 实现的事务模块. 这部分内容其实比较简单，**事务的核心功能都是通过 mysql 服务端提供的，客户端部分只需要将与事务有关的 tx、commit、rollback 等指令发往服务端**，并持有该连接实例即可.

## **4.1 开启事务**

database/sql 库中预定义的事务接口：

```go
type Tx interface {
    // 提交
    Commit() error
    // 回滚
    Rollback() error
}
```

**go-sql-driver/mysql** 库实现的事务类：

```go
type mysqlTx struct {
    mc *mysqlConn
}
```

事务的开启是 mysqlConn 的 Begin 方法实现的，该方法中通过将 **transaction 指令**发往服务端，**使用服务端的能力开启事务，并将该 conn 封装在 tx 实例中返回**：

```go
func (mc *mysqlConn) Begin() (driver.Tx, error) {
    return mc.begin(false)
}

func (mc *mysqlConn) begin(readOnly bool) (driver.Tx, error) {
    if mc.closed.Load() {
        mc.cfg.Logger.Print(ErrInvalidConn)
        return nil, driver.ErrBadConn
    }
    var q string
    if readOnly {
        q = "START TRANSACTION READ ONLY"
    } else {
        q = "START TRANSACTION"
    }
    err := mc.exec(q)
    if err == nil {
        return &mysqlTx{mc}, err
    }
    return nil, mc.markBadConn(err)
}
```

## **4.2 提交&回滚**

对应的提交事务和回滚事务的流程也都很简单，通过将 **commit 和 rollback 指令**借由 conn 的 exec 方法发放 mysql 服务端即可：

```go
func (tx *mysqlTx) Commit() (err error) {
    if tx.mc == nil || tx.mc.closed.Load() {
        return ErrInvalidConn
    }
    err = tx.mc.exec("COMMIT")
    tx.mc = nil
    return
}
```

```go
func (tx *mysqlTx) Rollback() (err error) {
    if tx.mc == nil || tx.mc.closed.Load() {
        return ErrInvalidConn
    }
    err = tx.mc.exec("ROLLBACK")
    tx.mc = nil
    return
}
```

## **5 执行结果**

最后要聊的是执行结果的部分.

## **5.1 查询类**

首先是对应于**查询类请求**的响应格式——**Rows，其本质是对应了多行数据的数据流**. database/sql 库中定义的接口标准如下：

```go
type Rows interface {
    // 返回所有列明
    Columns() []string

    // 关闭 rows 
    Close() error

    // 将下一行数据加载到 dest 中
    Next(dest []Value) error
}
```

在 **go-sql-driver/mysql** 库中实现的 rows 类如下：

```go
type mysqlRows struct {
    // mysql 数据库连接
    mc     *mysqlConn
    // 结果集，包含了各列的信息 
    rs     resultSet
    finish func()
}
```

结果集，包含了各列的信息 ：

```go
type resultSet struct {
    columns     []mysqlField
    columnNames []string
    done        bool
}
```

列字段信息：

```go
type mysqlField struct {
    tableName string
    name      string
    length    uint32
    flags     fieldFlag
    fieldType fieldType
    decimals  byte
    charSet   uint8
}
```

在 mysqlRows 实现的 Columns 方法中，直接从 rows 中内置的 resultSet 中读取列信息即可. 此前列信息数据已经通过 readColumns 方法预加载到 resultSet 中了：

```go
func (rows *mysqlRows) Columns() []string {
    if rows.rs.columnNames != nil {
        return rows.rs.columnNames
    }

    columns := make([]string, len(rows.rs.columns))
    if rows.mc != nil && rows.mc.cfg.ColumnsWithAlias {
        for i := range columns {
            if tableName := rows.rs.columns[i].tableName; len(tableName) > 0 {
                columns[i] = tableName + "." + rows.rs.columns[i].name
            } else {
                columns[i] = rows.rs.columns[i].name
            }
        }
    } else {
        for i := range columns {
            columns[i] = rows.rs.columns[i].name
        }
    }

    rows.rs.columnNames = columns
    return columns
}
```

关闭 mysqlRows 的核心是移除其中内置的 mysqlConn：

```go
func (rows *mysqlRows) Close() (err error) {
    if f := rows.finish; f != nil {
        f()
        rows.finish = nil
    }

    mc := rows.mc
    if mc == nil {
        return nil
    }
    if err := mc.error(); err != nil {
        return err
    }

    // ...
    // Remove unread packets from stream
    if !rows.rs.done {
        err = mc.readUntilEOF()
    }
    if err == nil {
        handleOk := mc.clearResult()
        if err = handleOk.discardResults(); err != nil {
            return err
        }
    }

    rows.mc = nil
    return err
}
```

有关于 Rows 的 Next 遍历方法，分别在 mysqlRows 的两个子类予以实现，分为 textRows 和 binaryRows 两种类型：

```
// prepare 模式的查询请求响应格式
type binaryRows struct {
    mysqlRows
}

// 普通的模式的查询请求响应格式
type textRows struct {
    mysqlRows
}
```

- textRows：普通模式下的查询请求使用的响应格式
- binaryRows：prepare statement 模式下的查询请求使用的响应格式

textRows 和 binaryRows 所实现的 Next 方法流程都是类似的，主要通过 readRow 方法读取一行数据解析到 dest 中，区别在于两者的解析协议不同（text protocol 和 binary protocol）.这部分内容本文不作展开，大家感兴趣可以自行深入学习：

```go
// http://dev.mysql.com/doc/internals/en/com-query-response.html#packet-ProtocolText::ResultsetRow
func (rows *textRows) readRow(dest []driver.Value) error {
    // ...

    return nil
}
```

```go
// http://dev.mysql.com/doc/internals/en/binary-protocol-resultset-row.html
func (rows *binaryRows) readRow(dest []driver.Value) error {
    // ...
    return nil
}
```

## **5.2 操作类**

mysqlResult 是用于操作类请求的响应格式.

database/sql 库中预定的接口规范如下：

```go
type Result interface {
    // 最后一笔插入的主键 id
    LastInsertId() (int64, error)

    // 影响的行数
    RowsAffected() (int64, error)
}
```

对应在 **go-sql-driver/mysql** 库中的实现为：

```go
type mysqlResult struct {
    affectedRows []int64
    insertIds    []int64
}

func (res *mysqlResult) LastInsertId() (int64, error) {
    return res.insertIds[len(res.insertIds)-1], nil
}

func (res *mysqlResult) RowsAffected() (int64, error) {
    return res.affectedRows[len(res.affectedRows)-1], nil
}
```

reference

[https://zhuanlan.zhihu.com/p/661304021](https://zhuanlan.zhihu.com/p/661304021)