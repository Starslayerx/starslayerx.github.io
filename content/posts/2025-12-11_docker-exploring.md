+++
date = '2025-12-11T8:00:00+08:00'
draft = true
title = 'Exploring Docker'
categories = ['Note']
tags = ['Docker']
+++

之前几篇关于 docker 的文章已经介绍了 image、container 方面的内容，这篇文章将介绍 Docker 其他方面的能力：

- Docker version
- Server information
- Downloading image updates
- Inspecting containers
- Entering a running container
- Returning a result
- Viewing logs
- Monitoring statistics

### Printing the Docker Version

```text
% docker version

Client:
 Version:           28.4.0
 API version:       1.51
 Go version:        go1.24.7
 Git commit:        d8eb465
 Built:             Wed Sep  3 20:56:26 2025
 OS/Arch:           darwin/arm64
 Context:           desktop-linux

Server: Docker Desktop 4.47.0 (206054)
 Engine:
  Version:          28.4.0
  API version:      1.51 (minimum version 1.24)
  Go version:       go1.24.7
  Git commit:       249d679
  Built:            Wed Sep  3 20:58:53 2025
  OS/Arch:          linux/arm64
  Experimental:     false
 containerd:
  Version:          1.7.27
  GitCommit:        05044ec0a9a75232cad458027ca83437aae3f4da
 runc:
  Version:          1.2.5
  GitCommit:        v1.2.5-0-g59923ef
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

注意到有 client 和 server 的区分，上面例子中 client 和 server 是匹配的版本，因为是一起安装的，但这并不总是相同的。
一般在生产环境会使用相同版本，在开发换即使版本有一些不大的区别，也会有很大问题。

### Server Information

通过 docker server 能够看到文件系统的类型、内核版本、操作相同类型，安装了哪些插件，使用的哪个 runtime，以及存储了多少容器和镜像。

```text
% docker system info

Client:
 Version:    28.4.0
 Context:    desktop-linux
 Debug Mode: false
 Plugins:
  ai: Docker AI Agent - Ask Gordon (Docker Inc.)
    Version:  v1.9.11
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-ai
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.28.0-desktop.1
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-buildx
  cloud: Docker Cloud (Docker Inc.)
    Version:  v0.4.29
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-cloud
  compose: Docker Compose (Docker Inc.)
    Version:  v2.39.4-desktop.1
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-compose
  debug: Get a shell into any image or container (Docker Inc.)
    Version:  0.0.42
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-debug
  desktop: Docker Desktop commands (Docker Inc.)
    Version:  v0.2.0
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-desktop
  extension: Manages Docker extensions (Docker Inc.)
    Version:  v0.2.31
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-extension
  init: Creates Docker-related starter files for your project (Docker Inc.)
    Version:  v1.4.0
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-init
  mcp: Docker MCP Plugin (Docker Inc.)
    Version:  v0.21.0
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-mcp
  model: Docker Model Runner (Docker Inc.)
    Version:  v0.1.41
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-model
  sbom: View the packaged-based Software Bill Of Materials (SBOM) for an image (Anchore Inc.)
    Version:  0.6.0
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-sbom
  scout: Docker Scout (Docker Inc.)
    Version:  v1.18.3
    Path:     /Users/starslayerx/.docker/cli-plugins/docker-scout

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 5
 Server Version: 28.4.0
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Using metacopy: false
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 CDI spec directories:
  /etc/cdi
  /var/run/cdi
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 05044ec0a9a75232cad458027ca83437aae3f4da
 runc version: v1.2.5-0-g59923ef
 init version: de40ad0
 Security Options:
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 6.10.14-linuxkit
 Operating System: Docker Desktop
 OSType: linux
 Architecture: aarch64
 CPUs: 6
 Total Memory: 3.828GiB
 Name: docker-desktop
 ID: 60fc8d35-0ffd-4da6-8abf-f8260e25c336
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 HTTP Proxy: http.docker.internal:3128
 HTTPS Proxy: http.docker.internal:3128
 No Proxy: hubproxy.docker.internal
 Labels:
  com.docker.desktop.address=unix:///Users/starslayerx/Library/Containers/com.docker.docker/Data/docker-cli.sock
 Experimental: false
 Insecure Registries:
  hubproxy.docker.internal:5555
  ::1/128
  127.0.0.0/8
 Live Restore Enabled: false
