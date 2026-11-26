<p align="center">
  <h1 align="center">🌉 WSB — WebSocket Bridge</h1>
  <p align="center">
    <strong>High-performance multi-protocol tunnel bridge over WebSocket, built in Rust.</strong>
  </p>
  <p align="center">
    <img alt="Platform" src="https://img.shields.io/badge/Platform-Linux%20x86__64-FCC624?logo=linux&logoColor=black">
    <img alt="Tokio" src="https://img.shields.io/badge/async-Tokio-blue?logo=rust">
    <img alt="License" src="https://img.shields.io/badge/license-MIT-green">
    <img alt="Version" src="https://img.shields.io/badge/version-0.2.0-blueviolet">
  </p>
  <p align="center">
    <strong>English</strong> ｜ <a href="README.zh-CN.md">中文</a> ｜ <a href="README.vi.md">Tiếng Việt</a>
  </p>
</p>

---

**WSB (`wsb` v0.2.0)** encapsulates and forwards **TCP**, **UDP**, **HTTP**, and **SOCKS5** traffic over **WebSocket** (`ws://` / `wss://`), allowing secure access to internal services on a remote host (such as **Kaggle Notebooks**, Google Colab, Docker containers, or Linux VPS/VMs) behind web tunnels like **Cloudflare Quick Tunnel** — **without opening raw TCP/UDP ports or needing a static IP**.

This package includes the pre-built **Linux x86_64** executable [`wsb.txt`](wsb.txt) (stored with a `.txt` extension for convenient transfer/download) along with sample server and client TOML configurations.

## Table of Contents

