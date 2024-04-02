# BST创想中心

# BST创想中心游戏后台k8s实践和思考

| 导语 本文介绍BST创想中心游戏后台上云-基础服务上云思考实践概览。主要内容包括基础架构思考和部署实践过程。希望对想了解游戏上云和即将准备尝试上云的同事有所帮助。

## **一. 前言**

**游戏业务可以理解为传统应用，本文我们讨论游戏业务和云原生。**

“上云“，是最近一年听的非常多的一个词。**云原生（Cloud+Native）：一种构建和运行应用程序的方法，是一套技术体系和方法论。** 其中技术体系主要包括了一下几个内容：

- DevOps
- 持续交付(Continuous Delivery)
- 微服务(Micro Services)
- 敏捷基础设施(Agile Infrastructure)
- 12要素(The Twelve-Factor App)
- 服务网格（Service Mesh）
- 声明式API

**与传统应用对比，有以下方面的不同**

[[后端开发/架构学习/BST创想中心/Untitled Database]]

（具体概念可以阅读[云原生浅析](http://km.oa.com/group/39344/articles/show/411074?kmref=search&from_page=1&no=1)详细了解。）

## **二. 游戏业务服务器后台特点和痛点**

### **游戏业务背景和特点**

针对游戏服务，腾讯游戏后台是各个项目独立的。整体技术体系如下图所示：

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143001_16_w836_h321.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143001_16_w836_h321.png&is_redirect=1)

> 这里列举了公共组件、公共服务、工作室具有代表性的产品和游戏，实际IEG有非常多的产品。
> 

 在工作室内各个游戏后台服务部署独立，各自的后台服务器负责各自的游戏产品。由于产品形态的不同，各游戏后台架构上各有千秋，技术栈也有所差别，工作室之间差异就更为明显。但不论是哪个游戏架构，模块划分和架构设计都会着重进行海量、容灾、灰度等等方面的设计，涉及海量服务的设计理念。

### **游戏后台痛点**

### **1. 后台服务架构方面**

为了便于理解，以下是几种常见的游戏后台架构拓扑：

a). 服务分层

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143031_32_w606_h519.jpg&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143031_32_w606_h519.jpg&is_redirect=1)

b). Proxy或Router转发内部消息

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143045_40_w629_h526.jpg&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143045_40_w629_h526.jpg&is_redirect=1)

实际后台服务拓扑更复杂，每一个模块功能看似简单，其实都有各自模块着重考虑的因素。 就从当前我们创想中心的游戏后台来看：三十多个服务为一体，提供全区全服的游戏后台服务： 这些服务里有重度状态服（gamesvrd）、轻度状态服（dbproxy）、还有无状态服（snssvrd）和主备服务（matchsvrd）。服务之间相互依赖，一个服务不可用可能造成整个功能的异常，甚至触发连锁异常。比如登录关键路径上的服务，一旦有不可用玩家就不能正常登录；PVPSvrd出现异常，玩家对局就会受影响，大量玩家不能正常对局，匹配服务可能引发雪崩。

游戏业务也属于传统应用。体现在：依赖关系、部署方式、运维方式、开发模式、扩展性等，那么我们逐一进行分析痛点。

[[后端开发/架构学习/BST创想中心/Untitled Database]]

### **2. 服务器框架设计方面**

由于游戏服务器存在的强状态和复杂逻辑，服务器框架设计也非常复杂。虽然各个项目组的技术差异；架构上有侧重点，但总体内容大概包括以下方面：

- 消息传递
    - 异步消息处理：多线程收发消息、消息分发
    - 消息队列和消费：消息缓存、下行包缓存
    - 事件注册和分发：注册消息、消费消息逻辑
    - 事件发布-订阅：如玩家事件的发布和订阅
    - 消息顺序保护：消息顺序不能乱，必须完全收发顺序一致
- 数据管理
    - 缓存缓写：LRU、内存池、对象池
    - 数据分片：多进程承载数据
    - 数据一致性：全Cache的业务逻辑，主要集中在消息路由和扩缩容期间、锁机制、版本号机制
    - 配置切片和管理：有可能有上百或上千个的配置文件，相互依赖。
