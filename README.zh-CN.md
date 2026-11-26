<p align="center">
  <h1 align="center">🌉 WSB — WebSocket Bridge</h1>
  <p align="center">
    <strong>用 Rust 编写的高性能多协议 WebSocket 隧道工具。</strong>
  </p>
  <p align="center">
    <img alt="Rust" src="https://img.shields.io/badge/Rust-1.70%2B-orange?logo=rust&logoColor=white">
    <img alt="Tokio" src="https://img.shields.io/badge/async-Tokio-blue?logo=rust">
    <img alt="License" src="https://img.shields.io/badge/license-MIT-green">
    <img alt="Version" src="https://img.shields.io/badge/version-0.2.0-blueviolet">
  </p>
  <p align="center">
    <a href="README.md">English</a> ｜ <strong>中文</strong> ｜ <a href="README.vi.md">Tiếng Việt</a>
  </p>
</p>

---

**WSB** 将 **TCP**、**UDP** 和 **HTTP** 数据流封装并通过 **WebSocket**（`ws://` / `wss://`）传输，实现安全访问远程主机上的任意内部服务——如 **Kaggle Notebooks**、Google Colab、Docker 容器或 Linux 虚拟机——通过 Web 隧道服务（例如 **Cloudflare Quick Tunnel**），**无需开放 TCP 原始端口或固定 IP 地址**。

## 目录

