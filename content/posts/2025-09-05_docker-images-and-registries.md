+++
date = '2025-09-05T8:00:00+08:00'
draft = false
title = 'Docker - Images and Registeries'
categories = ['Note']
tags = ['Docker']
+++

### The Docker Images | Docker 镜像

Image、OCI Image、Docker Image、Container Image 都是指同一个概念镜像的不容叫法.

镜像是一个轻量、只读且不可变的蓝图, 指定了应用运行所谁要的一切, 以及在 Docker 系统上如何运行.
就像是一份配方, 包括所有必要的原料, 诸如依赖、配置、环境设置和你的应用代码, 以及确保应用每次都能稳定运行的详细指令.

可以把镜像类比为面向对象编程中的类: 定义结构和行为, 但不能直接与类交互, 需要创建实例.

#### Pulling and Inspecting an Image 拉取并查看镜像

镜像其实就是一个 JSON 对象, 可以这样拉取一个镜像

```
% docker pull celery:latest
latest: Pulling from library/celery
ef0380f84d05: Pull complete
ada810c79ed7: Pull complete
4608a1c4fe47: Pull complete
58086cbb21fb: Pull complete
a7bccb4a3faa: Pull complete
9de06a08ec25: Pull complete
ad6feb8c6a6b: Pull complete
7568ca85d492: Pull complete
2d6f458f7411: Pull complete
Digest: sha256:5c236059192a0389a2be21fc42d8db59411d953b7af5457faf501d4eec32dc31
Status: Downloaded newer image for celery:latest
docker.io/library/celery:latest

What's next:
    View a summary of image vulnerabilities and recommendations → docker scout quickview celery:latest
```

现在查看镜像信息

```
% docker image inspect celery
```

这个命令会以 JSON 格式输出关于镜像的详细信息

```JSON
[
    {
        "Id": "sha256:e111a70eee6a42a68768ec0734a6b2f1ceab34b27e4c4325ed6a14d4f36c2568",
        "RepoTags": [
            "celery:latest"
        ],
        "RepoDigests": [
            "celery@sha256:5c236059192a0389a2be21fc42d8db59411d953b7af5457faf501d4eec32dc31"
        ],
        "Parent": "",
        "Comment": "",
        "Created": "2017-06-19T16:53:14.832853892Z",
        "DockerVersion": "17.03.1-ce",
        "Author": "",
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 215773711,
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/48895e03352004bc8ce3d6a639e808ba64b209349af83a3c75b481be2ed94938/diff:/var/lib/docker/overlay2/c845387bc00200c8fb7eb3e3305dba41fdb4ff2d9aa6c085fb0a6ad24c290c73/diff:/var/lib/docker/overlay2/17756520f50f2cbf3e556f44abb1c0b5b6789cd6f6df47246baaabcc3c14d8e0/diff:/var/lib/docker/overlay2/4d0e644e7332550ce97905e7784f5d9f96d9213c774a7204e2fd8b8aa3b11035/diff:/var/lib/docker/overlay2/5d44ed5515c803cd01048443fc3fa4d3372b48fb956d7574513f72a83ab83f66/diff:/var/lib/docker/overlay2/72ff0fa932bdc072f1beb1bc8ff782dc4b1fbde9a658307a698d4e0160d92797/diff:/var/lib/docker/overlay2/78b31a20c2e61a0d663de9c13830def1ae6d14c106dd3a78a581e0e40e7b4689/diff:/var/lib/docker/overlay2/d137d2107e48055ec9dcb33a81a136f0c6625f5e0a7439d1ccd972df8266edce/diff",
                "MergedDir": "/var/lib/docker/overlay2/4421a44d9ab8308305d3a051ca7672cb4d8e778bdbba66b8800348a805ee3dfc/merged",
                "UpperDir": "/var/lib/docker/overlay2/4421a44d9ab8308305d3a051ca7672cb4d8e778bdbba66b8800348a805ee3dfc/diff",
                "WorkDir": "/var/lib/docker/overlay2/4421a44d9ab8308305d3a051ca7672cb4d8e778bdbba66b8800348a805ee3dfc/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:007ab444b234be691d8dafd51839b7713324a261b16e2487bf4a3f989ded912d",
                "sha256:e2a576046222da3c29f237fa2d751d18c7a8f875ddc32fc803466aeaf998b9ee",
                "sha256:b5fa8e7e036448cb8c431f6b28eb1fbc4573c9330b197c7a0ba8e2292fa9fc2f",
                "sha256:4437df14034b8c6be5537d719f45f305b8e3365e01abe23a2207ba0963618214",
                "sha256:2b3426b547f7dade968deaaed73f75d25a883f6688f817ec0c3d1999b2a5c7cf",
                "sha256:df324182704a2a1f6229c5712619b2f2f8e5c0e79c1c90c97b99f2c9e8f0a22d",
                "sha256:bfac39c3713affaa50703d23c5a3dfd7a4f54bdae75af9ea23551cf221b3ae7c",
                "sha256:5c5a2dec46e80510abd744b0d469feaa20a5c10d70c937748509577c5a280dfb",
                "sha256:e4cff87cc854a86536342699faf3e67ce0c2ccf4b735d9998991a024e6fe4db6"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        },
        "Config": {
            "ArgsEscaped": true,
            "Cmd": [
                "celery",
                "worker"
            ],
            "Entrypoint": null,
            "Env": [
                "PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "LANG=C.UTF-8",
                "GPG_KEY=97FC712E4C024BBEA48A61ED3A5CA953F73C700D",
                "PYTHON_VERSION=3.5.3",
                "PYTHON_PIP_VERSION=9.0.1",
                "CELERY_VERSION=3.1.25",
                "CELERY_BROKER_URL=amqp://guest@rabbit"
            ],
            "Labels": null,
            "OnBuild": null,
            "User": "user",
            "Volumes": null,
            "WorkingDir": "/home/user"
        }
    }
]
```

