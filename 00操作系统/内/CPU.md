---
up:
  - "[[硬件结构]]"
---

# CPU

一个典型的CPU由运算器、控制器、寄存器等器件组成，这些器件靠内部总线相连。（外部总线是上一篇博客说的内存总线，数据总线，控制总线）

- 内部总线实现CPU内部各个器件之间的联系。
- 外部总线实现CPU和主板上其它器件的联系。

32位CPU所含有的寄存器有：

4个数据寄存器(EAX、EBX、ECX和EDX)

2个变址和指针寄存器(ESI和EDI)

2个指针寄存器(ESP和EBP)

6个段寄存器(ES、CS、SS、DS、FS和GS)

1个指令指针寄存器(EIP)

1个标志寄存器(EFlags)

[汇编语言--寄存器（CPU的工作原理 ax,bx,cx,dx通用寄存器 cs代码段寄存器）_weixin_30276935的博客-CSDN博客](https://blog.csdn.net/weixin_30276935/article/details/96846396)

[对所有CPU寄存器的简述（16位CPU14个，32位CPU16个）](http://t.zoukankan.com/findumars-p-4121962.html)

[[MMU和TLB]]