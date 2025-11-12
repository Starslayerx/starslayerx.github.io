+++
date = '2025-11-10T8:00:00+01:00'
draft = false
title = 'Solving SSH Lag Through Proxy: A Complete Guide to SSH on Port 443'
tags = ['Network']
+++

# 解决代理环境下的 SSH 卡顿：SSH over 443 端口完全指南

## 问题描述

从内地 SSH 连接到香港的云服务器时，出现严重的卡顿现象：

- 输入延迟明显，每个字符都有停顿
- 但通过服务商提供的 noVNC Console（Web 终端）却非常流畅

明明都是连接同一台香港服务器，为什么体验差距如此之大？

## 环境信息

- **本地位置**: 大陆
- **服务器位置**: 香港（IP: `<hk-server-ip>`）
- **代理工具**: Clash Verge（有香港代理节点）
- **操作系统**: macOS

## 问题诊断

### 第一步：基础网络测试

首先测试直连服务器的网络质量：

```bash
# 测试延迟和丢包率
ping -c 20 <hk-server-ip>

# 结果：
# - 平均延迟：170-180ms
# - 丢包率：4%
# - 延迟抖动：标准差 5-7ms
```

**分析**：对于大陆到香港的连接，这个延迟和丢包率虽然不理想，但也算"正常的糟糕"。真正的问题是：为什么 noVNC 不卡，而 SSH 卡？

### 第二步：路由追踪

```bash
traceroute -I <hk-server-ip>

# 结果显示只有1跳就到达目标
# 这说明中间路由器没有响应 ICMP，无法看到完整路径
```

### 第三步：IP 归属查询

```bash
curl -s "http://ip-api.com/json/<hk-server-ip>"
# 服务器：香港，ISP: Lucidacloud Limited

curl -s ifconfig.me
# 本地公网 IP 显示：香港，ISP: Nearoute Limited
```

**关键发现**：本地公网 IP 显示为香港，但我人在大陆！这说明当前网络已经走了 Clash 代理。

### 第四步：确认 SSH 是否走代理

```bash
ssh -v blog 2>&1 | grep -E "Connecting to|proxy"
# 输出：debug1: Connecting to <hk-server-ip> [<hk-server-ip>] port 22.
```

**问题定位**：SSH 并没有走代理，而是直连的！

虽然 Clash 开启了系统代理模式，但：

- 浏览器等应用会使用系统代理（走 Clash）
- SSH、终端命令默认**不使用系统代理**（直连）

这就是为什么：

- noVNC（走供应商内网或优化路由）→ 不卡
- SSH 直连（走普通国际出口，受 GFW 干扰）→ 卡

## 解决方案

### 方案1：开启 TUN 模式（失败）

尝试在 Clash Verge 中开启 TUN 模式，让所有流量自动走代理。

**结果**：开启后完全连不上服务器。

**原因**：TUN 模式可能导致 DNS 解析失败或路由配置错误。

### 方案2：配置 SSH 走 SOCKS5 代理（失败）

修改 `~/.ssh/config`：

```ssh-config
Host blog
    Hostname <hk-server-ip>
    User sx
    Port 22
    ProxyCommand ncat --proxy 127.0.0.1:7898 --proxy-type socks5 %h %p
```

**结果**：

```
Connection closed by UNKNOWN port 65535
```

**诊断过程**：

1. 测试 Clash 代理本身是否工作：

   ```bash
   curl -x socks5h://127.0.0.1:7898 https://www.google.com
   # ✅ 成功，代理正常工作
   ```

2. 测试代理连接服务器 SSH 端口：

   ```bash
   curl -x socks5h://127.0.0.1:7898 telnet://<hk-server-ip>:22
   # ✅ 可以连接
   ```

3. 查看 Clash 日志：
   ```
   [TCP] 127.0.0.1:53081(ncat) --> <hk-server-ip>:22
   match Match using Others[🇭🇰 香港 05]
   ```

**结论**：流量确实走了代理，但连接在 SSH 握手阶段被关闭。

**根本原因**：代理商**限制了 22 端口的 SSH 流量**。很多机场为了防止滥用，会限制 SSH、BT 等端口。

### 方案3：修改 SSH 监听端口到 443（成功✅）

#### 原理

代理商通常不会限制 443 端口（HTTPS），因为太多正常的 HTTPS 流量走这个端口。如果把 SSH 监听在 443 端口，流量就能伪装成 HTTPS，顺利通过代理。

#### 实施步骤

**服务器端配置：**

1. 检查系统 SSH 启动方式：

   ```bash
   sudo systemctl status ssh
   ```

   如果看到 `TriggeredBy: ● ssh.socket`，说明使用了 **systemd socket activation**。

> 在Ubuntu 22.10 或更高版本中，ssh 默认通过套接字激活

2. 对于 systemd socket 模式，需要修改 socket 配置文件：

   ```bash
   sudo vim /usr/lib/systemd/system/ssh.socket
   ```

   找到 `[Socket]` 部分，修改为：

   ```ini
   [Socket]
   ListenStream=22
   ListenStream=443
   ```

   > 对于 Ubuntu 24.04 中

   ```bash
   sudo systemctl restart ssh.service
   sudo systemctl daemon-reload
   sudo systemctl restart ssh.service
   ```

   有些云服务商为了启用远程密码登陆，会在 `/etc/ssh/sshd_config.d/` 自定义一个 conf 文件，修改 sshd 配置之前要先排除其干扰

   ```bash
   ls /etc/ssh/sshd_config.d/*.conf
   sudo nvim /etc/ssh/sshd_config.d/50-cloud-init.conf
   ```

   例如我这里九有`/etc/ssh/sshd_config.d/50-cloud-init.conf`，内容为

   ```text
   PasswordAuthentication yes
   ```

   配置完成后通过下面命令查看有效配置

   ```bash
   # Root 用户登录方式
   sudo sshd -T | grep -i "PermitRootLogin"
   # 密码认证
   sudo sshd -T | grep -i "PasswordAuthentication"
   # ssh 端口
   sudo sshd -T | grep -i "Port"
   ```

   如果看到输出 without-password 是正确的，因为 prohibit-password 就是其别名

