# DNS查询过程

想要理解DNS，首先就需要熟悉DNS的查询过程。DNS的查询按一定顺序进行，那就是host文件->DNS缓存->DNS服务器。

当DNS接收到域名解析请求时，会首先去检查本机的host文件（一般Linux和Macos系统位于`etc/hosts`，Windows系统位于`C:\Windows\System32\drivers\etc\hosts`），host文件存储的是域名与IP的键值对。

当host文件没有解析结果时，下一个查找的位置就是DNS缓存，DNS解析结果的缓存时间会根据不同的系统而不同。

当上面两个都无法返回解析结果时，DNS就会向DNS服务器查询结果。

根据上面的不同查询方法，DNS也分为内部DNS查询和外部DNS查询，从host文件或DNS缓存中查询出结果就是内部DNS查询，反之就是外部DNS查询。

### 递归查询

一个常见的面试题是，说一说从浏览器输入一个网址到返回页面这个过程中发生了哪些事。这里我们来重点看看这一过程中的DNS查找，之后的TCP连接不作详细讨论。

这个面试题答案第一步就是DNS查找，查找顺序如上所述，而当DNS开始外部查询时，又会发生什么事呢？

稍微了解一点DNS知识后就会知道，DNS服务器是一种树状架构，每个DNS服务器只负责自己zone内的域名解析，而这个树的根就是根DNS服务器。根DNS服务器用.代替，一般域名中是省略的，比如`www.baidu.com`的域名全貌是`www.baidu.com.`。

> 全世界有13台根DNS服务器，从a.root-servers.org到m.root-servers.org，可以在[root-servers.org/](https://link.juejin.cn/?target=https%3A%2F%2Froot-servers.org%2F) 这个网址看到分布的详细信息
> 

根DNS服务器只负责一些权威DNS服务器的IP解析，比如`.com`、`.cn`、`.org`等，所以让查询看上去挺费劲的，我明明问的是`www.baidu.com`的IP地址，你却不告诉我，让我去找`.com`权威DNS服务器。当然这样设计肯定是有原因的，DNS的特性就决定了架构一定是分布式的，也只有这样的树状架构才能满足海量的查询请求。

接下来就是递归查询的过程了，`.com`权威DNS服务器也不知道IP地址，却可以告诉请求你应该去问`baidu.com`的DNS服务器，这样流量就进入了百度的网络中。

整个过程如下图所示：

![Untitled](RPG从零开始/Web/DNS/DNS查询过程/Untitled.png)

### 迭代查询

这里有一个问题，DNS一定是递归查询吗？这当然不是的，实际上DNS还有一种查询方式，就是迭代查询，其实与递归查询差不多，区别在于递归查询是本地DNS服务器来依次去请求根DNS服务器、权威DNS服务器以及其他DNS服务器，而迭代查询则是由客户端来完成的。本地DNS服务器返回结果给客户端后，就不再参与查询了，由客户端去依次请求根DNS服务器、权威DNS服务器以及其他服务器。

一般情况下，为了减少资源的消耗，网络中客户端与所属的本地DNS服务器查询方式通常为递归查询，本地DNS服务器与外部的公共DNS服务器间的查询方式为迭代查询。

reference

[https://juejin.cn/post/6995465840732684295#heading-8](https://juejin.cn/post/6995465840732684295#heading-8)