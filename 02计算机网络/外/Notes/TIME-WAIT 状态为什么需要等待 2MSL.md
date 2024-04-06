---
up:
  - "[[TCP]]"
---

### **TIME-WAIT 状态为什么需要等待 2MSL**


## 答
![[Pasted image 20240405230757.png]]

2MSL，2 Maximum Segment Lifetime，即两个最大段生命周期

> ★
> 
> - 1个 MSL 保证四次挥手中主动关闭方最后的 ACK 报文能最终到达对端
> - 1个 MSL 保证对端没有收到 ACK 那么进行重传的 FIN 报文能够到达