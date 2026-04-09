# modules

# **Go Modules 的前世今生**

流行的现代编程语言一般都提供依赖库管理工具，如 Java 的 Maven 、Python 的 PIP、Node.js 的 NPM 和 Rust 的 Cargo 等。Go 最为一门新生代语言，自然也有其自己的库管理方式。

### **GOPATH**

在 Go 1.5 之前，Go 最原始的依赖管理使用的是 go get，执行命令后会拉取代码放入 GOPATH/src 下面。但是它是作为 GOPATH 下全局的依赖，并且 go get 还不能进行版本控制，以及隔离项目的包依赖。

而随着 Go 生态圈的快速壮大，无法进行版本控制，会导致项目中的依赖库经常出现 API broken 的情况。因为依赖的库相关接口改变了，导致我们的项目更新了依赖库后编译不过，我们不得不需要修改自己的代码以便适应依赖库的最新版本。更困难的是，如果多个依赖库分别依赖第三个依赖库的第三个版本，版本冲突就出现了

### **Go vendor**

Go 1.5 版本推出了 vendor 机制。但是需要手动设置环境变量 GO15VENDOREXPERIMENT= 1，Go 编译器才能启用。从 Go1.6 起，默认开启 vendor 机制。

所谓 vendor 机制，就是每个项目的根目录下可以有一个 vendor 目录，里面存放了该项目的依赖的 package。go build 的时候会先去 vendor 目录查找依赖，如果没有找到会再去 GOPATH 目录下查找。

但 vendor 也有缺点，那就是对外部依赖的第三方包的版本管理。

### **第三方管理工具**

