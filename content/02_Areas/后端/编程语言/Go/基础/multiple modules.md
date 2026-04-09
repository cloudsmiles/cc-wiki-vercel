# multiple modules

## 初步使用

目前go都是通过mod进行包管理，包主要分两类，一类是公网上的，直接引入用go mod管理即可。

一类是本地包，比如你自己写的包或者下载到本地的包，对于这类包，在go 1.18前，都是通过给go.mod中添加replace来修改引用包的实际路径。

这种包引用对于git提交就非常麻烦，每次提交前需要把replace去掉才行。

于是go work被发明了出来。

go work 即工作空间，就是一个目录(文件夹)，里边有一个go.work 文件。

go.work的内容如下：

```go
go 1.19
use(
    H:/information/goserver
    ./demo
)
```

注意两个目录路径，工作空间为 H:/work/ 但是模块引用的包路径是 H:/information/goserver，他们可以不在同一个目录下，**use后可以跟相对或者绝对路径**。

demo一旦放在工作空间下，那么，demo下的go.mod寻找就会通过go.work来实现，而不能像之前在demo下运行go run或者go build 就能自动寻找到go.mod， 所以必须在work目录下运行go use demo，把"./demo" 添加到go.work中，这样你才能在demo项目里引用demo模块中的包。

## 命令解析

### 工作空间初始化

> go work init
> 

该命令运行后会生成go.work文件。

### **添加项目到工作空间**

go work use [-r] [项目1 项目2 ...]

-r 表示递归查找。

项目指包含go.mod的项目

删除用

> go work -dropedit=项目
> 

# use replace

```bash

go 1.18
use (
  ./hello
  ./example
)
replace (
  github.com/link1st/example => ./example
)

```

use 和 replace都是指定本地项目目录

replace 则表示项目中的 github.com/link1st/example 在本地 ./example1中找

reference

[https://juejin.cn/post/7145855715565895710](https://juejin.cn/post/7145855715565895710)

[https://go.dev/doc/tutorial/workspaces](https://go.dev/doc/tutorial/workspaces)