- 业务扩展性设计
    - 进程线程框架：基础进程框架、线程管理
    - 业务逻辑层封装：框架层和业务层隔离性，Logic隔离
    - 事务逻辑：可能存在的事务框架或协程框架
    - 接口层封装：业务层中最上层的接口层封装，涉及消息统计、熔断和过载保护等
- 协议
    - 多端的协议设计：游戏客户端往往是多终端：移动端、web端
    - 协议向下兼容：协议必须支持向下兼容
    - 协议热更新：由于客户端版本更新的限制，协议最好考虑热更新，减少客户端更新频率
- 监控和告警
    - 存储相关监控告警：内存池、对象池阈值监控告警
    - 业务逻辑告警：异常业务逻辑告警
    - 日志和tracing：错误日志、FATAL日志、tlog、以及染色日志
- 进程管理和部署
    - 服务管理端：可以管理整个后台集群信息的管理端或配置
    - 变更和发布管理端：需要有变更和发布的管理工具以上内容，其他互联网产品技术也都会涉及，只是业务领域的不同。

## **三. 游戏服务和传统互联网主要差异**

对比传统互联网应用，游戏服务到底有哪些不同？（以下是个人对两者的理解）

本质上，游戏服务本身是互联网的产物，是互联网应用的重要组成。 既然是服务，那就有互联网服务的共性：

- 专注实际业务场景，领域发展
- 服务平台化
- 高可用架构
- 分布式环境
- 技术快速发展

不同点在于，从产品角度看游戏服务更具内聚性，往往聚焦于自身的游戏世界。另外游戏业务期待的是差异和新颖，玩家需要不停的给他们新的游戏体验。 具体游戏类型：面向数值的游戏系统和架构围绕数值系统展开；MMOG大世界、RPG游戏围绕任务社交系统和数值养成结合的系统展开；SLG游戏又是时间线上的成长养成系统；FPS游戏则是外围成长线轻量，注重单局玩法。 除了我们眼中常见的网络游戏，放眼海外游戏开发大厂，他们更着重于单机游戏或重度单局的游戏研发，玩家也更关注于游戏局内的游戏性，他们的游戏性更纯粹。这些游戏的外围则更像一个游戏服务平台。举个例子：EA的橘子平台包括了登录、支付、成就、任务、游戏好友等等。

再回到到技术层面，技术的设计开发又是围绕游戏玩法，涵盖单局和外围系统设计，各种类型游戏都有所侧重，也都有一整套技术实现和要点：大世界视野同步、帧同步、物理引擎、副本、公会、组队、养成等等。 在技术迭代过程中，横向看，各游戏系统又非常希望能够做到通用性：通用的大视野同步服务、帧同步sserver、通用的好友群组系统、匹配系统、通用的活动任务系统、哪怕完全不相干的数值系统都想抽象成一个抽象的数值系统。别介意，这些都是游戏开发的痛点（重复性的业务系统）。 纵向看，有些游戏服务或多或少总有设计不足，俗称“坑”。运营中的项目只有在足够稳定才抽出人力去优化重构，俗称“填坑”。新开项目则结合之前项目的经验，时间充裕就大刀阔斧的夯实地基；时间不充裕则在老代码基础上东拼西凑，满足开发进度为第一优先级。 我想，和其他互联网产品可能也类似。不能说不好，其实也优势，是技术的传承！毕竟开发者眼里的世界是单纯的、是越来越美好的~

回过头再看，同是运行在分布式环境下，试想技术上游戏后台能否向互联网方向靠拢？能否同互联网其他产品一样，采用一样的模式和方法论？ 在云原生的稳健的基础设施之上，实现统一服务化的业务模式？

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595249748_30_w842_h326.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595249748_30_w842_h326.png&is_redirect=1)

## **四. 技术基础**

### **基石-稳健的云基础设施**

得益于公司最近两年自研上云的推进，我们已经有了非常稳健的云基础设施。这里包括TKE、BCS、镜像仓库、开源协同的组件。以下简单介绍一下应用到的成熟的服务和组件：

