# TCP重传机制

Owner: cloudsmiles
Last edited time: April 25, 2023 5:33 AM

### **超时重传**

TCP 为了实现可靠传输，实现了重传机制。最基本的重传机制，就是**超时重传**，即在发送数据报文时，设定一个定时器，每间隔一段时间，没有收到对方的ACK确认应答报文，就会重发该报文。

这个间隔时间，一般设置为多少呢？我们先来看下什么叫**RTT（Round-Trip Time，往返时间）**。

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGQeePs2NCSbIdvl997a7mWGHUWic5kGghXVFpRNPwYtOggZytGywNMaw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGQeePs2NCSbIdvl997a7mWGHUWic5kGghXVFpRNPwYtOggZytGywNMaw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

RTT就是，一个数据包从发出去到回来的时间，即**数据包的一次往返时间**。超时重传时间，就是Retransmission Timeout ，简称**RTO**。

**RTO设置多久呢？**

- 如果RTO比较小，那很可能数据都没有丢失，就重发了，这会导致网络阻塞，会导致更多的超时出现。
- 如果RTO比较大，等到花儿都谢了还是没有重发，那效果就不好了。

一般情况下，RTO略大于RTT，效果是最好的。一些小伙伴会问，超时时间有没有计算公式呢?有的！有个标准方法算RTO的公式，也叫**Jacobson / Karels 算法**。我们一起来看下计算RTO的公式

**1. 先计算SRTT（计算平滑的RTT）**

```
SRTT = (1 - α) * SRTT + α * RTT  //求 SRTT 的加权平均

```

**2. 再计算RTTVAR (round-trip time variation)**

```
RTTVAR = (1 - β) * RTTVAR + β * (|RTT - SRTT|) //计算 SRTT 与真实值的差距

```

**3. 最终的RTO**

```
RTO = µ * SRTT + ∂ * RTTVAR  =  SRTT + 4·RTTVAR

```

其中，`α = 0.125，β = 0.25， μ = 1，∂ = 4`，这些参数都是大量结果得出的最优参数。

但是，超时重传会有这些缺点：

> ★
> 
> - 当一个报文段丢失时，会等待一定的超时周期然后才重传分组，增加了端到端的时延。
> - 当一个报文段丢失时，在其等待超时的过程中，可能会出现这种情况：其后的报文段已经被接收端接收但却迟迟得不到确认，发送端会认为也丢失了，从而引起不必要的重传，既浪费资源也浪费时间。
> 
> # ”
> 

并且，TCP有个策略，就是超时时间间隔会加倍。超时重传需要**等待很长时间**。因此，还可以使用**快速重传**机制。

### **快速重传**

**快速重传**机制，它不以时间驱动，而是以数据驱动。它基于接收端的反馈信息来引发重传。

一起来看下快速重传流程：

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGu7I5I3EJZTdlJMGMxibIkYquScTY5XibRicykIrEHAp7qBag50qH7I4UA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGu7I5I3EJZTdlJMGMxibIkYquScTY5XibRicykIrEHAp7qBag50qH7I4UA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

快速重传流程

发送端发送了 1，2，3，4，5,6 份数据:

- 第一份 Seq=1 先送到了，于是就 Ack 回 2；
- 第二份 Seq=2 也送到了，假设也正常，于是ACK 回 3；
- 第三份 Seq=3 由于网络等其他原因，没送到；
- 第四份 Seq=4 也送到了，但是因为Seq3没收到。所以ACK回3；
- 后面的 Seq=4,5的也送到了，但是ACK还是回复3，因为Seq=3没收到。
- 发送端连着收到三个重复冗余ACK=3的确认（实际上是4个，但是前面一个是正常的ACK，后面三个才是重复冗余的），便知道哪个报文段在传输过程中丢失了，于是在定时器过期之前，重传该报文段。
- 最后，接收到收到了 Seq3，此时因为 Seq=4，5，6都收到了，于是ACK回7.

但**快速重传**还可能会有个问题：ACK只向发送端告知最大的有序报文段，到底是哪个报文丢失了呢？**并不确定**！那到底该重传多少个包呢？

> ★
> 
> 
> 是重传 Seq3 呢？还是重传 Seq3、Seq4、Seq5、Seq6 呢？因为发送端并不清楚这三个连续的 ACK3 是谁传回来的。
> 
> # ”
> 

### **带选择确认的重传（SACK）**

为了解决快速重传的问题：**应该重传多少个包**? TCP提供了**SACK方法**（带选择确认的重传，Selective Acknowledgment）。

**SACK机制**就是，在快速重传的基础上，接收端返回最近收到的报文段的序列号范围，这样发送端就知道接收端哪些数据包没收到，酱紫就很清楚该重传哪些数据包啦。SACK标记是加在TCP头部**选项**字段里面的。

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGtnwicDafN34ibW12dBjvVgUdnEnDibWKkLBiafdjHnr0UNzZKIIMDoVsLA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGtnwicDafN34ibW12dBjvVgUdnEnDibWKkLBiafdjHnr0UNzZKIIMDoVsLA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

SACK机制

如上图中，发送端收到了三次同样的ACK=30的确认报文，于是就会触发快速重发机制，通过SACK信息发现只有`30~39`这段数据丢失，于是重发时就只选择了这个`30~39`的TCP报文段进行重发。

### **D-SACK**

D-SACK，即Duplicate SACK（重复SACK），在SACK的基础上做了一些扩展，，主要用来告诉发送方，有哪些数据包自己重复接受了。DSACK的目的是帮助发送方判断，是否发生了包失序、ACK丢失、包重复或伪重传。让TCP可以更好的做网络流控。来看个图吧：

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGRtETx7uS0rZNAIqvXqYLECxCEfyVPHamAJns9OBWXKO541eJau81pA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGRtETx7uS0rZNAIqvXqYLECxCEfyVPHamAJns9OBWXKO541eJau81pA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

D-SACK简要流程

reference

[https://mp.weixin.qq.com/s/-t6fS_Hif9jDPsuMlWWeyQ](https://mp.weixin.qq.com/s/-t6fS_Hif9jDPsuMlWWeyQ)