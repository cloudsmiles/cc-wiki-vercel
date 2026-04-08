# DNS消息报文

与其他的计算机网络协议相似，DNS协议也有自己的报文格式。

DNS的请求和响应的基本单位是DNS报文。请求和响应的DNS报文结构是完全相同的，每个报文都由以下三段（Section）构成：

[https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8d10571477649e99f58f036341619bd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8d10571477649e99f58f036341619bd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

### Header

DNS头部类似与TCP和UDP协议，也定义了一系列与DNS请求或响应有关的字段，具体结构如下：

[https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/826e747a59274c648fd5be04ec8faf7c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/826e747a59274c648fd5be04ec8faf7c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **ID**：由生成任何类型查询的程序分配的16位标识符。这个标识符被复制到相应的回复中，请求者可以使用它来匹配对未完成查询的回复。即：ID将由发起查询查找的DNS客户端生成，当响应到来时，可以使用ID将响应映射到查询
- **QR**：0表示查询报文，1表示响应报文
- **OPCODE**：一个4位字段，用于指定此消息中的查询类型。该值由查询的发起者设置并复制到响应中。例如：标准查询或反向查询
- **AA**：权威答案，该位在响应中有效，也表示该服务器是DNS请求的权威服务器
- **TC**：代表报文可截断
- **RD**：表示建议域名服务器进行递归解析
- **RA**：表示支持递归
- **Z**：保留以备将来使用
- **RCODE**：响应代码，4位字段设置为响应的一部分
RCODE | | REFERENCE |
| :---: | ---------------------------------------------------------- | ------------------------------------------- |
| 0 | 没有错误。 | [[RFC1035](https://link.juejin.cn/?target=http%3A%2F%2Fwww.iana.org%2Fgo%2Frfc1035)] |
| 1 | Format error：格式错误，服务器不能理解请求的报文格式。 | [[RFC1035](https://link.juejin.cn/?target=http%3A%2F%2Fwww.iana.org%2Fgo%2Frfc1035)] |
| 2 | Server failure：服务器失败，因为服务器的原因导致没办法处理这个请求。 | [[RFC1035](https://link.juejin.cn/?target=http%3A%2F%2Fwww.iana.org%2Fgo%2Frfc1035)] |
| 3 | Name Error：名字错误，该值只对权威应答有意义，它表示请求的域名不存在。 | [[RFC1035](https://link.juejin.cn/?target=http%3A%2F%2Fwww.iana.org%2Fgo%2Frfc1035)] |
| 4 | Not Implemented：未实现，域名服务器不支持该查询类型。 | [[RFC1035](https://link.juejin.cn/?target=http%3A%2F%2Fwww.iana.org%2Fgo%2Frfc1035)] |
| 5 | Refused：拒绝服务，服务器由于设置的策略拒绝给出应答。比如，服务器不希望对个请求者给出应答时可以使用此响应码。 | [[RFC1035](https://link.juejin.cn/?target=http%3A%2F%2Fwww.iana.org%2Fgo%2Frfc1035)]

**QDCOUNT**、**ANCOUNT**、**NSCOUNT**、**ARCOUNT**为无符号16bit整数，分别表示报文请求段、回答段、授权段以及附加段中记录数。

### Question

[https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b24675287eda4673944b90d77e5f57d2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b24675287eda4673944b90d77e5f57d2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **QNAME**：该字段包含我们希望解析的域名。

> 域名将表示为一系列标签。每个标签表示为一个八位字节长度字段，后跟该八位字节数。域名以根的空标签的零长度八位字节结束。例如：“example.com”，首先“example.com”由“example”和“com”两部分组成。然后“example”和“com”将分别被URL编码为“69 88 65 77 80 76 69”和“99 111 109”。这将被称为标签。标签前面将有一个整数字节，其中包含该段中的字节数，即：“example”编码成“7 69 88 65 77 80 76 69”。标签中的每个值都可以转换为单个八位字节值，即：(7) (69) (88)：00000111 01000101 01011000......最终数据可以放在QNAME问题部分。
> 
- **QTYPE**：这个2bit值指定了Query的类型
- **QCLASS**：指定Query的类别，主要是IN(internet)

### DNS Answers、Additional和Authority

Answers、Additional和Authority部分都共享相同的资源记录格式：

[https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1288de031540492d81846accc5919761~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1288de031540492d81846accc5919761~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- **TYPE**：指定查询的类型。例如：标准或反向查询
- **TTL**：这个32位字段表示资源记录（答案）可以被缓存的时间量，零值表示不应缓存资源记录。
- **RDLENGTH**：Response Data Length，16位字段表示Response Data字段中八位字节的长度。
- **RDATA**：描述资源的可变长度八位字节串。此信息的格式根据资源记录的TYPE和CLASS而有所不同。例如，如果TYPE是A(IPv4)并且CLASS是IN，RDATA字段是一个4个八位字节的ARPA Internet地址。

关于TYPE，这里简单列举一些DNS记录类型：

- A记录：记录域名对应的IPv4地址
- AAAA记录：记录域名对应的IPv6地址
- CNAME记录：将域名记录指向另一个域名记录
- NS记录：指定域名由哪个DNS服务器来解析
- PTR记录：A记录的反向解析，将IP映射到对应的域名
- MX记录：将域名记录指向邮件服务器
- TXT记录：某个主机名或域名的说明
- SOA记录：指定多个NS记录中的主服务器
- SRV记录：服务器资源记录

[[DNS记录类型]]