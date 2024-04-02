# Workspace

## **VSCode层次关系**

层次关系如下

![https://pic4.zhimg.com/80/v2-91da9ea2ecd7f2e5ddd42dba07f4281b_720w.webp](https://pic4.zhimg.com/80/v2-91da9ea2ecd7f2e5ddd42dba07f4281b_720w.webp)

层次很清晰，即

系统默认设置（不可修改）-用户设置-工作区设置-文件夹设置

后者的设置会覆盖前者的设置，若没有设置某一项，将继续使用前者的设置。

我们可以这样理解此层次

用户设置即全局设置，用户自行设定好后，每次打开VSCode即使用的此设定，若某项无设定即使用默认设置。

工作区设置即工作环境设置，可对不同的工作环境是用不同的工作环境，若某项无设定，即使用上一层设置。

文件夹设置即为项目设置，将一个文件夹当成一个项目，对同一个工作环境下的不同项目，使用不同的设置，若某项无设定，即使用上一层设置。

即 全局 - 工作环境 - 项目

## Go管理多module

源码目录下包括多个Go的子模块，VSCode 无法直接分析主目录下的子模块，需要添加工作空间配置文件（xx.code-workspace），告诉VSCode如何去解析子模块。

在xx.code-workspace的配置修改如下：

```json
{
    "folders": [
        {
            "path": "./boss-crm"
        },
        {
            "path": "./boss-lib"
        },
        {
            "path": "./boss-test"
        }
    ],
}
```

reference:

[关于VSCode中工作区的讲解与使用工作区还你一个轻量 的VSCode](https://zhuanlan.zhihu.com/p/54770077)