Docker 引擎在从镜像创建容器时, 会使用这个 JSON 对象, 该对象包含许多属性, 例如:

- `Id`: 镜像的唯一标识符
- `RepoTags`: 关联的标签
- `RepoDigests`: 镜像的分发摘要 (用于校验)
- `Created`: 镜像创建时间戳
- `Author`: 镜像创建者

当使用 `docker image pull` 命令而没有指定 tag 或版本时, Docker 会自动拉取以 `latest` 标记的镜像.  
这里一般是最新版本, 但注意 `latest` 知识一直约定, 并非所有镜像最新版本的 tag 都是 `latest`

#### The Layers Property | Layers 属性

镜像 JSON 对象中最关键的属性之一是 Layers, 如下

```JSON
{
    "Type": "layers",
    "Layers": [
        "sha256:007ab444b234be691d8dafd51839b7713324a261b16e2487bf4a3f989ded912d",
        "sha256:e2a576046222da3c29f237fa2d751d18c7a8f875ddc32fc803466aeaf998b9ee",
        "sha256:b5fa8e7e036448cb8c431f6b28eb1fbc4573c9330b197c7a0ba8e2292fa9fc2f",
        "sha256:4437df14034b8c6be5537d719f45f305b8e3365e01abe23a2207ba0963618214",
        "sha256:2b3426b547f7dade968deaaed73f75d25a883f6688f817ec0c3d1999b2a5c7cf",
        "sha256:df324182704a2a1f6229c5712619b2f2f8e5c0e79c1c90c97b99f2c9e8f0a22d",
        "sha256:bfac39c3713affaa50703d23c5a3dfd7a4f54bdae75af9ea23551cf221b3ae7c",
        "sha256:5c5a2dec46e80510abd744b0d469feaa20a5c10d70c937748509577c5a280dfb",
        "sha256:e4cff87cc854a86536342699faf3e67ce0c2ccf4b735d9998991a024e6fe4db6"
    ]
}
```

#### What is Layer? | 什么是 Layer?

文章开头提到镜像包含所有必要信息(依赖 配置等), 在 Docker 中, 模块化和解耦是关键原则.
Docker 中的 layer 本质是一组文件(或单个文件), 例如一个 Python 应用可能需要:

- Python 二进制文件
- apt 包管理器 / 所需系统工具
- 最后是应用文件 (又一个 layer)

layer 的好处在于: 一旦创建并存储了 Python 二进制的 layer, 任何需要它的镜像都可以直接引用它的位置.
Docker 会按需获取它, 如果世界另一端的某人也需要相同的 Python Layer, 他们不需要重新创建或重复上传该层, 可以直接使用相同的 Layer.
这种做法运行我们共享、重用并高效管理资源. 更棒的是, 分层让本地的更加高效, 如果机器的另一个镜像已经下载了一些 layer, Docker 会智能地复用这些已存在的 layer, 而不是重复下载.

简单来说: Layer 是一组文件, 而 Image 指明了它包含哪些层, 以及他们如何堆叠. 层的顺序很重要, 因为上层的文件可以屏蔽下层的文件.

但是, 容器本身并不知道层或文件在层中的组织方式, 容器看到的是一个完整的文件系统. 这由 Docker 的存储驱动完成: 它将所有层的文件收集合并, 为容器呈现一个单一的文件系统.

