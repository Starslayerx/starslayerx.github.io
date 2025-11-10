+++
date = '2025-11-05T8:00:00+08:00'
draft = false
title = 'PostgreSQL 01: Introduction'
tags = ['PostgreSQL']
+++

## PostgreSQL 的安装与配置

### 安装

brew 安装后 `brew services start postgresql@17` 可以启动服务，使用 `brew services list` 查看服务状态。
也可以通过设定环境变量 `export PGDATA=/opt/homebrew/var/postgresql@17` 后使用命令 `initdb` 创建数据库簇，然后 `pg_ctl start -D $PGDATA` 启动。
停止命令为 `pg_ctl stop -D $PGDATA [-m SHOWDOWN-MODE]`，其中 -m 是服务器停止方法：

- smart: 等所有连接终止后，关闭数据库。如果客户端连不上，则无法关闭数据库。

- fast: 快速关闭数据库，断开客户端连接，让已有的事务回滚，然后关闭数据库。相当于 Oracle 关闭时的 immediate 模式。

- immediate: 立即关闭数据库，相当于数据库进程立即停止，直接退出，下次启动数据库需要恢复。相当于 Oracle 关闭时的 abort 模式。

> PostgreSQL 数据库中的 immediate 关机模式相当于 Oracle 数据库中的 abort 关机模式，而 Oracle 中的 immediate 关机模式实际上对应的是 PostgreSQL 中的 fast 模式。

### 配置

PostgreSQL 数据库的配置主要通过修改数据目录下的 postgresql.conf 和 pg_hba.conf 文件来实现的。

pg_hda.conf 文件是一个很白名单的访问配置文件，可以控制允许哪些 IP 地址的机器访问数据库。
默认数据库无法接受远程连接，因此默认情况下 pg_hba.conf 中没有对应的配置项，可以在文件中加入下面命令

```
host    all    all    0/0    md5
```

该命令允许任何用户远程连接到本地数据库，连接时需要提供密码。

#### IP 和 端口

postgresql.conf 可以修改监听的 IP 和端口，找到以下内容：

```
# listen_address = 'localhost'
# port = 5432
```

其中，"listen_address" 默认监听 "localhost"，如果要接受远程登陆，修改为 "\*" 表示在本地所有地址上监听。
"port" 表示监听的数据库端口，默认 "5432"。

```
listen_address = '*'
port = 5432
```

修改这两个参数要重启数据库才能生效。

#### 日志

日志收集一般需要打开（默认开启），日志目录一般使用默认值即可

```
logging_collection = on
log_directory = 'pg_log'
```

日志切换和是否覆盖可以使用如下几种不同的方案：

1. 每天生成一个新的日志文件

   ```
   log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
   log_truncate_on_rotation = off
   log_rotation_age = 1d
   log_rotation_size  = 0
   ```

2. 每当日志写满一定的大小，切换一个日志

   ```
   log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
   log_truncate_on_rotation = off
   log_rotation_age = 0
   log_rotation_size = 10M
   ```

3. 只保留最近 7 天的日志，进行循环覆盖（默认）

   ```
   log_filename = 'postgresql-%a.log'
   log_truncate_on_rotation = on
   log_rotation_age = id
   log_rotation_size = 0
   ```

#### 内存参数

- shared_buffers: 共享内容的大小，默认 32MB
- work_mem: 单 SQL 执行时，以及排序、Hash Join 时使用的内存，SQL 运行完毕后，该内存就会释放

#### 其他功能

- 数据块 checksum 功能
  对于一些数据可靠性很高的场景，例如金融领域，建议打开数据块校验功能
  ```
  initdb -k
  ```
  使用 `pg_checksums -c` 检查当前数据库是否打开了 checksum 功能 （需要关闭数据库），
  命令 `pg_checksums -e -P` 将数据库转换成具有 checksum 功能的数据库，其中 `-P` 参数是为了显示进度。