- [功能特性](#-功能特性)
- [架构设计](#-架构设计与数据流)
- [安装](#-安装)
- [编译构建](#-编译构建)
- [快速开始](#-快速开始)
  - [服务端](#1-服务端kaggle--远程主机)
  - [客户端](#2-客户端本地机器)
- [使用示例](#-使用示例)
- [CLI 参考](#%EF%B8%8F-cli-参考)
- [TOML 配置](#-toml-配置)
- [项目结构](#%EF%B8%8F-项目结构)
- [许可证](#-许可证)

---

## ✨ 功能特性

| 特性 | 描述 |
|---|---|
| 🦀 **纯 Rust & Tokio** | 基于 `tokio` + `tokio-tungstenite` + `rustls` 构建——超低延迟、高 I/O 吞吐量、内存占用极少、零 C 依赖。 |
| 🔌 **TCP 隧道** | 连接任意 TCP 服务：MySQL、PostgreSQL、Redis、SSH、MongoDB、Kafka 等。 |
| 📡 **UDP 隧道** | 传输数据报（DNS、游戏服务器、Syslog），自动会话跟踪，60 秒空闲超时。 |
| 🌐 **HTTP 隧道** | 访问 Web 应用（phpMyAdmin、REST API、管理面板），**自动重写 `Host` 请求头**使 Apache/Nginx 正确识别虚拟主机。支持 keep-alive、chunked 传输、Upgrade/WebSocket 透传。 |
| 🧦 **SOCKS5 动态代理** | 内置本地 SOCKS5 代理——浏览器、`curl`、数据库管理工具、SSH 均可通过单一端口访问远程服务器上的**任意端口/服务**。支持 IPv4、IPv6 和域名。 |
| 🔒 **安全性** | 基于 Token 的身份认证，握手时采用恒定时间比较。自动生成 256 位加密安全随机 Token，或使用自定义 Token。服务端支持 `allowed_targets` 白名单。 |
| ⚙️ **灵活配置** | TOML 配置文件 + CLI 参数 + 环境变量——可自由组合使用。 |
| ☁️ **Cloudflare Tunnel 兼容** | 与 `cloudflared tunnel --url http://127.0.0.1:8766` 完美配合。 |
| 🏗️ **跨平台 TLS** | Windows 使用 `native-tls`；Linux/macOS 使用 `rustls`——所有平台均可干净编译。 |
| 🫀 **WebSocket 心跳保活** | 每 25 秒发送 Ping 帧，确保连接在代理/CDN 后保持活跃。TCP 半关闭语义端到端保留。 |
| 🔄 **IPv4 优先回退** | 客户端自动优先选择 IPv4，回退到 IPv6，每个地址超时 10 秒。 |

---

## 📐 架构设计与数据流

```text
  本地机器（客户端）                               远程主机（服务端）
  ================                               ================

  [ MySQL / DBeaver   ] ─ TCP :13306 ─┐
  [ Redis CLI         ] ─ TCP :16379 ─┤
  [ Web 浏览器        ] ─ HTTP :18080 ┤
  [ DNS 客户端        ] ─ UDP :10053 ─┤
  [ SOCKS5 应用       ] ─ SOCKS5 :1080┤
                                      │
                              ┌───────▼───────────────┐
                              │    wsb client (Rust)   │
                              └───────┬───────────────┘
                                      │  WS 二进制帧 (WSS)
                              ┌───────▼───────────────┐
                              │  Cloudflare Tunnel /   │
                              │  Web 反向代理          │
                              │  (https://*.trycloudflare.com)
                              └───────┬───────────────┘
                                      │
                              ┌───────▼───────────────┐
                              │    wsb server (Rust)   │  :8766
                              └───────┬───────────────┘
                                      │
                                      ├──► MySQL      :3306
                                      ├──► Redis      :6379
                                      ├──► Apache/Web :80
                                      ├──► SSH        :22
                                      └──► DNS        :53
```

**工作流程：**

1. **客户端**在本地端口上监听（TCP/UDP/HTTP/SOCKS5）。
2. 当连接到达时，客户端向服务端打开 WebSocket 连接，发送包含 `token`、`protocol` 和 `target` 的**握手 JSON**。
3. **服务端**验证 Token（恒定时间比较）、检查白名单、连接目标服务，然后返回成功响应。
4. 数据通过 WebSocket 二进制帧**双向桥接**传输，每 25 秒发送 Ping 心跳保活。

---

## 📦 安装

### 系统要求
- **Rust 工具链** ≥ 1.70（edition 2021）

### 安装 Rust

<details>
<summary><strong>Linux / Kaggle / Google Colab / Ubuntu</strong></summary>

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```
</details>

<details>
<summary><strong>Windows</strong></summary>

从 [https://rustup.rs/](https://rustup.rs/) 下载并运行 `rustup-init.exe`（选择默认选项 `1`）。
</details>

<details>
<summary><strong>macOS</strong></summary>

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
# 或者：brew install rust
```
</details>

验证安装：
```bash
cargo --version && rustc --version
```

---

## 🚀 编译构建

```bash
cargo build --release
```

| 平台 | 可执行文件 |
|---|---|
| Linux / macOS | `target/release/wsb` |
| Windows | `target\release\wsb.exe` |

> [!TIP]
> 复制到 PATH 目录以便全局使用：
> ```bash
> cp target/release/wsb ~/.local/bin/   # 或 sudo cp ... /usr/local/bin/
> ```

---

## 📖 快速开始

### 1. 服务端（Kaggle / 远程主机）

**使用配置文件：**
```bash
wsb server -c config.server.toml
```

**使用 CLI 参数：**
```bash
# 自动生成 256 位安全 Token（推荐）：
wsb server --listen 0.0.0.0:8766

# 使用自定义固定 Token：
wsb server --listen 0.0.0.0:8766 --token "my-secret-token"

# 生成 Token 保存到配置文件：
wsb gen-token
```

**通过 Cloudflare Tunnel 暴露到互联网：**
```bash
cloudflared tunnel --url http://127.0.0.1:8766 --no-autoupdate
# → https://xxx-xxx-xxx.trycloudflare.com
```

### 2. 客户端（本地机器）

**使用配置文件：**
```bash
wsb client -c config.client.toml
```

**使用 CLI 参数：**
```bash
wsb client \
  --ws "wss://xxx-xxx-xxx.trycloudflare.com" \
  --token "my-secret-token" \
  --tcp 127.0.0.1:13306=127.0.0.1:3306 \
  --tcp 127.0.0.1:16379=127.0.0.1:6379 \
  --http 127.0.0.1:18080=127.0.0.1:80 \
  --udp 127.0.0.1:10053=1.1.1.1:53 \
  --socks5 127.0.0.1:1080
```

---

## 🎯 使用示例

### 通过 TCP 隧道连接 MySQL

打开 DBeaver、Navicat 或 MySQL Workbench：
- **主机：** `127.0.0.1` &nbsp;|&nbsp; **端口：** `13306`
- 输入远程服务器上的 MySQL 账户凭据。

或使用自带的 Python 测试脚本：
```bash
python test/test_mysql.py
```

### 通过 HTTP 隧道访问 phpMyAdmin / Web

```
http://127.0.0.1:18080/phpmyadmin/
```

> [!NOTE]
> WSB 自动将 `Host` 请求头重写为目标地址（如 `127.0.0.1:80`），使 Apache/Nginx 正确解析虚拟主机，避免重定向错误。

### 通过 UDP 隧道进行 DNS 查询

```bash
nslookup google.com 127.0.0.1 -port=10053
```

### SOCKS5 动态代理

仅需**一个端口** `127.0.0.1:1080`，即可访问远程服务器上的任意服务：

```bash
# 通过 curl 访问 Web
curl --socks5-hostname 127.0.0.1:1080 http://127.0.0.1:80/phpmyadmin/

# 通过代理 SSH
ssh -o "ProxyCommand=nc -X 5 -x 127.0.0.1:1080 %h %p" root@127.0.0.1
```

**浏览器：** 将 SOCKS5 代理配置为 `127.0.0.1:1080`（Chrome 使用 SwitchyOmega 扩展，或 Firefox 网络设置）。

---

## ⚙️ CLI 参考

### `wsb server`

| 参数 | 环境变量 | 默认值 | 描述 |
|---|---|---|---|
| `-c, --config <PATH>` | — | — | TOML 配置文件路径 |
| `-l, --listen <ADDR>` | `WS_BRIDGE_LISTEN` | `0.0.0.0:8766` | WebSocket 监听地址 |
| `-t, --token <SECRET>` | `WS_BRIDGE_TOKEN` | `auto`（256 位） | 客户端认证 Token |
| `--allow <TARGET>` | — | 全部 | 目标白名单（可重复指定） |

### `wsb client`

| 参数 | 环境变量 | 默认值 | 描述 |
|---|---|---|---|
| `-c, --config <PATH>` | — | — | TOML 配置文件路径 |
| `-w, --ws <URL>` | `WS_BRIDGE_URL` | — | 远程 WebSocket URL（`wss://...`） |
| `-t, --token <SECRET>` | `WS_BRIDGE_TOKEN` | `12345` | 认证 Token |
| `--tcp <LISTEN=TARGET>` | — | — | TCP 隧道（例：`127.0.0.1:13306=127.0.0.1:3306`） |
| `--udp <LISTEN=TARGET>` | — | — | UDP 隧道（例：`127.0.0.1:10053=1.1.1.1:53`） |
| `--http <LISTEN=TARGET>` | — | — | HTTP 隧道（含 Host 重写） |
| `--socks5 <LISTEN>` | — | — | SOCKS5 动态代理（例：`127.0.0.1:1080`） |

### `wsb gen-token`

生成一个加密安全的 256 位随机 Token（64 位十六进制字符）。

---

## 📄 TOML 配置

<details>
<summary><strong>config.server.toml</strong></summary>

```toml
listen = "0.0.0.0:8766"

# "auto" = 每次启动时生成新的 256 位 Token
token = "auto"

# 目标白名单（取消注释以启用）
# allowed_targets = [
#     "127.0.0.1:3306",
#     "127.0.0.1:6379",
#     "127.0.0.1:80",
#     "127.0.0.1:22",
#     "1.1.1.1:53",
# ]
```
</details>

<details>
<summary><strong>config.client.toml</strong></summary>

```toml
ws_url = "wss://xxx-xxx-xxx.trycloudflare.com"
token = "<your_token>"

# SOCKS5 动态代理（可选）
socks5 = "127.0.0.1:1080"

# --- 隧道规则 ---

[[tunnels]]
name = "mysql"
protocol = "tcp"
listen = "127.0.0.1:13306"
target = "127.0.0.1:3306"

[[tunnels]]
name = "redis"
protocol = "tcp"
listen = "127.0.0.1:16379"
target = "127.0.0.1:6379"

[[tunnels]]
name = "web_phpmyadmin"
protocol = "http"
listen = "127.0.0.1:18080"
target = "127.0.0.1:80"

[[tunnels]]
name = "ssh"
protocol = "tcp"
listen = "127.0.0.1:10022"
target = "127.0.0.1:22"

[[tunnels]]
name = "dns"
protocol = "udp"
listen = "127.0.0.1:10053"
target = "1.1.1.1:53"
```
</details>

---

## 🗂️ 项目结构

```
ws_bridge/
├── Cargo.toml              # 包清单和依赖项
├── config.server.toml      # 服务端配置示例
├── config.client.toml      # 客户端配置示例
├── src/
│   ├── main.rs             # CLI 入口点 (clap)
│   ├── config.rs           # TOML 配置解析和 Token 生成
│   ├── protocol.rs         # 握手请求/响应和 TunnelProtocol 枚举
│   ├── bridge.rs           # TCP↔WS 双向桥接（Ping 保活 & 半关闭）
│   ├── server/
│   │   ├── mod.rs          # WS 服务端监听、认证、分发
│   │   ├── tcp.rs          # 服务端 TCP 目标连接器
│   │   ├── udp.rs          # 服务端 UDP 目标连接器
│   │   └── http.rs         # 服务端 HTTP 目标连接器
│   └── client/
│       ├── mod.rs          # WS 客户端连接器、握手、隧道生成器
│       ├── tcp.rs          # 客户端 TCP 监听 → WS 隧道
│       ├── udp.rs          # 客户端 UDP 监听 → WS 隧道（会话跟踪）
│       ├── http.rs         # 客户端 HTTP 监听 → WS 隧道（Host 重写）
│       └── socks5.rs       # 客户端 SOCKS5 代理 → WS 隧道
└── test/
    └── test_mysql.py       # MySQL 连接测试脚本
```

---

## 📝 许可证

MIT © [conghoa004](https://github.com/conghoa004)
