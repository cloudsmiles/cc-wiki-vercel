# Docker镜像瘦身

总而言之我们有下面的几种方式来共同作用减少容器的体积：

- 使用合适的小体积的基础镜像
- 减少docker镜像的layer的数量
- 镜像只包含程序运行所需的最小的资源
    - 排除不必要的资源
    - 按照程序语言的运行特性做对应的资源裁剪
- 使用多阶段构建技术

# 使用合适的小体积的基础镜像

这个是最容易做到并且很容易理解的一种常见策略,也就是将`FROM xxx`语句中使用的基础镜像在满足程序运行性能和稳定性的前提下越小越好。

举个例子，我有一个Java项目需要运行，那么我需要引入一个openjdk镜像做基础镜像，那么我们就需要考虑用那种openjdk镜像做为基础最合适.

下面我列举如下的几个选项,这些选项可以在[docker hub openjdk](https://link.juejin.cn/?target=https%3A%2F%2Fhub.docker.com%2Fsearch%3Fq%3Dopenjdk%26type%3Dimage)上检索到.

| TAG | Compressed Size |
| --- | --- |
| openjdk:11-jre-slim | 76.39 MB |
| openjdk:11-jre | 117.76 MB |
| openjdk:11-slim | 225.52 MB |
| openjdk:11 | 318.71 MB |
| openjdk:17 | 231.66 MB |
| openjdk:17-slim | 210.62 MB |
| openjdk:17-alpine | 181.71 MB |

很明显，我们当我们运行的项目需要`JDK11`的情况下,那么`openjdk:11-jre-slim`是最佳选择,如果我们的项目是`JDK17`的情况下，那么`openjdk:17-slim`是最佳选择。

# 减少docker镜像的layer的数量

Docker镜像是由很多镜像层(Layers)组成的(最多127层),Dockerfile中的每条指定都会创建镜像层，但是只有`RUN`, `COPY`, `ADD`会使镜像的体积增加.

> Only the instructions RUN, COPY, ADD create layers. Other instructions create temporary intermediate images, and do not increase the size of the build.
> 

所以我们在使用上面讲的三个指令的使用，就需要特别注意尽量把它们合并为一条shell语句，而不是每个语句一行

例如下面的用法

```docker

...

RUN apt update -y
RUN apt install curl -y

...

```

我们就可以合并为

```docker

RUN apt update -y && apt install curl -y

```

这样在理论上减少了一个layer. 此外对于上面的2个指令，合并还能获取避免意料之外的问题

# 镜像只包含程序运行所需的最小的资源

我们在使用Dockerfile构建docker镜像的时候，可能会产生一些中间文件或者由于考虑步骤意外的引入了程序运行无关的文件，这个也需要考虑

## 首先是排除不必要的资源

我们的程序只需要它需要的资源，一切与他无关的文件，比如某些图片,文档等，我们可以使用[.dockerignore.](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.docker.com%2Fdevelop%2Fdevelop-images%2Fdockerfile_best-practices%2F%23exclude-with-dockerignore)排除出去，原理和`.gitignore`一样

## 按照程序语言的运行特性做对应的资源裁剪

有时我们在构建的时候需要安装某种linux工具，比如为有些`slim`版本的linux安装`curl`执行内部的健康检查，这是我们执行`apt`/`dnf`包管理软件的时候，可能无意间产生了大量的临时缓存，这很容易被无意间打包到镜像中，所以我建议使用结合指令合并技术加上`rm -rf /var/lib/apt/lists/*`来立即在软件安装完后清理缓存，比如我安装`curl`时会这样写`RUN apt update -y && apt install -y curl && rm -rf /var/lib/apt/lists/*`

此外在经验不足的情况下，我们没有具体的语言具体分析，比如`golang`它是一种纯粹的编译型语言，在程序被编译出来后，它只需要linux基础环境即可运行

有些人可能会这样写`FROM golang:1.17`并以为它作为golang的运行环境是最合适的，其实未必，golang编译出的二进制可执行程序，只需linux环境即可，我们就可以先编译golang并`FROM debian:bullseye-slim` 即可，它能有更小的体积并完全不影响正常的运行性能.

还有就是nodejs,python3这类解释性语言，我们需要在运行前执行包管理程序下载依赖包，然后程序才能运行，但是实际上程序的运行只需要有程序本身和依赖包即可，包管理器和包管理器执行过程产生的缓存以及临时文件是我们不需要的，我们也可以使用分阶段构建技术(后文会讲)来在后续的阶段移除这些运行无关的文件，哪怕它们在构建早期是如此重要。

此外解释性语言在安装包时可能会附带一些调试和附加工具,它们在开发阶段很实用但是正式环境运行时不再必要，我们最好也是要注意一下只安装生产环境级别的依赖，排除一些辅助性的开发组件。

# 使用多阶段构建技术

[multi-stage builds](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.docker.com%2Fdevelop%2Fdevelop-images%2Fdockerfile_best-practices%2F%23use-multi-stage-builds)是我们构建过程中相对来将复杂一段的技术，但是也异常的实用，比如前面讲的nodejs等解释性语言就可以充分利用本技术来瘦身。当然java,go等编译性语言也能受益于此技术

它的核心思想是：将Docker镜像的构建分为多个阶段，下一个阶段依赖上一个阶段的输出，将上一个阶段的输出作为输入来排除掉上个阶段构建过程中无法排除的废弃文件

通常会分为2个大致的阶段 `编译/依赖下载`->`实际的镜像构建`

以官方的例子为例

golang的构建，第一阶段使用golang基础镜像做依赖的下载和编译工作，第二阶段只复制上一阶段产生的二进制文件并使用更加精简的`debian scratch`镜像作为实际的运行环境

```docker
# syntax=docker/dockerfile:1FROM golang:1.16-alpine AS build

# Install tools required for project# Run `docker build --no-cache .` to update dependenciesRUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock# These layers are only re-built when Gopkg files are updatedCOPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependenciesRUN dep ensure -vendor-only

# Copy the entire project and build it# This layer is rebuilt when a file changes in the project directoryCOPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer imageFROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]

```

nodejs构建，第一步下载依赖，第二步只将上一阶段的依赖复制过来，排除了上一阶段的npm和npm运行过程中的中间文件，同时使用更精简的`slim`镜像来提供实际的运行环境

```docker

FROM node:14 AS build
WORKDIR /srv
ADD package.json .
RUN npm install

FROM node:14-slim
COPY --from=build /srv .ADD . .
EXPOSE 3000CMD ["node", "index.js"]
```

reference

[Docker 镜像瘦身技巧 - 掘金](https://juejin.cn/post/7074981052233711647)