```

Docker本身由许多不同的插件共同协作构成。
这种设计极具优势，因为它意味着用户还可以安装由社区成员贡献的其他多种插件。
即使您只是想确认Docker是否已识别最近添加的插件，能够查看已安装的插件列表也非常实用。

在大多数情况下，默认的根目录 `/var/lib/docker` 将用来存储镜像和容器。
如果要修改这个位置，可以使用 `--data-root` 指定其他位置

```shell
sudo dockerd \
  -H unix://var/run/docker.sock \
  --data-root="/data/docker"
```

默认的 docker 配置文件在 `/etc/docker/daemon.json`，该文件的配置会直接传递给 dockerd，建议使用该文件永久配置。

### Downloading Image Updates

下面例子都使用 ubuntu 镜像。

```text
% docker image pull ubuntu:latest
latest: Pulling from library/ubuntu
97dd3f0ce510: Pull complete
Digest: sha256:c35e29c9450151419d9448b0fd75374fec4fff364a27f176fb458d472dfc9e54
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest
```

需要记住的是，即使拉取了最新版本，Docker 也不会自动更新本地镜像。
这需要自己负责处理。
不过，如果部署基于较新版本 `ubuntu:latest` 的镜像，Docker 客户端会在部署过程中下载缺失的层，正如预期的那样。
注意，这是 Docker 客户端的行为，其他库或 API 工具可能不会以这种方式运行。
强烈建议始终使用固定版本标签而非 `latest` 标签来部署生产代码。这有助于确保获得预期的版本，避免意外情况。

也可以使用镜像的 sha256 内容地址 tag，像这样：

```text
sha256:c35e29c9450151419d9448b0fd75374fec4fff364a27f176fb458d472dfc9e54
```

对应的命令有些不同，注意符号 @

```text
docker image pull ubuntu@sha256:c35e29c9450151419d9448b0fd75374fec4fff364a27f176fb458d472dfc9e54
```

这里的 sha256 地址不能简写，想要完整的地址

### Inspecting a Container

容器创建后，无论是否运行，都可以使用 docker 查看其配置。
这在调试容器和辨别容器上很有用。

先运行一个容器：

```shell
% docker container run --rm -t -d ubuntu /bin/bash
591db9e9ec1008cc6e5c1a8e64167454386ff3768a8b74c452e8ed545c38e704
```

- `--rm`: 容器退出后自动删除
- `-t`: 为容器分配一个伪终端 pseudo-TTY
- `-d`: 以后台模式 detached mode 运行容器

docker 检查信息有点冗余

```text
% docker container inspect 591db9e9ec10

