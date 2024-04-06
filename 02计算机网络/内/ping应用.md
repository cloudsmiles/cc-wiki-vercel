---
up:
  - "[[00计算机网络知识目录]]"
---

断网的时候，在控制台输入ping 127.0.0.1

```jsx
$ ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.080 ms
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.093 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.074 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.079 ms
64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.079 ms
^C
--- 127.0.0.1 ping statistics ---
5 packets transmitted, 5 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.074/0.081/0.093/0.006 ms
```

说明拔了网线，ping 127.0.0.1是能ping通的，这是为什么呢？

## 什么是127.0.0.1

首先，这是个 `IPV4` 地址

`IPV4` 地址有 `32` 位，一个字节有 `8` 位，共 `4` 个字节。

其中**127 开头的都属于回环地址**，也是 `IPV4` 的特殊地址，没什么道理，就是人为规定的。

而`127.0.0.1`是**众多**回环地址中的一个。之所以不是 `127.0.0.2` ，而是 `127.0.0.1`，是因为源码里就是这么定义的，也没什么道理。

```jsx
/* Address to loopback in software to local host.  */
#define    INADDR_LOOPBACK     0x7f000001  /* 127.0.0.1   */
```

## 什么是ping

ping 是应用层命令，它的功能比较简单，就是**尝试**发送一个小小的消息到目标机器上，判断目的机器是否**可达**，其实也就是判断目标机器网络是否能连通。

ping应用的底层，用的是网络层的**ICMP协议**。

