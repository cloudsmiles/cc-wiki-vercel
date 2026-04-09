# go build

Go语言的程序编写基本以源码方式，无论是自己的代码还是第三方代码，并且以 GOPATH 作为工作目录和一套完整的工程目录规则。因此Go语言中日常编译时无须像[C++](http://c.biancheng.net/cplus/)一样配置各种包含路径、链接库地址等。

Go语言中使用 go build 命令主要用于编译代码。在包的编译过程中，若有必要，会同时编译与之相关联的包。

go build 有很多种编译方法，如无参数编译、文件列表编译、指定包编译等，使用这些方法都可以输出可执行文件。

## go build+文件列表

```jsx
代码相对于 GOPATH 的目录关系如下：
.
└── src
    └── chapter11
        └── gobuild
            ├── lib.go
            └── main.go
```

编译同目录的多个源码文件时，可以在 go build 的后面提供多个文件名，go build 会编译这些源码，输出可执行文件，“go build+文件列表”的格式如下：

```jsx
go build file1.go file2.go……
```

### 提示

使用“go build+文件列表”方式编译时，可执行文件默认选择文件列表中第一个源码文件作为可执行文件名输出。

如果需要指定输出可执行文件名，可以使用-o

## go build+包

“go build+包”在设置 GOPATH 后，可以直接根据包名进行编译，即便包内文件被增（加）删（除）也不影响编译指令。

相对于GOPATH的目录关系如下：

```jsx
.
└── src
    └── chapter11
        └──goinstall
            ├── main.go
            └── mypkg
                └── mypkg.go
```

**按包编译命令**

执行以下命令将按包方式编译 goinstall 代码：

```jsx
$ export GOPATH=/home/davy/golangbook/code
$ go build -o main chapter11/goinstall
$ ./main
call CustomPkgFunc
hello world
```

## **go build 编译时的附加参数**

go build 还有一些附加参数，可以显示更多的编译信息和更多的操作，详见下表所示。

| 附加参数 | 备  注 |
| --- | --- |
| -v | 编译时显示包名 |
| -p n | 开启并发编译，默认情况下该值为 CPU 逻辑核数 |
| -a | 强制重新构建 |
| -n | 打印编译时会用到的所有命令，但不真正执行 |
| -x | 打印编译时会用到的所有命令 |
| -race | 开启竞态检测 |

reference:

[go build命令（go语言编译命令）完全攻略](http://c.biancheng.net/view/120.html)