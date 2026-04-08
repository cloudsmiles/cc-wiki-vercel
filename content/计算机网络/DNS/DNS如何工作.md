# DNS如何工作

DNS 解析过程涉及将主机名（例如 [www.example.com](http://www.example.com/)）转换为计算机友好的 IP 地址（例如 192.168.1.1）。Internet 上的每个设备都被分配了一个 IP 地址，必须有该地址才能找到相应的 Internet 设备 - 就像使用街道地址来查找特定住所一样。当用户想要加载网页时，用户在 Web 浏览器中键入的内容（[example.com](http://example.com/)）与查找 [example.com](http://example.com/) 网页所需的机器友好地址之间必须进行转换。

为理解 DNS 解析过程，务必了解 DNS 查询必须通过的各种硬件设备。对于 Web 浏览器而言，DNS 查询是“在幕后”发生的，除了初始请求外，不需要从用户的计算机进行任何交互。

# 域名结构

在了解DNS服务器之前，需要先了解因特网上的域名空间结构，具体如下图所示：

![https://pic2.zhimg.com/80/v2-59cf070e13da779b42fa6780bdbece01_1440w.webp](https://pic2.zhimg.com/80/v2-59cf070e13da779b42fa6780bdbece01_1440w.webp)

顶级域名是域名的最后一个部分，即是域名最后一点之后的字母，例如在http://example.com这个域名中，顶级域是.com（或.COM），大小写视为相同。

二级域名是域名的倒数第二个部分，例如在http://example.com这个域名中，二级域名是example。以此类推。

# **加载网页涉及 4 个 DNS 服务器：**

- **[DNS 解析器](https://www.cloudflare.com/learning/dns/dns-server-types/)** - 该解析器可被视为被要求去图书馆的某个地方查找特定图书的图书馆员。DNS 解析器是一种服务器，旨在通过 Web 浏览器等应用程序接收客户端计算机的查询。然后，解析器一般负责发出其他请求，以便满足客户端的 DNS 查询。
- **根域名服务器** - [根域名服务器](https://www.cloudflare.com/learning/dns/glossary/dns-root-server/)是将人类可读的主机名转换（解析）为 IP 地址的第一步。可将其视为指向不同书架的图书馆中的索引 - 一般其作为对其他更具体位置的引用。
- **[TLD 名称服务器](https://www.cloudflare.com/learning/dns/dns-server-types/)** —— 顶级域名服务器（[TLD](https://www.cloudflare.com/learning/dns/top-level-domain/)）可看做是图书馆中一个特殊的书架。这个域名服务器是搜索特定 IP 地址的下一步，其上托管了主机名的最后一部分（例如，在 example.com 中，TLD 服务器为 “com”）。
- **[权威性域名服务器](https://www.cloudflare.com/learning/dns/dns-server-types/)** - 可将这个最终域名服务器视为书架上的字典，其中特定名称可被转换成其定义。权威性域名服务器是域名服务器查询中的最后一站。如果权威性域名服务器能够访问请求的记录，则其会将已请求主机名的 IP 地址返回到发出初始请求的 DNS 解析器（图书管理员）。

# **权威性 DNS 服务器与递归 DNS 解析器之间的区别是什么？**

这两个概念都是指 DNS 基础设施不可或缺的服务器（服务器组），但各自担当不同的角色，并且位于 DNS 查询管道内的不同位置。考虑二者差异的一种方式是，[递归](https://www.cloudflare.com/learning/dns/what-is-recursive-dns/)解析器位于 DNS 查询的开头，而权威性域名服务器位于末尾。

## **递归 DNS 解析器**

递归解析器是一种计算机，其响应来自客户端的递归请求并花时间追踪 [DNS 记录](https://www.cloudflare.com/learning/dns/dns-records/)。为执行此操作，其发出一系列请求，直至到达用于所请求的记录的权威性 DNS 域名服务器为止（或者超时，或者如果未找到记录，则返回错误）。幸运的是，递归 DNS 解析器并不总是需要发出多个请求才能追踪响应客户端所需的记录；[缓存](https://www.cloudflare.com/learning/cdn/what-is-caching/)是一种数据持久性过程，可通过在 DNS 查找中更早地服务于所请求的资源记录来为所需的请求提供捷径。

![Untitled](RPG从零开始/Web/DNS/DNS如何工作/Untitled.png)

## **权威性 DNS 服务器**

简言之，权威性 DNS 服务器是实际持有并负责 DNS 资源记录的服务器。这是位于 DNS 查找链底部的服务器，其将使用所查询的资源记录进行响应，从而最终允许发出请求的 Web 浏览器达到访问网站或其他 Web 资源所需的 IP 地址。权威性域名服务器从自身数据满足查询需求，无需查询其他来源，因为这是某些 DNS 记录的最终真实来源。

![https://cf-assets.www.cloudflare.com/slt3lc6tev37/6Cxvsc4NOvmU4pPkKbkDmP/a7588a4c8a3c187e9175a40fa1b3d548/dns_record_request_sequence_authoritative_nameserver.png](https://cf-assets.www.cloudflare.com/slt3lc6tev37/6Cxvsc4NOvmU4pPkKbkDmP/a7588a4c8a3c187e9175a40fa1b3d548/dns_record_request_sequence_authoritative_nameserver.png)

值得一提的是，在查询对象为子域（例如 foo.example.com 或 [blog.cloudflare.com](https://blog.cloudflare.com/)）的情况下，将向权威性域名服务器之后的序列添加一个附加域名服务器，其负责存储该子域的 [CNAME 记录](https://www.cloudflare.com/learning/dns/dns-records/dns-cname-record/)。

![Untitled](RPG从零开始/Web/DNS/DNS如何工作/Untitled%201.png)

reference

[https://zhuanlan.zhihu.com/p/38282917](https://zhuanlan.zhihu.com/p/38282917)

[[[[TLD(top-level-domain)]]]]