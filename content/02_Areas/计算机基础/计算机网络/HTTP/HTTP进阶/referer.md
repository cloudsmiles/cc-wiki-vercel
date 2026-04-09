# referer

### **Referrer Policy 和 Referrer**

[https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnlAaoSXY0YM0UFgz62fPnrETULWrTtorwZiauv1NhQ1roIQsHMZRicfYrDZOqp0JqRgHXpqFcXUugCg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnlAaoSXY0YM0UFgz62fPnrETULWrTtorwZiauv1NhQ1roIQsHMZRicfYrDZOqp0JqRgHXpqFcXUugCg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

### **Referrer是什么**

Referrer 是HTTP请求header的报文头，用于指明当前流量的来源参考页面，常被用于分析用户来源等信息。通过这个信息，我们可以知道访客是怎么来到当前页面的。比如在上面的请求截图里，可以看出我是使用`https://www.bilibili.com/`访问的视频资源。

### **Referrer Policy 是什么**

- Referrer 字段，会用来指定该请求是从哪个页面跳转页来的，里面的信息是浏览器填的。
- 而 Referrer Policy 则是用于控制Referrer信息传不传、传哪些信息、在什么时候传的策略。

为什么要这么麻烦呢？因为有些网站一些用户敏感信息，比如 sessionid 或是 token 放在地址栏里，如果当做Referrer字段全部传递的话，那第三方网站就会拿到这些信息，会有一定的安全隐患。所以就有了 Referrer Policy，用于过滤 Referrer 报头内容。

比如在上面的请求截图里，可以看出我是使用`strict-origin-when-cross-origin`策略，含义是跨域时将当前页面URL过滤掉参数及路径部分，仅将协议、域名和端口（如果有的话）当作 Referrer。否则 Referrer 还是传递当前页的全路径。同时当发生降级（比如从 https:// 跳转到 http:// ）时，不传递 Referrer 报头。

reference

[https://mp.weixin.qq.com/s/wNRoDoW_VEqiq8JelePj2g](https://mp.weixin.qq.com/s/wNRoDoW_VEqiq8JelePj2g)