- 腾讯云容器服务（Tencent Kubernetes Engine ，TKE）作为自研上云和开源协同的统一容器服务底层提供者，助力自研业务实现云化架构。腾讯云容器服务完全兼容原生 kubernetes API ，扩展了腾讯云的 CBS、CLB 等 kubernetes 插件，为容器化的应用提供高效部署、资源调度、服务发现和动态伸缩等一系列完整功能
- 蓝鲸容器服务(BCS，Blueking Container Service) 高度可扩展、灵活易用的容器管理服务。BCS支持很多种集群模式，当然也支持TKE。 BCS同时提供了完善的基于Docker的服务生态、服务管理、配置中心、监控和日志检索。
- 腾讯云CLB（Cloud Load Balancer）CLB是对多台云服务器进行流量分发的服务，旨在帮助企业通过流量分发扩展应用系统对外的服务能力，以满足企业消除单点故障提升应用系统的可用性的需求。借助TKE的service-controller和l7-lb-controller让用户能够快速便捷地使用腾讯云clb的能力
- 蓝鲸配置BSCP蓝鲸配置服务([BSCP](http://km.oa.com/articles/show/451178?kmref=author_post)), 是蓝鲸体系内的针对微服务的配置原子服务。结合蓝鲸容器服务、蓝鲸管控平台、蓝盾流水线等服务, 提供服务配置的版本管理、版本发布和配置渲染等功能
- 镜像仓库存放版本镜像，云原生的交付内容。[http://bk.artifactory.oa.com/](http://bk.artifactory.oa.com/)

### **游戏服务器架构基础-真正适应分布式环境**

### **架构挑战**

我们游戏服务器是以TSF4G为基础组件的服务器框架。TBusd是作为基础的服务通信中间件，业务服务划分为逻辑的地址，通过TCM配置进行管理。

Tbus原理：

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143136_10_w322_h420.jpg&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143136_10_w322_h420.jpg&is_redirect=1)

通过TCM配置的服务数和部署节点的部署方式在过去十多年，对腾讯游戏发挥了非常大的作用。可以说这一套部署和执行是腾讯游戏成功的功臣，实至名归！

本质上，TCM配置服务的方式是建立在物理机或CVM的环境的产物，很好的满足了固定拓扑场景下的业务需要。静态的服务拓扑配置对在稳定的分布式环境下非常稳定，没有故障就没有任何问题。 只是如果出现机器故障则需要人工干预，从设计上不支持自动切换。

目前游戏后台要做到服务化，就要适应分布式环境的不稳定性。能够支持一定的自愈能力，支持必要的健康检查、负载调节等能力。根据以往项目经验，我们有几种方案：

[[后端开发/架构学习/BST创想中心/Untitled Database]]

### **架构调整**

proxysvrd 我们当前ago仓库已经有了一定的实现，也尝试用gRPC做底层的消息中间件，这些都是实验中的项目，应用到生产环境还是有比较大的风险。 从项目框架角度看，我们希望选择改动成本最低最安全有效的方式， 因此最终我们首先采用了TBuspp的调整。也符合tbus产品的发展趋势。

### **TBuspp提供服务之间的通信能力**

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143171_75_w625_h514.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143171_75_w625_h514.png&is_redirect=1)

### **TBuspp提供名字服务**

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143208_12_w539_h408.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143208_12_w539_h408.png&is_redirect=1)

- 采用tbuspp支持路由和链路通信
- Tbuspp NameSvr 提供名字服务
- Tbuspp-agent之间网状tcp直连
- TbusppNs和kubernetes打通，支持k8s部署

服务通信框架修改后，tbuspp托管了统一的名字服务和链路通信代理。目前应用层未做修改，开发上和之前的方式相同。服务的上下线、扩缩容能够做到动态实施。

### **两者结合**

Kubernetes 提供了基础设施来构建一个真正以容器为中心的开发环境，Kubernetes 可以在物理或虚拟机集群上调度和运行应用程序容器，实现容器管理、编排和调度。 Kubernetes满足了生产中运行应用程序的许多常见的需求，支持 应用健康检查、横向自动扩缩容、负载均衡、滚动更新等。

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595249784_3_w1011_h396.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595249784_3_w1011_h396.png&is_redirect=1)

## **五. 技术实践过程**

**上云首先的目标是将以上服务模块部署到云环境中，各模块能够实现云上的快速扩缩容。以下内容从概念开始一步步实践（编程）**

### **Step1: 容器服务（k8s基础设施）**

容器服务，对应互娱的oteam

[BCS容器服务](https://iwiki.oa.tencent.com/display/BCS)

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143284_32_w511_h305.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143284_32_w511_h305.png&is_redirect=1)

