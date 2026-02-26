+++
date = '2025-12-27T8:00:00+08:00'
draft = false
title = 'Taskwarrior Server Config Guide'
categories = ['Blog']
tags = ['Tools']
+++

# Taskwarrior 3.x 同步服务器配置指南

本文介绍如何使用 Docker 部署 taskchampion-sync-server，为 Taskwarrior 3.x 提供跨设备同步功能。

## 背景知识

Taskwarrior 是一款强大的命令行任务管理工具。从 3.0 版本开始，官方不再支持 taskd 服务器，改用新的 taskchampion-sync-server。

与 taskd 相比，新同步服务器的优势：

- 无需手动配置 SSL 证书
- 无需预先创建用户账户
- 客户端数据端到端加密
- 部署和维护更简单

## 服务器端配置

### 使用 Docker 部署

创建数据目录并启动容器：

```bash
sudo mkdir -p /var/lib/taskchampion-sync-server
sudo chmod 777 /var/lib/taskchampion-sync-server

docker run -d \
  --name taskchampion \
  -p 53589:8080 \
  -e RUST_LOG=info \
  -v taskchampion-data:/var/lib/taskchampion-sync-server \
  --restart unless-stopped \
  ghcr.io/gothenburgbitfactory/taskchampion-sync-server:main
```

**端口说明**：容器内部使用 8080，映射到宿主机的 53589（可自定义）。

### 配置防火墙

确保服务器防火墙开放相应端口。以常见的云服务器防火墙为例：

- 类型：入站
- 行动：允许
- 协议：TCP
- 目的端口：53589

### 验证服务运行

```bash
docker ps | grep taskchampion
docker logs taskchampion
```

正常运行时应看到：

```
[INFO] Serving on port 8080
[INFO] starting service: "actix-web-service-0.0.0.0:8080"
```

## 客户端配置

### 生成配置信息

```bash
# 生成唯一的客户端 ID
CLIENT_ID=$(uuidgen)
echo "Client ID: $CLIENT_ID"

# 生成加密密钥
ENCRYPTION_SECRET=$(openssl rand -base64 32)
echo "Encryption Secret: $ENCRYPTION_SECRET"
```

**重要提示**：请妥善保存这两个值，多设备同步时需要使用相同的配置。

### 配置 Taskwarrior

```bash
# 设置服务器地址
task config sync.server.url http://你的服务器IP:53589

# 设置客户端 ID
task config sync.server.client_id $CLIENT_ID

# 设置加密密钥
task config sync.encryption_secret "$ENCRYPTION_SECRET"
```

### 首次同步

```bash
task sync
```

成功时会显示：

```
Syncing with sync server at http://你的服务器IP:53589
Success!
```

## 日常使用

### 同步任务

```bash
# 手动同步
task sync

# 添加任务后自动同步（可选）
echo 'sync.on=all' >> ~/.taskrc
```

### 监控服务

查看服务器实时日志：

```bash
docker logs -f taskchampion
```

每次同步都会产生类似记录：

```
[INFO] 客户端IP "GET /v1/client/get-child-version/..." 200
[INFO] 客户端IP "POST /v1/client/add-version/..." 200
```

### 多设备同步

在其他设备上使用**相同的配置**：

- 相同的 `sync.server.url`
- 相同的 `sync.server.client_id`
- 相同的 `sync.encryption_secret`

这样所有设备将共享同一份任务数据。

## 常见问题

### Docker 权限问题

如果遇到 "permission denied" 错误：

```bash
# 将用户添加到 docker 组
sudo usermod -aG docker $USER

# 完全退出并重新登录 SSH
exit

# 如果还不行，清理 SSH 连接缓存
ssh -o "ControlPath=none" 服务器地址
```

### 容器数据持久化

推荐使用 Docker 命名卷（如上面的 `taskchampion-data`），Docker 会自动处理权限问题。

如果需要访问数据库文件：

```bash
# 查看数据卷位置
docker volume inspect taskchampion-data

# 进入容器
docker exec -it taskchampion sh
```

### 查看同步日志

启用详细日志可以帮助诊断问题：

```bash
docker run -d \
  --name taskchampion \
  -e RUST_LOG=debug \
  ...其他参数
```

日志级别：`error` < `warn` < `info` < `debug` < `trace`

## 安全建议

虽然数据在客户端加密，但仍建议：

1. 使用防火墙限制访问来源 IP
2. 配置反向代理添加 HTTPS（如 Nginx + Let's Encrypt）
3. 定期备份数据卷：

```bash
docker run --rm \
  -v taskchampion-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/taskchampion-backup.tar.gz /data
```

## 总结

taskchampion-sync-server 为 Taskwarrior 3.x 提供了简洁高效的同步方案。相比旧的 taskd，配置过程大幅简化，日常维护也更轻松。配合 Docker 部署，几分钟即可搭建完成。

关键配置要点：

- 服务器使用 Docker 部署，注意端口映射
- 客户端需要三个配置项：服务器地址、客户端 ID、加密密钥
- 多设备使用相同的客户端 ID 和加密密钥
- 数据端到端加密，服务器无法读取明文

现在你可以在任何设备上使用 `task sync` 保持任务列表同步了。
