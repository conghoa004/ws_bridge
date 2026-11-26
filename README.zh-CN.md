<p align="center">
  <h1 align="center">🌉 WSB — WebSocket Bridge</h1>
  <p align="center">
    <strong>用 Rust 构建的高性能多协议 WebSocket 隧道工具。</strong>
  </p>
  <p align="center">
    <img alt="Platform" src="https://img.shields.io/badge/Platform-Linux%20x86__64-FCC624?logo=linux&logoColor=black">
    <img alt="Tokio" src="https://img.shields.io/badge/async-Tokio-blue?logo=rust">
    <img alt="License" src="https://img.shields.io/badge/license-MIT-green">
    <img alt="Version" src="https://img.shields.io/badge/version-0.2.0-blueviolet">
  </p>
  <p align="center">
    <a href="README.md">English</a> ｜ <strong>中文</strong> ｜ <a href="README.vi.md">Tiếng Việt</a>
  </p>
</p>

---

**WSB（`wsb` v0.2.0）** 将 **TCP**、**UDP**、**HTTP** 和 **SOCKS5** 数据流封装并通过 **WebSocket**（`ws://` / `wss://`）传输，让您能够通过 Web 隧道（例如 **Cloudflare Quick Tunnel**）安全访问远程主机（如 **Kaggle Notebooks**、Google Colab、Docker 容器或 Linux VPS/虚拟机）上的任意内部服务——**无需开放原始 TCP/UDP 端口或固定 IP 地址**。

当前项目目录包含预编译的 **Linux x86_64** 可执行文件 [`wsb.txt`](wsb.txt)（保存为 `.txt` 扩展名以便上传下载传输）以及服务端和客户端示例 TOML 配置文件。

## 目录