在 Go 1.11 之前，很多优秀的第三方包管理工具起到了举足轻重的作用，弥补了 Go 在依赖管理方面的不足，比如 [godep](https://github.com/tools/godep)、[govendor](https://github.com/kardianos/govendor)、[glide](https://github.com/Masterminds/glide)、[dep](https://github.com/golang/dep) 等。其中 dep 拥趸众多，而且也得到了 Go 官方的支持，项目也放在 Golang 组织之下 [golang/dep](https://github.com/golang/dep)。

### **Go Modules 横空出世**

从 2018 年 Go 1.11 开始，Go 官方推出了 Go Modules。为了保持向后兼容，Go 官方旧的依赖管理方式依然存在。启用 Go Modules 需要显示通过设置一个环境变量 GO111MODULE=on。在之后的 go 1.12 正式推出后，Go Modules 成为默认的依赖管理方式。

先前，我们的库都是以 package 来组织的，package 以一个文件或者多个文件实现单一的功能。一个项目包含一个package 或者多个 package。Go modules 就是一个统一打版和发布的 package 的集合，在项目根文件下有 go.mod 文件定义 module path 和依赖库的版本，还有一个 go.sum 的文件，该文件包含特定依赖包的版本内容的散列哈希值。

一般我们项目都是单 module 的形式，项目根目录下包含 go.mod 和 go.sum 文件，子文件夹定义 package，或者主文件夹也是一个 package。但是一个项目也可以包含多个 module，只不过这种方式不常用而已。

# **go.mod 文件**

go modules 最重要的是 go.mod 文件的定义，它用来标记一个 module 和它的依赖库以及依赖库的版本。会放在 module 的主文件夹下，一般以 go.mod 命名。

一个 go.mod 内容类似下面的格式：

```jsx
module github.com/dablelv/go-huge-util

go 1.17

replace github.com/coreos/bbolt => ../r

require (
	github.com/cenk/backoff v2.2.1+incompatible
	github.com/edwingeng/doublejump v0.0.0-20200330080233-e4ea8bd1cbed
	github.com/go-sql-driver/mysql v1.5.0
	github.com/spf13/cast v1.4.1
	github.com/stretchr/testify v1.7.0
	golang.org/x/text v0.3.2
)

require (
	github.com/davecgh/go-spew v1.1.1 // indirect
	github.com/pmezard/go-difflib v1.0.0 // indirect
	gopkg.in/yaml.v3 v3.0.0-20200313102051-9f266ea9e77c // indirect
)

exclude (
	go.etcd.io/etcd/client/v2 v2.305.0-rc.0
	go.etcd.io/etcd/client/v3 v3.5.0-rc.0
)

retract (
    v1.0.0 // 废弃的版本，请使用v1.1.0
)
```

### **语义化版本 2.0.0**

Go module 遵循[语义化版本 2.0.0](https://semver.bootcss.com/)。语义化版本规范 2.0.0 规定了版本号的格式，每个字段的意义以及版本号比较的规则等等。

![Untitled](RPG从零开始/Program%20Language/Go/基础/modules/Untitled.png)

### **module path**

go.mod 的第一行是 module path，一般采用“仓库+module name” 的方式定义。这样我们获取一个 module 的时候，就可以到它的仓库中去查询，或者让 go proxy 到仓库中去查询。

`module github.com/dablelv/go-huge-util`

如果你的版本已经 >=2.0.0，按照 Go 的规范，你应该加上 major 的后缀，module path 改成下面的方式：

`module github.com/dablelv/go-huge-util/v2

module github.com/dablelv/go-huge-util/v3`

### g**o directive**

第二行是 go directive。格式是 go 1.xx，它并不是指你当前使用的 Go 版本，而是指名你的代码所需要的 Go 的最低版本。

`go 1.17`复制

因为 Go 的标准库在不断迭代，一些新的 API 会陆续被加进来。如果你的代码用到了这些新的 API，你可能需要指明它依赖的 Go 版本。

### **require**

require段中列出了项目所需要的各个依赖库以及它们的版本，除了正规的v1.3.0这样的版本外，还有一些奇奇怪怪的版本和注释，那么它们又是什么意思呢？

**伪版本号**

`github.com/edwingeng/doublejump v0.0.0-20200330080233-e4ea8bd1cbed`复制

上面这个库中的版本号就是一个伪版本号`v0.0.0-20200330080233-e4ea8bd1cbed`，这是 go module 为它生成的一个类似符合语义化版本 2.0.0 版本，实际这个库并没有发布这个版本。

**indirect** 

```jsx
indirect 注释
require (
	github.com/davecgh/go-spew v1.1.1 // indirect
	github.com/pmezard/go-difflib v1.0.0 // indirect
	gopkg.in/yaml.v3 v3.0.0-20200313102051-9f266ea9e77c // indirect
)
```

有些库后面加了 indirect 注释，这又是什么意思呢？

如果用一句话总结，间接的使用了这个库，但是又没有被列到某个 go.mod 中，当然这句话也不算太准确，更精确的说法是下面的情况之一就会对这个库加 indirect 注释。

- 当前项目依赖 A，但是 A 的go.mod 遗漏了 B，那么就会在当前项目的 go.mod 中补充 B，加 indirect 注释；
- 当前项目依赖 A，但是 A 没有 go.mod，同样就会在当前项目的 go.mod 中补充 B，加 indirect 注释；
- 当前项目依赖 A，A 又依赖 B。当对 A 降级的时候，降级的 A 不再依赖 B，这个时候 B 就标记 indirect 注释。我们可以执行`go mod tidy`来清理不依赖的 module。

**incompatible**

有些库后面加了 incompatible 后缀，但是你如果看这些项目，它们只是发布了 v2.2.1 的 tag，并没有`+incompatible`后缀。

`github.com/cenk/backoff v2.2.1+incompatible`复制

这些库采用了 go.mod 的管理，但是不幸的是，虽然这些库的版 major 版本已经 >=2 了，但是他们的 module path 中依然没有添加 v2、v3 这样的后缀，不符合 Go 的 module 管理规范。

**exclude**

如果你想在你的项目中跳过某个依赖库的某个版本，你就可以使用这个段。

```jsx
exclude (
	go.etcd.io/etcd/client/v2 v2.305.0-rc.0
	go.etcd.io/etcd/client/v3 v3.5.0-rc.0
)
```

这样，Go 在版本选择的时候，就会主动跳过这些版本，比如你使用`go get -u ......`或者`go get github.com/xxx/xxx@latest`等命令时，会执行`version query`的动作，这些版本不在考虑的范围之内。

**replace**

replace 也是常用的一个手段，用来解决一些错误的依赖库的引用或者调试依赖库。

```jsx
replace github.com/coreos/bbolt => go.etcd.io/bbolt v1.3.3
replace github.com/panicthis/A v1.1.0 => github.com/panicthis/R v1.8.0
replace github.com/coreos/bbolt => ../r
```

比如 etcd v3.3.x 的版本中错误地使用了`github.com/coreos/bbolt`作为 bbolt 的 module path，其实这个库在它自己的go.mod 中声明的 module path 是 go.etcd.io/bbolt。又比如 etcd 使用的 grpc 版本有问题，你也可以通过 replace 替换成所需的 grpc 版本。

甚至你觉得某个依赖库有问题，自己 fork 到本地做修改，想调试一下，你也可以替换成本地的文件夹。

replace 可以替换某个库的所有版本到另一个库的特定版本，也可以替换某个库的特定版本到另一个库的特定版本。

# **go.sum 文件**

上面我们说到，Go 在做依赖管理时会创建两个文件，go.mod 和 go.sum。

相比于 go.mod，关于 go.sum 的资料明显少得多。自然，go.mod 的重要性不言而喻，这个文件几乎提供了依赖版本的全部信息。而 go.sum 则是记录了所有依赖的 module 的校验信息，以防下载的依赖被恶意篡改，主要用于安全校验。

每行的格式如下：

```jsx
<module> <version> <hash>
<module> <version>/go.mod <hash>
```

比如：

```jsx
github.com/spf13/cast v1.4.1 h1:s0hze+J0196ZfEMTs80N7UlFt0BDuQ7Q+JDnHiMWKdA=
github.com/spf13/cast v1.4.1/go.mod h1:Qx5cxh0v+4UWYiBimWS+eyWzqEqokIECu5etghLkUJE=
```

其中 module 是依赖的路径。

version 是依赖的版本号。如果 version 后面跟`/go.mod`表示对哈希值是 module 的 `go.mod` 文件；否则，哈希值是 module 的`.zip`文件。

hash 是以`h1:`开头的字符串，表示生成 checksum 的算法是第一版的 HASH 算法（SHA256）。如果将来在 SHA-256 中发现漏洞，将添加对另一种算法的支持，可能会命名为 h2。

reference: 

[深入理解 Go Modules 的 go.mod 与 go.sum-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2020911)

[[go.mod中indirect含义]]