1. API和CLI，用户可以通过BCS产品页面或者直接使用API操作集群和服务。在某些特殊场合也可以通过CLI来操作
2. 容器服务后台模块，包括集群管理服务、网络服务、健康管理、Metrics管理、路由调度、存储
3. 容器及微服务生态工具，包括DNS、Loadbalance、服务发现、分布式配置中心和镜像仓库
4. 基于原生Kubernetes的集群，包含了分布式驱动接口，及Kubernetes相关的组件
5. 基于Mesos的集群，包含了分布式驱动接口，自研的调度器，以及Mesos相关的组件
6. 容器网络相关插件，包含了主流的CNI工具实现

### **Step2: Docker容器和镜像**

Docker是Docker Inc公司开源的一项基于Ubuntu LXC技术构建的应用容器引擎，源代码托管在GitHub上，完全基于go语言开发并遵守Apache2.0协议开源。Docker可以让开发者打包应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的Linux版本机器上，也可以实现虚拟化。 Docker基于 Linux 内核的cgroup，namespace和unionFS。namespace 是用来做资源隔离, cgroup 是用来做资源限制。

我们项目中采用docker镜像的交付方式。docker的应用要着重理解cgroup和namesapce的限制和隔离，限制和隔离会影响容器内服务的能力（如网络管理、进程PID共享、ptrace等）。

这里简单介绍Dockfile：

`FROM bk.artifactory.oa.com:8080/paas/public/tlinux2.2:latest
MAINTAINER denrenchen

ARG TIMEZONE=Asia/Shanghai
#ARG TIMEZONE=UTCRUN ln -sf /usr/share/zoneinfo/$TIMEZONE /etc/localtime
RUN echo $TIMEZONE > /etc/timezone

WORKDIR /pangu
COPY bin/ /pangu/RUN chmod 777 -R /pangu
RUN chmod +x /bin/killall
RUN chmod +x /bin/strace
RUN useradd -d /data -g docker -m  docker
RUN chown -R docker:docker /pangu
USER docker

CMD ["/pangu/start.sh"]`

说明：

- FROM: 定制的镜像都是基于 FROM 的镜像
- COPY: 打包文件到镜像
- ENV: 设置环境内环境变量
- ARG: 指定一些参数
- RUN: 构建镜像时运行的Shell命令
- CMD: 容器启动后执行的指令
- VOLUME:指定容器挂载点到宿主机自动生成的目录或其他容器
- USER:为RUN、CMD和ENTRYPOINT执行Shell命令指定运行用户
- WORKDIR:设置工作目录

以上Dockfile，是将我们bin目录完整的Copy到容器的/pangu 目录，切换用户docker执行start.sh. 我们通过docker push 到[http://bk.artifactory.oa.com/](http://bk.artifactory.oa.com/) 镜像仓库。 后续k8s部署将用到此镜像仓库。

### **Step3: Kubernetes应用部署**

K8S 里所有的资源或者配置文件都可以用yaml 或Json 定义。

对于容器镜像，我们可以通过yaml文件描述某个应用镜像如何部署。在我们应用里主要用Deployment 、StatefulSet、Service、DaemonSet 对象。

- Deployment可以理解面向无状态服务，所管理的Pod的IP、名字，启停顺序等都是随机的。
- StatefulSet则是面向有状态服务，有固定的Pod名称，启停顺序。以下是部分yaml配置：

Deployment:

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143349_17_w455_h464.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143349_17_w455_h464.png&is_redirect=1)

StatefulSet:

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143365_88_w554_h485.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595143365_88_w554_h485.png&is_redirect=1)

具体yaml配置建议逐一理解，一般第一次配置比较费力（可以用helm create 生成基础配置），后续都是在此基础上不断调整。

在我们服务中，一个Pod会有几种k8s对象yaml配置：

	`deployment.yaml
	hpa.yaml
	ingress.yaml
	service.yaml
	destinationrule.yaml
	vituralservice.yaml`

后台服务一共33个应用，其中的yaml文件中有非常多的参数变量，一共33*6个yaml配置

### **Step4: Helm 应用部署管理**