[
    {
        "Id": "591db9e9ec1008cc6e5c1a8e64167454386ff3768a8b74c452e8ed545c38e704",
        "Created": "2025-12-11T09:20:40.095487792Z",
        "Path": "/bin/bash",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 616,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2025-12-11T09:20:40.157077708Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:9a84ec2d5dd7ad691e3df0d6e7c6e7ebd6ba67fdd1b6172588c009c12359d307",
        "ResolvConfPath": "/var/lib/docker/containers/591db9e9ec1008cc6e5c1a8e64167454386ff3768a8b74c452e8ed545c38e704/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/591db9e9ec1008cc6e5c1a8e64167454386ff3768a8b74c452e8ed545c38e704/hostname",
        "HostsPath": "/var/lib/docker/containers/591db9e9ec1008cc6e5c1a8e64167454386ff3768a8b74c452e8ed545c38e704/hosts",
        "LogPath": "/var/lib/docker/containers/591db9e9ec1008cc6e5c1a8e64167454386ff3768a8b74c452e8ed545c38e704/591db9e9ec1008cc6e5c1a8e64167454386ff3768a8b74c452e8ed545c38e704-json.log",
        "Name": "/cool_bose",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "bridge",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": true,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "ConsoleSize": [
                35,
                124
            ],
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "private",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": [],
            "BlkioDeviceWriteBps": [],
            "BlkioDeviceReadIOps": [],
            "BlkioDeviceWriteIOps": [],
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": null,
            "PidsLimit": null,
            "Ulimits": [],
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/interrupts",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware",
                "/sys/devices/virtual/powercap"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "ID": "591db9e9ec1008cc6e5c1a8e64167454386ff3768a8b74c452e8ed545c38e704",
                "LowerDir": "/var/lib/docker/overlay2/54fe3ed56bbca61cf784634cd23beb85f874788aee7f5404c89ea8057df5a62f-init/diff:/var/lib/docker/overlay2/b1a16eb3a83938d4bb52a8279e87a23c919793de6bac7691f7282fa6311c9f62/diff",
                "MergedDir": "/var/lib/docker/overlay2/54fe3ed56bbca61cf784634cd23beb85f874788aee7f5404c89ea8057df5a62f/merged",
                "UpperDir": "/var/lib/docker/overlay2/54fe3ed56bbca61cf784634cd23beb85f874788aee7f5404c89ea8057df5a62f/diff",
                "WorkDir": "/var/lib/docker/overlay2/54fe3ed56bbca61cf784634cd23beb85f874788aee7f5404c89ea8057df5a62f/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [],
        "Config": {
            "Hostname": "591db9e9ec10",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": true,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/bash"
            ],
            "Image": "ubuntu",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "org.opencontainers.image.ref.name": "ubuntu",
                "org.opencontainers.image.version": "24.04"
            }
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "079b975620af085dc5d2498a4d63f0db8005ef5d3faeb8e14809d6e27c2dae8b",
            "SandboxKey": "/var/run/docker/netns/079b975620af",
            "Ports": {},
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "41ced3762ad84154bdd497801be5db07337ca54cac82d183f65349603c7e9a13",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "fde6:dc51:edf::2",
            "GlobalIPv6PrefixLen": 64,
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "fde6:dc51:edf::1",
            "MacAddress": "da:6e:ae:6f:d5:c8",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "MacAddress": "da:6e:ae:6f:d5:c8",
                    "DriverOpts": null,
                    "GwPriority": 0,
                    "NetworkID": "19a3c6274752109c74d9c9eabbb47ac2b4bb1e5b7d670d797b089022cf58caa9",
                    "EndpointID": "41ced3762ad84154bdd497801be5db07337ca54cac82d183f65349603c7e9a13",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "fde6:dc51:edf::1",
                    "GlobalIPv6Address": "fde6:dc51:edf::2",
                    "GlobalIPv6PrefixLen": 64,
                    "DNSNames": null
                }
            }
        }
    }
]
```

- 长 Id 字符串：完整版本的容器唯一 id，也可以看到容器创建的具体时间，并比 `docker container ls` 要详细的多

还有以下在容器启动时可以配置的内容：

- top level command
- environment
- base image
- container hostname

常见的给容器传递配置的方法是环境变量，因此通过 `docker container inspect` 能看到大量信息用于调试。

### Exploring the shell

```shell
docker container run --rm -it ubuntu:20.04 /bin/bash
```

这条命令会启动进入一个 ubuntu 20.04 版本，并开启 bash shell 作为顶层进程。

查看容器的进程信息如下：

```bash
root@d0ba261b2d2c:/# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 12:07 pts/0    00:00:00 /bin/bash
root         9     1  0 12:09 pts/0    00:00:00 ps -ef
```

也就是说，该容器内只有一个 bash 进程，没有任何其他进程。
这也是如何在容器中运行 shell 的方法，如果要运行其他软件，使用 apt update 和 apt install 来安装运行，但这些内容只会在容器的生命周期存在，这只是在修改容器本身，而不是镜像。

如果直接退出 shell 会自然关闭容器

```bash
exit
```

### Returning a result

```bash
docker container run --rm ubuntu:22.04 /bin/false
echo $?
```

1

```bash
docker container run --rm ubuntu:22.04 /bin/true
echo $?
```

0

上面命令中

- `/bin/true`: 什么都不做，立即以退出码 0 退出（成功）
- `/bin/false`: 什么都不做，立即以退出码 1 结束（失败）

它们常用于测试、脚本控制流等

```text
% docker container run --rm ubuntu:20.04 /bin/cat /etc/passwd | wc -l
      19
```

上面命令是将容器中的 cat 输出，通过管道传递到本地的 wc 命令，而不是容器内的 wc 命令。
管道本身不会传递到容器内，如果想要在容日内使用管道，需要使用 `bash -c "<your_command> | <something_else>"` 这样的命令。
对应上面命令就是：

```text
docker container run ubuntu:20.04 /bin/bash -c "/bin/cat /etc/passwd | wc -l"
```

### Getting Inside a Running Container

使用 `docker container exec` 进入一个新的交互式容器的命令，但有一种更加 linux 原生的方式，叫做 `nsenter`。

下面先创建一个容器

```bash
docker container run --rm ubuntu:20.04 sleep 600
```

- `sleep` 是一个 Linux 用户态程序，让当前进程休眠指定时间（秒）

然后进入容器，命令类似 `docker container exec`

```bash
docker container exec -it 03eeebc5ca2d /bin/bash
```

- `-i`: 报持 STDIN 打开
- `-t`: 分配一个伪终端 pseudo-TTY

进入容器后查看一下进程

```text
root@03eeebc5ca2d:/# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 02:26 ?        00:00:00 sleep 600
root         6     0  0 02:27 pts/0    00:00:00 /bin/bash
root        14     6  0 02:28 pts/0    00:00:00 ps -ef
```

也可以使用 `docker container exec` 在容器中启动额外的后台进程，类似 `docker run` 命令，使用参数 `-d` (detached)。
但对于调试之外，应该谨慎使用该机制，因为这会破坏镜像部署的可重复性。
因为其他人必须要知道向 `docker container exec` 传入哪些命令，才能获取预期的功能。
更好的方式应该是重新构建容器镜像，如果只是需要通知容器内的软件执行某个动作，更加合理的方式是使用 `docker container kill -s <SIGNAL>`，通过标准的 Unix 信号向容器内的进程传递信息。

---

Docker 支持一个 volume 命令，可用于列出存储在根目录中的所有卷，并可进一步查看他们的详细信息，包括这些卷在服务器上的物理存储位置。

这些 volumes 不是绑定挂载的，相反，他们是用于持久化数据的特殊数据容器。

```shell
% docker volume ls

