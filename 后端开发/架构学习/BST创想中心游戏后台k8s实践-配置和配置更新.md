# BST创想中心游戏后台k8s实践-配置和配置更新

| 导语 容器技术在部署、交付阶段给人们带来了非常大的便利，本篇结合BST创想中心游戏后台k8s实践，总结七彩石配置中心应用工具，供大家参考。希望对想了解游戏上云和即将准备尝试上云的同事有所帮助。

# **1. 七彩石**

官网 ： [https://rainbow.oa.com/](https://rainbow.oa.com/)

用户案例：[七彩石配置中心](https://iwiki.woa.com/pages/viewpage.action?pageId=264887332)

# **2. 项目配置相关背景**

项目主要应用七彩石 文件类型 做配置和脚本管理。 首先简单介绍一下项目中配置相关的划分：

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202112%2F1639384507-7308-61b705bbb26f4-843001.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202112%2F1639384507-7308-61b705bbb26f4-843001.png&is_redirect=1)

### **部署环境配置(yaml)**

- yaml文件中的 values.yaml 相关配置：发布相关，一般一个环境确定了 values 直接在环境中修改。 目前采用流水线更新的方式
- 流水线生成Infra配置：这部分是在创建一个环境后，根据流水线参数生成的配置，不会变化。

### **业务脚本和配置**

- 业务脚本代码：
- 目前全部托管到七彩石管理，有对应的流水线进行全量和增量更新。
- 需要区分AB版本
- template模板生成的配置： 生成全量配置时，不同的环境业务配置通过业务自定义的tempate生成 。

## **镜像内容**

镜像中目前包括 二进制bin/so 和bash脚本，不存在配置内容。

# **3. 七彩石应用**

七彩石容器应用我们和火影的应用实施一样，采用七彩石推荐的做法。详细可以参考 curryxie的文章:

[游戏项目如何使用配置中心 -- 七彩石在RED中的实践](https://km.woa.com/articles/show/473971)

# **4. 一些项目中的应用工具**

## **4.1 分组创建**

在多环境下，每新增一个分组都要到七彩石管理端创建分组。 像我们项目的应用，每一个命名空间都需要创建一个独立的分组。 七彩石提供了admin工具，可以通过admin创建分组，发布等操作（相关代码见附件，改改应该就能用）：

`./aptoolkits rainbow --config=../rainbow/config/rainbow_config.toml --creator=denrenchen create_group --group_name=k8s.devtest-v2.pangusvr`

## **4.2 文件变更diff**

七彩石配置变更，对于文件类型，提交一次配置更新，rainbow-agent是增量更新，将变化的配置同步到临时目录：

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202112%2F1639383499-9957-61b701cbf31a0-135987.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202112%2F1639383499-9957-61b701cbf31a0-135987.png&is_redirect=1)

这里有个小问题， 由于我们业务检查文件变化是根据文件修改时间检查的，如果直接cp整个临时目录下的文件到 volumes，那相当于整个目录文件都发生了变化。 我们的做法是，给七彩石添加一个工具，专门做目录下文件diff。 我们只cp diff差异的文件。原理很简单：

- 1. BeforeShell中 计算volumes 目录下文件MD5
- 2. AfterShell中，对比临时目录文件和volumes 差异，对差异文件进行cp；修改锚点文件，触发业务进程reload

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202112%2F1639383859-202-61b7033331578-434334.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202112%2F1639383859-202-61b7033331578-434334.png&is_redirect=1)

## **4.3 扩展rainbow-agent**

以上BeforeShell和AfterShell相关工具 我们打包到新的rainbow镜像

`FROM mirrors.tencent.com/rainbow/rainbow-agent:v1.2.5RUN mkdir -p /usr/local/rainbow-agent/apgame_tools
COPY apgame_tools /usr/local/rainbow-agent/apgame_tools`

最后更新于 2021-12-14 13:01