- [Features](#-features)
- [Architecture & Data Flow](#-architecture--data-flow)
- [Getting Started](#-getting-started)
- [Quick Start](#-quick-start)
  - [1. Server (Kaggle / Remote Linux Host)](#1-server-kaggle--remote-linux-host)
  - [2. Client (Local Linux Machine)](#2-client-local-linux-machine)
- [Usage Examples](#-usage-examples)
- [CLI Reference](#%EF%B8%8F-cli-reference)
- [TOML Configuration](#-toml-configuration)
- [Project Structure](#%EF%B8%8F-project-structure)
- [License](#-license)

---

## ✨ Features

| Feature | Description |
|---|---|
| ⚡ **Pre-Built Linux Binary** | Ships as a standalone Linux `x86_64` ELF executable (`wsb.txt` → `wsb` v0.2.0) powered by Rust + Tokio (`tokio-tungstenite`) — low latency, minimal memory usage, no Rust toolchain required. |
| 🔌 **TCP Tunnel** | Forward any TCP service: MySQL, PostgreSQL, Redis, SSH, MongoDB, Kafka, etc. |
| 📡 **UDP Tunnel** | Forward UDP datagrams (up to 64 KB per packet, e.g., DNS, Syslog, game servers) with automatic per-peer session tracking (100-packet queue) and a 60s idle timeout. |
| 🌐 **HTTP Tunnel** | Access remote web apps (phpMyAdmin, REST APIs, admin dashboards) with **automatic `Host` header rewriting** to the target address, supporting HTTP/1.1 Keep-Alive, `Content-Length`, `Transfer-Encoding: chunked`, and `Upgrade`/WebSocket passthrough. |
| 🧦 **SOCKS5 Dynamic Proxy** | Built-in SOCKS5 proxy (`CONNECT` with no-auth) supporting IPv4, IPv6, and domain names — route browsers, `curl`, database GUIs, and SSH to **any remote port/host** through one local port. |
| 🔒 **Security** | Constant-time token verification during handshake, auto-generated 256-bit (64-hex) tokens or custom static tokens, plus an optional server-side `allowed_targets` whitelist. |
| ⚙️ **Flexible Configuration** | Combine TOML config files (`config.server.toml`, `config.client.toml`), CLI flags, and environment variables (`WS_BRIDGE_LISTEN`, `WS_BRIDGE_URL`, `WS_BRIDGE_TOKEN`, `RUST_LOG`). |
| ☁️ **Cloudflare Tunnel Ready** | Works seamlessly with `cloudflared tunnel --url http://127.0.0.1:8766`. |
| 🔐 **TLS & Network Resilience** | Uses `rustls` (WebPKI roots) on Linux/macOS with no OpenSSL runtime dependency. Client resolves DNS and attempts **IPv4 addresses before IPv6** (10s timeout per address) with `TCP_NODELAY` enabled. |
| 🫀 **Keepalive & Half-Close** | Sends WebSocket `Ping` frames every 25s to keep idle tunnels alive behind proxies/CDNs, and preserves TCP half-close semantics via `wsb:eof` control messages. |

---

## 📐 Architecture & Data Flow

```text
  Local Machine (Client)                           Remote Linux Host (Server)
  ======================                           ==========================

  [ MySQL / DBeaver   ] ─ TCP :13306 ─┐
  [ Redis CLI         ] ─ TCP :16379 ─┤
  [ Web Browser       ] ─ HTTP :18080 ┤
  [ DNS Client        ] ─ UDP :10053 ─┤
  [ SOCKS5 App        ] ─ SOCKS5 :1080┤
                                      │
                              ┌───────▼───────────────┐
                              │    wsb client         │
                              └───────┬───────────────┘
                                      │  WS Binary Frames (ws:// or wss://)
                              ┌───────▼───────────────┐
                              │  Cloudflare Tunnel /   │
                              │  Reverse Proxy (opt.)  │
                              │  (https://*.trycloudflare.com)
                              └───────┬───────────────┘
                                      │
                              ┌───────▼───────────────┐
                              │    wsb server         │  :8766
                              └───────┬───────────────┘
                                      │
                                      ├──► MySQL      :3306
                                      ├──► Redis      :6379
                                      ├──► Apache/Web :80
                                      ├──► SSH        :22
                                      └──► DNS        :53
```

**How it works:**

1. The **client** (`wsb client`) binds local listeners for configured TCP, UDP, HTTP, and/or SOCKS5 rules.
2. When traffic arrives, the client opens a WebSocket connection (`ws://` or `wss://`) using IPv4-first fallback and sends a JSON **handshake** containing `token`, `protocol` (`tcp`, `udp`, or `http`; SOCKS5 uses `tcp`), and `target`.
3. The **server** (`wsb server`) validates the token (constant-time), checks `allowed_targets` (if configured), **connects to the target service first** (10s timeout), and returns a JSON `HandshakeResponse`.
4. Payload bytes are bridged bidirectionally using WebSocket binary frames, with `Ping` frames every 25s and TCP half-close (`EOF`) propagation.

---

## 📦 Getting Started

No compiler or Rust toolchain is needed on Linux `x86_64`. Rename [`wsb.txt`](wsb.txt) to `wsb` and make it executable:

```bash
mv wsb.txt wsb
chmod +x wsb
```

*(Or use `cp wsb.txt wsb && chmod +x wsb` if you want to keep `wsb.txt` in the repository directory.)*

Verify the binary:

```bash
./wsb --version
# Output: wsb 0.2.0
```

> [!TIP]
> Install `wsb` to your `PATH` for global access:
> ```bash
> cp wsb ~/.local/bin/   # or: sudo cp wsb /usr/local/bin/
> ```

> [!NOTE]
> The Linux `wsb` binary supports both `server` and `client` modes. If your local machine runs Windows, use the Windows `wsb.exe` release on Windows as the client while running `./wsb server` on your remote Linux host (e.g., Kaggle, Google Colab, Ubuntu VPS).

---

## 📖 Quick Start

### 1. Server (Kaggle / Remote Linux Host)

**Using the sample config file ([`config.server.toml`](config.server.toml)):**
```bash
./wsb server -c config.server.toml
```

**Using CLI flags:**
```bash
# Auto-generate a 256-bit secure token on startup (recommended):
./wsb server --listen 0.0.0.0:8766

# Or specify a static token and target whitelist:
./wsb server --listen 0.0.0.0:8766 --token "my-secret-token" --allow 127.0.0.1:3306 --allow 127.0.0.1:80

# Generate a 256-bit (64-hex) token to paste into config files:
./wsb gen-token
```

**Optional — Expose the server via Cloudflare Quick Tunnel:**
```bash
cloudflared tunnel --url http://127.0.0.1:8766 --no-autoupdate
# → https://xxx-xxx-xxx.trycloudflare.com
```
*(Use `wss://xxx-xxx-xxx.trycloudflare.com` as `ws_url` on the client, or `ws://<server-ip>:8766` for direct connections.)*

### 2. Client (Local Linux Machine)

**Using the sample config file ([`config.client.toml`](config.client.toml)):**

Edit `ws_url` and `token` in `config.client.toml`, then run:
```bash
./wsb client -c config.client.toml
```

**Using CLI flags:**
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
> You can combine `-c config.client.toml` with `--ws`, `--token`, `--socks5`, or extra `--tcp` / `--udp` / `--http` flags.

---

## 🎯 Usage Examples

### MySQL via TCP Tunnel

Connect with DBeaver, Navicat, MySQL Workbench, or the `mysql` CLI:
```bash
mysql -h 127.0.0.1 -P 13306 -u <your_user> -p
```
- **Host:** `127.0.0.1` &nbsp;|&nbsp; **Port:** `13306`

### phpMyAdmin / Web via HTTP Tunnel

Open in your browser:
```text
http://127.0.0.1:18080/phpmyadmin/
```

> [!NOTE]
> `wsb` automatically rewrites the `Host` request header to the remote `target` (e.g., `127.0.0.1:80`) so Apache/Nginx matches the expected virtual host and avoids redirect loops.

### DNS via UDP Tunnel

```bash
dig @127.0.0.1 -p 10053 google.com
# or: nslookup -port=10053 google.com 127.0.0.1
```

### SOCKS5 Dynamic Proxy

With a single local port (`127.0.0.1:1080`), forward TCP connections to any remote host and port dynamically:

```bash
# HTTP request through SOCKS5 (remote DNS resolution)
curl --socks5-hostname 127.0.0.1:1080 http://127.0.0.1:80/phpmyadmin/

# SSH through SOCKS5
ssh -o "ProxyCommand=nc -X 5 -x 127.0.0.1:1080 %h %p" root@127.0.0.1
```

**Browser:** Configure your SOCKS5 proxy as `127.0.0.1:1080` (e.g., SwitchyOmega or Firefox proxy settings).  
*(Limitation: the built-in SOCKS5 server supports `CONNECT` with no-authentication only; `BIND` and `UDP ASSOCIATE` are not supported.)*

---

## ⚙️ CLI Reference

### `wsb server`

| Flag | Env | Default | Description |
|---|---|---|---|
| `-c, --config <CONFIG>` | — | — | Path to server TOML configuration file |
| `-l, --listen <LISTEN>` | `WS_BRIDGE_LISTEN` | `0.0.0.0:8766` | WebSocket listen address |
| `-t, --token <TOKEN>` | `WS_BRIDGE_TOKEN` | `auto` | Shared auth token (`auto` / `generate` / empty generates a 256-bit token on startup) |
| `--allow <ALLOWED>` | — | All targets | Allowed `host:port` target whitelist (repeatable; overrides `allowed_targets` in config) |

### `wsb client`

| Flag | Env | Default | Description |
|---|---|---|---|
| `-c, --config <CONFIG>` | — | — | Path to client TOML configuration file |
| `-w, --ws <WS_URL>` | `WS_BRIDGE_URL` | *Required* | Remote WebSocket server URL (`wss://...` or `ws://...`) |
| `-t, --token <TOKEN>` | `WS_BRIDGE_TOKEN` | `12345` | Shared secret token matching the server |
| `--tcp <LISTEN=TARGET>` | — | — | Add a TCP tunnel rule (repeatable, e.g., `127.0.0.1:13306=127.0.0.1:3306`) |
| `--udp <LISTEN=TARGET>` | — | — | Add a UDP tunnel rule (repeatable, e.g., `127.0.0.1:10053=1.1.1.1:53`) |
| `--http <LISTEN=TARGET>` | — | — | Add an HTTP tunnel rule with `Host` rewrite (repeatable) |
| `--socks5 <LISTEN_ADDR>` | — | — | Enable SOCKS5 dynamic proxy (e.g., `127.0.0.1:1080`; overrides config `socks5`) |

> [!TIP]
> Run with `RUST_LOG=debug ./wsb ...` to inspect per-connection tunnel and handshake logs. Default filter is `wsb=info,info`.

### `wsb gen-token`

Prints a cryptographically secure 256-bit random token (64 lowercase hexadecimal characters) to stdout.

---

## 📄 TOML Configuration

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

## 🗂️ Project Structure

```text
ws_bridge/
├── wsb.txt                 # Pre-built Linux x86_64 binary (v0.2.0; rename to wsb & chmod +x)
├── config.server.toml      # Sample server configuration
├── config.client.toml      # Sample client configuration
├── README.md               # Documentation (English)
├── README.vi.md            # Documentation (Tiếng Việt)
└── README.zh-CN.md         # Documentation (简体中文)
```

---

## 📝 License

MIT © [conghoa004](https://github.com/conghoa004)
