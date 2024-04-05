---
up:
  - "[[ssh]]"
reference: https://blog.csdn.net/qq_18617433/article/details/125818669
relate: "[[known_hosts文件]]"
---

## 问题描述
ssh登陆报错“IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!“

## 原因
这个报错主要是因为远程主机的ssh公钥发生了变化，两边不一致导致的。

## 解决方法
删除本地对应ip的在known_hosts相关信息