DRIVER    VOLUME NAME
local     8ae9318b841d9590eb9bb4325ee8bcd65ad7255e71561189dd4b56965b4cfb43
local     160507e738bcb4d769e42bdad8597b4a2e0e7fe3ed73ccaa56368c9980c8cdcf
local     b6800a72157489e14f94b287b8cee35a7e725ca83afd4ccde866f6942cec83c8
local     deploy_postgres_data_cloud
local     deploy_redis_data_cloud
local     f8c40ca914edbf435c2de5e5d7ae7a382eb0be948b9d2c253c03281c82a9b5d7
```

可以自己创建卷

```shell
docker volume create my-data
```

还可以检查数据卷

```shell
% docker volume inspect my-data

[
    {
        "CreatedAt": "2025-12-13T03:27:41Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/my-data/_data",
        "Name": "my-data",
        "Options": null,
        "Scope": "local"
    }
]
```

现在就可以在启动容器的时候使用该该数据卷了

```shell
docker container run --rm \
  --mount source=mydata, target=/app \
  ubuntu:latest touch /app/my-persistent-data
```

也可以自己删除相关数据卷

```shell
docker volume rm my-data
```

## Logging

Logging 日志是一个在生产环境中至关重要的部分，当出问题的时候，日志是恢复服务的重要工具。
如果你运行在一台主机上，你可能会将日志输出到一个本地的 logfile 文件中。
或者将输出记录在内核缓存 (kernel buffer) 中，然后从 dmesg 读取。
或者像许多使用 systemd 的现代 Linux 发行版那样，期望日志可以通过 journalctl 来查看。

- journalctl 是 systemd 提供的统一日志查询工具，用于读取 systemd-journald 收集的日志。

由于容器的限制和 docker 的构建方式，这些方法在不配置的情况下都不会正常工作。
但没有关系，因为 logging 在 docker 内有一流支持。

Docker 会捕获容器内所有的文本输出，任何 stdout 或 stderr 都会被 Docker daemon 捕获，并将其推送至一个后端日志中。

### docker container logs

docker 内置了 `json-file` 日志插件，每个容器内应用的日志会被 Docker daemon 流式传输到一个 JSON 文件中。
可以使用 `docker container logs` 命令查看日志

先拉取一个镜像并创建容器

```text
docker container run --rm -d --name nginx-test --rm nginx:latest
```

然后查看日志

```text
% docker container logs 7bdfc9e334d8

/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2025/12/13 06:37:18 [notice] 1#1: using the "epoll" event method
2025/12/13 06:37:18 [notice] 1#1: nginx/1.29.4
2025/12/13 06:37:18 [notice] 1#1: built by gcc 14.2.0 (Debian 14.2.0-19)
2025/12/13 06:37:18 [notice] 1#1: OS: Linux 6.10.14-linuxkit
2025/12/13 06:37:18 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2025/12/13 06:37:18 [notice] 1#1: start worker processes
2025/12/13 06:37:18 [notice] 1#1: start worker process 29
2025/12/13 06:37:18 [notice] 1#1: start worker process 30
2025/12/13 06:37:18 [notice] 1#1: start worker process 31
2025/12/13 06:37:18 [notice] 1#1: start worker process 32
2025/12/13 06:37:18 [notice] 1#1: start worker process 33
2025/12/13 06:37:18 [notice] 1#1: start worker process 34
```

这种方式对于小容量的日志十分有用。

要将日志输出限制为较新的内容，可以使用 `--since` 选项，显示在指定直接后的日志，时间可以使用以下格式：

- RFC 3339 格式：2020-10-02T10:00:00-05:00
- Unix 时间戳：1450071961
- 标准日期格式：20220731
- Go 时间段字符串：5m45s

还可以使用 `--tail` 选项显示指定的若干行

实际的日志存储地址默认在 `/var/lib/docker/containers/<container_id>`

```text
$ sudo ls /var/lib/docker/containers

