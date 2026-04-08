# go generate

go generate命令是在Go语言 1.4 版本里面新添加的一个命令，当运行该命令时，它将扫描与当前包相关的源代码文件，找出所有包含//go:generate的特殊注释，提取并执行该特殊注释后面的命令。

使用go generate命令时有以下几点需要注意：

- 该特殊注释必须在 .go 源码文件中；
- 每个源码文件可以包含多个 generate 特殊注释；
- 运行`go generate`命令时，才会执行特殊注释后面的命令；
- 当`go generate`命令执行出错时，将终止程序的运行；
- 特殊注释必须以`//go:generate`开头，双斜线后面没有空格。

在下面这些场景下，我们会使用go generate命令：

- yacc：从 .y 文件生成 .go 文件；
- protobufs：从 protocol buffer 定义文件（.proto）生成 .pb.go 文件；
- Unicode：从 UnicodeData.txt 生成 Unicode 表；
- HTML：将 HTML 文件嵌入到 go 源码；
- bindata：将形如 JPEG 这样的文件转成 go 代码中的字节数组。

reference:

[go generate命令——在编译前自动化生成某类代码](http://c.biancheng.net/view/4442.html)