# WireGuard VPN 搭建与使用教程 / WireGuard VPN Setup & Usage Guide

> **中英双语 | Bilingual (Chinese & English)**
>
> 一份从零开始搭建 WireGuard VPN 的完整指南，涵盖安装、配置、Docker 部署、客户端管理及故障排除。
>
> A complete guide to setting up WireGuard VPN from scratch, covering installation, configuration, Docker deployment, client management, and troubleshooting.

---

## 📖 目录 / Table of Contents

1. [什么是 WireGuard？ / What is WireGuard?](#-什么是-wireguard--what-is-wireguard)
2. [安装 WireGuard / Installation](#-安装-wireguard--installation)
3. [密钥生成 / Key Generation](#-密钥生成--key-generation)
4. [服务端配置 / Server Configuration](#-服务端配置--server-configuration)
5. [客户端配置 / Client Configuration](#-客户端配置--client-configuration)
6. [Docker 部署 / Docker Deployment](#-docker-部署--docker-deployment)
7. [Docker Compose 部署 / Docker Compose Setup](#-docker-compose-部署--docker-compose-setup)
8. [管理 Peer / Peer Management](#-管理-peer--peer-management)
9. [路由与防火墙 / Routing & Firewall](#-路由与防火墙--routing--firewall)
10. [手机端二维码配置 / QR Code for Mobile](#-手机端二维码配置--qr-code-for-mobile)
11. [常见问题排查 / Troubleshooting](#-常见问题排查--troubleshooting)
12. [性能调优 / Performance Tuning](#-性能调优--performance-tuning)
13. [支持这个项目 / Support This Project](#-支持这个项目--support-this-project)

---

## 🔍 什么是 WireGuard？ / What is WireGuard?

WireGuard 是一个**极简、高速、现代化的 VPN 协议**，于 2020 年合并到 Linux 5.6 内核中。它由 Jason A. Donenfeld 开发，目标是在提供强加密的同时做到极致简单和高效。

WireGuard is a **minimal, high-speed, modern VPN protocol** that was merged into the Linux 5.6 kernel in 2020. Developed by Jason A. Donenfeld, it aims to deliver strong encryption while being extremely simple and efficient.

### 为什么 WireGuard 比 OpenVPN 更快更轻？/ Why is WireGuard Faster & Lighter than OpenVPN?

| 对比维度 | WireGuard | OpenVPN |
|---------|-----------|---------|
| **代码量** | ~4,000 行内核代码 | ~600,000+ 行代码 |
| **加密套件** | 仅 3 种：Curve25519、ChaCha20、Poly1305 | 大量加密套件组合 |
| **内核集成** | Linux 内核原生集成，零上下文切换 | 用户态，需频繁内核/用户切换 |
| **连接方式** | 无连接 UDP，静默时零流量 | TLS 握手保活，持续心跳 |
| **密钥交换** | Noise 协议框架，1-RTT | TLS 多轮握手 |
| **并发处理** | 每个 CPU 核心独立队列 | 共享状态锁竞争 |

**核心原因 / Core reasons:**

1. **代码极简** — 仅约 4,000 行代码，攻击面小，审计容易。Extremely small codebase (~4K lines), small attack surface, easy to audit.
2. **内核级运行** — 直接运行在内核态，避免数据在用户态和内核态之间多次拷贝。Runs in kernel space, avoiding costly user-space/kernel-space data copies.
3. **现代加密原语** — 使用 Curve25519 ECDH + ChaCha20-Poly1305 AEAD，硬件加速友好。Modern crypto primitives with hardware acceleration support.
4. **无连接设计** — 像 UDP 一样无状态，长时间空闲不消耗资源。Connectionless design — no state kept during idle, zero resource consumption.
5. **平滑漫游** — IP 地址变更自动重连，无需重启隧道。Seamless roaming — automatically reconnects when IP changes.

---

## 📦 安装 WireGuard / Installation

### Linux (Ubuntu / Debian)

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install wireguard wireguard-tools resolvconf -y

# 或者安装最新的主线内核版本 (可选)
# Or install the latest mainline kernel version (optional)
sudo add-apt-repository ppa:wireguard/wireguard
sudo apt update
sudo apt install wireguard -y
```

### Linux (CentOS / RHEL / Oracle Linux 8+)

```bash
# CentOS 8 / RHEL 8 / Oracle Linux 8
sudo dnf install elrepo-release epel-release -y
sudo dnf install kmod-wireguard wireguard-tools -y

# Oracle Linux 9 (内核已内置 WireGuard)
# Oracle Linux 9 (kernel has built-in WireGuard)
sudo dnf install wireguard-tools -y
```

> ⚠️ **注意 / Note**: 如果内核版本低于 5.6，需要通过 ELRepo 安装 WireGuard 内核模块。Oracle Linux 7 需启用 `ol7_UEKR6` 仓库。
> If kernel version < 5.6, install WireGuard kernel module via ELRepo. OL7 needs `ol7_UEKR6` repo.

### macOS

```bash
# 使用 Homebrew / Using Homebrew
brew install wireguard-tools

# 或从官网下载 GUI 应用
# Or download GUI app from: https://www.wireguard.com/install/
```

### Windows

1. 从官网下载安装包：https://www.wireguard.com/install/
2. 运行安装程序，按照向导完成安装
3. 安装后系统托盘会出现 WireGuard 图标

Download from: https://www.wireguard.com/install/ — run the installer, and a tray icon will appear.

### iOS / Android

| 平台 | 下载方式 | 获取途径 |
|------|---------|---------|
| **iOS** | App Store | 搜索 "WireGuard" 官方应用 |
| **Android** | Google Play / F-Droid | 搜索 "WireGuard" 官方应用 |
| **APK 直装** | GitHub Releases | https://github.com/WireGuard/wireguard-android/releases |

---

## 🔑 密钥生成 / Key Generation

WireGuard 使用公钥加密（Curve25519 ECDH）。每个 Peer 需要一对密钥。

WireGuard uses public-key cryptography (Curve25519 ECDH). Each peer needs a key pair.

### 生成服务端密钥 / Generate Server Keys

```bash
cd /etc/wireguard
umask 077  # 确保密钥文件权限为 600 / Ensure key file permission is 600

# 生成私钥 / Generate private key
wg genkey | tee server.key

# 生成公钥 / Generate public key
cat server.key | wg pubkey | tee server.pub
```

### 生成客户端密钥 / Generate Client Keys

```bash
# 为每个客户端单独生成 / Generate for each client
wg genkey | tee client1.key
cat client1.key | wg pubkey | tee client1.pub

wg genkey | tee client2.key
cat client2.key | wg pubkey | tee client2.pub
```

### 快速生成（一行命令）/ Quick One-Liner

```bash
# 一键生成私钥和公钥到变量 / Generate both into variables
PRIVKEY=$(wg genkey)
PUBKEY=$(echo "$PRIVKEY" | wg pubkey)
echo "Private: $PRIVKEY"
echo "Public:  $PUBKEY"
```

> 🔐 **安全提醒 / Security Reminder**: 私钥绝不要泄露！公钥可以自由分享。Never expose private keys! Public keys are safe to share.

---

## 🖥️ 服务端配置 / Server Configuration

### 配置文件位置 / Config File Location

```
/etc/wireguard/wg0.conf
```

### 示例配置 / Example Configuration

```ini
[Interface]
# 服务端本机地址 / Server's VPN address
Address = 10.0.0.1/24

# 监听端口 / Listen port
ListenPort = 51820

# 服务端私钥 / Server private key (请替换为你的密钥 / REPLACE WITH YOUR KEY)
PrivateKey = <SERVER_PRIVATE_KEY>

# 启用 IP 转发 / Enable IP forwarding
#PostUp = sysctl -w net.ipv4.ip_forward=1
#PostDown = sysctl -w net.ipv4.ip_forward=0

# NAT 转发（如果需要客户端通过 VPN 上网）
# NAT masquerade (if clients need internet via VPN)
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# 可选：设置 MTU / Optional: Set MTU
# MTU = 1420

# DNS 服务器（可选）/ DNS servers (optional)
# DNS = 1.1.1.1, 8.8.8.8

# ========== Peer 1: Client 1 ==========
[Peer]
# 客户端公钥 / Client public key
PublicKey = <CLIENT1_PUBLIC_KEY>

# 允许的 IP 段 / Allowed IPs (client's VPN IP)
AllowedIPs = 10.0.0.2/32

# ========== Peer 2: Client 2 ==========
[Peer]
PublicKey = <CLIENT2_PUBLIC_KEY>
AllowedIPs = 10.0.0.3/32
```

### 启动服务 / Start the Service

```bash
# 启动 WireGuard 接口 / Bring up the WireGuard interface
sudo wg-quick up wg0

# 设为开机自启 / Enable on boot
sudo systemctl enable wg-quick@wg0

# 查看接口状态 / Check interface status
sudo wg show

# 查看路由 / Check routing
ip route show

# 停止接口 / Bring down the interface
sudo wg-quick down wg0
```

---

## 💻 客户端配置 / Client Configuration

### Linux 客户端 / Linux Client

**配置文件**: `/etc/wireguard/wg0.conf`

```ini
[Interface]
# 客户端 VPN 地址 / Client VPN address
Address = 10.0.0.2/24

# 客户端私钥 / Client private key
PrivateKey = <CLIENT1_PRIVATE_KEY>

# DNS 服务器 / DNS servers
DNS = 1.1.1.1, 8.8.8.8

[Peer]
# 服务端公钥 / Server public key
PublicKey = <SERVER_PUBLIC_KEY>

# 服务端公网地址和端口 / Server public address and port
Endpoint = your-server-ip:51820

# 允许所有流量走 VPN（全隧道）/ Route all traffic through VPN (full tunnel)
AllowedIPs = 0.0.0.0/0, ::/0

# 保持连接（NAT 穿透）/ Keepalive (NAT traversal)
PersistentKeepalive = 25
```

```bash
# 启动客户端 / Start client
sudo wg-quick up wg0

# 验证连接 / Verify connection
sudo wg show
ping 10.0.0.1
```

### 分隧道配置 / Split Tunnel Configuration

如果只想 VPN 流量去内网，普通上网走本地：

If you only want VPN traffic for the private network and regular internet via local connection:

```ini
[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = your-server-ip:51820
# 仅内网流量走 VPN / Only internal traffic via VPN
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24
PersistentKeepalive = 25
```

### macOS / Windows 客户端

1. 下载并安装 WireGuard GUI 客户端
2. 点击 "Add Tunnel" → "Add empty tunnel..."
3. 粘贴客户端配置内容
4. 点击 "Activate"

Or create a `.conf` file and import it via "Import tunnel(s) from file...".

---

## 🐳 Docker 部署 / Docker Deployment

使用 `linuxserver/wireguard` 镜像，一键部署 WireGuard 服务端。

Deploy WireGuard server in one command with the `linuxserver/wireguard` Docker image.

### 快速启动 / Quick Start

```bash
docker run -d \
  --name=wireguard \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Asia/Shanghai \
  -e SERVERURL=your-server-ip \
  -e SERVERPORT=51820 \
  -e PEERS=3 \
  -e PEERDNS=1.1.1.1 \
  -e INTERNAL_SUBNET=10.13.13.0 \
  -e ALLOWEDIPS=0.0.0.0/0 \
  -p 51820:51820/udp \
  -v /path/to/wireguard-config:/config \
  -v /lib/modules:/lib/modules \
  --sysctl="net.ipv4.conf.all.src_valid_mark=1" \
  --sysctl="net.ipv4.ip_forward=1" \
  --restart=unless-stopped \
  linuxserver/wireguard
```

### 参数说明 / Parameter Explanation

| 参数 | 说明 | Description |
|------|------|-------------|
| `SERVERURL` | 服务端公网 IP 或域名 | Server public IP or domain |
| `SERVERPORT` | 监听端口（默认 51820） | Listen port (default 51820) |
| `PEERS` | 自动创建的客户端数量 | Number of clients to auto-create |
| `PEERDNS` | 客户端 DNS 服务器 | Client DNS server |
| `INTERNAL_SUBNET` | VPN 内网子网 | VPN internal subnet |
| `ALLOWEDIPS` | 允许的路由（`0.0.0.0/0` = 全隧道） | Allowed routes (`0.0.0.0/0` = full tunnel) |

### 查看客户端配置 / View Client Config

```bash
# Docker 容器会自动生成客户端配置
# Docker container auto-generates client configs
docker exec -it wireguard cat /config/peer1/peer1.conf

# 容器日志中包含二维码 / Container logs contain QR codes
docker logs wireguard
```

客户端配置文件在宿主机 `<config-path>/peer1/peer1.conf`，可直接导入客户端。

---

## 📋 Docker Compose 部署 / Docker Compose Setup

创建 `docker-compose.yml`:

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  wireguard:
    image: linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - SERVERURL=your-server-ip    # ← 替换为你的服务器 IP
      - SERVERPORT=51820
      - PEERS=3                     # ← 客户端数量
      - PEERDNS=1.1.1.1
      - INTERNAL_SUBNET=10.13.13.0
      - ALLOWEDIPS=0.0.0.0/0        # 全隧道；内网用 10.13.13.0/24
    volumes:
      - ./config:/config
      - /lib/modules:/lib/modules
    ports:
      - "51820:51820/udp"
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
    restart: unless-stopped
```

启动 / Start:

```bash
docker compose up -d

# 查看状态 / Check status
docker compose logs -f

# 查看生成的客户端配置 / View generated client configs
ls -la config/peer*/
cat config/peer1/peer1.conf
```

---

## 👥 管理 Peer / Peer Management

### 手动添加客户端 / Add a Client Manually

如果你不使用 Docker，需要手动管理 Peer。

If you're not using Docker, manage peers manually:

```bash
# 1. 客户端生成密钥 / Client generates keys
# 2. 服务端添加 Peer / Add peer on server

# 编辑服务端配置 / Edit server config
sudo vim /etc/wireguard/wg0.conf

# 添加以下内容 / Add the following:
[Peer]
PublicKey = <NEW_CLIENT_PUB_KEY>
AllowedIPs = 10.0.0.4/32

# 3. 重载配置 / Reload config
sudo wg addconf wg0 <(wg-quick strip wg0)
# 或重启 / Or restart
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

### 使用 wg 命令添加（无需重启）/ Add via wg Command (No Restart)

```bash
# 动态添加 Peer，无需重启 / Dynamically add peer without restart
sudo wg set wg0 peer <CLIENT_PUB_KEY> allowed-ips 10.0.0.5/32

# 验证 / Verify
sudo wg show
```

### 删除客户端 / Remove a Client

```bash
# 从配置文件中删除对应的 [Peer] 段落
# Delete the corresponding [Peer] section from config file

# 然后移除 Peer（无需重启）/ Then remove peer (no restart)
sudo wg set wg0 peer <CLIENT_PUB_KEY> remove

# 验证 / Verify
sudo wg show
```

### Docker 环境中的 Peer 管理 / Peer Management in Docker

```bash
# Docker 容器会自动管理 PEERS 参数
# 如果 PEERS=3，会生成 peer1, peer2, peer3

# 增加客户端数量 / Increase number of clients
# 停止容器 → 修改 PEERS=N → 启动容器
# Stop container → change PEERS=N → start container

# 新增客户端配置在宿主机路径下
# New client configs appear under the host config path
ls ./config/peer*/

# 重新生成二维码 / Regenerate QR codes
docker exec -it wireguard app.sh
```

---

## 🔥 路由与防火墙 / Routing & Firewall

### 启用 IP 转发 / Enable IP Forwarding

```bash
# 临时启用 / Temporary
sudo sysctl -w net.ipv4.ip_forward=1

# 永久启用 / Permanent
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 防火墙配置 / Firewall Configuration

```bash
# 开放 WireGuard UDP 端口 / Open WireGuard UDP port
# ufw
sudo ufw allow 51820/udp
sudo ufw reload

# firewalld (CentOS / Oracle Linux)
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload

# iptables
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -A FORWARD -i wg0 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# 保存 iptables 规则 / Save iptables rules
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### 数据包转发规则说明 / Packet Forwarding Rules

```
客户端 (10.0.0.2) → wg0 → iptables FORWARD ACCEPT → eth0 → 互联网
                                                 ↓
                                       MASQUERADE (SNAT) → 源 IP 替换为服务器公网 IP
```

`MASQUERADE` 将客户端内网 IP (10.0.0.x) 转换为服务器公网 IP，使互联网能正确回复数据包。

---

## 📱 手机端二维码配置 / QR Code for Mobile

WireGuard 官方客户端支持通过扫描二维码导入配置。使用 `qrencode` 工具生成二维码。

The WireGuard official app supports importing configs via QR codes. Use `qrencode` to generate them.

### 安装 qrencode / Install qrencode

```bash
# Ubuntu/Debian
sudo apt install qrencode -y

# macOS
brew install qrencode

# CentOS/RHEL
sudo dnf install qrencode -y
```

### 生成二维码 / Generate QR Code

```bash
# 从客户端配置文件生成 / Generate from client config file
qrencode -t png -o peer1-qr.png -r /path/to/peer1.conf

# 直接在终端打印 ASCII 二维码 / Print ASCII QR code in terminal
qrencode -t ansiutf8 < /path/to/peer1.conf

# 从字符串生成 / Generate from string
cat peer1.conf | qrencode -t ansiutf8
```

### Docker 环境自动生成二维码 / Docker Auto-Generated QR

```bash
# Docker 容器启动时自动在日志中输出 ASCII 二维码
# Docker container outputs ASCII QR codes in logs on startup
docker logs wireguard 2>&1 | grep -A 30 "peer1"

# 或保存到文件 / Or save to file
docker logs wireguard 2>&1 | sed -n '/peer1/,/peer2/p' > peer1-qr.txt
```

---

## 🔧 常见问题排查 / Troubleshooting

### 问题 1: 连接失败 / Connection Failure

**症状 / Symptoms**: `ping 10.0.0.1` 无响应，`wg show` 显示 `latest handshake: (none)`

**排查步骤 / Steps**:
1. 检查防火墙是否开放 `51820/udp` 端口
2. 检查服务端公网 IP 是否正确
3. 确认客户端和服务端公钥配对正确
4. 查看 WireGuard 日志:

```bash
# Linux
sudo journalctl -u wg-quick@wg0 -f

# 或查看内核日志 / Or check kernel logs
sudo dmesg | grep wireguard

# Docker
docker logs wireguard
```

### 问题 2: 能 Ping 通 VPN IP 但无法上网 / Can Ping VPN IP but No Internet

**原因 / Cause**: 未启用 IP 转发或 iptables MASQUERADE

**解决 / Solution**:
```bash
# 检查 IP 转发 / Check IP forwarding
sysctl net.ipv4.ip_forward
# 如果为 0，则启用 / If 0, enable it

# 检查 MASQUERADE 规则 / Check MASQUERADE rule
sudo iptables -t nat -L POSTROUTING -v
```

### 问题 3: 传输速度慢 / Slow Transfer Speeds

**原因 / Cause**: MTU 设置不当或 UDP 被 QoS 限速

**解决 / Solution**:
- 尝试降低 MTU 至 1280-1420
- 检查运营商是否对 UDP 做 QoS 限制
- 服务端网卡支持硬件校验和卸载时关闭 GorMSS 平摊

### 问题 4: 客户端连接一段时间后断开 / Client Drops After Some Time

**原因 / Cause**: NAT 超时或连接保活不足

**解决 / Solution**:
```bash
# 在客户端配置中添加 / Add to client config:
PersistentKeepalive = 25
```

### 问题 5: Docker 容器启动报错 "Operation not permitted"

**原因 / Cause**: 缺少 `NET_ADMIN` 或 `SYS_MODULE` cap_add

**解决 / Solution**: 确保 docker-compose.yml 中包含:
```yaml
cap_add:
  - NET_ADMIN
  - SYS_MODULE
```

### 问题 6: WireGuard 模块未加载 / Module Not Loaded

```bash
# 检查内核模块 / Check kernel module
lsmod | grep wireguard

# 手动加载 / Manually load
sudo modprobe wireguard

# 如果 modprobe 失败，需要安装内核模块
# If modprobe fails, install kernel module
sudo apt install wireguard-dkms  # Ubuntu/Debian
```

### 常用调试命令 / Useful Debug Commands

```bash
# 查看 WireGuard 状态 / View WireGuard status
sudo wg show

# 查看详细接口信息 / Verbose interface info
sudo wg show wg0

# 查看路由表 / View routing table
ip route show table all | grep wg

# 抓包调试 / Packet capture debugging
sudo tcpdump -i wg0 -n
sudo tcpdump -i eth0 port 51820 -n

# 查看连接状态 / Check connection state
ip addr show wg0
ip link show wg0
```

---

## ⚡ 性能调优 / Performance Tuning

### MTU 优化 / MTU Optimization

WireGuard 默认 MTU 为 1420。根据网络环境调整可提升性能。

Default MTU is 1420. Adjusting based on network conditions can improve performance.

```ini
# wg0.conf 中的 Interface 段 / In the Interface section
MTU = 1420
```

| 网络环境 | 推荐 MTU | 说明 |
|---------|---------|------|
| 以太网 | 1420 | 默认值，适合大多数场景 |
| PPPoE | 1412 | 减去 8 字节 PPPoE 头部 |
| 蜂窝网络 / LTE | 1280-1350 | 蜂窝网络可能有较小 MTU |
| 卫星网络 | 1280 | 高延迟场景使用较低 MTU |
| 有线 / 千兆 | 1420-1500 | 可尝试加大 MTU 测试 |

### 并发连接调优 / Concurrency Tuning

```bash
# 设置网络队列长度 / Set network queue length
sudo ip link set dev wg0 txqueuelen 1000

# 调整内核网络缓冲区 / Tune kernel network buffers
echo "net.core.rmem_max = 2500000" | sudo tee -a /etc/sysctl.conf
echo "net.core.wmem_max = 2500000" | sudo tee -a /etc/sysctl.conf

# 调整 UDP 接收缓冲区 / Tune UDP receive buffer
echo "net.core.rmem_default = 2500000" | sudo tee -a /etc/sysctl.conf
echo "net.core.wmem_default = 2500000" | sudo tee -a /etc/sysctl.conf

sudo sysctl -p
```

### 多核利用 / Multi-Core Utilization

WireGuard 从内核 5.6 开始原生支持多核并行。每个 CPU 核心有自己的发送队列，无需额外配置。

WireGuard natively supports multi-core parallelism since kernel 5.6. Each CPU core has its own transmit queue — no extra configuration needed.

```bash
# 检查队列数（应等于 CPU 核心数）/ Check queue count (should equal CPU cores)
ethtool -l wg0 2>/dev/null || echo "wg0 is a virtual interface"

# 查看 CPU 使用 / Check CPU usage
top -1
```

### UDP 优化 / UDP Optimization

```bash
# 开启 GRO (Generic Receive Offload) / Enable GRO
sudo ethtool -K eth0 gro on

# 开启 GSO (Generic Segmentation Offload) / Enable GSO
sudo ethtool -K eth0 gso on

# 开启 TSO (TCP Segmentation Offload) / Enable TSO
sudo ethtool -K eth0 tso on

# 增大 UDP 接收队列 / Increase UDP receive queue
echo "net.core.netdev_max_backlog = 5000" | sudo tee -a /etc/sysctl.conf

sudo sysctl -p
```

### 服务端并发限制 / Server Concurrency Limits

```bash
# 查看系统文件描述符限制 / Check system file descriptor limit
ulimit -n

# 增加最大打开文件数 / Increase max open files
echo "fs.file-max = 100000" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# /etc/security/limits.conf
# * soft nofile 100000
# * hard nofile 100000
```

### 测速工具 / Speed Testing Tools

```bash
# 安装 iperf3 / Install iperf3
sudo apt install iperf3 -y

# 服务端（VPN 地址）/ Server (VPN address)
iperf3 -s -B 10.0.0.1

# 客户端 / Client
iperf3 -c 10.0.0.1

# 双向测试 / Bidirectional test
iperf3 -c 10.0.0.1 -d
```

### 性能对比参考 / Performance Reference

| 场景 | OpenVPN (AES-256) | WireGuard (ChaCha20) | 提升比例 |
|------|-------------------|----------------------|---------|
| 100M 宽带 | ~80 Mbps | ~95 Mbps | ~19% |
| 1G 宽带 | ~300 Mbps | ~900 Mbps | ~200% |
| 10G 宽带 | ~800 Mbps | ~3.5 Gbps | ~340% |
| 树莓派 4 | ~80 Mbps | ~400 Mbps | ~400% |
| 手机端 | ~20 Mbps | ~100 Mbps | ~400% |

*(数据基于常见测试环境，实际结果因硬件和网络条件而异 / Numbers based on common test environments, actual results vary)*

---

## 📄 完整示例配置 / Complete Example Configs

### 服务端完整配置 / Full Server Config (`/etc/wireguard/wg0.conf`)

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>
MTU = 1420

# NAT 与转发 / NAT & forwarding
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# 手机客户端 / Phone client
PublicKey = <CLIENT1_PUB_KEY>
AllowedIPs = 10.0.0.2/32

[Peer]
# 笔记本客户端 / Laptop client
PublicKey = <CLIENT2_PUB_KEY>
AllowedIPs = 10.0.0.3/32

[Peer]
# 办公室内网路由 / Office LAN
PublicKey = <CLIENT3_PUB_KEY>
AllowedIPs = 10.0.0.4/32, 192.168.1.0/24
```

### 客户端完整配置 / Full Client Config

```ini
[Interface]
PrivateKey = <CLIENT1_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 1.1.1.1, 8.8.8.8
MTU = 1420

[Peer]
PublicKey = <SERVER_PUB_KEY>
Endpoint = your-server-ip:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

---

## ✅ 快速安装脚本 / Quick Install Script

如果你想要一个自动化安装脚本：

If you want an automated install script:

```bash
#!/bin/bash
# WireGuard 一键安装脚本 / WireGuard Quick Install Script
# 用法 / Usage: bash install-wireguard.sh

set -e

# 检测系统 / Detect system
if [ -f /etc/debian_version ]; then
    apt update && apt install wireguard wireguard-tools resolvconf -y
elif [ -f /etc/redhat-release ]; then
    dnf install -y elrepo-release epel-release
    dnf install -y kmod-wireguard wireguard-tools
fi

# 启 IP 转发 / Enable IP forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# 生成密钥 / Generate keys
cd /etc/wireguard
umask 077
wg genkey | tee server.key | wg pubkey > server.pub

echo "====== WireGuard 安装完成 / Installation Complete ======"
echo "Server Private Key: $(cat server.key)"
echo "Server Public Key:  $(cat server.pub)"
```

---

## 📚 参考资源 / References

- [WireGuard 官方网站 / Official Site](https://www.wireguard.com/)
- [WireGuard 白皮书 / Whitepaper](https://www.wireguard.com/papers/wireguard.pdf)
- [WireGuard GitHub](https://github.com/WireGuard/wireguard-linux)
- [linuxserver/wireguard Docker 镜像](https://docs.linuxserver.io/images/docker-wireguard/)
- [Arch Wiki: WireGuard](https://wiki.archlinux.org/title/WireGuard)

---

## ☕ 支持这个项目 / Support This Project

如果你觉得这个教程对你有帮助，欢迎请我喝杯咖啡 ☕  
你的支持是我持续更新的最大动力！非常感谢！🙏

If you find this tutorial helpful, feel free to buy me a coffee ☕  
Your support is my biggest motivation to keep improving! Thank you so much! 🙏

**USDT (TRC20)**
```
TVbQerV1SF4MXB1JCcAzQxarewHwEPYTKm
```