- [功能特性](#-功能特性)
- [架构设计与数据流](#-架构设计与数据流)
- [准备工作](#-准备工作)
- [快速开始](#-快速开始)
  - [1. 服务端（Kaggle / 远程 Linux 主机）](#1-服务端kaggle--远程-linux-主机)
  - [2. 客户端（本地 Linux 机器）](#2-客户端本地-linux-机器)
- [使用示例](#-使用示例)
- [CLI 参考](#%EF%B8%8F-cli-参考)
- [TOML 配置](#-toml-配置)
- [项目结构](#%EF%B8%8F-项目结构)
- [许可证](#-许可证)

---

## ✨ 功能特性

| 特性 | 描述 |
|---|---|
| ⚡ **预编译 Linux 二进制** | 提供独立的 Linux `x86_64` ELF 可执行文件（`wsb.txt` → `wsb` v0.2.0），基于 Rust + Tokio（`tokio-tungstenite`）构建——低延迟、低内存占用、无需安装 Rust 工具链。 |
| 🔌 **TCP 隧道** | 转发任意 TCP 服务：MySQL、PostgreSQL、Redis、SSH、MongoDB、Kafka 等。 |
| 📡 **UDP 隧道** | 传输 UDP 数据报（单包最大 64 KB，如 DNS、Syslog、游戏服务器），支持按来源地址自动会话跟踪（100 个数据包缓冲队列）及 60 秒空闲超时。 |
| 🌐 **HTTP 隧道** | 访问远程 Web 应用（phpMyAdmin、REST API、管理后台），**自动重写 `Host` 请求头**为目标地址，支持 HTTP/1.1 Keep-Alive、`Content-Length`、`Transfer-Encoding: chunked` 以及 `Upgrade`/WebSocket 透传。 |
| 🧦 **SOCKS5 动态代理** | 内置本地 SOCKS5 代理（无认证 `CONNECT`），支持 IPv4、IPv6 和域名——通过单一本地端口即可让浏览器、`curl`、数据库 GUI、SSH 访问远程**任意端口/服务**。 |
| 🔒 **安全性** | 握手阶段使用恒定时间比较校验 Token，支持启动时自动生成 256 位（64 位十六进制）安全 Token 或自定义静态 Token，服务端支持 `allowed_targets` 白名单。 |
| ⚙️ **灵活配置** | 支持 TOML 配置文件（`config.server.toml`、`config.client.toml`）、CLI 参数与环境变量（`WS_BRIDGE_LISTEN`、`WS_BRIDGE_URL`、`WS_BRIDGE_TOKEN`、`RUST_LOG`）自由组合。 |
| ☁️ **Cloudflare Tunnel 兼容** | 与 `cloudflared tunnel --url http://127.0.0.1:8766` 无缝配合。 |
| 🔐 **TLS 与网络容错** | Linux/macOS 构建使用 `rustls`（WebPKI 根证书），运行时不依赖外部 OpenSSL。客户端 DNS 解析后**优先尝试 IPv4 再回退 IPv6**（每个地址超时 10 秒），并启用 `TCP_NODELAY`。 |
| 🫀 **心跳保活与半关闭** | 每 25 秒发送一次 WebSocket `Ping` 帧防止代理/CDN 断连，并通过 `wsb:eof` 控制消息端到端保留 TCP 半关闭语义。 |

---

## 📐 架构设计与数据流

```text
  本地机器（客户端）                               远程 Linux 主机（服务端）
  ================                               ========================

  [ MySQL / DBeaver   ] ─ TCP :13306 ─┐
  [ Redis CLI         ] ─ TCP :16379 ─┤
  [ Web 浏览器        ] ─ HTTP :18080 ┤
  [ DNS 客户端        ] ─ UDP :10053 ─┤
  [ SOCKS5 应用       ] ─ SOCKS5 :1080┤
                                      │
                              ┌───────▼───────────────┐
                              │     wsb client        │
                              └───────┬───────────────┘
                                      │  WS 二进制帧 (ws:// 或 wss://)
                              ┌───────▼───────────────┐
                              │  Cloudflare Tunnel /   │
                              │  反向代理（可选）      │
                              │  (https://*.trycloudflare.com)
                              └───────┬───────────────┘
                                      │
                              ┌───────▼───────────────┐
                              │     wsb server        │  :8766
                              └───────┬───────────────┘
                                      │
                                      ├──► MySQL      :3306
                                      ├──► Redis      :6379
                                      ├──► Apache/Web :80
                                      ├──► SSH        :22
                                      └──► DNS        :53
```

**工作流程：**

1. **客户端**（`wsb client`）按配置监听本地 TCP、UDP、HTTP 和/或 SOCKS5 端口。
2. 当有连接到达时，客户端以 IPv4 优先回退方式建立 WebSocket 连接（`ws://` 或 `wss://`），发送包含 `token`、`protocol`（`tcp`、`udp` 或 `http`；SOCKS5 使用 `tcp`）和 `target` 的 **JSON 握手请求**。
3. **服务端**（`wsb server`）校验 Token（恒定时间比较）、检查 `allowed_targets` 白名单（如已配置）、**先连接目标服务**（超时 10 秒），再返回 `HandshakeResponse` JSON。
4. 数据通过 WebSocket 二进制帧双向转发，每 25 秒发送 Ping 心跳保活，并同步 TCP 半关闭状态。

---

## 📦 准备工作

在 Linux `x86_64` 上无需安装编译器或 Rust 工具链。将 [`wsb.txt`](wsb.txt) 重命名为 `wsb` 并赋予可执行权限：

```bash
mv wsb.txt wsb
chmod +x wsb
```

*（如果您希望在仓库目录中保留 `wsb.txt`，可使用 `cp wsb.txt wsb && chmod +x wsb`。）*

验证程序：

```bash
./wsb --version
# 输出: wsb 0.2.0
```

> [!TIP]
> 复制到 `PATH` 目录以便在任意位置运行：
> ```bash
> cp wsb ~/.local/bin/   # 或: sudo cp wsb /usr/local/bin/
> ```

> [!NOTE]
> Linux 版 `wsb` 同时支持 `server` 与 `client` 模式。如果本地电脑为 Windows，可在本地使用 Windows 版 `wsb.exe` 作为客户端，并在远程 Linux 主机（如 Kaggle、Google Colab、Ubuntu VPS）上运行 `./wsb server`。

---

## 📖 快速开始

### 1. 服务端（Kaggle / 远程 Linux 主机）

**使用示例配置文件（[`config.server.toml`](config.server.toml)）：**
```bash
./wsb server -c config.server.toml
```

**使用 CLI 参数：**
```bash
# 启动时自动生成 256 位随机安全 Token（推荐）：
./wsb server --listen 0.0.0.0:8766

# 或指定固定 Token 与目标白名单：
./wsb server --listen 0.0.0.0:8766 --token "my-secret-token" --allow 127.0.0.1:3306 --allow 127.0.0.1:80

# 生成一个 256 位（64 位十六进制）Token 用于写入配置文件：
./wsb gen-token
```

**可选 — 通过 Cloudflare Quick Tunnel 暴露服务：**
```bash
cloudflared tunnel --url http://127.0.0.1:8766 --no-autoupdate
# → https://xxx-xxx-xxx.trycloudflare.com
```
*（客户端将 `ws_url` 设为 `wss://xxx-xxx-xxx.trycloudflare.com`，直连时可使用 `ws://<服务器IP>:8766`。）*

### 2. 客户端（本地 Linux 机器）

**使用示例配置文件（[`config.client.toml`](config.client.toml)）：**

修改 `config.client.toml` 中的 `ws_url` 和 `token`，然后运行：
```bash
./wsb client -c config.client.toml
```

**使用 CLI 参数：**
```bash
./wsb client \
  --ws "wss://xxx-xxx-xxx.trycloudflare.com" \
  --token "my-secret-token" \
  --tcp 127.0.0.1:13306=127.0.0.1:3306 \
  --tcp 127.0.0.1:16379=127.0.0.1:6379 \
  --http 127.0.0.1:18080=127.0.0.1:80 \
  --udp 127.0.0.1:10053=1.1.1.1:53 \
  --socks5 127.0.0.1:1080
```

> [!TIP]
> 您也可以将 `-c config.client.toml` 与 `--ws`、`--token`、`--socks5` 或额外的 `--tcp` / `--udp` / `--http` 参数组合使用。

---

## 🎯 使用示例

### 通过 TCP 隧道连接 MySQL

使用 DBeaver、Navicat、MySQL Workbench 或 `mysql` 命令行客户端连接：
```bash
mysql -h 127.0.0.1 -P 13306 -u <your_user> -p
```
- **主机：** `127.0.0.1` &nbsp;|&nbsp; **端口：** `13306`

### 通过 HTTP 隧道访问 phpMyAdmin / Web

在浏览器中打开：
```text
http://127.0.0.1:18080/phpmyadmin/
```

> [!NOTE]
> `wsb` 会自动将请求的 `Host` 请求头重写为目标地址（例如 `127.0.0.1:80`），使 Apache/Nginx 正确匹配虚拟主机并避免重定向错误。

### 通过 UDP 隧道进行 DNS 查询

```bash
dig @127.0.0.1 -p 10053 google.com
# 或: nslookup -port=10053 google.com 127.0.0.1
```

### SOCKS5 动态代理

仅需**一个本地端口** `127.0.0.1:1080`，即可动态转发 TCP 流量至远程服务器上的任意主机和端口：

```bash
# 通过 curl 经 SOCKS5 访问 Web（远程 DNS 解析）
curl --socks5-hostname 127.0.0.1:1080 http://127.0.0.1:80/phpmyadmin/

# 通过 SOCKS5 代理 SSH
ssh -o "ProxyCommand=nc -X 5 -x 127.0.0.1:1080 %h %p" root@127.0.0.1
```

**浏览器：** 将 SOCKS5 代理配置为 `127.0.0.1:1080`（Chrome 使用 SwitchyOmega，或使用 Firefox 网络设置）。  
*（限制说明：内置 SOCKS5 仅支持无认证模式的 `CONNECT` 命令，不支持 `BIND` 和 `UDP ASSOCIATE`。）*

---

## ⚙️ CLI 参考

### `wsb server`

| 参数 | 环境变量 | 默认值 | 描述 |
|---|---|---|---|
| `-c, --config <CONFIG>` | — | — | 服务端 TOML 配置文件路径 |
| `-l, --listen <LISTEN>` | `WS_BRIDGE_LISTEN` | `0.0.0.0:8766` | WebSocket 监听地址 |
| `-t, --token <TOKEN>` | `WS_BRIDGE_TOKEN` | `auto` | 客户端认证 Token（`auto` / `generate` / 空值将在启动时自动生成 256 位 Token） |
| `--allow <ALLOWED>` | — | 全部目标 | 允许连接的 `host:port` 白名单（可多次指定；会覆盖配置中的 `allowed_targets`） |

### `wsb client`

| 参数 | 环境变量 | 默认值 | 描述 |
|---|---|---|---|
| `-c, --config <CONFIG>` | — | — | 客户端 TOML 配置文件路径 |
| `-w, --ws <WS_URL>` | `WS_BRIDGE_URL` | *必填* | 远程 WebSocket 服务 URL（`wss://...` 或 `ws://...`） |
| `-t, --token <TOKEN>` | `WS_BRIDGE_TOKEN` | `12345` | 与服务端一致的认证 Token |
| `--tcp <LISTEN=TARGET>` | — | — | 添加 TCP 隧道规则（可重复，例：`127.0.0.1:13306=127.0.0.1:3306`） |
| `--udp <LISTEN=TARGET>` | — | — | 添加 UDP 隧道规则（可重复，例：`127.0.0.1:10053=1.1.1.1:53`） |
| `--http <LISTEN=TARGET>` | — | — | 添加 HTTP 隧道规则并重写 `Host`（可重复） |
| `--socks5 <LISTEN_ADDR>` | — | — | 启用 SOCKS5 动态代理（例：`127.0.0.1:1080`；会覆盖配置中的 `socks5`） |

> [!TIP]
> 使用 `RUST_LOG=debug ./wsb ...` 运行可查看每条连接的隧道与握手详细日志，默认级别为 `wsb=info,info`。

### `wsb gen-token`

向标准输出打印一个加密安全的 256 位随机 Token（64 位小写十六进制字符）。

---

## 📄 TOML 配置

<details>
<summary><strong>config.server.toml</strong></summary>

```toml
# ===============================================
# WS-BRIDGE SERVER CONFIGURATION
# ===============================================

# Port to listen for incoming WebSocket connections
listen = "0.0.0.0:8766"

# Secret token required from clients to authenticate.
# Set to "auto" (default) to automatically generate a cryptographically secure 256-bit token on startup,
# or specify your own static secret token.
token = "auto"

# Optional security whitelist of permitted target addresses.
# If omitted or commented out, client can connect to any target address.
# allowed_targets = [
#     "127.0.0.1:3306",  # MySQL
#     "127.0.0.1:6379",  # Redis
#     "127.0.0.1:80",    # HTTP / phpMyAdmin
#     "127.0.0.1:22",    # SSH
#     "1.1.1.1:53",      # DNS
# ]
```
</details>

<details>
<summary><strong>config.client.toml</strong></summary>

```toml
# ===============================================
# WS-BRIDGE CLIENT CONFIGURATION
# ===============================================

# Remote WebSocket URL (e.g. Cloudflare tunnel URL or direct domain)
ws_url = "wss://xxx-xxx-xxx.trycloudflare.com"

# Secret token matching the server config
token = "<your_token>"

# Optional: SOCKS5 Dynamic Proxy
# Enables transparent tunneling to ANY remote host/port through a single local port!
socks5 = "127.0.0.1:1080"

# --- Static Port Forwarding Rules ---

# 1. MySQL TCP Tunnel
[[tunnels]]
name = "mysql"
protocol = "tcp"
listen = "127.0.0.1:13306"
target = "127.0.0.1:3306"

# 2. Redis TCP Tunnel
[[tunnels]]
name = "redis"
protocol = "tcp"
listen = "127.0.0.1:16379"
target = "127.0.0.1:6379"

# 3. HTTP Web Server / phpMyAdmin
[[tunnels]]
name = "web_phpmyadmin"
protocol = "http"
listen = "127.0.0.1:18080"
target = "127.0.0.1:80"

# 4. SSH TCP Tunnel
[[tunnels]]
name = "ssh"
protocol = "tcp"
listen = "127.0.0.1:10022"
target = "127.0.0.1:22"

# 5. DNS UDP Tunnel
[[tunnels]]
name = "dns"
protocol = "udp"
listen = "127.0.0.1:10053"
target = "1.1.1.1:53"
```
</details>

---

## 🗂️ 项目结构

```text
ws_bridge/
├── wsb.txt                 # 预编译 Linux x86_64 二进制文件 (v0.2.0；重命名为 wsb 并 chmod +x)
├── config.server.toml      # 服务端配置示例
├── config.client.toml      # 客户端配置示例
├── README.md               # 说明文档（English）
├── README.vi.md            # 说明文档（Tiếng Việt）
└── README.zh-CN.md         # 说明文档（简体中文）
```

---

## 📝 许可证

MIT © [conghoa004](https://github.com/conghoa004)
