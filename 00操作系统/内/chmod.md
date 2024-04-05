---
up:
  - "[[01Linux命令目录]]"
relate: "[[chown&chgrp]]"
---
## 改变文件权限——chmod

### 文件权限

ls -l中显示的内容如下：

```javascript
-rwxrw-r‐-1 root root 1213 Feb 2 09:39 abc
```

- 第一个字符代表文件（-）、目录（d），链接（l）
- 10个字符确定不同用户能对文件干什么
- 其余字符每3个一组（rwx），读（r）、写（w）、执行（x）
- 第一组rwx：文件所有者的权限是读、写和执行
- 第二组rw-：与文件所有者同一组的用户的权限是读、写但不能执行
- 第三组r–：不与文件所有者同组的其他用户的权限是读不能写和执行
>也可用数字表示为：r=4，w=2，x=1 因此rwx=4+2+1=7

- 1 表示连接的文件数
- root 表示用户
- root表示用户所在的组
- 1213 表示文件大小（字节）
- Feb 2 09:39 表示最后修改日期
- abc 表示文件名

### 用数字来改变文件权限

我们已经了解了`-rw-r--r--`所表示含义，linux为每一个权限分配一个固定的数字：

> r： 4（读权限） 
> w：2（写权限）
> x： 1（执行权限）

我们再将这些数字相加，就得到每一组的权限值，例如

```javascript
-rw-r--r--  1 myy groupa 0 Sep 26 06:07 filed
```

第一组（user）：rw- = 4+2+0 = 6 
第二组（group）：r-- = 4+0+0 = 4 
第三组（others）：r-- = 4+0+0 = 4 
那么644就是fileb权限的数字表示值。

chmod语法：

> chmod xyz 文件/目录

例子：chmod 777 文件/目录


### 用字符来改变文件权限

还有一种改变权限的方法，我们已经了解到，文件权限分为三组，分别是user，group，others，那么我们可以用u，g，o分别代表三组，另外，a（all）代表全部，而权限属性即可用r，w，x三个字符来表示，那么请看下面的语法：

>chmod u/g/o/a +(加入)/-(除去)/=(设定) r/w/x 文件或者目录

```javascript
chmod u=rwx 文件或者目录
chmod u+rwx 文件或者目录
chmod u-rwx 文件或者目录
chmod u=rwx,go=rx 文件或者目录
chmod u=rwx，g=rx，o=rx 文件或者目录
```

例： 我们想使filed文件得到： u：可读，可写，可执行 g，o：可读，可执行
```javascript
[root@redhat zgzdir]# ls -l
total 8
-rwxrwxrwx  1 myy groupa 0 Sep 26 06:07 filec
-rwxr-x---  1 myy groupa 0 Sep 26 06:07 filed

#--修改filed的文件属性
[root@redhat zgzdir]# chmod u=rwx,go=rx filed
[root@redhat zgzdir]# ls -l
total 8
-rwxrwxrwx  1 myy groupa 0 Sep 26 06:07 filec
-rwxr-xr-x  1 myy groupa 0 Sep 26 06:07 filed
```

**其中g和o也可以用“，”分开来分别设定。**

假设目前我不知道各组权限如何，只是想让所有组都增加“x”权限，那么我们可以用`chmod a+x filename`来实现，例如：

```javascript
[root@redhat zgz]# ls -l
total 24
-rw-r--r--  1 myy groupa    0 Sep 26 05:48 filea
drwxr-xr-x  2 myy groupa 4096 Sep 26 06:07 zgzdir

# --修改filea的文件属性，所有组都增加“x”权限
[root@redhat zgz]# chmod a+x filea
[root@redhat zgz]# ls -l
total 24
-rwxr-xr-x  1 myy groupa    0 Sep 26 05:48 filea
drwxr-xr-x  2 myy groupa 4096 Sep 26 06:07 zgzdir
```

如果想除去某一权限，可以用“-”来操作， 例如：

```javascript
[root@redhat zgz]# ls -l
total 24
-rwxr-xr-x  1 myy groupa    0 Sep 26 05:48 filea
drwxr-xr-x  2 myy groupa 4096 Sep 26 06:07 zgzdir

# 修改filea文件属性所有组都除去“x”权限
[root@redhat zgz]# chmod a-x filea
[root@redhat zgz]# ls -l
total 24
-rw-r--r--  1 myy groupa    0 Sep 26 05:48 filea
drwxr-xr-x  2 myy groupa 4096 Sep 26 06:07 zgzdir
[root@redhat zgz]#
```