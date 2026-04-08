# DNS

# **DNS 查找有哪些步骤？**

大多数情况下，DNS 与正被转换为相应 IP 地址的域名有关。要了解此过程的工作方式，在 DNS 查找从 Web 浏览器经过 DNS 查找过程然后再返回时，跟踪 DNS 查找的路径会有所帮助。我们来看一下这些步骤。

注意：通常，DNS 查找信息将本地缓存在查询计算机内，或者远程缓存在 DNS 基础设施内。DNS 查找通常有 8 个步骤。缓存 DNS 信息时，将从 DNS 查找过程中跳过一些步骤，从而使该过程更快。以下示例概述了不缓存任何内容时的所有 8 个步骤。

## **DNS 查找的 8 个步骤：**

1. 用户在 Web 浏览器中键入 “example.com”，查询传输到 Internet 中，并被 DNS 递归解析器接收。
2. 接着，解析器查询 DNS 根域名服务器（.）。
3. 然后，根服务器使用存储其域信息的顶级域（TLD）DNS 服务器（例如 .com 或 .net）的地址响应该解析器。在搜索 example.com 时，我们的请求指向 .com TLD。
4. 然后，解析器向 .com TLD 发出请求。
5. TLD 服务器随后使用该域的域名服务器 example.com 的 IP 地址进行响应。
6. 最后，递归解析器将查询发送到域的域名服务器。
7. example.com 的 IP 地址而后从域名服务器返回解析器。
8. 然后 DNS 解析器使用最初请求的域的 IP 地址响应 Web 浏览器。

DNS 查找的这 8 个步骤返回 example.com 的 IP 地址后，浏览器便能发出对该网页的请求：

1. 浏览器向该 IP 地址发出 [HTTP](https://www.cloudflare.com/learning/ddos/glossary/hypertext-transfer-protocol-http/) 请求。
2. 位于该 IP 的服务器返回将在浏览器中呈现的网页（第 10 步）。

![Untitled](RPG从零开始/Web/DNS/Untitled.png)

# **什么是 DNS 高速缓存？DNS 高速缓存发生在哪里？**

缓存的目的是将数据临时存储在某个位置，从而提高数据请求的性能和可靠性。DNS 高速缓存涉及将数据存储在更靠近请求客户端的位置，以便能够更早地解析 DNS 查询，并且能够避免在 DNS 查找链中进一步向下的额外查询，从而缩短加载时间并减少带宽/CPU 消耗。DNS 数据可缓存到各种不同的位置上，每个位置均将存储 DNS 记录并保存由[生存时间（TTL）](https://www.cloudflare.com/learning/cdn/glossary/time-to-live-ttl/)决定的一段时间。

## **浏览器 DNS 缓存**

现代 Web 浏览器设计为默认将 DNS 记录缓存一段时间。目的很明显；越靠近 Web 浏览器进行 DNS 缓存，为检查缓存并向 IP 地址发出正确请求而必须采取的处理步骤就越少。发出对 DNS 记录的请求时，浏览器缓存是针对所请求的记录而检查的第一个位置。

在 Chrome 浏览器中，您可以转到 chrome://net-internals/#dns 查看 DNS 缓存的状态。

## **操作系统（OS）级 DNS 缓存**

操作系统级 DNS 解析器是 DNS 查询离开您计算机前的第二站，也是本地最后一站。操作系统内旨在处理此查询的过程通常称为“存根解析器”或 DNS 客户端。当存根解析器获取来自某个应用程序的请求时，其首先检查自己的缓存，以便查看是否有此记录。如果没有，则将本地网络外部的 DNS 查询（设置了递归标记）发送到 Internet 服务提供商（ISP）内部的 DNS 递归解析器。

# DNS的应用方式

第一种，重定向，当主机有需要切换，下线迁移时，可以更改DNS记录，让域名指向其他的机器

第二种，内部使用域名，可以不用写死IP地址

第三种，因为域名解析可以返回多个IP地址，所以一个域名可以对应多台主机，客户端收到多个IP地址后，就可以根据轮询请求，实现负载均衡

第四种，域名解析可以配置内部的策略，返回离客户端最近的主机，或者返回当前服务质量最好的主机，这样在 DNS 端把请求分发到不同的服务器，实现负载均衡。

参考

[https://time.geekbang.org/column/article/99665](https://time.geekbang.org/column/article/99665)

[https://www.cloudflare.com/zh-cn/learning/dns/what-is-dns/](https://www.cloudflare.com/zh-cn/learning/dns/what-is-dns/)

[[DNS是什么]]

[[DNS如何工作]]

[[DNS查询过程]]

[[DNS消息报文]]

[[实践—-dig命令&go实现dns服务器]]