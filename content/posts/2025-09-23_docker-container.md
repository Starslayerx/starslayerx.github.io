+++
date = '2025-09-23T8:00:00+08:00'
draft = false
title = 'Docker - Containers'
categories = ['Blog']
tags = ['Docker']
+++

像 VMware 或 KVM 这类虚拟化系统, 他们运行在虚拟化层上运行完整的 Linux 内核与操作系统.
这种架构能提供极强的隔离性, 因为每个虚拟机都搭载独立的内核, 这些内核各自运行在硬甲虚拟化层之上的隔离内存空间中.

而容器技术有着根本性差异, 所有容器共享同一个内核, 工作负载间的隔离性全通过内核机制实现, 这种模式被称为操作系统级虚拟化 _operating system virtualization_.

[runc/libcontainer](https://github.com/opencontainers/runc/blob/main/libcontainer/README.md) 提供了一个很好的定义:
A container is a self-contained execution environment that shares the kernel of the host system and is isolated from other containers in the system.

容器最大的优势就在于性能, 当运行一个进程的时候, 只有小部分的代码在内核中用于管理容器.
如今, 容器几乎在任何地方运行. Docker 和 OCI 镜像提供了生成环境中软件的打包格式, 并为 Kubernetes 和大多数 "serverless" 云技术打下了基础.

- 所谓的 serverless 技术并不是真的没有服务器: 它们依赖其他人的服务器来完成工作, 这样应用开发者就无需关心管理硬甲和操作系统了.

## Creating a Container

创建容器的命令 `docker container run` 实际上是包装在一起的两条命令.
第一件事是从基本的镜像中创建一个容器, 可以通过 `docker container create` 命令实现.
第二件事是执行容器, 同样地, 可以通过 `docker container start` 命令实现.

之前的命令参数中, 端口参数 `-p/--publish argument` 和环境变量参数 `-e/--env` 都只能在创建容器的时候设置.

### Basic Configuration

#### Conatiner name

当创建一个容器的时候, 默认情况下会使用 Dockerfile 设置的值, 但是可以在创建的时候通过命令行参数覆盖.
默认情况下, Docker 会使用名人名称的组合随机命名容器, 这就会出现类似 _ecstatic-babbage_ 和 _serene-albattani_ 这样的容器名.
如果要给容器一个具体的名字, 使用参数 `--name`

```
docker container create --name="awesome-service" ubuntu:latest sleep 120
```

创建后就可以启动容器了

```
docker container start awesome-service
```

它将会在 120 秒后自动退出, 但也可以通过命令提前关闭

```
docker container stop awesome-service
```

> 任何一个名称都只能给一个容器使用. 如果运行两次命令, 将会得到一个报错.
> 要么使用 `docker container rm` 删除之前的容器, 要么换一个名字.

#### Labels

Labels 是可以作为 Docker 镜像和容器元数据的键值对.
当 Linux 容器创建后, 他们将自动继承父镜像的标签.

当然也可以为容器添加新的标签:

```
docker container run --rm -d --name has-some-labels \
    -l deployer=Ahmed -l tester=Asako \
    ubuntu:latest sleep 1000
```

然后就可以基于 metadata 来搜索和过滤容器, 通过 `docker container ls` 命令实现:

```
docker container ls -a -f label=deployer=Ahmed

CONTAINER ID  IMAGE         COMMAND       … NAMES
845731631ba4  ubuntu:latest "sleep 1000"  … has-some-labels
```

可以使用 `docker container inspect` 命令查看容器所有的 labels

```
docker container inspect has-some-labels
...
"Labels": {
    "deployer": "Ahmed",
    "tester": "Asako"
}
...
```

这个容器运行了命令 sleep 1000, 这样 1000 秒后容器就会自动停止.

#### Hostname

默认情况下，当开启一个容器的时候，Docker 会将 host 宿主机上特定的系统文件复制到 host 上面的容器配置目录中，包括 `/etc/hostname`，并使用挂载将文件复制到容器中。可以下面这样启动一个没有额外配置的默认容器：

```Docker
docker container run --rm -ti ubuntu:latest /bin/bas
```

这条命令 `docker container run` 实际在背后运行了 `docker container create` 和 `docker container start` 两条命令。

- `--rm` 参数告诉 Docker 当容器存在时将其删除
- `-t` 参数告诉 Docker 分配一个伪终端
- `-t` 参数告诉 Docker 这将是一次交互式的会话，以及我们想将 STDIN 开启

如果镜像中没有定义 `ENTRYPOINT`，那么命令中的最后一个参数就是在容器中运行的可执行文件及其命令行参数，在本例中为 `/bin/bash`。
如果镜像中定义了 `ENTRYPOINT`，那么最后一个参数将作为命令行参数列表传递给 `ENTRYPOINT` 进程。

如果到容器内部运行 mount 命令，会看到下面这样的输出：

```Docker
root@4464f3966c8c:/# mount
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/QGANRL2RXF5OAGF6BR3VJBYKV7:/var/lib/docker/overlay2/l/FTZKGGCRTA67I47MV56GWNKXTJ,upperdir=/var/lib/docker/overlay2/9dc4a47989317b4297d540bb99e6d25b14b4db15eaa18e444f43877dfa6088f8/diff,workdir=/var/lib/docker/overlay2/9dc4a47989317b4297d540bb99e6d25b14b4db15eaa18e444f43877dfa6088f8/work)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,size=65536k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
sysfs on /sys type sysfs (ro,nosuid,nodev,noexec,relatime)
cgroup on /sys/fs/cgroup type cgroup2 (ro,nosuid,nodev,noexec,relatime)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
shm on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k)
/dev/vda1 on /etc/resolv.conf type ext4 (rw,relatime,discard)
/dev/vda1 on /etc/hostname type ext4 (rw,relatime,discard)
/dev/vda1 on /etc/hosts type ext4 (rw,relatime,discard)
devpts on /dev/console type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
proc on /proc/bus type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/fs type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/irq type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/sys type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/sysrq-trigger type proc (ro,nosuid,nodev,noexec,relatime)
tmpfs on /proc/interrupts type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/kcore type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/keys type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/timer_list type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/scsi type tmpfs (ro,relatime)
tmpfs on /sys/firmware type tmpfs (ro,relatime)
```

当看到 `root@hashID` 时，一般说明在容器内部运行

- 默认 hostname 是容器 ID，但可以使用 `--name` 参数指定容器名称（用于管理，如 docker ps 看到的名称）
- 默认用户为 root，可以通过 `--user` 参数指定进入容器后的用户

上面的绑定挂载中，我们感兴趣的是这一行：

```
/dev/vda1 on /etc/hostname type ext4 (rw,relatime,discard)
```

前面的设备名称可能不同，但挂载点都是 `/etc/hostname`，这回将容器的 `/etc/hostname` 链接到 Docker 为容器准备的主机文件，该文件默认包含容器 ID，并没有完全限制域名。（容器的主机名不带域名后缀，例如为 `4464f3966c8c` 而不是 `4464f3966c8c.example.com`）

如果要设置具体的 hostname，可以使用 `--hostname` 参数传入一个具体的值：

```Docker
docker container run --rm -ti --hostname="mycnotainer.example.com" \
    ubuntu:latest /bin/bash
```

#### Domain Name Services

如同 `/etc/hostname`，`resolv.conf` 文件设置的 Domain Name Service (DNS) 通过主机和容器的绑定挂载管理。

```Docker
/dev/vda1 on /etc/resolv.conf type ext4 (rw,relatime,discard)
```

默认情况下，会直接复制宿主机的 `resolv.conf` 文件，如果不想要默认配置，可以使用 `--dns` (nameserver) 和 `--dns-search` (search) 参数复写容器的行为：

```Docker
docker container run --rm -ti --dns=8.8.8.8 --dns=8.8.4.4 \
    --dns-search=exmaple.com --dns-search=example2.com \
    ubuntu:latest /bin/bash
```

如果不想设置 search domain，使用 `--dns-search=.`

容器中的文件为下面这样：

```
root@97b634b9bb5b:/# more /etc/resolv.conf
# Generated by Docker Engine.
# This file can be edited; Docker Engine will not make further changes once it
# has been modified.

nameserver 8.8.8.8
nameserver 8.8.4.4
search exmaple.com example2.com

# Based on host file: '/etc/resolv.conf' (legacy)
# Overrides: [nameservers search]
```

#### MAC address

另一个可以设置的重要信息是容器的 media access control (MAC) 地址。

如果没有任何配置，容器会接受一个计算出的 MAC 地址，并以 `02:42:ac:11` 为前缀（以 02 开头的 MAC 表示“本地管理的（Locally Administered）”地址，不与全球 OUI 冲突）。
如果要设置这个值，使用类似下面方法：

```Docker
docker container run --rm -ti --mac-address="a2:11:aa:22:bb:33" \
    ubuntu:latest /bin/bash
```

> WARNING 警告

自定义 MAC 地址的时候要小心，当两个系统使用同样 MAC 地址的时候，可能会引起 ARP contention（ARP 冲突）。
如果确实有自定义 MAC 地址的需求，建议使用 `x2-`, `x6-`, `xA-` 和 `xE-` 开头的地址。

#### Storage Volumes

有时使用容器默认分配的磁盘空间，或者容器是暂时的，这种并不适合手头的工作，因此需要在容器部署之间的持久化存储。

这时候，可以使用 `--mount/-v` 参数将 host 上的目录或单独的文件挂载到容器中。
下面的例子将 `/mnt/session_data` 挂载到容器的 `/data` 目录：

```Docker
docker container run --rm -ti \
    --mount type=bind, target=/mnt/session_data,source=/data \
    ubuntu:latest /bin/bash
```

可以使用 `-v` 参数简化，使用冒号 `:` 将源文件与目标文件分开，还在末尾添加 `ro` 以只读方式挂载

```Docker
docker container run --rm -ti \
    -v /mnt/session_data:/data:ro \
    ubuntu:latest /bin/bash
```

host 和容器中挂载的文件都无需预先存在，如果主机挂载点的文件不存在，会直接创建对应的目录文件，这可能会导致一些问题。

> SELINUX AND VOLUME MOUNTS

SELINUX (Secuirty-Enhanced Linux) 是一个内置于 Linux 内核的强制访问控制器 (MAC) 安全系统。
其核心思想是“默认拒绝”。

- 传统 Linux 使用自主访问控制 (DAC)，基于用户/组/权限 (rwx)。
- SELinux 在此基础上，为每个进程、文件、目录等对象都打上了一个安全上下文标签。策略规则规定了 “哪个进程的标签” 可以访问 “哪个文件的标签”。

如果在 Docker host 启动了 SELinux，挂载文件目录的时候，可能会遇到一个 "Permission Denied" 报错。
可以使用 Docker `z/Z` 选择来处理挂载问题：

- 小写的 `z` 选项表示绑定挂载内容可多容器共享
- 大写的 `Z` 选项表示绑定挂载内容是私有且不共享的

例如下面这样

```Docker
docker container run --rm -v /app/dhcpd/etc:/etc/dhcpd:z dhcpd
```

还可以告诉 Docker 容器以只读方式运行，这样任何进程就无法修改根目录文件了（任何在 `/` 下尝试写入的操作都会失败，除非该路径被单独挂载为可写卷）。
在之前的例子中，可以使用 `--read-only=true` 命令：

```Docker
docker container run --rm -ti --read-only=true -v /mnt/session_data:/data \
    ubuntu:latest /bin/bash
```

有时，即使容器为可读的，也有必要使 `/tmp` 这样目录可写。
这种情况下，可以使用 `docker container run` 命令配置 `--mount type=tmpfs` 参数，已将 tmpfs 文件系统挂载到容器中。
tmpfs 文件系统完全在内存中，速度非常快，同时也是临时的，关闭容器就会释放。
下面示例展示启动一个容器，在 `/tmp` 处挂载一个 256 MB 的 tmpfs 文件系统：

```Docker
docker container run --rm -ti --read-only=true \
    --mount type=tmpfs,destination=/tmp,tmpfs-size=256M \
    ubuntu:latest /bin/bash
```

容器如下：

```
root@a1b02391f02d:/# df -h /tmp
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           256M     0  256M   0% /tmp

root@a1b02391f02d:/# grep /tmp /etc/mtab
tmpfs /tmp tmpfs rw,nosuid,nodev,noexec,relatime,size=262144k 0 0
```

> WARNING

容器应该尽可能设计为无状态的，管理存储会有不必要的依赖，使得部署场景更加复杂。

#### Resource Quotas

虚拟机通常可以细致的控制 OS 的内容和 CPU 以及其他资源。

当使用 Docker 时，必须利用 linux 内核的 cgroup 功能来控制资源可用性。
Docker 容器的运行命令 `docker container run` 命令在创建容器时直接支持设置 CPU, 内存, swap 和 I/O 限制。

> NOTE

这些限制通常在容器创建的时候设置，如果需要修改这些配置，使用 `docker container update` 命令或部署一个新的容器。

Docker 支持多种资源的限制，但是必须要在内核中开启相应的功能提供给 Docker。
可能需要通过命令行将这些功能开启，运行命令 `docker system info` 来查看内核相关限制支持。
如果缺失任何支持，将会有类似下面这样的输出：

```Docker
WARNING: No swap limit support
```

##### CPU shares

docker 有好几种限制 CPU 使用的容器应用。原始的最常用的方法是 cpu shares。

系统中所有 CPU 核心的计算能力被视为总份额池。Docker 分配数字 1024 代表完整的池。
通过设置容器的 CPU 共享，可以设置容器获取多少 CPU shares。
如果希望容器最多获取系统一半的 CPU shares，那么就要分配 512 的份额。
这并发独占份额，即使将 1024 份额分配给一个容器，也不会影响其他容器的运行。

更准确的说，它提示调度器容器运行的时间。
假如有一个容器分配了 1024 份额，另外两个分配 512 份额，他们都将被调度相同次数。
但是如果每个进程的正常 CPU 时间是 100 微秒，那么 512 份额的容器将运行 50 微秒，1024 份额的容器将运行 100 微秒。

下面使用 `stress` 命令在容器内进行压力测试，下面命令将创建 2 条 CPU 密集计算，1 条 I/O 密集计算和两个内存分配程序，并分配 6 个 CPU 核心。

```Docker
docker container run --rm -ti spkane/train-os \
    stress -v --cpu 6 --io 1 --vm 2 --vm-bytes 128M --timeout 120s
```

如果想运行同样的压力测试，但只分配一般的 CPU 运行时间，可以这样做：

```Docker
docker container run --rm -ti --cpu-shares 512 spkane/train-os \
    stress -v --cpu 6 --io 1 --vm 2 --vm-bytes 128M --timeout 120s
```

与虚拟机不同，Docker基于cgroup对CPU份额的限制可能会产生意想不到的后果。
它不是硬性限制，而是相对限制，类似于nice命令。
例如，一个被限制为一半CPU份额的容器，如果运行在一个不怎么繁忙的系统上，由于CPU不繁忙，CPU份额的限制效果有限，因为调度器池中没有竞争。
当第二个大量使用CPU的容器部署到同一个系统时，突然间，第一个容器上的限制效果就会变得明显。
在限制容器和分配资源时，应仔细考虑这一点。

##### CPU pinning

可以将一个容器绑定到一个或多个 CPU 核心。
这意味着这个容器的工作只会调度到分配给他的核心上。
如果要隔离 CPU 或者提高缓存效率，这会很有用。

下面命令将容器绑定到前 2 个 CPU 上：

```Docker
docker container run --rm -ti \
    --cpu-shares 512 --cpu-set=0 spkane/train-os \
    stress -v --cpu 2 --io 1 --vm 2 --vm-bytes 128M --timeout 120s
```

> WARNING

`--cpu-set` 参数是从 0 开始的，因此第一个 CPU 是 0。
如果让 Docker 使用一个不存在的 CPU 核心，将会得到一个无法启动容器的报错。

在 Linux 内核中使用 CPU Completely Fair Scheduler (CFS)，可以使用 `--cpu-quota` 参数更改 CPU quota 来启动容器。

CFS 完全公平调度器：

- Linux 内核中的默认 CPU 调度器
- 目标是为所有进程提供“公平”的 CPU 时间
- 通过时间片轮转和优先级来分配 CPU 资源

CPU quota:

- 不同于 CPU shares 这是一个硬性限制
- 限制一个容器在一个时间周期内可以使用的 CPU 时间
- 即使系统空闲，也不能超过这个限制

限制最多使用 50% CPU:

```Docker
docker container run --cpu-period=100000 --cpu-quota=50000 image
```

cpu-period: 时间周期（默认 100000 微秒 = 100ms）

CPU 使用率 = (cpu-quota / cpu-period) × 100%

##### Simplifying CPU quotas

CPU shares 和 CPU quotas 是 Docker 中管理 CPU 限制的基本机制，但现在 Docker 已经发展了很多。
只需要简单地告诉 Docker 希望容器提供多少 CPU，它就会自动完成底层 cgroups 所需的计算。

参数 `--cpus` 可以设置一个 0.01 至 CPU 核心数的浮点数：

```Docker
docker container run --rm -ti --cpus=".25" spkane/train-os \
    stress -v --cpu 2 --io 1 --vm 2 --vm-bytes 128M --timout 60s
```

如果要调整容器的资源限制，可以使用 `update` 命令，如下：

```Docker
docker container update --cpus="1.5" 092c5dc8504 92b797f12af1
```

##### Memory

内存可以使用类似 CPU 限制的功能。但不同的是，内存是硬性限制，分配多少就只能用那么多，且会有一定的默认分配。
由于虚拟内存的存在，实际上可以为容器分配比系统内存更多的内存，容器将使用 swap 存储这些内容，就像 linux 中的进程一样。

下面使用 `--memory` 参数命令开启一个限制内存的命令：

```Docker
docker conatiner run --rm -ti --memory 512m spkane/train-os \
    stress -v --cpu 2 --io 1 --vm 2 --vm-bytes 128M --timout 10s
```

当单独使用 `--memory` 参数时，会同时设置 RAM 和 swap 的大小，这里就设置了 512m 的 RAM 和 512m 的 swap 大小。
Docker 支持 b, k, m, g 单位。

如果要单独设置 swap 大小，或者关闭它，使用 `--memory-swap` 参数：

```Docker
docker container run --rm -ti --memory 512m --memory-swap=768m \
    spkane/train-os stress -v --cpu 2 --io 1 --vm 2 --vm-bytes 128M \
    --timeout 10s
```

如果将 `--memory-swap` 设置为 -1，那么容器将可以使用系统上任意大的 swap 空间。
如果将 `--memory-swap` 设置为和 `--memory` 相同的正数，则容器将无法使用任何 swap 空间。

当内存快满的时候，Docker 容器将会让 linux 内核认为系统内存快满了，它将会尝试杀死进程以释放内存。
例如下面命令：

```Docker
docker container run --rm -ti --memory 100m spkane/train-os \
    stress -v --cpu 2 --io 1 --vm 2 --vm-bytes 128M --timeout 10s
```

会有类似下面的报错

```Docker
stress: FAIL: [1] (461) failed run completed in 0s
```

这是因为容器尝试分配的内存大于运行的内存， Linux OOM (kernel out-of-memory) killer 被激活，并杀死子进程，父进程清理进程，并报错退出。

可以使用命令 `sudo dmesg` 查看相关信息，该 OOM 事件也会被 Docker 记录，可以使用 `docker system events` 监控

```Docker
docker system events
```

该命令是一个阻塞命令，它会持续监听 Docker daemon 守护进程的事件，因此直接开启该命令看不到之前的报错信息。

##### Block I/O

很多容器实际上都是无状态容器，大多数都不需要 I/O 限制。
但 Docker 还是提供了基于 cgroup 机制的限制功能。

第一种方式是为容器的 block I/O 设备设置优先级。
可以通过设置默认的 `blkio.weight` cgroup 状态实现，该属性设置为 0 表示禁止，或者 10~1000 之间的一个数，默认为 500。
该限制类似 CPU shares，系统会将所有可用的 I/O 划分为 1000 份，并在 cgroup 切片中的每个进程之间进行分配，其中分配的权重会影响每个进程可用的 I/O 量。

通过参数 `--blkio-weight` 设置权重，也可以使用参数 `--blkio-weight-device` 指定特殊的设备。
和 CPU shares 一样，在实际操作中调整合适的权重比较困难，但是可以通过限制每秒最大的字节数/操作数来使管理变得更加容易。
例如下面参数：

- `--devices-read-bps`: 限制设备读取速率 (bytes per second)
- `--devices-read-iops`: 限制设备读取速率 (IO per second)
- `--devices-write-bps`: 限制设备写速率 (bytes per second)
- `--devices-write-iops`: 限制设备写速率 (IO per second)

可以使用 Linux I/O tester [bonnie](https://www.coker.com.au/bonnie) 来测试对容器性能的影响：

```Docker
time docker container run --rm -ti spkane/train-os:latest \
    bonnie++ -u 500:500 -d /tmp -r 1024 -s 2048 -x 1

real 0m27.715s
user 0m0.027s
sys  0m0.030s
```

这个根据经验，更加推荐使用 `--device-read-iops` 和 `--dvice-write-iops` 参数来设置 block I/O 限制。

##### ulimits

在 Linux 引入 cgroups 之前，还有另一种方式可以限制进程可用的系统资源：
通过 `ulimit` 命令 设置用户级资源限制。
这种机制至今仍然可用，并且在许多传统使用场景下依然非常有用。

下面的示例展示了可以通过 `ulimit` 命令设置“软限制（soft limit）”和“硬限制（hard limit）”的系统资源类型：

```bash$ ulimit -a
core file size (blocks, -c) 0
data seg size (kbytes, -d) unlimited
scheduling priority (-e) 0
file size (blocks, -f) unlimited
pending signals (-i) 5835
max locked memory (kbytes, -l) 64
max memory size (kbytes, -m) unlimited
open files (-n) 1024
pipe size (512 bytes, -p) 8
POSIX message queues (bytes, -q) 819200
real-time priority (-r) 0
stack size (kbytes, -s) 10240
cpu time (seconds, -t) unlimited
max user processes (-u) 1024
virtual memory (kbytes, -v) unlimited
file locks (-x) unlimited
```

可以在 Docker 守护进程（daemon） 的启动配置中设置默认的用户限制，使其应用到每一个容器上。
例如，下面的命令告诉 Docker 守护进程：启动所有容器时，将打开文件数的软限制设为 50，硬限制设为 150：

```Docker
sudo dockerd --default-ulimit nofile=50:150
```

然后，可以在启动特定容器时使用 `--ulimit` 参数覆盖这些默认值，例如：

```Docker
docker container run --rm -d --ulimit nofile=150:300 nginx
```

此外，还有一些更高级的命令可用于创建容器时设置限制，但上述内容涵盖了多数常见场景。
Docker 客户端文档中列出了所有可用选项，并会随着每次 Docker 发布而更新。

### Starting a Container

创建容器后，并不会默认运行，这是一个设置的过程，而不是运行过程。

```Docker
$ docker container create -p 6379:6379 redis:2.8

Unable to find image 'redis:2.8' locally
2.8: Pulling from library/redis
51f5c6a04d83: Pull complete
6c8ccd839b1d: Pull complete
0fded1c9651d: Pull complete
7f1aa6a73799: Pull complete
fbe8a4f1aa87: Pull complete
1a9852d2edd3: Pull complete
128182e1e85d: Pull complete
b94de088b6d8: Pull complete
Digest: sha256:e507029ca6a11b85f8628ff16d7ff73ae54582f16fd757e64431f5ca6d27a13c
Status: Downloaded newer image for redis:2.8
d14e339f67f70c21783e242ce706b705e5355facf0a5d97a797a9f3bbc701ff7
```

上面最后一行就是容器的哈希，可以使用该哈希来启动容器，如果没有记录下来该哈希，可以使用下面命令查看容器：

```Docker
$ docker container ls -a --filter ancestor=redis:2.8

CONTAINER ID   IMAGE       COMMAND                  CREATED         STATUS    PORTS     NAMES
d14e339f67f7   redis:2.8   "docker-entrypoint.s…"   3 minutes ago   Created             pensive_cori
```

可以使用下面命令来启动容器

```Docker
docker container start d14e339f67f7
```

> NOTE

这里的哈希可以是完整的，也可以是部分的，甚至无论长度只要满足唯一性即可。

现在容器应该正常运行了，但运行在后台无法知道是否报错：

```Docker
docker container ls
```

### Auto-Restarting a Container

在许多情况下，我们希望容器在退出后自动重启。
许多容器存活时间很短，但对于生产环境，可能希望他们能够始终保持运行。
如果要运行一个复杂系统，可以使用 scheduler。

最简单是情况下，使用 `--restart` 参数告诉 docker 容器重新运行命令，该参数有 4 类可选值：

- `no`: 永不重启
- `always`: 总是重启
- `no-failure`: 当以非 0 退出码的时候重启，如果设置成 `no-failure:3` 则会在尝试重启 3 次失败后放弃重启
- `unless-stopped`: 总是重启，除非有意停止，例如 `docker container stop`

可以重新运行之前的内存受限的例子来展示，这里使用 `--restart` 参数而不是 `--rm` 参数：

```Docker
docker container run -ti --restart=on-failure:3 --memory 100m spkane/train-os \
    stress -v --cpu 2 --io 1 --vm 2 --vm-bytes 128M --timeout 120s
```

### Stoping a Container

容器可以停止与启动，你可能认为容器的暂停和启动是类似的，但实际上两者并不相同。
当进程停止的时候，并不是暂停了，而是退出了。
而容器停止 stop 的时候，它不再在 `docker container ls` 输出中显示，一但重启 docker 就会尝试启动所有关机时关闭的容器。
即容器进程退出，但会保留容器状态，重启的使用再运行。
如果只是想暂停容器，而不停止任何进程，应该使用 pause 命令，`docker container pause` 和 `docker container unpause`。

如果要查看已经停止的容器，使用 `docker container ls -a` 命令查看。
这意味着，虽然内存和临时文件 temporary file system (tmpfs) 会暂时丢失，但所有其他的文件内容和元数据，包括环境变量和端口信息都会在容器重启的时候恢复。

容器与服务器上任何其他进程以基本相同的方式与系统交互，这意味着我们可以向容器中的进程发送 Unix 信号，而它们可以做出响应。
在之前的 `docker container stop` 示例中，向容器发送了 SIGTERM 信号，并等待容器优雅地退出。
容器遵循与Linux上任何其他进程组接收到的相同的过程组信号传播。

一个正常的 `docker container stop` 会向进程发送 SIGTERM 信号。
如果希望容器在经过一定时间后仍未停止时被强制杀死，可以使用 -t 参数，例如：

```Docker
docker container stop -t 25 d14e339f67f7
```

这样在 25 秒后，如果容器仍未关闭，则会发送一个 SIGKILL 信号将其强制关闭。

### Killing a Container

当一个进程不正常的时候，`docker container stop` 可能无法解决问题。
如果希望容器立刻退出，可以使用 `docker container kill`，类似 linux kill 命令，该命令也可以发送信号

```Docker
docker container kill --signal=USR1 092c5dc85044
```

任何标准的 Unix signal 都可以通过该方法传入容器

### Pausing and Unpausing Container

暂停容器使用了 [cgroup freezer](https://www.kernel.org/doc/Documentation/cgroup-v1/freezer-subsystem.txt) 实现，这样基本上是组织进程被调度，直到 unfreeze 此进程。
这将阻止容器进行任何操作，保持状态不变，包括内存内容。
不同于 stop 命令通过 SIGSOTP 信号停止容器，而暂停容器不会向容器发送任何关于其状态变化的信息，这是一个重要区别。

命令 `docker container pause 092c5dc85044` 暂停容器，之后开使用 `docker container unpause 092c5dc85044` 继续运行容器。

### Cleaning Up Containers and Images

当运行很多命令后，会在系统上积累大量的镜像层和容器层。

可以使用命令 `docker container ls -a` 查看所有容器，在删除镜像之前，需要先暂停使用该镜像的所有容器才行。

也可以列出系统上的所有镜像，使用命令 `docker image ls`，然后使用下面命令删除：

```Docker
docker image rm 0256c63af7db
```

有时，尤其是再部署循环的时候，需要将整个系统的镜像或容器都从系统中剃除，最简单的就是使用下面的方法：

```Docker
docker system prune
```

如果要将所有未使用的镜像都剃除，而不只是悬空镜像，使用 `-a` 参数：

```Docker
docker system prune -a
```

如果要删除所有的容器或镜像，可以使用下面的组合命令

```Docker
docker container rm $(docker container ls -a -q)
docker image rm $(docker image -q)
```

上面两个命令都支持一个过滤参数，用于微调删除命令或特定情况

例如移除所有非 0 状态的容器

```Docker
docker contaienr rm $(docker container ls -a -q --filter 'exited!=0')
```

或者删除所有没标签的容器

```Docker
docker image rm $(docker images -q -f "dangling=true")
```

### Windows Containers

自从 2016 年以来 Windows 已经支持运行含本地原生应用的 Windows 容器。
包含原生 Windows 应用的 Windows 容器可以使用特殊的 Docker 命令管理。
下面将演示 Windows 10+ 通过 Hyper-V 使用 Docker。

第一件需要做的事情就是从 Linux 容器切换到 Windows 容器，选择“Switch to Windows Containers...”，这个过程会消耗一定的时间。
可以使用下面命令在 PowerShell 测试 Windows 容器：

```Docker
docker container run --rm -it mrc.microsoft.com/powershell
```

要实现同样功能，可以编写下面的 Dockerfile

```Dockerfile
FROM mcr.microsoft.com/powershell
SHELL ["pwsh", "-command"]
RUN Add-Content C:\helloworld.ps1`
    'Write-Host "Hello World from Windows"'`
CMD ["pwsh", "C:\\helloworld.ps1"]
```

再次点击 "Switch to Linux Containers..." 切换回 Linux 容器。
