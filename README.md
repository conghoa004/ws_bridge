<p align="center">
  <h1 align="center">🌉 WSB — WebSocket Bridge</h1>
  <p align="center">
    <strong>High-performance multi-protocol tunnel over WebSocket, written in Rust.</strong>
  </p>
  <p align="center">
    <img alt="Rust" src="https://img.shields.io/badge/Rust-1.70%2B-orange?logo=rust&logoColor=white">
    <img alt="Tokio" src="https://img.shields.io/badge/async-Tokio-blue?logo=rust">
    <img alt="License" src="https://img.shields.io/badge/license-MIT-green">
    <img alt="Version" src="https://img.shields.io/badge/version-0.2.0-blueviolet">
  </p>
  <p align="center">
    <strong>English</strong> ｜ <a href="README.zh-CN.md">中文</a> ｜ <a href="README.vi.md">Tiếng Việt</a>
  </p>
</p>

---

**WSB** encapsulates and transmits **TCP**, **UDP**, and **HTTP** data streams through **WebSocket** (`ws://` / `wss://`), enabling secure access to any internal service on a remote host — such as **Kaggle Notebooks**, Google Colab, Docker containers, or Linux VMs — via web tunnel services (e.g., **Cloudflare Quick Tunnel**), **without opening raw TCP ports or requiring a static IP**.

## Table of Contents

