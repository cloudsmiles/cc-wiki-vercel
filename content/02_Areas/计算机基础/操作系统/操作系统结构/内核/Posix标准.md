# Posix标准

![Untitled](RPG从零开始/Operation%20System/操作系统结构/内核/Posix标准/Untitled.png)

### POSIX是什么

POSIX是Unix的标准，是为了统一Unix和Unix衍生系统而制定的规范，由于各个系统的系统调用实现不一致，所以需要制定规范约束

### Linux为什么流行

GNU/Linux系统脱颖而出，重要的原因是Linux系统采用glibc库函数，glibc库函数按照POSIX的标准而编写，所以可以兼容各种Unix系统，可移植性好

### 如何跨平台

一个编程语言的可移植性取决于

1. 不同平台编译器的数量
2. 对特殊硬件或操作系统的依赖性

我们都是将C，C++等各种语言当作中间层，以实现其一定程度上的可移植。如今，语言的跨平台的程序都是以这样的方式实现的。但是在不同的平台下，仍需要重新编译。

参考

[posix是什么都不知道，还好意思说你懂Linux？](https://zhuanlan.zhihu.com/p/392588996)