# HTTP是什么

# HTTP是什么

HTTP (HyperText Transfer Protocol)是一种用于传输超文本的协议。它是Web的基础，允许客户端和服务器之间进行通信。HTTP协议使用TCP/IP协议来传输数据，它被设计用于Web浏览器和Web服务器之间的通信。

HTTP协议定义了客户端和服务器之间交换的消息格式和规则。当客户端发送HTTP请求到服务器时，服务器会用HTTP响应来回应请求。HTTP请求通常由一个方法、一个URL和一个HTTP版本组成。常用的HTTP方法有GET、POST、PUT和DELETE。

GET方法用于获取资源，而POST方法用于提交数据。PUT方法用于更新资源，而DELETE方法用于删除资源。HTTP还定义了其他方法，例如OPTIONS和HEAD，但这些方法不太常用。

除了请求方法和URL之外，HTTP消息还包括一个标头和一个消息体。标头包含有关消息的元数据，例如内容类型和内容长度。消息体包含实际的数据。

HTTP消息还可以使用Cookie和Session来跟踪客户端状态。Cookie是一种在客户端存储的数据，它可以在HTTP请求中发送给服务器。Session是一种在服务器端存储的数据，它可以在多个HTTP请求之间保存客户端状态。

总之，HTTP是一种用于传输超文本的协议，它是Web的基础。它定义了客户端和服务器之间交换的消息格式和规则，包括请求方法、URL、标头、消息体、Cookie和Session等内容。

## HTTP不是什么

- HTTP 不是编程语言。
    
    编程语言是人与计算机沟通交流所使用的语言，而 HTTP 是计算机与计算机沟通交流的语言，我们无法使用 HTTP 来编程，但可以反过来，用编程语言去实现 HTTP，告诉计算机如何用 HTTP 来与外界通信。很多流行的编程语言都支持编写 HTTP 相关的服务或应用，例如使用 Java 在 Tomcat 里编写 Web 服务，使用 PHP 在后端实现页面模板渲染，使用 JavaScript 在前端实现动态页面更新，你是否也会其中的一两种呢？
    
- HTTP 不是 HTML，
    
    这个可能要特别强调一下，千万不要把 HTTP 与 HTML 混为一谈，虽然这两者经常是同时出现。HTML 是超文本的载体，是一种标记语言，使用各种标签描述文字、图片、超链接等资源，并且可以嵌入 CSS、JavaScript 等技术实现复杂的动态效果。单论次数，在互联网上 HTTP 传输最多的可能就是 HTML，但要是论数据量，HTML 可能要往后排了，图片、音频、视频这些类型的资源显然更大。
    
- HTTP 不是一个孤立的协议。
    
    俗话说“一个好汉三个帮”，HTTP 也是如此。在互联网世界里，HTTP 通常跑在 TCP/IP 协议栈之上，依靠 IP 协议实现寻址和路由、TCP 协议实现可靠数据传输、DNS 协议实现域名查找、SSL/TLS 协议实现安全通信。此外，还有一些协议依赖于 HTTP，例如 WebSocket、HTTPDNS 等。这些协议相互交织，构成了一个协议网，而 HTTP 则处于中心地位。
    

参考 

[https://time.geekbang.org/column/article/98128](https://time.geekbang.org/column/article/98128)