Helm是一个应用于K8s的包管理器，类似于YUM或者APT，用来简化应用的部署和管理。我们知道在k8s中各种资源是用yaml来描述，而helm可以把yaml模板化，让运营人员根据需要配置并实例化应用。 Helm将原生应用程序涉及到的众多K8s资源对象打包成一个所谓的Chart，以此实现统一的管理 对于应用发布者而言，可以通过Helm来打包应用，管理应用依赖关系，管理应用版本，发布到应用仓库 对于应用使用者而言，使用Helm后无需手动编写Manifests文件，通过简单的操作即可完成对应用的安装，升级，回滚，卸载 具体可以参考以下链接详细理解：

1. [https://helm.sh/docs/](https://helm.sh/docs/)
2. [https://iwiki.oa.tencent.com/pages/viewpage.action?pageId=188780849](https://iwiki.oa.tencent.com/pages/viewpage.action?pageId=188780849)
3. [http://devops.oa.com/console/bcs/agame/helm/list](http://devops.oa.com/console/bcs/agame/helm/list)

Helm 本质上是一个面向k8s对象的模板化工具，将应用中的变量模板化:

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595170324_48_w1366_h829.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595170324_48_w1366_h829.png&is_redirect=1)

这里为了简化部署，我们将所有服务的yaml配置放到一个Chart下，通过values.yaml 对服务进行配置。BCS helm本地环境配置好后，通过helm push 推送到Chart仓库：

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595157181_57_w1616_h556.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595157181_57_w1616_h556.png&is_redirect=1)

BCS web页面上可以进行chart的部署和更新，以下是几个测试环境的部署Release：

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595157219_41_w1623_h831.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595157219_41_w1623_h831.png&is_redirect=1)

Deployment

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595144163_45_w554_h474.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595144163_45_w554_h474.png&is_redirect=1)

至此业务服务都已经部署到集群内。那么如何进入容器查看服务是否运行正常？

实际上我们在部署过程中也遇到了非常多的问题，期间也是不断学习理解和修改。过程中经常去查找错误日志和异常信息。 下面介绍一些k8s常用的管理工具。

### **Step5: K8s管理工具和问题定位**

### **日常pod操作(master node上root执行)**

- 查询Pods列表:kubectl get pods --namespace=apgame-k1
- Pod详细信息:kubectl get statefulset/pods -n apgame-k1kubectl describe pod --namespace=apgame-k1 apgame-2006240659-gamesvr-0
- 调试pods:kubectl exec -n apgame-k1 -it -c=gamesvr apgame-2006240659-gamesvr-0 -- /bin/bash
- 快捷指令 :kube_alias.bash: kubepods kubedesc kubeexe

### **异常信息查询**

- 启动rs时就出现问题 (在master上执行)kubectl describe rs apgame-2006150421-aassvr -n apgame-k1kubectl describe replicaset -n apgame-k1 apgame-2006220940-lobby-56d5d756bf-xz6gg
- pod 直接Crash 查看docker日志或者是pod 起来了，但其中某个容器没有异常，怎么查日志？
    1. kubectl describe 查看对应pod在哪个节点上，以及错误的容器id
    2. 到对应的node机器上
    3. docker ps -a 找到上次crash的容器的日志 还留在本地如 tbuspp日志：docker cp ${dockerId}:/data/tbuspp/server/log ./或者给tbuspp的容器加cmd sleep 100000

### **Step6: 上云部署流程和流水线**

综合以上内容描述，总结下我们当前项目上云部署流程：

1. Dockerfile打包镜像、推送到仓库变更版本：AServer/dockers/build/config.sh打包镜像：AServer/dockers/build/build.sh推送镜像：AServer/dockers/build/push.sh
2. Helm values.yaml 修改镜像版本、chat版本、推送到仓库镜像版本：tag: 1.1.49 -> tag: 1.1.50Chats版本：version: 1.1.103 -> version: 1.1.104推送仓库：AServer/dockers/deployments/helm/push_helm.sh
3. BCS helm 部署更新

核心流水线执行步骤：

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595144193_96_w302_h431.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595144193_96_w302_h431.png&is_redirect=1)

[[后端开发/架构学习/BST创想中心/Untitled Database]]

### **Step7: 上云部署网络方案**

服务环境都部署到k8s后，如何能够真正用起来，如何验证我们的服务？ 这个问题首先要解决外部访问集群的网络方案。 这里直接抛出我们应用目前的网络方案。