- [Features](#-features)
- [Architecture](#-architecture--data-flow)
- [Installation](#-installation)
- [Build](#-build)
- [Quick Start](#-quick-start)
  - [Server](#1-server-kaggle--remote-host)
  - [Client](#2-client-local-machine)
- [Usage Examples](#-usage-examples)
- [CLI Reference](#%EF%B8%8F-cli-reference)
- [TOML Configuration](#-toml-configuration)
- [Project Structure](#%EF%B8%8F-project-structure)
- [License](#-license)

---

## ✨ Features

| Feature | Description |
|---|---|
| 🦀 **Pure Rust & Tokio** | Built on `tokio` + `tokio-tungstenite` + `rustls` — ultra-low latency, high I/O throughput, minimal memory footprint, zero C dependencies. |
| 🔌 **TCP Tunnel** | Connect to any TCP service: MySQL, PostgreSQL, Redis, SSH, MongoDB, Kafka, etc. |
| 📡 **UDP Tunnel** | Transport datagrams (DNS, game servers, Syslog) with automatic session tracking and 60s idle timeout. |
| 🌐 **HTTP Tunnel** | Access web applications (phpMyAdmin, REST APIs, admin dashboards) with **automatic `Host` header rewriting** so Apache/Nginx correctly identifies the virtual host. Supports keep-alive, chunked transfer, and Upgrade/WebSocket passthrough. |
| 🧦 **SOCKS5 Dynamic Proxy** | Built-in local SOCKS5 proxy — browsers, `curl`, database GUIs, and SSH can access **any port/service** on the remote server through a single port. Supports IPv4, IPv6, and domain names. |
| 🔒 **Security** | Token-based authentication with constant-time comparison at handshake. Auto-generated 256-bit cryptographically secure tokens, or bring your own. Server-side `allowed_targets` whitelist. |
| ⚙️ **Flexible Configuration** | TOML config files + CLI flags + environment variables — mix and match freely. |
| ☁️ **Cloudflare Tunnel Ready** | Works seamlessly with `cloudflared tunnel --url http://127.0.0.1:8766`. |
| 🏗️ **Cross-Platform TLS** | Windows uses `native-tls`; Linux/macOS uses `rustls` — clean builds on all platforms. |
| 🫀 **WebSocket Keepalive** | Ping frames every 25s to keep connections alive through proxies/CDNs. TCP half-close semantics are preserved end-to-end. |
| 🔄 **IPv4-First Fallback** | Client automatically prefers IPv4, falls back to IPv6, with a 10s timeout per address. |

---

## 📐 Architecture & Data Flow

```text
  Local Machine (Client)                           Remote Host (Server)
  ======================                           ====================

  [ MySQL / DBeaver   ] ─ TCP :13306 ─┐
  [ Redis CLI         ] ─ TCP :16379 ─┤
  [ Web Browser       ] ─ HTTP :18080 ┤
  [ DNS Client        ] ─ UDP :10053 ─┤
  [ SOCKS5 App        ] ─ SOCKS5 :1080┤
                                      │
                              ┌───────▼───────────────┐
                              │    wsb client (Rust)   │
                              └───────┬───────────────┘
                                      │  WS Binary Frames (WSS)
                              ┌───────▼───────────────┐
                              │  Cloudflare Tunnel /   │
                              │  Web Reverse Proxy     │
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

**How it works:**

1. The **client** listens on local ports (TCP/UDP/HTTP/SOCKS5).
2. When a connection arrives, the client opens a WebSocket to the server and sends a **handshake JSON** containing `token`, `protocol`, and `target`.
3. The **server** authenticates the token (constant-time), checks the whitelist, connects to the target, and responds with success.
4. Data is **bridged bidirectionally** via WebSocket binary frames, with Ping keepalive every 25s.

---

## 📦 Installation

### Requirements
- **Rust toolchain** ≥ 1.70 (edition 2021)

### Install Rust

<details>
<summary><strong>Linux / Kaggle / Google Colab / Ubuntu</strong></summary>

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```
</details>

<details>
<summary><strong>Windows</strong></summary>

Download and run `rustup-init.exe` from [https://rustup.rs/](https://rustup.rs/) (select default option `1`).
</details>

<details>
<summary><strong>macOS</strong></summary>

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
# or: brew install rust
```
</details>

Verify:
```bash
cargo --version && rustc --version
```

---

## 🚀 Build

```bash
cargo build --release
```

| Platform | Binary |
|---|---|
| Linux / macOS | `target/release/wsb` |
| Windows | `target\release\wsb.exe` |

> [!TIP]
> Copy to your PATH for global access:
> ```bash
> cp target/release/wsb ~/.local/bin/   # or sudo cp ... /usr/local/bin/
> ```

---

## 📖 Quick Start

### 1. Server (Kaggle / Remote Host)

**Using config file:**
```bash
wsb server -c config.server.toml
```

**Using CLI flags:**
```bash
# Auto-generate a 256-bit secure token (recommended):
wsb server --listen 0.0.0.0:8766

# With a custom static token:
wsb server --listen 0.0.0.0:8766 --token "my-secret-token"

# Generate a token for your config file:
wsb gen-token
```

**Expose to the internet via Cloudflare Tunnel:**
```bash
cloudflared tunnel --url http://127.0.0.1:8766 --no-autoupdate
# → https://xxx-xxx-xxx.trycloudflare.com
```

### 2. Client (Local Machine)

**Using config file:**
```bash
wsb client -c config.client.toml
```

**Using CLI flags:**
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

## 🎯 Usage Examples

### MySQL via TCP Tunnel

Open DBeaver, Navicat, or MySQL Workbench:
- **Host:** `127.0.0.1` &nbsp;|&nbsp; **Port:** `13306`
- Enter the MySQL credentials from the remote server.

Or test with the included Python script:
```bash
python test/test_mysql.py
```

### phpMyAdmin / Web via HTTP Tunnel

```
http://127.0.0.1:18080/phpmyadmin/
```

> [!NOTE]
> WSB automatically rewrites the `Host` header to the target address (e.g., `127.0.0.1:80`) so Apache/Nginx correctly resolves the virtual host, preventing redirect errors.

### DNS via UDP Tunnel

```bash
nslookup google.com 127.0.0.1 -port=10053
```

### SOCKS5 Dynamic Proxy

With just **one port** `127.0.0.1:1080`, access any service on the remote server:

```bash
# Web via curl
curl --socks5-hostname 127.0.0.1:1080 http://127.0.0.1:80/phpmyadmin/

# SSH via proxy
ssh -o "ProxyCommand=nc -X 5 -x 127.0.0.1:1080 %h %p" root@127.0.0.1
```

**Browser:** Configure SOCKS5 proxy to `127.0.0.1:1080` (use SwitchyOmega for Chrome or Firefox network settings).

---

## ⚙️ CLI Reference

### `wsb server`

| Flag | Env | Default | Description |
|---|---|---|---|
| `-c, --config <PATH>` | — | — | TOML configuration file |
| `-l, --listen <ADDR>` | `WS_BRIDGE_LISTEN` | `0.0.0.0:8766` | WebSocket listen address |
| `-t, --token <SECRET>` | `WS_BRIDGE_TOKEN` | `auto` (256-bit) | Client authentication token |
| `--allow <TARGET>` | — | All | Target whitelist (repeatable) |

### `wsb client`

| Flag | Env | Default | Description |
|---|---|---|---|
| `-c, --config <PATH>` | — | — | TOML configuration file |
| `-w, --ws <URL>` | `WS_BRIDGE_URL` | — | Remote WebSocket URL (`wss://...`) |
| `-t, --token <SECRET>` | `WS_BRIDGE_TOKEN` | `12345` | Authentication token |
| `--tcp <LISTEN=TARGET>` | — | — | TCP tunnel (e.g., `127.0.0.1:13306=127.0.0.1:3306`) |
| `--udp <LISTEN=TARGET>` | — | — | UDP tunnel (e.g., `127.0.0.1:10053=1.1.1.1:53`) |
| `--http <LISTEN=TARGET>` | — | — | HTTP tunnel with Host rewrite |
| `--socks5 <LISTEN>` | — | — | SOCKS5 dynamic proxy (e.g., `127.0.0.1:1080`) |

### `wsb gen-token`

Generate a cryptographically secure 256-bit random token (64 hex characters).

---

## 📄 TOML Configuration

<details>
<summary><strong>config.server.toml</strong></summary>

```toml
listen = "0.0.0.0:8766"

# "auto" = generate a new 256-bit token on each startup
token = "auto"

# Target whitelist (uncomment to enable)
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

# SOCKS5 Dynamic Proxy (optional)
socks5 = "127.0.0.1:1080"

# --- Tunnel rules ---

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

## 🗂️ Project Structure

```
ws_bridge/
├── Cargo.toml              # Package manifest & dependencies
├── config.server.toml      # Sample server configuration
├── config.client.toml      # Sample client configuration
├── src/
│   ├── main.rs             # CLI entry point (clap)
│   ├── config.rs           # TOML config parser & token generation
│   ├── protocol.rs         # Handshake request/response & TunnelProtocol enum
│   ├── bridge.rs           # Bidirectional TCP↔WS bridge with Ping keepalive & half-close
│   ├── server/
│   │   ├── mod.rs          # WS server listener, auth, dispatch
│   │   ├── tcp.rs          # Server-side TCP target connector
│   │   ├── udp.rs          # Server-side UDP target connector
│   │   └── http.rs         # Server-side HTTP target connector
│   └── client/
│       ├── mod.rs          # WS client connector, handshake, tunnel spawner
│       ├── tcp.rs          # Client TCP listener → WS tunnel
│       ├── udp.rs          # Client UDP listener → WS tunnel (session tracking)
│       ├── http.rs         # Client HTTP listener → WS tunnel (Host rewrite)
│       └── socks5.rs       # Client SOCKS5 proxy → WS tunnel
└── test/
    └── test_mysql.py       # MySQL connection test script
```

---

## 📝 License

MIT © [conghoa004](https://github.com/conghoa004)
