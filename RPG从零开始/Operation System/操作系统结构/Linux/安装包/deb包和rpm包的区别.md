# deb包和rpm包的区别

## rpm

RPM全称是Red Hat Package Manager(Red Hat包管理器)，是由红帽公司最先发布的一种用来打包软件的的文件格式，是专门用来安装，卸载软件等操作的，它里面打包的内容必定是一个可以使用的具体软件。

RPM本质上就是一个包，包含可以立即在特定机器体系结构上安装和运行的Linux软件。在红帽LINUX、SUSE、Fedora可以直接进行安装，但在Ubuntu中却无法识别

## deb

deb是Debian Linux提供的一个包管理器，类似与RPM。但由于RPM出现得早，并且应用广泛，所以在各种版本的Linux中都常见到，而Debian的包管理器dpkg只出现在Debian Linux中。它的优点是不用被严格的依赖性检查所困扰，缺点是只在Debian Linux发行版中才能见到这个包管理工具。因此，在Ubuntu系统中，我们才可以直接双击deb包自动进入安装进程，而在其他的linux系统中，就不能直接使用安装。

reference

[https://www.cnblogs.com/longchang/p/12530697.html](https://www.cnblogs.com/longchang/p/12530697.html)