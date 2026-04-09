# go.mod中indirect含义

在执行命令`go mod tidy`时，Go module 会自动整理`go.mod` 文件，如果有必要会在部分依赖包的后面增加`// indirect`注释。一般而言，被添加注释的包肯定是间接依赖的包，而没有添加`// indirect`注释的包则是直接依赖的包，即明确的出现在某个`import`语句中。

然而，这里需要着重强调的是：并不是所有的间接依赖都会出现在 `go.mod`文件中。

间接依赖出现在`go.mod`文件的情况，可能符合下面所列场景的一种或多种：

- 直接依赖未启用 Go module
- 直接依赖go.mod 文件中缺失部分依赖

## **直接依赖未启用 Go module**

如下图所示，Module A 依赖 B，但是 B 还未切换成 Module，也即没有`go.mod`文件，此时，当使用`go mod tidy`命令更新A的`go.mod`文件时，B的两个依赖B1和B2将会被添加到A的`go.mod`文件中（前提是A之前没有依赖B1和B2），并且B1 和B2还会被添加`// indirect`的注释。

[https://imgconvert.csdnimg.cn/aHR0cHM6Ly9vc2NpbWcub3NjaGluYS5uZXQvb3NjbmV0L3VwLTExZTdhMTE4ZTA0YzNlZTRmZmNiMjU4YmQ3NDRhYjFhYjEzLnBuZw?x-oss-process=image/format,png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9vc2NpbWcub3NjaGluYS5uZXQvb3NjbmV0L3VwLTExZTdhMTE4ZTA0YzNlZTRmZmNiMjU4YmQ3NDRhYjFhYjEzLnBuZw?x-oss-process=image/format,png)

此时Module A的`go.mod`文件中require部分将会变成：

```go
require (
	B vx.x.x
	B1 vx.x.x// indirect
	B2 vx.x.x// indirect
)
```

依赖B及B的依赖B1和B2都会出现在`go.mod`文件中。

## **直接依赖 go.mod 文件不完整**

如上面所述，如果依赖B没有`go.mod`文件，则Module A 将会把B的所有依赖记录到A 的`go.mod`文件中。即便B拥有`go.mod`，如果`go.mod`文件不完整的话，Module A依然会记录部分B的依赖到`go.mod`文件中。

如下图所示，Module B虽然提供了`go.mod`文件中，但`go.mod`文件中只添加了依赖B1，那么此时A在引用B时，则会在A的`go.mod`文件中添加B2作为间接依赖，B1则不会出现在A的`go.mod`文件中。

[https://imgconvert.csdnimg.cn/aHR0cHM6Ly9vc2NpbWcub3NjaGluYS5uZXQvb3NjbmV0L3VwLWYxODVlNGEwMWM2M2ZmY2U3MDc2N2VjZGYwNjU4MTkxMDBjLnBuZw?x-oss-process=image/format,png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9vc2NpbWcub3NjaGluYS5uZXQvb3NjbmV0L3VwLWYxODVlNGEwMWM2M2ZmY2U3MDc2N2VjZGYwNjU4MTkxMDBjLnBuZw?x-oss-process=image/format,png)

此时Module A的`go.mod`文件中require部分将会变成：

```go
require (
	B vx.x.x
	B2 vx.x.x// indirect
)
```

由于B1已经包含进B的`go.mod`文件中，A的`go.mod`文件则不必再记录，只会记录缺失的B2。

## 总结

### **如何查找间接依赖来源**

Go module提供了`go mod why` 命令来解释为什么会依赖某个软件包，若要查看`go.mod`中某个间接依赖是被哪个依赖引入的，可以使用命令`go mod why -m <pkg>`来查看。

reference

[https://blog.csdn.net/juzipidemimi/article/details/104441398](https://blog.csdn.net/juzipidemimi/article/details/104441398)