3. 重新加载配置并重启：

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart ssh.socket
   ```

4. 确认监听端口：

   ```bash
   sudo ss -tlnp | grep -E ':(22|443)'
   # 应该看到 sshd 监听在 22 和 443 两个端口
   ```

5. 配置防火墙：
   ```bash
   sudo ufw allow 443/tcp
   ```

**客户端配置：**

修改 `~/.ssh/config`：

```ssh-config
Host blog
    Hostname <hk-server-ip>
    User sx
    Port 443  # 改用 443 端口
    ProxyCommand ncat --proxy 127.0.0.1:7898 --proxy-type socks5 %h %p
    ServerAliveInterval 30
    Compression yes
```

**测试连接：**

```bash
ssh blog
```

✅ **成功！流畅无卡顿！**

## 效果对比

| 指标     | 直连（优化前）               | 通过代理 443 端口（优化后）          |
| -------- | ---------------------------- | ------------------------------------ |
| 平均延迟 | 170-180ms                    | 30-50ms                              |
| 丢包率   | 4%                           | ~0%                                  |
| 连接路径 | 本地 → 国际出口（GFW）→ 香港 | 本地 → Clash → 香港代理节点 → 服务器 |
| 体验     | 明显卡顿                     | 流畅                                 |

## 进一步优化

### 1. SSH 连接复用

为了加快后续 SSH 会话（scp、rsync、git 等）的连接速度，可以启用连接复用：

```ssh-config
Host blog
    Hostname <hk-server-ip>
    User sx
    Port 443
    ProxyCommand ncat --proxy 127.0.0.1:7898 --proxy-type socks5 %h %p

    # 连接复用配置
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 10m

    ServerAliveInterval 30
    Compression yes
```

创建 socket 目录：

```bash
mkdir -p ~/.ssh/sockets
```

**工作原理**：首次 SSH 连接会建立一个主连接，后续的 SSH、scp、git 等操作会复用这个连接，避免重复握手和认证。

### 2. 修复 macOS 上的其他代理配置

还有在 GitHub SSH 配置中使用了：

```ssh-config
ProxyCommand nc -x 127.0.0.1:7897 -X 5 %h %p
```

这个命令在 **macOS 上不工作**，因为 macOS 的 `nc` 不支持 `-X` 选项。

正确的配置应该是：

```ssh-config
Host github.com
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_rsa
    ProxyCommand ncat --proxy 127.0.0.1:7898 --proxy-type socks5 %h %p
```

## 技术要点总结

### 1. systemd socket activation

传统的 SSH 服务启动方式是：`sshd` 进程始终运行，监听指定端口。

而 systemd socket activation 模式下：

- systemd 先监听端口
- 有连接时才启动 `sshd` 进程处理
- 更节省资源

在这种模式下，监听端口由 `/usr/lib/systemd/system/ssh.socket` 控制，而不是 `/etc/ssh/sshd_config` 中的 `Port` 配置。

### 2. 为什么代理商限制 SSH？

- **滥用防护**：SSH 可用于端口转发、隧道，容易被滥用
- **流量特征**：SSH 流量特征明显，容易识别
- **商业考虑**：鼓励用户使用 HTTPS 等"正常"协议

### 3. 端口 443 的特殊性

- 443 是标准 HTTPS 端口
- 大量正常流量使用此端口
- 代理商、防火墙通常不会限制
- 成为"万能端口"（SSH over 443、VPN over 443 等）

### 4. DPI（深度包检测）

有些代理商可能使用 DPI 技术，即使改成 443 端口，仍然能通过流量特征（如 SSH 握手时的 `SSH-2.0-OpenSSH` 字符串）识别出 SSH 流量。

本文的方案在大多数情况下有效，但如果遇到严格的 DPI 检测，可能需要：

- 使用 obfsproxy 混淆 SSH 流量
- 使用 V2Ray/Xray 的 WebSocket + TLS 伪装
- 换用更宽松的代理服务

## 故障排查检查清单

如果遇到类似问题，按以下步骤诊断：

- [ ] 测试直连延迟和丢包：`ping <server>`
- [ ] 确认 SSH 是否走代理：`ssh -v <host> 2>&1 | grep proxy`
- [ ] 测试代理本身是否工作：`curl -x socks5h://127.0.0.1:7898 https://www.google.com`
- [ ] 测试代理连接服务器端口：`curl -x socks5h://127.0.0.1:7898 telnet://<server>:22`
- [ ] 检查 Clash 日志，确认流量路由
- [ ] 如果连接被拒绝，尝试改用 443 端口
- [ ] 检查服务器是否真的监听了新端口：`ss -tlnp | grep :443`
- [ ] 注意 systemd socket activation 的特殊配置方式

## 总结

SSH 连接卡顿看似简单，实际涉及多个层面：

1. **网络层**：直连路由质量差
2. **应用层**：SSH 默认不走系统代理
3. **策略层**：代理商限制特定端口
4. **系统层**：不同的服务启动机制（systemd socket）

通过系统化的诊断，找到了真正的瓶颈：代理商对 22 端口的限制。通过将 SSH 迁移到 443 端口，既绕过了限制，又利用了代理的加速效果，最终实现了流畅的 SSH 体验。
