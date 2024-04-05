# TIME-WAIT 状态为什么需要等待 2MSL

Owner: cloudsmiles
Last edited time: April 25, 2023 5:25 AM

### **TIME-WAIT 状态为什么需要等待 2MSL**

[https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGcmrVMle4KZubY5Tciae8HO8wnpzPUZthmTXY8PpmoYjZ4FXC9ibRz0ug/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpwwTDZWIWFC9LakSgxrYMZGcmrVMle4KZubY5Tciae8HO8wnpzPUZthmTXY8PpmoYjZ4FXC9ibRz0ug/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

2MSL，2 Maximum Segment Lifetime，即两个最大段生命周期

> ★
> 
> - 1个 MSL 保证四次挥手中主动关闭方最后的 ACK 报文能最终到达对端
> - 1个 MSL 保证对端没有收到 ACK 那么进行重传的 FIN 报文能够到达