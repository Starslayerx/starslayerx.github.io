+++
date = '2025-09-07T8:00:00+08:00'
draft = false
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

ENV AP=/data/app
ENV SCPATH=/etc/supervisor/conf.d

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
ENV API=/data/app
ENV SCPATH=/etc/supervisor/conf.d
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

> Note 注意

虽然这不是严格要求, 但一般认为最佳实践是每个容器只运行一个进程.
这里的核心想法是, 一个容器内部应该只提供一个功能, 这样便于水平扩展.
在这个例子中, 使用 supervisord 作为一个进程管理, 从而帮助提升 node 应用的弹性, 保证容器的正常运行.
这样做对于开发其间的错误处理也很有用, 这样可以方便地重启服务, 而不是整个容器.

### Building an image 构建一个容器
下面通过一个例子来说明, 首先克隆这个 git 仓库
```
git clone http://github.com/spkane/docker-node-hello.git \
--config core.autocrlf=input
```
上面 `--config core.autocrlf=input` 这个参数用于设置如何处理文本文件的结束符:
- Git 在提交代码时会将行结束符转换为 LF
- 但在检出代码时不会进行任何转换

之后使用命令显示所有文件
```
tree -a -I .git
```
其中  
- `-a` 表示 all, 显示所有文件.
- `-I .git` 表示 Ignore, 忽略 `.git` 模式的文件或目录

得到如下目录结构
```
.
├── .dockerignore
├── .gitignore
├── Dockerfile
├── index.js
├── package.json
└── supervisord
    └── conf.d
        ├── node.conf
        └── supervisord.conf
```
- `.dockerignore` 文件定义不想上传到 docker 镜像中的文件或目录, 文件中写了 `.git`, 这会指示 docker 在 build 的时候, 排除 `.git` 目录.
- `packages.json` 定义了 Node.js 应用并列出了它所依赖的包
- `index.js` 是应用程序的主源代码
- `supervisord` 目录包含用于启动和监控该应用的 supervisord 配置文件

下面运行构建命令
```
docker image build -t example/docker-node-hello:latest .
```
为了提升构建速度, docker 在认为安全的时候会使用本地缓存.
但有时一些底层的修改, 并不会被检测到.
当看到 `⇒ CACHED [2/8] RUN apt-get -y update` 这样的内容, 就说明 docker 在使用缓存.
可以使用 `--no-cache` 命令禁止使用缓存构建.


### Running Your Image 运行镜像
一但成功构建镜像, 就可以使用如下命令运行镜像
```
docker container run --rm -d -p 8080:8080 example/docker-node-hello:latest
```
其中  
- `--rm`: 是使容器退出时自动删除, 避免系统中堆集大量已停止的容器, 适用测试或临时运行容器
- `-d`: detached 模式, 让容器在后台运行
- `-p 8080:8080`: 端口映射 `主机端口:容器端口`, 这里并不依赖 Dockerfile 里面的 EXPOSE 命令

如果一切正常, Node.js 应用程序就会在宿主机上的容器中运行.
可以通过执行 `docker container ls` 来检测容器是否在运行.

通过可以通过以下方式确定 Docker 主机的 IP 地址:
- 查看 `docker context list` 命令输出中带有星号 `*` 标记的条目
- 或者检查环境变量 `DOCKER_HOST` 的值

