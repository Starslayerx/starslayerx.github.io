+++
date = '2025-11-25T8:00:00+08:00'
draft = false
title = 'Docker Context'
tags = ['Docker']
+++

Docker Context 是 Docker 2019 年引入的一个特性，用来管理多个 Docker 主机的上下文，通过切换 context 就能让本地的 docker 命令作用在不同的 Docker 主机上。

本地开发机：

```shell
docker context use default
```

远程服务器要配置好 ssh 免密登陆，然后使用下面命令添加 context：

```shell
docker context create my-server --docker "host=ssh://root@1.2.3.4"

docker context create my-server --docker "host=ssh://CompanyServer1"
```

这里的 `CompanyServer1` 是 ssh 配置，例如这样

```ssh
Host CompanyServer1
    Hostname 192.168.0.106
    User root
    Port 22
    IdentityFile ~/.ssh/id_rsa_company
```

其中 `IdentityFile` 是存放无密码密钥的地方，如果你的密钥密钥密码就不需要这一行，否则需要设置一个没有密码的密钥，例如这样

```shell
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_company -N ""
```

配置好后就可以在本地连接服务器 docker 了

```shell
docker context list
```

输出类似这样

```
NAME              DESCRIPTION                               DOCKER ENDPOINT                                     ERROR
company-server                                              ssh://CompanyServer1
default           Current DOCKER_HOST based configuration   unix:///***/docker.sock
desktop-linux *   Docker Desktop                            unix:///***/docker.sock
```

使用命令 use 切换 context

```shell
docker context use company-server
```

如果要删除 context，使用 remove 命令

```shell
docker context remove company-server
```

那么使用 docker context 和登陆服务器再用 docker 相比有什么好处呢？

一大优势就是，当前的 AI Agent 还不能登陆到服务器里面去，如果你的 docker 镜像出了问题，想要 ai 帮你排查，那么使用 context 就很方便，ai 运行 docker 命令直接作用于服务器。