13470e5fef4b71232d1cb18e342cab1884ba36dd42661772cea321737f27f2ad
17518b5d83f8b9dec547373a65f8505d76204b1c7e73cbb70accb20b14b99e3a
197ad199c73e6b0d118cfcf2c12e66df1f969920d730f0ae147f89687c1e0361
8e333a8c7241c332dd65751b0bbc1a0c8e93b9f1d44c1f307ce5b1a16dcc867a
f471f0bc60bdaf943f41c9c97255262d843e760676c7fa4b78d5a27b500013c6
fb6fdf34f33213953eba036a48089b08b9513e027fafb6f410e1433e87679487
```

上面列出的是每个容器的目录，若要查询日志，需要查看目录下面的 `<container_id>-json.log` 文件

```text
$ sudo tail -3 /var/lib/docker/containers/13470e5fef4b71232d1cb18e342cab1884ba36dd42661772cea321737f27f2ad/13470e5fef4b71232d1cb18e342cab1884ba36dd42661772cea321737f27f2ad-json.log

{"log":"2025-12-13 05:54:51.691 UTC [27] LOG:  checkpoint complete: wrote 6 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.503 s, sync=0.078 s, total=0.765 s; sync files=6, longest=0.055 s, average=0.013 s; distance=19 kB, estimate=22 kB; lsn=0/1B96448, redo lsn=0/1B96410\n","stream":"stderr","time":"2025-12-13T05:54:51.69140269Z"}
{"log":"2025-12-13 06:09:50.890 UTC [27] LOG:  checkpoint starting: time\n","stream":"stderr","time":"2025-12-13T06:09:50.890460875Z"}
{"log":"2025-12-13 06:09:51.669 UTC [27] LOG:  checkpoint complete: wrote 6 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.505 s, sync=0.126 s, total=0.780 s; sync files=6, longest=0.103 s, average=0.021 s; distance=20 kB, estimate=22 kB; lsn=0/1B9B4E0, redo lsn=0/1B9B4A8\n","stream":"stderr","time":"2025-12-13T06:09:51.66995443Z"}
```

使用参数 `-f` 可以进入交互式环境，持续查看最近的日志信息

```
docker container logs -f nginx-test
```

对于单一主机的日志而言，这种方法很不错。
其不足之处在于日志轮转、轮转后远程访问日志以及高流量日志记录时的磁盘空间占用问题。
如果你有更加复制的环境，你将需要更加鲁棒和中心化的日志功能。

Docker daemon 守护进程的默认配置并不会启用日志轮转。
因此，如果在生成环境中运行容器，务必通过命令行参数或 `daemon.json` 配置文件显示指定 `--log-opt max-size` 和 `--log-opt max-file` 这两个选项。

其中，`max-size` 用于限定单个日志文件在触发轮转之前所允许的最大大小，而 `max-file` 用于限定日志轮转后最多保留的日志文件数量。
要注意的是，单独配置 `max-file` 并不会生效，因为 Docker 只有在知道合适需要进行日志轮转的情况下，才会创建多个日志文件。
当日志轮转启用后，`docker container logs` 只会返回当前正在写入的日志文件中的数据，而不会读取已经归档的旧日志文件。

### More Advanced Logging

当默认的机制不够用的时候，尤其是在大规模场景下，Docker 还支持可配置的日志后端。
这些日志后端的插件列表在不断增长，除了前面提到的 json-file，还有 syslog、fluentd、journald、gelf、awslgos、splunk、gcplogs、local、logentries 等，这些后端用于把容器日志发送到各种主流的日志框架和日志服务中。
可以使用命令参数 `--log-driver=syslog` 来指定后端。

> TIPS

在 docker compose 里面可以这样配置

```yaml
services:
  # PostgreSQL 数据库
  postgres:
    image: postgres:16-alpine
    container_name: postgres_db
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "10"
```

这样表示使用 `json-file` 作为后端，单个容器日志文件最多 10 个，每个最大 10 MiB。

如果使用 `daemon.json` 配置，其一般位于 `/etc/docker` 下面，如果使用 docker desktop 则可以使用图像界面去修改配置。
