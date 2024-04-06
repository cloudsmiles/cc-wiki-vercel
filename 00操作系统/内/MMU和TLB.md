---
up:
  - "[[CPU]]"
---

# 什么是MMU和TLB

## MMU(Memory Management Unit)内存管理单元

- 一种硬件电路单元负责将虚拟内存地址转换为物理内存地址
- 所有的内存访问都将通过MMU进行转换

## TLB(Transalation Lookaside Buffer)转译后备缓冲器

本质上是MMU用于虚拟地址到物理地址转换表的缓存

![Untitled](MMU.png)