例如下面两个数据库镜像

```
$ docker pull redis
        Using default tag: latest
        latest: Pulling from library/redis
        af302e5c37e9: Pull complete  # <<<<-----  [1]
        01b95e092fd0: Pull complete
        c111ca53a743: Pull complete
        f7d6cf14046e: Pull complete
        589f36d317d9: Pull complete
        94041d0cae8f: Pull complete
        4f4fb700ef54: Pull complete
        4f5f785c9703: Pull complete
        Digest: sha256:ca65ea36ae16e709b0f1c7534bc7e5b5ac2e5bb3c97236e4fec00e3625eb678d
        Status: Downloaded newer image for redis:latest
        docker.io/library/redis:latest

$ docker pull postgres
        Using default tag: latest
        latest: Pulling from library/postgres
        af302e5c37e9: Already exists # <<<<-----  Check this out, this layer was already fetched in [1]
        23db180a1f67: Pull complete
        dc59dd9c8eb3: Pull complete
        aec09e638045: Pull complete
        4dd47a683737: Pull complete
        7cebbe7849b3: Pull complete
        dc4330b02129: Pull complete
        498cc40b9fe9: Pull complete
        6d3411bb4696: Pull complete
        8f14f34d54d3: Pull complete
        88d4f7416643: Pull complete
        e91ad5cfb8d0: Pull complete
        e0c4d5055fb9: Pull complete
        254ee626d709: Pull complete
        Digest: sha256:87ec5e0a167dc7d4831729f9e1d2ee7b8597dcc49ccd9e43cc5f89e808d2adae
        Status: Downloaded newer image for postgres:latest
        docker.io/library/postgres:latest
```

#### Base Layer 基础层

镜像中的 base layer 就像建筑的地基, 它是后续构建的起点.
通常 base layer 包含操作系统或运行环境所需的基础文件和工具.
每个 Docker 镜像都以 base layer 开始, 后续层在其基础上构建.
这个基础层保证了一致性, 并为应用提供了稳定的运行环境.
由于 base layer 可以在做个镜像之间重复使用, Docker 只需下载并存储一份, 从而提高效率并节省空间.
在上面的 Redis / Postgres 示例中, `af302e5c37e9` 就是 base layer.

#### Digests 摘要

如前所述, layer 是可重用的, 所以 Docker 需要一种系统来高效存储、组织和定位他们.
这些数字就是 layer 的唯一地址, 使得他们可以被识别和重用, 每个 layer 都有一个基于其内容计算出来的哈希值, 准确的说是该层的 content-addressable identifier 内容寻址.

Docker 会为 layer 中的每个文件计算哈希, 然后生成该 layer 的合并哈希.
这个哈希确保 layer 是唯一可寻址的, 从而让镜像可以指向它

如果某个 layer 中的文件被更新了, 文件的哈希会改变, layer 的合并哈希也会改变, 而产生一个全新的、唯一的 layer.

#### Image Digests 镜像摘要

```
$ docker pull postgres
    Using default tag: latest
    latest: Pulling from library/postgres
    af302e5c37e9: Already exists
    23db180a1f67: Pull complete
    ...
    254ee626d709: Pull complete
    Digest: sha256:87ec5e0a167dc7d4831729f9e1d2ee7b8597dcc49ccd9e43cc5f89e808d2adae #<----- Distribution Hash
    Status: Downloaded newer image for postgres:latest
    docker.io/library/postgres:latest
```

每个镜像由一系列 layer 构成, 每个 layer 有自己的哈希, 这意味着整个镜像也可以有一个用于识别它的唯一哈希, 在 layer 列表后看到的那个值就是

```
Digest: sha256:87ec5e0a167dc7d4831729f9e1d2ee7b8597dcc49ccd9e43cc5f89e808d2adae
```

这个就是 image digest, 是整个镜像的唯一哈希值.

镜像通常按名字拉取, 但 digest 在安全性方面非常关键, 用于确认拉取的镜像没有被篡改, 保证镜像的安全与真实性.

镜像有两种类型的 digest 值:

- Distribution Digest: 压缩后镜像的哈希, 用于传输阶段
- Content Digest: 每个未压缩 layer 的内容哈希

#### The Role of Comparession and Digest Verification 压缩与摘要检验的角色

在将镜像推送到 registry 之前, Docker 会对 layer 和镜像进行压缩以节省存储和网络流量.
压缩后会计算一个哈希, 并将其发送给 registry.
registry 会基于接收到的内容, 重新计算哈希并与 DOcker 发来的哈希比较.
这个哈下就是 Distribution Digest.
当另一端拉取镜像时, Docker 会计算接收到内容的哈希并与原始的 Distribution Digest 比对, 确保镜像在传输过程中未被篡改.

