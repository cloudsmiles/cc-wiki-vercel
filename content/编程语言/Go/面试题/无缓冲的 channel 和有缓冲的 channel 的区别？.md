# 无缓冲的 channel 和有缓冲的 channel 的区别？

对于无缓冲区channel：

发送的数据如果没有被接收方接收，那么**发送方阻塞；**如果一直接收不到发送方的数据，**接收方阻塞**；

有缓冲的channel：

发送方在缓冲区满的时候阻塞，接收方不阻塞；接收方在缓冲区为空的时候阻塞，发送方不阻塞。

可以类比生产者与消费者问题。

![Untitled](RPG从零开始/Program%20Language/Go/面试题/无缓冲的%20channel%20和有缓冲的%20channel%20的区别？/Untitled.png)