# csv乱码问题

由于csv文件格式可能是utf8或者是gbk，所以需要读取的时候指定encoding。还需要判断读取的字符的编码

reference:

[https://juejin.cn/s/golang csv 中文乱码](https://juejin.cn/s/golang%20csv%20%E4%B8%AD%E6%96%87%E4%B9%B1%E7%A0%81)

[golang学习之判断字符串编码是否是UTF8或GBK - 小白的分享](https://www.yoby123.cn/2021/03/11/2021-3-11-2.html)

[golang处理excel打开csv乱码问题_golang 打开csv文件不是utf-8编码_羁士的博客-CSDN博客](https://blog.csdn.net/dianxin113/article/details/111227305)