如果 Docker 端点设置为 Unix socket, 那么 IP 地址就通常是 `127.0.0.1`
```
NAME              DESCRIPTION                               DOCKER ENDPOINT                                     ERROR
default           Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux *   Docker Desktop                            unix:///Users/starslayerx/.docker/run/docker.sock
```
之后访问 [http://127.0.0.1:8080](http://127.0.0.1:8080) 就可以访问服务, 看到以下内容
```
Hello World. Wish you were here.
```

#### Build Arguments 构建参数
如果 inspect 检查刚刚构建的镜像, 可以看到 maintainer 的 label 是 anna@example.com:
```
docker image inspect \
    example/docker-node-hello:latest | grep maintainer
```
如果想要修改 maintainer 这个标签, 可以简单的运行 build 命令, 并使用 `--build-arg` 参数附带一个新的 email RAG, 例如这样:
```
docker image build --build-arg email=me@example.com \
    -t example/docker-node-hello:latest .
```
构建完成后可以再次 inspect 会发现 maintainer 的 label 就被修改了

#### Environment Variables as Configuration 环境变量作为设置
阅读 `index.js` 文件, 可以看到涉及到了 `$WHO` 变量, 这里根据环境变量决定 Hello 后面跟什么:
```Javascript
let DEFAULT_WHO = "World";
let WHO = process.env.WHO || DEFAULT_WHO

app.get('/', function(req, res) {
    res.send('Hello' + WHO + '.Wish you were here.\n');
});
```

> Note 注意

可以通过 [Go template](https://developer.hashicorp.com/nomad/tutorials/archive/go-template-syntax) 来格式化 `docker container ls` 的输出只感兴趣的部分.
例如 `docker container ls --format "table {{.ID}}\t{{.Image}}\t{{.Status}}"`.
此外, 还可以使用 `docker container ls --quiet` 不带 format options 将会限制只输出容器 ID.

然后, 使用之前输出得到的 container ID 可以停止运行容器:
```
docker container stop bc26bfd2b4a8
```

> TIP 提示

使用 `docker container ls` 命令和使用 `docker ps` 命令在功能上是相同的.

可以通过 `--env` 参数重启容器来添加环境变量
```
docker container run --rm -d \
    --publish model=ingress,published=8080,target=8080 \
    --env WHO="Sean and Karl" \
    example/docker-node-hello:latest
```
这时候刷新浏览器, 应该能够看到这样的内容
```
Hello Sean and Karl. Wish you were here.
```

> Note 注意

可以这样缩短上面命令
```
docker container run --rm -d -p 8080:8080 \
    -e WHO="Sean and Karl" \
    example/docker-node-hello:latest
```

### Custom Base Images 自定义基础镜像
基础镜像经常基于常见 Linux 发行版的 minimal installs, 例如 Ubuntu, Fedora, Alpine Linux.
对于大多数人而言, 使用 Linux 发行版官方基础镜像是一个很好的选择.

但是, 有时会需要构建自己的基础镜像, 而不是使用现成的.
一个原因就是为了在所有的硬件, 虚拟机, 容器部署保持一样的镜像.
另一个原因是为了降低镜像的大小.
完全没有必要使用完整的 Ubuntu 发行版, 例如 C 或 Go 应用.
你可能会发现, 你只需要一些常用的工具进行调试, 以及其他一些 shell 命令.

另一种位于中间的方法是使用 Alpine Linux, 该发行版很小, 作为基础镜像十分流行.
为了保持小型化, Alpine Linux 基于现代的轻量库 [musl standard library](https://musl.libc.org/) 而不是传统的 [GNU C Library (glibc)](https://www.gnu.org/software/libc).
由于镜像极小, Alpine Linux 内默认使用 `/bin/sh` 而不是 `/bin/bash`.
但如果需要, 也可以自己安装 glibc 和 bash, 这经常在 JVM 容器中发生.

### Storing Images 存储镜像
当构建完镜像后, 往往需要存储该镜像, 以便未来的部署.
一般不会在生产环境服务器上面去构建, 然后运行镜像, 而是从某个地方拉取已经构建好的镜像.

#### Public Registries 公共仓库
Docker 提供一个镜像仓库给社区的共享镜像.
包括官方 Linux 发行版镜像, WordPress 镜像之类.

如果你要发布一个镜像到互联网上, 最好的仓库是 [Docker Hub](https://hub.docker.com/).
但也有其他一些选择, 例如 Docker 刚刚流行起来, Docker Hub 还不存在时, 为了填补社区的需要, [Quay.io](https://quay.io/) 被创建了.
在那之后, Quay.io 被收购了好几次, 现在属于 RedHat.
像 Google, Github 和 SaaS 公司都有他们自己的仓库.

对于大量使用 Docker 的公司来说, 这些集中化仓库最大的缺点之一就是他们并不在应用部署所在的网络中. 这意味着应用部署时, 每一层镜像都可能需要跨跃整个互联网来传输. 网络延迟会对软件部署产生非常真实的影响, 而这些镜像一但出故障, 就可能严重影响公司按时完成部署. 这个问题可以通过良好的镜像设计来缓解, 例如将镜像分割成精简的层, 以便更容易在互联网上传输.


#### Private Registries 私有仓库
另一种很多公司采用的方法, 是在内容网络部署 Docker 镜像.
开源的 [Distribution](https://github.com/distribution/distribution) 项目提供了基础的功能.
其他私有仓库的竞争者包括 [Harbor](https://goharbor.io/) 和 [Red Hat Quay](https://www.redhat.com/en/technologies/cloud-computing/quay), 他们除了基础功能, 还提供了一套 GUI 界面和其他功能, 例如镜像校验.

#### Authenticating to a Registry 仓库认证
Docker 默认使用 Docker Hub 作为镜像仓库.

- Creating a Docker Hub acount 创建一个 Docker Hub 帐号  
  如果只是拉取镜像, 不需要登陆帐号. 但如果要避免限速, 以及上传构建的容器就需要登陆.  
  创建帐号后, 可以向 public registry 上传镜像. 在 [Account Settings](https://hub.docker.com/settings/default-privacy) 选项下面有一个 Default Privacy 选项, 可以将可见性修改有私有.

> WARNING 警告

为了更好的安全性, 应该使用 [access token](https://docs.docker.com/security/access-tokens/) 登陆 Docker Hub.

- Logging in to a registry 登陆注册中心

  使用下面命令登陆 Docker Hub
  ```
  docker login
  ```
  当成功登陆后, Docker 会在家目录下面创建一个 dotfile 来缓存信息.
  ```
  % cat ${HOME}/.docker/config.json
  ─────┬─────────────────────────────────────────────
       │ File: /Users/starslayerx/.docker/config.json
  ─────┼─────────────────────────────────────────────
    1  │ {
    2  │     "auths": {
    3  │         "https://index.docker.io/v1/": {}
    4  │     },
    5  │     "credsStore": "desktop",
    6  │     "currentContext": "desktop-linux",
    7  │     "plugins": {
    8  │         "debug": {
    9  │             "hooks": "exec"
   10  │         },
   11  │         "scout": {
   12  │             "hooks": "pull,buildx build"
   13  │         }
   14  │     },
   15  │     "features": {
   16  │         "hooks": "true"
   17  │     }
   18  │ }
  ─────┴─────────────────────────────────────────────
  ```
  可以看到, 该文件以 JSON 格式存储了用户的凭据, 该配置文件支持存储多个镜像仓库的凭据. 从现在起, 当镜像仓库需要身份验证时, Docker 会自动查询该文件, 检查是否存储了主机对应的凭据. 若存在, Docker 将会自动提交这些凭据. 值得注意的是, 这里缺失了"时间戳", 这些凭据会永久缓存, 除非主动清楚他们.  
  与登录操作类似, 若不再需要缓存某镜像仓库的凭据, 也可以执行注销操作:
  ```
  docker logout
  ```
  如果要登陆其他非 Docker Hub 的仓库, 需要提供 hostname:
  ```
  docker login someregistry.example.com
  ```
  这样将会为 `${HOME}/.docker/config.json` 文件添加该仓库信息

- Pushing images into a repository 推送镜像到仓库  

  推送镜像的第一步要确保已经登陆到了 Docker 仓库.  
  一但登陆进去, 就可以上传镜像了.
  早期使用 `docker image build -t example/docker-node-hello:latest .` 命令构建镜像, 但实际上 Docker client 将 `example/docker-node-hello:latest` 视为 `docker.io/example/docker-node-hello:latest`. 这里 `docker.io` 表示 registry 名称, 而 `example/docker-node-hello` 是 registry 里面的 repository, 包含了各种镜像.

  可以使用下面命令来方便的给镜像打标签, 使用你的 Docker Hub 用户名称替换 `${<myuser>}`
  ```
  docker image tag example/docker-node-hello:latest \
      docker.io/${<myuser>}/docker-node-hello:latest
  ```
  如果要使用新的命名重新构建镜像, 通过下面命令完成 (`-t` 参数为镜像打标签)
  ```
  docker image build -t docker.io/${<myuser>}/docker-node-hello:latest .
  ```
  第一次构建的时候会花一些时间, 但如果是重新构建, 会非常块.
  因为大多数层都已经存在 Docker server 里面了.

  接下来可以使用下面命令推送镜像:
  ```
  docker image push ${<myuser>}/docker-node-hello:latest
  ```

- Exploring images in Docker Hub  
  除了简单的流览 Docker Hub 网络流览镜像, 还可以使用 docker 搜索命令来查找镜像
  ```
  docker search node

  NAME               DESCRIPTION                                     STARS     OFFICIAL
  node               Node.js is a JavaScript-based platform for s…   13999     [OK]
  cimg/node          The CircleCI Node.js Docker Convenience Imag…   25
  circleci/node      Node.js is a JavaScript-based platform for s…   135
  bitnami/node       Bitnami container image for NodeJS              83
  kindest/node       https://sigs.k8s.io/kind node image             114
  okteto/node                                                        2
  eclipse/node       Node 0.12.9                                     1
  chainguard/node    Build, ship and run secure software with Cha…   0
  sitespeedio/node   Node base template                              3
  corpusops/node     https://github.com/corpusops/docker-images/     0
  rootpublic/node                                                    0
  ubuntu/node        Ubuntu-based Node.js image for server-side a…   2
  setupphp/node      Docker images to run setup-php GitHub Action    0
  joxit/node         Slim node docker with some utils for dev        1
  treehouses/node                                                    2
  activestate/node   ActiveState's customizable, low-to-no vulner…   12
  alpine/node                                                        5
  vmware/node        Node.js base built on top of Photon OS          0
  wayofdev/node                                                      0
  vulhub/node                                                        0
  systemsdk/node     Docker environment with node 16 for Laravel/…   0
  openizr/node       Safer, non-root, nodeJS environment             0
  openeuler/node                                                     0
  presearch/node     Run a search node in Presearch's decentraliz…   25
  iron/node          Tiny Node image                                 28
  ```

  右侧的 OFFICIAL 表明该镜像是官方认证的镜像.  
  这意味着该镜像往往是由该应用开发的公司或者社区维护.  
  AUTOMATED 代表该镜像是通过 CI/CD 触发, 自动构建的.


- Running a Private Registry 运行一个私有的注册中心  
  建立一个基础的 registry 并不难, 但对于生产环境中使用, 应该花时间去了解一下可用的配置选项 [the open source Docker Registry (Distribution)](https://docs.docker.com/retired/). 下面使用 SSL 和 HTTP 认证构建一个简单的 registry.

  首先克隆一个 Git 仓库:
  ```
  git clone https://github.com/spkane/basic-registry --config core.autocrlf=input
  ```
  查看仓库文件
  ```
  ls basic-registry

  config.yaml  config.yml.sample  Dockerfile  htpasswd.sample  README.md  registry.crt.sample  registry.key.sample
  ```
  这个 Dockerfile 只是简单的将 Docker Hub 的上游镜像和一些本地配置写入新的镜像.  
  作为测试, 可以使用示例文件, 但不要在生产中使用.  
  如果在本地直接复制文件即可
  ```
  cp config.yaml.sample config.yaml
  cp registry.key.sample registry.key
  cp registry.crt.sample registry.crt
  cp htpasswd.sample htpasswd
  ```
  如果 Docker server 在远程服务器, 则需要修改一下 `config.yaml` 文件, 将 ip 地址修改为服务器地址, 例如这样:
  ```
  http:
    host: https://172.17.41.10:5000
  ```
  然后需要为 registry 的 IP 创建一个 SSL keypair, 一种方法是使用 OpenSSL 命令
  ```
  openssl req -x509 -nodes -sha256 -newkey ras:4096 \
      -keyout registry.key -out registry.crt \
      -days 14 -subj '{/CN=172.17.42.10}'
  ```
  最后复制 `htpasswd.sample` 文件内容到 `htpasswd`, 或者也可以使用下面命令来自定义用户名 `<username>` 和密码 `<password>`
  ```
  docker container run --rm --entrypoint htpasswd g \
      -Bbn ${<username>} ${<password>} > htpasswd
  ```
  如果一切正确, 就应该能够构建和运行 registry 了 (mac 的 5000 端口可能会被控制中心占用, 这里使用 5001 端口)
  ```
  docker image build -t my-registry .
  docker container run --rm -d -p 5001:5000 --name registry my-registry
  docker container logs registry
  ```

#### Testing the private registry 测试私有注册中心
运行后首先登陆该 registry
```Docker
% docker login 127.0.0.1:5001

Username: myuser
Password:
Login Succeeded
```

> WARNING 警告

这个 registry 容器内置了一个 SSL 密钥, 且没有使用任何外部存储.
也就是说, 其内部包含一个密钥, 而且当删除正在运行的容器时, 内部存储的镜像也会被删除.

在生产环境中, 需要让容器从密钥管理系统中获取密钥, 并且使用某种冗余的外部存储.
如果希望在开发环境中, 不同容器之间保留 registry 镜像, 可以在运行 docker 容器时加上类似参数:
```Docker
--mount type=bind, source=/tmp/registry-data, target=/var/lib/registry
```
这里是将宿主机的 `/tmp/registry-data` 目录映射到容器里的 `/var/lib/registry`, 这样镜像就会存储到宿主机, 即使容器被删除也不会丢失


现在来测试能否在本地私有 registry 上传镜像
```
% docker image tag my-registry 127.0.0.1:5001/my-registry

% docker image push 127.0.0.1:5001/my-registry
Using default tag: latest
The push refers to repository [127.0.0.1:5001/my-registry]
aa2afd4dffb6: Pushed
f4596b202dbb: Pushed
e971abc09286: Pushed
63c05075324e: Pushed
19c15db2ec23: Pushed
7ad8ac1e6be1: Pushed
97599ee5c821: Pushed
014ffb8901a5: Pushed
171a26c7bc56: Pushed
latest: digest: sha256:445f229035a87428b9e9dd1f816a29f7aaf8f8eabab220942d1aeae545e3f2f1 size: 2193
```
现在就可以 pull 同样的镜像了
```Docker
% docker image pull 127.0.0.1:5001/my-registry
Using default tag: latest
latest: Pulling from my-registry
Digest: sha256:445f229035a87428b9e9dd1f816a29f7aaf8f8eabab220942d1aeae545e3f2f1
Status: Image is up to date for 127.0.0.1:5001/my-registry:latest
127.0.0.1:5001/my-registry:latest
```
可以使用下面命令停止该容器
```Docker
docker container stop registry
```

> TIP 提示

当熟悉 Docker Distribution 后可能考虑流览 Cloud Native Computing Foundation (CNCF) 叫做 [Harbor](https://goharbor.io/) 的项目, 该项目比 Docker Distribution 更加安全可靠.


### Optimizing Images 镜像优化
这一章介绍一些镜像优化技巧, 减少内存使用并提升构建速度

#### Keeping Images Small 保持小镜像
从互联网下载 1G 的文件是人们经常担心的问题, 尤其是要部署 100+ 的结点, 且每天要多次部署新发行版的时候.
下载这些大文件很容易造成网络堵塞 network congestion, 并且部署速度慢会对产品有实在的影响.

为了方便, 大量的 Linux 容器会继承一个最小化的 Linux 发行版的基础镜像.
虽然这样很方便上手, 但并不是必须的.
容器只需要包含在宿主机内核上运行应用所需的文件, 除此之外什么都不要.

Go 是一门编译语言, 可以很方便地生成静态编译的二进制文件.
作为示例, 这里使用一个 Go 编写的 Web 应用, 该应用可以在 [Github](https://github.com/spkane/scratch-helloworld) 上找到.
运行下面命令, 然后在浏览器打开
```Docker
docker container run --rm -d -p 8080:8080 spkane/scratch-helloworld
```
一般情况下可能认为容器内会有项目文件, 但实际上并不是这样, 使用下面类似命令将容器打包
```Docker
docker container export 19931dd0fc21 -o web-app.tar
```
再使用 tar 命令, 检查容器内容
```
tar -tvf web-app.tar
```
首先注意到, 该容器内几乎没有什么文件, 且大部分都是 0 字节.
这些大小为 0 的文件, 在每个 Linux 文件中都存在, 他们在容器第一次创建时, 会自动从宿主机绑定挂载 bind-mount 进来.
除了 `.dockerenv` 之外, 这些文件都是内核正常运行所必须的文件.
在这个容器中, 唯一真正有实际大小的, 与应用相关的文件就静态编译好的 helloworld 二进制程序.

从这个实验中可以知道:  
容器只需要包含运行在底层内核之上的最小文件集和, 其他的一切都是不必要的.
不过, 在很多情况下, 为了方便排查问题, 会希望容器内有一个可用的 shell, 因此很多人会折中一下, 选择一个轻量级的 Linux 发行版来构建镜像.

> TIP 提示

如果需要经常查看镜像文件, 可以尝试一下这个工具 [dive](https://github.com/wagoodman/dive), 其提供了一个 CLI 接口便捷地查看镜像内容.

尽管可以使用 `docker container run -ti alpine:latest /bin/sh` 查看 apline image, 但对于 `spkane/scratch-helloworld` 这个镜像不行, 因为这个容器中不含 bash 或者 SSH. 这之前是用 `docker container export` 命令生成了一个 `.tar` 文件, 其中包含容器内所有文件的副本, 但本次将直接连接到 Docker 服务器并查看容器自身的 文件系统来检查它. 要做到这一点, 需要找出文件在服务器磁盘上的存放位置, 可以使用下面命令获取:
```Docker
docker image inspect <container_name>:<container_tag>
```
下面使用这个例子来演示
```Docker
docker container run --rm -it --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh
```
alpine 是一个非常小的基础镜像, 只有 4.5 MB, 非常适合在其之上构建容器.  
然而, 可以看到, 即使在还没有基于它构建任何东西之前, 这个容器里仍然包含了不少内容.

接下来, 让查看 `spkane/scratch-helloworld` 镜像中的文件. 这种情况下, 查看 `docker image inspect` 输出中 LowerDir 项的第一个目录, 注意到它第一个名为 `diff` 的目录名称:
```bash
# ls -lFh /var/lib/docker/overlay2/37…4d/diff
total 3520
-rwxr-xr-x    1 root     root        3.4M Jul  2  2014 helloworld*

# exit
```
这个目录里只有一个文件, 大小为 3.4 MB, 这个 helloworld 二进制文件是容器中唯一包含的文件, 而且它比 alpine 镜像在添加任何应用文件之前的初始大小还要小.

> Note 注意

在 Docker 服务上, 可以直接从该目录运行 helloworld 应用程序, 因为它不依赖任何文件, 除了在开发环境里, 绝不应该怎么做.
这样做能够直观地说明, 这对静态编译应用程序有多么实用.


#### Multistage Builds 多阶段构建
在很多情况下, 可以通过多阶段构建 (multistage builds) 将容器限制得更小.
这也是构建大多数生产环境容器的推荐方式.
你不必担心构建引入额外资源, 同时你仍然可以运行一个 精简的(lean)生产容器.
多阶段容器还鼓励在 Docker 内部完成构建, 这是实现构建系统可重复性的一种极佳模式.

正如 [scratch-helloworld] 应用的原作者写的那样, Docker 本身引入对多阶段构建的支持, 使得创建小型容器的过程比过去容易的多.

在过去, 要实现在多阶段构建, 必须先构建一个镜像来编译代码, 然后提取生成的二进制文件, 再构建第二个不包含构建依赖的镜像, 并把那个二进制文件注入进入. 这种方式往往很难配置, 并且在标准化的部署流水线中也不总是能开箱即用.

而如今, 只需要一个像下面这样简单的 Dockerfile 就能达到类似的效果
```Dockerfile
# Build Container
FROM docker.io/golang:apline as builder
RUN apk update && \
    apk add git && \
    CGO_ENABLED=0 go install -a -ldflags '-' \
    github.com/spkane/scratch-helloworld@latest

# Production Container
FROM scratch
COPY --from=builder /go/bin/scratch-helloworld/helloworld
EXPOSE 8080
CMD ["/helloworld"]
```
首先注意到, 这个 Dockerfile 看上去像是两个 Dockerfile 拼接起来的.
确实是这样, 但不止如此. 第一条 FROM 命令在构建阶段将变量命名为 *builder*.
之后的 FROM 语句使用 scratch 这个特殊的特殊的镜像名, 叫做 scratch, 告诉 Docker 从空镜像开始构建, 这将不会包含额外的文件.
下一行 `COPY --from=builder /go/bin/scratch-helloworld /helloworld` 将 *builder* 镜像的二进制文件直接复制到当前镜像中.

`EXPOST 8080` 这行是为了告诉用户这个服务使用的端口 ports 和协议 protocols.

这里可以添加更多阶段, 实际上, 多阶段构建之间并非必须要有联系, 他们会按顺序构建.
可以一个阶段构建 Go web API 服务的镜像, 另一个阶段构建 Angular web UI 的镜像.
最后阶段将两个镜像输出结合起来.

> TIP 提示

当构建更加复杂的镜像时, 可能会发现构建单个上下文很有挑战性.
这个 docker-buildx 插件能够支持多上下文构建, 可以用来支持一些进阶的工作流.
(build context 构建上下文是 `docker build` 时指定的目录内容, Docker 在构建镜像时, 会先把这个目录的文件打包上传给 Docker 引擎, 作为上下文, 之后 Dockerfile 里的指令就只能访问这个 `build context` 内的文件)


#### Layers Are Additive 层具有叠加性
只有深入探究镜像的构建原理, 才会发现一个不显而易见的事实: 组成镜像的文件系统图层在设计上严格遵循叠加原则.
虽然可以通过覆盖/屏蔽先前图层中的文件, 但无法真正删除这些文件.
这意味着无法通过简单删除前期步骤生成的文件来缩小镜像体积.

> Note 注意

若在 Docker 中启用试验性功能, 可以通过 `docker image build --squash` 命令将多个层压缩为单一层.
这将使所有在中间层被删除的文件最终从镜像中彻底消失, 从而有效回收被浪费的存储空间.
但问题是这样即使只更新了一行源码, 整个层都需要重新下载.

可以使用 `docker image history <image>` 命令查看镜像构建阶段的文件层信息, 例如下面这样:
```Docker
% docker image history ...
IMAGE        CREATED            CREATED BY                             SIZE
543d61c95677 About a minute ago CMD ["/usr/sbin/httpd" "-DFOREGROU…"]  0B
<missing>    About a minute ago RUN /bin/sh -c dnf install -y httpd …  273MB
<missing>    6 weeks ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]…  0B
<missing>    6 weeks ago        /bin/sh -c #(nop)  ADD file:58865512c… 163MB
<missing>    3 months ago       /bin/sh -c #(nop)  ENV DISTTAG=f36co…  0B
<missing>    15 months ago      /bin/sh -c #(nop)  LABEL maintainer=…  0B
```
上面有 3 层没有添加额外的大小, 有两层添加了很多 MB, 对于 ADD 那条命令可以理解, 是添加原始的镜像, 但 RUN 这条命令就比较奇怪了, 为什么添加一个 Apache web server 会占用这么多空间?

需要理解的核心要点是, 镜像层本质上严格遵守叠加原则. 图层一但建立, 内容就无法移除.
这意味着, 无法通过在后继图层中删除文件来缩限先前图层的体积.
当在后续图层中编辑或删除文件时, 实际上知识在新图层中用修改或删除标记覆盖了旧版本.
因此, 缩小图层体积的唯一方法是在保存图层前移除文件.

最长将的处理方式是在 Dockerfile 的单行命令中串连多个命令, 通过使用 `&&` 运算符可以轻松的实现, 该运算符作为布尔 AND 语句使用, 基本上等于"如果前一个命令执行成功，就执行这个命令". 此外, 还可以利用 `\` 运算符在命令换行后继续, 这能够提升长命令的可读性.

掌握这些后可以重写 Dockerfile:
```Dockerfile
FROM docker.io/fedora
RUN dnf install -y httpd && \
    dnf clean all
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```
现在可以重写构建镜像, 再查看大小会发现从 273 MB 缩减到了 44.8 MB.
这是非常大的空间提升, 尤其是当有多个服务器拉取镜像的时候.


#### Utilizing the Layer Cache
最后一个要介绍的构建技巧, 是关于如何让构建速度尽可能的块.
DevOps 的重要目标之一就是如何保持反馈循环尽量的紧密.
这意味着尽快使问题被发现很重要, 这样开发者就可以专注于问题代码, 而不是等待已经去做其他不相关的任务了.

在标准构建过程中, Docker 会使用层缓存机制, 尽可能避免重建已经构建且未包含明确变更的层.
由于这个机制, 在Dockerfile 中安排操作指令的顺序会显著影响构建过程的耗时.

对于需要根据代码安装依赖的项目, 例如 npm, bundle, 可以研究一下如何优化 Docker 在这些平台的构建.
这通常包括锁定依赖版本和存储代码, 这样就无需每次 build 的时候都下载依赖.

#### Directory Caching 目录缓存
BuildKit 增加到镜像构建的一个特性是目录缓存.
目录缓存是一个非常有用的, 能够加速构建速度的工具, 且不保留运行时非必要的文件到镜像中.
其核心原理是, 允许将镜像内某个目录的内容保存在特殊层中, 该层在构建时可通过绑定挂载方式使用, 并在生成镜像快照前解除挂载.
此功能常用于处理 Linux 包管理器 (apt/apk/dnf 等) 和语言依赖处理器 (npm/bundler/pip 等) 存放数据库及归档文件的目录.


> TIP 提示

[bind mount](https://docs.docker.com/engine/storage/bind-mounts/) 绑定挂载是指将宿主机上的一个现有文件或目录直接"映射"到容器内部的一个路径上. 这是一个非常简单的共享方式. 有以下特点:
- 直接访问宿主及文件系统
- 高性能, 绕过了 Docker 的存储驱动, 直接使用宿主机的文件系统, I/O 性能高
- 依赖宿主机系统, 需要一个宿主机上的绝对路径, 可移植性差

为了支持目录缓存必须开启 BuildKit (较新版本一般都默认开启).
可以使用环境变量在客户端强制使用 `DOCKER_BUILDKIT=1`
- 无缓存
```
% time docker build --no-cache -t docker.io/spkane/open-mastermind:latest .
[+] Building 49.9s (12/12) FINISHED                                                                                 docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 209B                                                                                                0.0s
 => [internal] load metadata for docker.io/library/python:3.9.15-slim-bullseye                                                      4.4s
 => [auth] library/python:pull token for registry-1.docker.io                                                                       0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [1/6] FROM docker.io/library/python:3.9.15-slim-bullseye@sha256:ffc6cb648d6993e7c90abb95c2481eb688a6842dfac29bf19e3755454632d  17.6s
 => => resolve docker.io/library/python:3.9.15-slim-bullseye@sha256:ffc6cb648d6993e7c90abb95c2481eb688a6842dfac29bf19e3755454632da  0.0s
 => => sha256:662193c72f528eec2405a0519d470f5ba9091b8f430b22a3b49c55a8447af08a 1.37kB / 1.37kB                                      0.0s
 => => sha256:34a0c56576530bce4b6eb782044f19c72533a5ae02b2b8df527164f3a3eb0f5b 7.50kB / 7.50kB                                      0.0s
 => => sha256:6064e7e5b6afa4dc711228eddfd250aebac271830dc184c400ce640020bc2cb0 30.06MB / 30.06MB                                   16.1s
 => => sha256:23e07e2954939698377b8fe1a859b2d8d0ed4999c7a6da4c983084d50ca4dbe3 1.07MB / 1.07MB                                      6.4s
 => => sha256:b151283f362b5371164a098a8d4d6d6fdac38a5b1dbd8a89b05e200df14e32c3 11.59MB / 11.59MB                                   15.9s
 => => sha256:ffc6cb648d6993e7c90abb95c2481eb688a6842dfac29bf19e3755454632daa2 1.86kB / 1.86kB                                      0.0s
 => => sha256:bc8cbd54c67de1e886c60718641e2ca3d998c39507cf1912dfa7174999ab8497 234B / 234B                                          7.2s
 => => sha256:41d4a1e4080eccd6dce7443cfcf0bd3555cd73e8a962fd672e4a786434739775 3.18MB / 3.18MB                                     14.2s
 => => extracting sha256:6064e7e5b6afa4dc711228eddfd250aebac271830dc184c400ce640020bc2cb0                                           0.9s
 => => extracting sha256:23e07e2954939698377b8fe1a859b2d8d0ed4999c7a6da4c983084d50ca4dbe3                                           0.1s
 => => extracting sha256:b151283f362b5371164a098a8d4d6d6fdac38a5b1dbd8a89b05e200df14e32c3                                           0.3s
 => => extracting sha256:bc8cbd54c67de1e886c60718641e2ca3d998c39507cf1912dfa7174999ab8497                                           0.0s
 => => extracting sha256:41d4a1e4080eccd6dce7443cfcf0bd3555cd73e8a962fd672e4a786434739775                                           0.2s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 69.19kB                                                                                                0.0s
 => [2/6] RUN mkdir /app                                                                                                            0.4s
 => [3/6] WORKDIR /app                                                                                                              0.0s
 => [4/6] COPY . /app                                                                                                               0.0s
 => [5/6] RUN pip install -r requirements.txt                                                                                      26.9s
 => [6/6] WORKDIR /app/mastermind                                                                                                   0.0s
 => exporting to image                                                                                                              0.5s
 => => exporting layers                                                                                                             0.5s
 => => writing image sha256:d1c84698c57b65e2d4e646c179ad47a15bad22386c6468b3c5c73d88be065dd9                                        0.0s
 => => naming to docker.io/spkane/open-mastermind:latest                                                                            0.0s

View build details: docker-desktop://dashboard/build/desktop-linux/desktop-linux/clviefrt28zp2uvhc9utddc09

What's next:
    View a summary of image vulnerabilities and recommendations → docker scout quickview
docker build --no-cache -t docker.io/spkane/open-mastermind:latest .  0.52s user 0.41s system 1% cpu 50.602 total
```
- 有缓存
```
% time docker build -t docker.io/spkane/open-mastermind:latest .
[+] Building 1.1s (11/11) FINISHED                                                                                  docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 209B                                                                                                0.0s
 => [internal] load metadata for docker.io/library/python:3.9.15-slim-bullseye                                                      1.1s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [1/6] FROM docker.io/library/python:3.9.15-slim-bullseye@sha256:ffc6cb648d6993e7c90abb95c2481eb688a6842dfac29bf19e3755454632da  0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 2.46kB                                                                                                 0.0s
 => CACHED [2/6] RUN mkdir /app                                                                                                     0.0s
 => CACHED [3/6] WORKDIR /app                                                                                                       0.0s
 => CACHED [4/6] COPY . /app                                                                                                        0.0s
 => CACHED [5/6] RUN pip install -r requirements.txt                                                                                0.0s
 => CACHED [6/6] WORKDIR /app/mastermind                                                                                            0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:d1c84698c57b65e2d4e646c179ad47a15bad22386c6468b3c5c73d88be065dd9                                        0.0s
 => => naming to docker.io/spkane/open-mastermind:latest                                                                            0.0s

View build details: docker-desktop://dashboard/build/desktop-linux/desktop-linux/fum8hqxpzp4ezege0qoxrd0q3

What's next:
    View a summary of image vulnerabilities and recommendations → docker scout quickview
docker build -t docker.io/spkane/open-mastermind:latest .  0.25s user 0.53s system 24% cpu 3.223 total
```
从上面可以看出, 无缓存使用了 50.6 秒, 有缓存使用了 3.2 秒.

下面将 Docker 修改成这样
```Dockerfile
# syntax=docker/dockerfile:1
FROM python:3.9.15-slim-bullseye

RUN mkdir /app
WORKDIR /app

COPY . /app

RUN --mount=type=cache,target=/root/.cache pip install -r requirements.txt

WORKDIR /app/mastermind

CMD ["python", "mastermind.py"]
```
这里有两个修改:
- `#syntax=docker/dockerfile:1` 这个告诉 Docker 使用新版的 [Dockerfile frontend](https://hub.docker.com/r/docker/dockerfile), 这个版本提供了对 BuildKit 的新功能.
- `RUN --mount=type=cache,target=/root/.cache pip install -r requirements.txt` 这一行告诉 BuildKit 在构建期间, 将缓存层挂载到容器的 `/root/.cache` 目录. 这样既能让最终生成的镜像不包含该目录内容, 又能在后续构建时重新挂载缓存供 pip 使用.
