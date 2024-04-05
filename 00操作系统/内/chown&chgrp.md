---
up:
  - "[[01Linux命令目录]]"
---

## 修改文件所属组群——chgrp

修改文件所属组群很简单`chgrp命令`，就是change group的缩写

语法：
>chgrp 组群 文件名/目录

## 修改文件拥有者——chown

修改组群的命令使chgrp，即change group，那么修改文件拥有者的命令自然就是chown，即change owner。chown功能很多，不仅仅能更改文件拥有者，还可以修改文件所属组群。如果需要将某一目录下的所有文件都改变其拥有者，可以使用-R参数。

语法如下：

> chown -R 账号名称 文件/目录 
> chown -R 账号名称:组群 文件/目录


