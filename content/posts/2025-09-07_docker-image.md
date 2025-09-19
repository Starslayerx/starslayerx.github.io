+++
date = '2025-09-07T8:00:00+08:00'
draft = true
title = 'Docker - Images'
tags = ['Docker']
+++
每个 Linux 容器都基于一个镜像, 镜像重新构建运行中容器的底层定义.
要启动一个容器, 需要下载公共镜像或者创建自己的镜像.
每个镜像由一个或多个相互关联的文件系统层 layer 组成, 这些层通常与创建镜像的每个构建步骤大致一一对应.

由于镜像由独立的层构建而成, 这就对 Linux 内核提出了特殊要求: 内核必须提供 Docker 所需的驱动, 以便运行存储后端.
在镜像管理, Docker 高度依赖这个存储后端, 该后端通过与底层 Linux 文件系统通信, 用来构建并管理多个层并将它们组合成一个可用的镜像.
主要支持的后端存储有以下类型:
- Overlay2
- B-Tree File System
- Device Mapper

每一个后端都提供一个快速的 copy-on-write (CoW) 系统用于镜像管理. 包含:
- Building images
- Uploading (pushing) images to an image registry
- Downloading (pulling) images from an image registry
- Creating and running containers from an image

### Anatomy of a Dockerfile | 剖析 Dockerfile
这个文件描述了所有构建一个容器所需的步骤, 并且通常存储在项目源码的根目录里.

一个典型的 Dockerfile 看起来像下面这样, 这里是创建一个 `Node.js` 的应用镜像:
```Dockerfile
FROM node:18.13.0

ARG emali="anna@example.com"
LABEL "maintainer"=$email
LABEL "rating"="Five Stars" "class"="First Class"

USER root

ENV AP /data/app
ENV SCPATH /etc/supervisor/conf.d

# The daemons
RUN apt-get -y install supervisor
RUN mkdir -p /var/log/supervisor

# Supervisor Configuration
COPY ./supervisord/conf.d/* $SCPATH/

# Application Code
COPY *.js $AP/

WORKDIR $AP

RUN npm install

CMD ["supervisord", "-n"]
```

Dockerfile 中的每一行都会创建一个新的镜像层并由 Docker 存储.
该层包含了该命令执行后产生的所有更改, 这意味着当构建新镜像时, Docker 只需构建与先前构建不一致的层: 那些未改变的层可以重复利用.

虽然可以从一个普通的基础 Linux 镜像构建一个 Node 实例, 但也可以在 Docker Hub 上查找官方的 Node 镜像.
如果想要缩小到某个版本, 可以指定诸如 `node:18.13.0` 之类的标签, 下面这个基础镜像会提供一个在 Ubuntu Linux 上运行 Node 11.11.x 的环境:
```Dockerfile
FROM docker.io/node:18.13.0
```
其中 ARG 参数可以设置变量及其默认值, 但只在镜像构建过程中可用:
```Dockerfiles
ARG email="anna@example.com"
```
为镜像和容器添加标签 label, 可以通过键值对的形式为他们附加元数据, 这些元数据之后可以用于搜索和识别 Docker 镜像与容器.
可以使用 `docker image inspect` 命令查看某个应用的标签.
对于 maintainer 标签, 这里使用了上一行中定义的 `email` 参数的值. 在构建该镜像时, 可以随时更改这个标签:
```
LABEL "maintainer"=$email
LABEL "rating"="Five stars" "calss"="First Class"
```
默认情况下, Docker 使用 root 运行所有容器中的进程, 但是可以使用 `USER` 命令来修改:
```Dockerfile
USER root
```

> CAUTION 注意

尽管容器与底层操作系统提供了一定程度的隔离, 但他们仍运行在主机的内核之上.
由于安全风险, 在生产环境中应该总是在非特权用户的上下文中运行.

不同于 `ARG` 命令, `ENV` 命令运行你设置 shell 变量, 用于运行中的容器应用设置:
```Dockerfile
ENV API /data/app
ENV SCPATH /etc/supervisor/conf.d
```
在下面的代码中, 使用一系列 `RUN` 命令安装所需要的依赖, 以及创建所需的文件结构:
```Dockerfile
RUN apt-get -y update

# The daemons
RUN apt-get -y install supervisor
RUN mkdir -p /var/log/supervisor
```

> WARNING 警告

虽然上面的展示中更新了容器依赖, 但一般不推荐这么做.
因为爬取最新的仓库列表不同, 在 build 的时候包的版本号不一定都相同, 导致生成的镜像不是可以重复使用的.
相反, 应该使用一个已经包含所需更新的镜像, 这样更快且可重复使用.

`COPY` 命令用于从本地文件系统复制文件到镜像中. 经常会包含应用代码和任何要求的文件.
```Dockerfile
# Supervisor Configuration
COPY ./supervisord/conf.d/* $SCPATH/

# Application Code
COPY *.js* $AP/
```

> TIP 提示

之前说过, Dockerfile 中的每条命令都会创建一个新的 Docker image layer.
因此将多条命令结合到同一行中是合理的, 甚至可以使用 `COPY` 复制一个脚本, 然后使用 `RUN` 命令去运行这个脚本, 这样就使用两行命令实现了复杂的操作.

通过 `WORKDIR` 命令, 可以修改镜像的工作目录:
```
WORKDIR $AP
RUN npm install
```
> CAUTION 注意

Dockerfile 中命令的顺序会对后续的构建时间产生非常显著是影响,
应该尽量将那些每次构建都会发生变化的步骤放在 Dockerfile 靠下的位置.
也就是说, 像添加代码这样的步骤应该尽量放到最后, 因为每次构件新镜像时,
从第一个发生变化的层考试, 之后的层都必须重新构建.

最终, 可以通过 `CMD` 命令在容器中运行服务了
```Dockerfile
CMT ["supervisord", "-n"]
```

> Note 笔记

虽然这不是严格要求, 但一般认为最佳实践是每个容器只运行一个进程.
这里的核心想法是, 一个容器内部应该只提供一个功能, 这样便于水平扩展.
在这个例子中, 使用 supervisord 作为一个进程管理, 从而帮助提升 node 应用的弹性, 保证容器的正常运行.
这样做对于开发其间的错误处理也很有用, 这样可以方便地重启服务, 而不是整个容器.

### Building an image 构建一个容器
