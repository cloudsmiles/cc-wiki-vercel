# go get

go get 命令可以借助代码管理工具通过远程拉取或更新代码包及其依赖包，并自动完成编译和安装。整个过程就像安装一个 App 一样简单。

这个命令可以动态获取远程代码包，目前支持的有 BitBucket、GitHub、Google Code 和 Launchpad。在使用 go get 命令前，需要安装与远程包匹配的代码管理工具，如 Git、SVN、HG 等，参数中需要提供一个包名。

这个命令在内部实际上分成了两步操作：第一步是下载源码包，第二步是执行 go install。下载源码包的 go 工具会自动根据不同的域名调用不同的源码工具，对应关系如下：

```jsx
BitBucket (Mercurial Git)
GitHub (Git)
Google Code Project Hosting (Git, Mercurial, Subversion)
Launchpad (Bazaar)
```

## 常用go get flag

### -d

表示只下载包，不安装包

### -f

只有当-u使用时才有效，强制让get -u 不要去验证每一个已经从源库check out的包，这由它的导入路径暗示，这对于源是本地分支时是很有用

### -t

同时也下载需要为运行测试所需要的包

### -u

指示get通过网络去更新包，默认情况下，get只是通过网络下载包，但不会去更新已经存在的包

### -v

输出进度和debug信息

referenece:

[go get命令——一键获取代码、编译并安装](http://c.biancheng.net/view/123.html)