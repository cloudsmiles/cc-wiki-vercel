# HTTP请求方法

# 标准请求方法

目前 HTTP/1.1 规定了八种方法，单词都必须是大写的形式

1. GET：获取资源，可以理解为读取或者下载数据；
2. HEAD：获取资源的元信息；
3. POST：向资源提交数据，相当于写入或上传数据；
4. PUT：类似 POST；
5. DELETE：删除资源；
6. CONNECT：建立特殊的连接隧道；
7. OPTIONS：列出可对资源实行的方法；
8. TRACE：追踪请求 - 响应的传输路径。

## GET/HEAD

它的含义是请求从服务器获取资源，这个资源既可以是静态的文本、页面、图片、视频，也可以是由 PHP、Java 动态生成的页面或者其他格式的数据。

GET 方法虽然基本动作比较简单，但搭配 URI 和其他头字段就能实现对资源更精细的操作。

例如，在 URI 后使用“#”，就可以在获取页面后直接定位到某个标签所在的位置；使用 If-Modified-Since 字段就变成了“有条件的请求”，仅当资源被修改时才会执行获取动作；使用 Range 字段就是“范围请求”，只获取资源的一部分数据。

## POST/PUT

GET 和 HEAD 方法是从服务器获取数据，而 POST 和 PUT 方法则是相反操作，向 URI 指定的资源提交数据，数据就放在报文的 body 里。

POST 也是一个经常用到的请求方法，使用频率应该是仅次于 GET，应用的场景也非常多，只要向服务器发送数据，用的大多数都是 POST。

PUT 的作用与 POST 类似，也可以向服务器提交数据，但与 POST 存在微妙的不同，通常 POST 表示的是“新建”“create”的含义，而 PUT 则是“修改”“update”的含义。

## OPTIONS

它用于获取当前URL所支持的方法。若请求成功，则它会在HTTP响应头部中带上给各种“Allow”的头，表明某个请求在对应的服务器中都支持哪种请求方法。比如下图：

[https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnlAaoSXY0YM0UFgz62fPnrEcPHg8kxmK0OWwicbic1jduvicafhOXgHhcmAsLntSULkNahoU6vrKDyRQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnlAaoSXY0YM0UFgz62fPnrEcPHg8kxmK0OWwicbic1jduvicafhOXgHhcmAsLntSULkNahoU6vrKDyRQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这里面需要关注的点有两个

- Request Header里的关键字段
    
    [https://mmbiz.qpic.cn/mmbiz_jpg/FmVWPHrDdnlAaoSXY0YM0UFgz62fPnrEUicpNnu4UKJptMXMDM4etg3x2zffQicpk5uSGC2uK1gOTb9bmpQibN6dw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_jpg/FmVWPHrDdnlAaoSXY0YM0UFgz62fPnrEUicpNnu4UKJptMXMDM4etg3x2zffQicpk5uSGC2uK1gOTb9bmpQibN6dw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)
    
- Response Header里的关键字段
    
    [https://mmbiz.qpic.cn/mmbiz_jpg/FmVWPHrDdnlAaoSXY0YM0UFgz62fPnrEn0mxNyyCMvUqQwAJoxHLN6rR022bZIOg09RSISGLiahtj19JRhxDCzA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_jpg/FmVWPHrDdnlAaoSXY0YM0UFgz62fPnrEn0mxNyyCMvUqQwAJoxHLN6rR022bZIOg09RSISGLiahtj19JRhxDCzA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)
    

`Options`堪称是网络协议中的老实人，就好像老实人刚谈了个女朋友，每次牵手前都要问下人家 “我可以牵你的手吗？”， “我可以抱你吗？”，得到了答应后才会下手。差点被这老实人气质感动得留下了不争气的泪水。

### **什么时候需要使用options**

在**跨域**（记住这个词，待会解释）的情况下，浏览器发起**复杂请求前**会**自动**发起 options 请求。跨域共享标准规范要求，对那些可能对服务器数据产生副作用的 HTTP 请求方法（特别是 GET 以外的 HTTP 请求，或者搭配某些 MIME 类型的 POST 请求），浏览器必须首先使用 options 方法发起一个预检请求，从而获知服务端是否允许该跨域请求。服务器确认允许之后，才发起实际的 HTTP 请求。

### **什么是简单请求和复杂请求。**

某些请求不会触发 CORS 预检请求，这样的请求一般称为"简单请求"，而会触发预检的请求则为"复杂请求"。

1.简单请求

- 请求方法为`GET、HEAD、POST`
- 只有以下`Headers`字段
    - `Accept`
    - `Accept-Language`
    - `Content-Language`
    - `Content-Type`
    - `DPR/Downlink/Save-Data/Viewport-Width/Width` (这些不常见，放在一起)
- `Content-Type` 只有以下三种
    - `application/x-www-form-urlencoded`
    - `multipart/form-data`
    - `text/plain`
- 请求中的任意 XMLHttpRequestUpload 对象均没有注册任何事件监听器；
- 请求中没有使用 ReadableStream 对象。

2.复杂请求

- 不满足简单请求的，都是复杂请求

由此可见，因为上述请求在获取B站资源的请求Headers里带有 `Access-Control-Request-Headers: range` , 而`range`正好不在简单请求的条件2中提到的Headers范围里，因此属于**复杂请求**，于是触发预检options请求。

### **options带来什么问题**

由此可见，复杂请求的条件其实非常容易满足，而一旦满足复杂请求的条件，则浏览器便会发送2次请求（一次预检options，一次复杂请求），这一次options就一来一回（一个RTT），显然会导致延迟和不必要的网络资源浪费，高并发情况下则可能为服务器带来严重的性能消耗。

### **如何优化options**

每次复杂请求前都会调用一次options，这其实非常没有必要。因为大部分时候相同的请求，短时间内获得的结果是不会变的，是否可以通过浏览器缓存省掉这一次查询？

`Access-Control-Max-Age`就是优化这个流程中使用的一个Header。它的作用是当你每次请求`options`方法时，服务端返回调用支持的方法（Access-Control-Allow-Methods ）和Headers（Access-Control-Allow-Headers）有哪些，同时告诉你，它在接下来 `Access-Control-Max-Age`时间（单位是秒）里都支持，则这段时间内，不再需要使用options进行请求。特别注意的是，当`Access-Control-Max-Age`的值为-1时，表示禁用缓存，每一次请求都需要发送预检请求，即用OPTIONS请求进行检测。

## 安全与幂等

所谓的“幂等”实际上是一个数学用语，被借用到了 HTTP 协议里，**意思是多次执行相同的操作，结果也都是相同的，即多次“幂”后结果“相等”。**

很显然，GET 和 HEAD 既是安全的也是幂等的，DELETE 可以多次删除同一个资源，效果都是“资源不存在”，所以也是幂等的。POST 和 PUT 的幂等性质就略费解一点。

按照 RFC 里的语义，POST 是“新增或提交数据”，多次提交数据会创建多个资源，所以不是幂等的；而 PUT 是“替换或更新数据”，多次更新一个资源，资源还是会第一次更新的状态，所以是幂等的。

我对你的建议是，你可以对比一下 SQL 来加深理解：把 POST 理解成 INSERT，把 PUT 理解成 UPDATE，这样就很清楚了。多次 INSERT 会添加多条记录，而多次 UPDATE 只操作一条记录，而且效果相同。

参考

[https://time.geekbang.org/column/article/101518](https://time.geekbang.org/column/article/101518)