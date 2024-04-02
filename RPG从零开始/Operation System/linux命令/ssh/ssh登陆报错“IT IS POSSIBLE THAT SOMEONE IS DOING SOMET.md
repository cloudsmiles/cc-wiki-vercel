# ssh登陆报错“IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!“问题原因及解决方法

这个报错主要是因为远程主机的ssh公钥发生了变化，两边不一致导致的。删除本地对应ip的在known_hosts相关信息

reference:

[https://blog.csdn.net/qq_18617433/article/details/125818669](https://blog.csdn.net/qq_18617433/article/details/125818669)