### **CLB介绍和能力：**

### **腾讯云CLB（Cloud Load Balancer）**

CLB是对多台云服务器进行流量分发的服务，旨在帮助企业通过流量分发扩展应用系统对外的服务能力，以满足企业消除单点故障提升应用系统的可用性的需求。 借助TKE的service-controller和l7-lb-controller让用户能够快速便捷地使用腾讯云clb的能力

### **自研云CLB 的使用：**

> 蓝鲸容器服务clb使用指引
> 
> 
> [蓝鲸容器平台(TKEx-IEG)上服务暴露的几种方式](http://km.oa.com/group/26966/articles/show/437620)
> 
- 支持clb直接转发到underlay IP容器，支持clb转发到服务的nodePort
- 通过Service方式使用clb功能
- 通过Ingress方式使用clb功能
- [clb-controller](https://github.com/Tencent/bk-bcs/tree/1.15.x/docs/features/bcs-clb-controller)
    - clb controller根据服务发现结果，动态绑定clb到服务实例，从而实现集群流量的导入
    - 同时clb-controller支持Statefulset的端口转发，让每个Statefulset pod映射到clb实例的独立端口
    - 通过自定义的clb ingress规则，可以支持腾讯云clb可设置的大部分参数
    - 支持clb直接转发到underlay IP容器，支持clb转发到服务的nodePort
    - 周期性收敛后端服务信息变化，防止大面积容器漂移的时候引发的阻塞和抖动
    - 支持statefulset映射端口段
    - **目前使用 bcs-ingress-controller**

[https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595157012_58_w554_h161.png&is_redirect=1](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fpictures%2F202007%2F1595157012_58_w554_h161.png&is_redirect=1)

对于我们游戏业务，主要对外端口是tconnd和ds端口，tconnd目前和lobby进程部署在同一pod，ds进程和gamesvr部署在同一pod。

- pod-lobby 采用clb-controller NodePort的方式对外端口映射，后续会改为statefulset映射端口段。
- pod-gamesvr 用statefulset的应用方式，通过clb stateful映射端口段实现外部到内部的访问。

### **Step8: 配置方案**

在一切环境就绪之后，我们同时着手配置方案的选型，涉及到代码热更新方案、挂载卷、分发校验和效率等等复杂的问题，后续有同事专题分享，这里就抛砖引玉~ 首先需求背景是，

- 我们游戏中有大量的配置需要更新。这里的配置包括策划的配置也有代码，实际上我们项目大范围的使用lua脚本。配置即代码，代码也可以作为配置去热更新。
- 由于游戏后台环境相对复杂，一般研发中的项目后台会有十几个服务器环境，包括：开发服、日构建、测试1-N、策划独占1-N，CE服、xxx冒烟服等等。 这些不同环境的配置有些是有差异的。
- 配置更新需要支持灰度、回滚等操作

以往上云之前，我们是用tcm或夸克等工具进行配置分发，在k8s上这些工具有些吃力。

### **内部配置系统比较**

[[后端开发/架构学习/BST创想中心/Untitled Database]]

七彩石1.10版本已经支持文件配置和批量tar包的配置方式，目前项目组使用七彩石配置。

### **Step10: 精华内容**

讲到这里，实际上只是我们游戏上云的Stage1，我们上云项目分为三个大的Stage：能用、好用、完美。

后面两个Stage涉及：

- [BST创想中心游戏后台k8s实践-镜像制作](https://km.woa.com/articles/view/527512)
- [BST创想中心游戏后台k8s实践-配置和配置更新](https://km.woa.com/group/35679/articles/show/495926)
- 监控方案
- [BST创想中心游戏后台k8s实践-日志采集](https://km.woa.com/articles/show/521054?ts=1631077795) (BCS+骏鹰日志采集)
- 蓝绿发布、灰度发布
- [BST创想中心游戏后台k8s实践-无损扩缩容](https://km.woa.com/articles/show/532462?ts=1639463550)
- [BST创想中心游戏后台k8s实践-DS上云](https://km.woa.com/articles/show/543367?ts=1648785487)

**这些具体技术精华是其他几位同事分别突破，期间一个多月做了大量的研究和尝试，最终初步满足我们制定的目标。这些内容都非常丰富，需要相关同事专题总结分享。**