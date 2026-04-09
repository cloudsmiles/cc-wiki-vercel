# 安装MySQL

## 查看MySQL版本

> 
> 
> 
> 访问DokcerHub中的MySQL镜像库地址：[https://hub.docker.com/_/mysql/tags](https://hub.docker.com/_/mysql/tags)
> 
> 可以通过 Sort by 查看其他版本的MySQL，查看最新版本MySQL镜像(mysql:latest`)：[https://hub.docker.com/_/mysql/tags?page=1&name=latest](https://hub.docker.com/_/mysql/tags?page=1&name=latest)`
> 

> 此外，我们还可以用`docker search mysql`命令来查看可用版本：
> 

![Untitled](RPG从零开始/Cloud%20Native/Docker/使用/安装MySQL/Untitled.png)

## 拉取最新的MySQL版本

`docker pull mysql:latest`

> 注意：tag是可选的，tag表示标签，多为软件的版本，默认是latest版本（最新版）
> 

![https://img2022.cnblogs.com/blog/1336199/202209/1336199-20220912224805976-1673035520.png](https://img2022.cnblogs.com/blog/1336199/202209/1336199-20220912224805976-1673035520.png)

## **验证MySQL镜像是否成功拉取到本地:**

使用以下命令来查看mysql镜像是否成功拉取到本地：

`docker images`

![Untitled](RPG从零开始/Cloud%20Native/Docker/使用/安装MySQL/Untitled%201.png)

## **创建并运行一个MySQL容器：**

`docker run --name=mysql-test -itd -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root123456 -d mysql`

**参数说明：**

- -name：指定了容器的名称，方便之后进入容器的命令行。
- itd：其中，i是交互式操作，t是一个终端，d指的是在后台运行。
- p：指在本地生成一个随机端口，用来映射mysql的3306端口。
- e：设置环境变量。
- MYSQL_ROOT_PASSWORD=root123456：指定了MySQL的root密码
- d mysql：指运行mysql镜像，设置容器在在后台一直运行。

![Untitled](RPG从零开始/Cloud%20Native/Docker/使用/安装MySQL/Untitled%202.png)

## **验证MySQL容器是否创建并运行成功：**

`docker ps`

![Untitled](RPG从零开始/Cloud%20Native/Docker/使用/安装MySQL/Untitled%203.png)

**1、进入MySQL容器：**

`docker exec -it mysql-test /bin/bash`

![Untitled](RPG从零开始/Cloud%20Native/Docker/使用/安装MySQL/Untitled%204.png)

**2、进入MySQL：**

`mysql -uroot -pEnter password：123456`

![Untitled](RPG从零开始/Cloud%20Native/Docker/使用/安装MySQL/Untitled%205.png)

参考

[https://www.cnblogs.com/Can-daydayup/p/16653879.html](https://www.cnblogs.com/Can-daydayup/p/16653879.html)