```
# Content Hash
$ docker image inspect redis
...
 "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:f5fe472da25334617e6e6467c7ebce41e0ae5580e5bd0ecbf0d573bacd560ecb", # <--- Content Digest
                "sha256:3e62a323d3749bd1fc58ef850f0d6e5a9c901f71cc160426da9f6fb7507293ef",
                "sha256:b5a15fcf554ab5310928776fceacdc57d225c145d96a3d3b62382a31684a6c7e",
                                ...,
                "sha256:ca15e9bb0d852a5e8c1d4c3264fa085f68197056b80da09003764bfa6dda5067"
            ]
        },
```

```
# Distribution hash
$ docker images --digests redis
redis        latest    sha256:ca65ea36ae16e709b0f1c7534bc7e5b5ac2e5bb3c97236e4fec00e3625eb678d   4075a3f8c3f8   13 days ago   117MB
```

#### Image Registry 镜像仓库

镜像仓库是容器镜像的存储中心, 各类软件, 像 Redis、Postgres、Node.js 这样的热门软件会把他们的容器化版本存放在这里.
这些仓库让用户可以方便地随时访问这些镜像, 虽然存在许多仓库, 但最广泛的是 Docker 的镜像仓库 - Docker Hub.

当运行类似 `docker pull <image-name>` 的命令时, 其实在使用一个简写.
完整的命令包含注册中心 registry, 组织 organization 和标签 tag, 如下所示:

```
docker pull <registry>/<organization>/<repository>:<tag>

# 例如
# docker pull docker.io/library/redis:latest
# docker pull docker.io/fredj/sample:1.0
```

当省略某些细节时, Docker 会做一些默认假设来补全:

- Registry & Organization: 如果不指定, docker 默认使用 Docker Hub (`docker.io`), 并假设官方镜像位于 `library` 组织下
- Tag: 如果不提供 tag, Docker 会拉取标记为 `latest` 的镜像

#### Repository: The Heart of the Registry | 仓库: 镜像仓库的核心

仓库 repository 是镜像仓库中的核心单位.
每个 registry 托管多个 repository, 每个 repository 代表一个应用或工具, 例如 redis、postgres、alpine、node.
在一个 repository 下可以有多个 tag, 每个 tag 对应镜像的一个具体版本.

有点类似 Git: Docker 的 repository 有点类似, 可以包含多个带标签的版本. 例如:

- Redis 在 GitHub 上带有标签的发布
- 类似的, Docker 上的 Redis 仓库也有标签

#### Two Types of Repositories on Docker Hub | Docker Hub 上的两类仓库

- 官方仓库 Offical Repositories  
   这些是经过 Docker 官方策划与验证的高质量镜像, 位于仓库的顶层, 因此可以不写组织名直接拉取, 例如:

  ```
  docker pull ubuntu
  ```

  这些官方仓库包含像 Redis、Ubuntu、Python、Node.js 等流行工具的镜像

- Regular Repositories 普通仓库  
   这些由个人、团队或组织创建, 拉取普通仓库的镜像时通常需要指定组织, 例如:
  ```
  docker pull mycompany/sample:1.0
  ```

#### Other Registries 其他仓库

虽然 Docker Hub 是默认选项, 但他并非唯一选择, 还有很多其他仓库, 例如:

- AWS Elastic Container Registry (ECR)
- Google Container Registry (GCR)
- GitHub Container Registry (GHCR)

从这些仓库拉取镜像时, 需要在命令中指定 registry 名称, 例如:

```
docker pull <registry>/<organization>/<repository>:<tag>

# 例如
# docker pull ghcr.io/myteam/myapp:1.0
```

不同的镜像仓库之所以能无缝工作, 是因为遵循了 OCI 标准, 它定义了镜像的创建与分发方式. 两个关键标准是:

- Image Specification: 定义容器镜像如何创建, 像一个通用的镜像蓝图规范
- Distribution Specification: 规定镜像如何在注册中心之间共享和分发

只要注册中心和镜像遵循这些标准, 使用什么工具制作镜像, 把镜像发布到哪个 registry, 都能互相兼容, 像魔法一样地协同工作.

### Conclusion 结论

Docker 镜像与层构成了容器化应用开发的骨干. 理解镜像如何构建, 存储与分发是掌握 Docker 的重要一步.
这些概念不仅是理论上的, 还是让容器轻量、可重用并高效的实用工具.