[https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnkr3eLdxxIK0eujAOibyGS3abT1NHNLtfz0KV8jBOVDFtaEb7ZqpQ48XEzZ1eAnbXrDXmXPuMelJ6A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnkr3eLdxxIK0eujAOibyGS3abT1NHNLtfz0KV8jBOVDFtaEb7ZqpQ48XEzZ1eAnbXrDXmXPuMelJ6A/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

虽然ICMP协议和IP协议**都属于网络层协议**，但其实**ICMP也是利用了IP协议进行消息的传输**。

## TCP发数据和ping的区别

一般情况下，我们会使用 TCP 进行网络数据传输，那么我们可以看下它和 ping 的区别。

[https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnkr3eLdxxIK0eujAOibyGS3aTZVia58LjFhfprOVOg8y4WclVdG4Y7tD8IGhPQia65nPey5TDWARgGNA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnkr3eLdxxIK0eujAOibyGS3aTZVia58LjFhfprOVOg8y4WclVdG4Y7tD8IGhPQia65nPey5TDWARgGNA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

                                                        ping和普通发消息的关系

ping和其他应用层软件都属于**应用层**。

为了发送消息，那就得先知道往哪发。linux里万物皆文件，那你要发消息的目的地，也是个文件，这里就引出了socket 的概念。

要使用 `socket` , 那么首先需要创建它。

在 TCP 传输中创建的方式是  `socket(AF_INET, SOCK_STREAM, 0);`，其中 `AF_INET` 表示将使用 IPV4 里 **host:port** 的方式去解析待会你输入的网络地址。`SOCK_STREAM` 是指使用面向字节流的 TCP 协议，**工作在传输层**。

创建好了 `socket` 之后，就可以愉快的把要传输的数据写到这个文件里。调用 socket 的`sendto`接口的过程中进程会从**用户态进入到内核态**，最后会调用到 `sock_sendmsg` 方法。

然后进入传输层，带上`TCP`头。网络层带上`IP`头，数据链路层带上 `MAC`头等一系列操作后。进入网卡的**发送队列 ring buffer** ，顺着网卡就发出去了。

回到 `ping` ， 整个过程也基本跟 `TCP` 发数据类似，差异的地方主要在于，创建 `socket` 的时候用的是  `socket(AF_INET,SOCK_RAW,IPPROTO_ICMP)`，`SOCK_RAW` 是原始套接字 ，**工作在网络层**， 所以构建`ICMP`（网络层协议）的数据，是再合适不过了。ping 在进入内核态后最后也是调用的  `sock_sendmsg` 方法，进入到网络层后加上**ICMP和IP头**后，数据链路层加上**MAC头**，也是顺着网卡发出。因此 本质上ping 跟 普通应用发消息 在程序流程上没太大差别。

这也解释了**为什么当你发现怀疑网络有问题的时候，别人第一时间是问你能ping通吗？**因为可以简单理解为ping就是自己组了个数据包，让系统按着其他软件发送数据的路径往外发一遍，能通的话说明其他软件发的数据也能通。

## **为什么断网了还能 ping 通 127.0.0.1**

前面提到，有网的情况下，ping 最后是**通过网卡**将数据发送出去的。

那么断网的情况下，网卡已经不工作了，ping 回环地址却一切正常，我们可以看下这种情况下的工作原理。

[https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnkr3eLdxxIK0eujAOibyGS3a3njA3Ysh4YWH7owWreIy81oOqpAr5vOfdp5QkPjPIiarZ5Tkg7MZUYQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/FmVWPHrDdnkr3eLdxxIK0eujAOibyGS3a3njA3Ysh4YWH7owWreIy81oOqpAr5vOfdp5QkPjPIiarZ5Tkg7MZUYQ/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

                                                              ping回环地址

从应用层到传输层再到网络层。这段路径跟ping外网的时候是几乎是一样的。到了网络层，系统会根据目的IP，在路由表中获取对应的**路由信息**，而这其中就包含选择**哪个网卡**把消息发出。

当发现**目标IP是外网IP**时，会从"真网卡"发出。

当发现**目标IP是回环地址**时，就会选择**本地网卡**。

本地网卡，其实就是个**"假网卡"**，它不像"真网卡"那样有个`ring buffer`什么的，"假网卡"会把数据推到一个叫 `input_pkt_queue` 的 **链表** 中。这个链表，其实是所有网卡共享的，上面挂着发给本机的各种消息。消息被发送到这个链表后，会再触发一个**软中断**。

专门处理软中断的工具人**"ksoftirqd"** （这是个**内核线程**），它在收到软中断后就会立马去链表里把消息取出，然后顺着数据链路层、网络层等层层往上传递最后给到应用程序。

ping 回环地址和**通过TCP等各种协议发送数据到回环地址**都是走这条路径。整条路径从发到收，都没有经过"真网卡"。**之所以127.0.0.1叫本地回环地址，可以理解为，消息发出到这个地址上的话，就不会出网络，在本机打个转就又回来了。**所以断网，依然能 `ping` 通 `127.0.0.1`。

## ping回环地址和ping本机地址有什么区别

我们在mac里执行 `ifconfig` 。

```jsx
$ ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
    inet 127.0.0.1 netmask 0xff000000
    ...
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
    inet 192.168.31.6 netmask 0xffffff00 broadcast 192.168.31.255
    ...
```

能看到 **lo0**，表示本地回环接口，对应的地址，就是我们前面提到的 **127.0.0.1** ，也就是**回环地址**。

和 **eth0**，表示本机第一块网卡，对应的IP地址是**192.168.31.6**，管它叫**本机IP**。

ping 本机IP 跟 ping 回环地址一样，相关的网络数据，都是走的  **lo0**，本地回环接口，也就是前面提到的**"假网卡"**。

只要走了本地回环接口，那数据都不会发送到网络中，在本机网络协议栈中兜一圈，就发回来了。因此 **ping回环地址和ping本机地址没有区别**。

## **127.0.0.1 和 localhost 以及 0.0.0.0 有区别吗**

首先 `localhost` 就不叫 `IP`，它是一个域名，就跟 `"baidu.com"`,是一个形式的东西，只不过默认会把它解析为 `127.0.0.1` ，当然这可以在 `/etc/hosts` 文件下进行修改。

所以默认情况下，使用 `localhost`  跟使用  `127.0.0.1`  确实是没区别的。

其次就是 `0.0.0.0`，执行 ping 0.0.0.0  ，是会失败的，因为它在`IPV4`中表示的是无效的**目标地址**。

```jsx
$ ping 0.0.0.0
PING 0.0.0.0 (0.0.0.0): 56 data bytes
ping: sendto: No route to host
ping: sendto: No route to host
```

但它还是很有用处的，回想下，我们启动服务器的时候，一般会 `listen` 一个 IP 和端口，等待客户端的连接。

如果此时 `listen` 的是本机的 `0.0.0.0` , 那么它表示本机上的**所有IPV4地址**。

```jsx
/* Address to accept any incoming messages. */
#define    INADDR_ANY      ((unsigned long int) 0x00000000) /* 0.0.0.0   */
```

reference

[https://mp.weixin.qq.com/s/9tVsCqp7y2xvzT1mwA9EBg](https://mp.weixin.qq.com/s/9tVsCqp7y2xvzT1mwA9EBg)