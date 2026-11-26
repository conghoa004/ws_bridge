<p align="center">
  <h1 align="center">🌉 WSB — WebSocket Bridge</h1>
  <p align="center">
    <strong>Công cụ tạo đường hầm đa giao thức qua WebSocket, hiệu năng cao, viết bằng Rust.</strong>
  </p>
  <p align="center">
    <img alt="Rust" src="https://img.shields.io/badge/Rust-1.70%2B-orange?logo=rust&logoColor=white">
    <img alt="Tokio" src="https://img.shields.io/badge/async-Tokio-blue?logo=rust">
    <img alt="License" src="https://img.shields.io/badge/license-MIT-green">
    <img alt="Version" src="https://img.shields.io/badge/version-0.2.0-blueviolet">
  </p>
  <p align="center">
    <a href="README.md">English</a> ｜ <a href="README.zh-CN.md">中文</a> ｜ <strong>Tiếng Việt</strong>
  </p>
</p>

---

**WSB** đóng gói và truyền tải toàn bộ luồng dữ liệu **TCP**, **UDP** và **HTTP** thông qua **WebSocket** (`ws://` / `wss://`), cho phép kết nối an toàn tới bất kỳ dịch vụ nội bộ nào trên máy chủ từ xa — như **Kaggle Notebooks**, Google Colab, Docker container, Linux VM — qua các dịch vụ web tunnel (ví dụ **Cloudflare Quick Tunnel**), **không cần mở cổng TCP thô hay IP tĩnh**.

## Mục lục

- [Tính năng](#-tính-năng)
- [Kiến trúc](#-kiến-trúc--luồng-dữ-liệu)
- [Cài đặt](#-cài-đặt)
- [Biên dịch](#-biên-dịch)
- [Khởi chạy nhanh](#-khởi-chạy-nhanh)
  - [Server](#1-server-kaggle--máy-chủ-từ-xa)
  - [Client](#2-client-máy-tính-cá-nhân)
- [Ví dụ sử dụng](#-ví-dụ-sử-dụng)
- [CLI Reference](#%EF%B8%8F-cli-reference)
- [Cấu hình TOML](#-cấu-hình-toml)
- [Cấu trúc dự án](#%EF%B8%8F-cấu-trúc-dự-án)
- [License](#-license)

---

## ✨ Tính năng

| Tính năng | Mô tả |
|---|---|
| 🦀 **Rust thuần & Tokio** | Nền tảng `tokio` + `tokio-tungstenite` + `rustls` — độ trễ cực thấp, thông lượng I/O cao, tiêu tốn ít RAM, không phụ thuộc thư viện C bên ngoài. |
| 🔌 **TCP Tunnel** | Kết nối tới mọi dịch vụ TCP: MySQL, PostgreSQL, Redis, SSH, MongoDB, Kafka, v.v. |
| 📡 **UDP Tunnel** | Truyền tải datagram (DNS, game server, Syslog) với session tracking tự động và idle timeout 60s. |
| 🌐 **HTTP Tunnel** | Truy cập ứng dụng web (phpMyAdmin, REST API, dashboard) với **auto-rewrite `Host` header** để Apache/Nginx nhận diện đúng virtual host. Hỗ trợ keep-alive, chunked transfer, Upgrade/WebSocket passthrough. |
| 🧦 **SOCKS5 Dynamic Proxy** | Proxy cục bộ SOCKS5 — trình duyệt, `curl`, database GUI, SSH truy cập **mọi cổng/dịch vụ** trên remote server qua một cổng duy nhất. Hỗ trợ IPv4, IPv6, domain name. |
| 🔒 **Bảo mật** | Xác thực bằng token bí mật (constant-time compare) ngay khi handshake. Token tự sinh 256-bit (crypto-secure) hoặc tự chỉ định. Danh sách trắng `allowed_targets` trên server. |
| ⚙️ **Cấu hình linh hoạt** | File TOML + CLI flags + biến môi trường — kết hợp tùy ý. |
| ☁️ **Cloudflare Tunnel** | Hoạt động mượt mà với `cloudflared tunnel --url http://127.0.0.1:8766`. |
| 🏗️ **Cross-platform TLS** | Windows dùng `native-tls`, Linux/macOS dùng `rustls` — build sạch trên mọi nền tảng. |
| 🫀 **WebSocket Keepalive** | Ping frame mỗi 25s giữ kết nối sống qua proxy/CDN. TCP half-close semantics được bảo toàn end-to-end. |
| 🔄 **IPv4-first fallback** | Client tự động ưu tiên IPv4, fallback IPv6, với timeout 10s mỗi địa chỉ. |

---

## 📐 Kiến trúc & Luồng dữ liệu

```text
  Máy cục bộ (Client)                              Máy chủ từ xa (Server)
  ====================                              ======================

  [ MySQL / DBeaver   ] ─ TCP :13306 ─┐
  [ Redis CLI         ] ─ TCP :16379 ─┤
  [ Trình duyệt web   ] ─ HTTP :18080 ┤
  [ DNS Client        ] ─ UDP :10053 ─┤
  [ SOCKS5 app        ] ─ SOCKS5 :1080┤
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

**Luồng hoạt động:**

1. **Client** lắng nghe trên các cổng local (TCP/UDP/HTTP/SOCKS5).
2. Khi có kết nối đến, client mở một WebSocket tới server, gửi **handshake JSON** chứa `token`, `protocol`, `target`.
3. **Server** xác thực token (constant-time), kiểm tra whitelist, kết nối tới target, rồi phản hồi thành công.
4. Dữ liệu được **bridge hai chiều** (bidirectional) qua WebSocket binary frames, kèm Ping keepalive mỗi 25s.

---

## 📦 Cài đặt

### Yêu cầu
- **Rust toolchain** ≥ 1.70 (edition 2021)

### Cài Rust (nếu chưa có)

<details>
<summary><strong>Linux / Kaggle / Google Colab / Ubuntu</strong></summary>

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```
</details>

<details>
<summary><strong>Windows</strong></summary>

Tải và chạy `rustup-init.exe` từ [https://rustup.rs/](https://rustup.rs/) (chọn tùy chọn mặc định `1`).
</details>

<details>
<summary><strong>macOS</strong></summary>

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
# hoặc: brew install rust
```
</details>

Kiểm tra:
```bash
cargo --version && rustc --version
```

---

## 🚀 Biên dịch

```bash
cargo build --release
```

| Nền tảng | File thực thi |
|---|---|
| Linux / macOS | `target/release/wsb` |
| Windows | `target\release\wsb.exe` |

> [!TIP]
> Copy vào PATH để gõ `wsb` từ bất kỳ đâu:
> ```bash
> cp target/release/wsb ~/.local/bin/   # hoặc sudo cp ... /usr/local/bin/
> ```

---

## 📖 Khởi chạy nhanh

### 1. Server (Kaggle / Máy chủ từ xa)

**Dùng file cấu hình:**
```bash
wsb server -c config.server.toml
```

**Dùng CLI flags:**
```bash
# Token tự sinh 256-bit (khuyên dùng):
wsb server --listen 0.0.0.0:8766

# Token cố định:
wsb server --listen 0.0.0.0:8766 --token "my-secret-token"

# Sinh token để lưu vào config:
wsb gen-token
```

**Public qua Cloudflare Tunnel:**
```bash
cloudflared tunnel --url http://127.0.0.1:8766 --no-autoupdate
# → https://xxx-xxx-xxx.trycloudflare.com
```

### 2. Client (Máy tính cá nhân)

**Dùng file cấu hình:**
```bash
wsb client -c config.client.toml
```

**Dùng CLI flags:**
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

## 🎯 Ví dụ sử dụng

### MySQL qua TCP tunnel

Mở DBeaver, Navicat hoặc MySQL Workbench:
- **Host:** `127.0.0.1` &nbsp;|&nbsp; **Port:** `13306`
- Nhập tài khoản MySQL trên remote server.

Hoặc test bằng script Python:
```bash
python test/test_mysql.py
```

### phpMyAdmin / Web qua HTTP tunnel

```
http://127.0.0.1:18080/phpmyadmin/
```

> [!NOTE]
> WSB tự động rewrite `Host` header thành target address (vd: `127.0.0.1:80`) để Apache nhận diện đúng virtual host, tránh lỗi redirect.

### DNS qua UDP tunnel

```bash
nslookup google.com 127.0.0.1 -port=10053
```

### SOCKS5 Dynamic Proxy

Chỉ cần **1 cổng** `127.0.0.1:1080` để truy cập mọi dịch vụ trên remote:

```bash
# Web qua curl
curl --socks5-hostname 127.0.0.1:1080 http://127.0.0.1:80/phpmyadmin/

# SSH qua proxy
ssh -o "ProxyCommand=nc -X 5 -x 127.0.0.1:1080 %h %p" root@127.0.0.1
```

**Trình duyệt:** Cấu hình SOCKS5 proxy `127.0.0.1:1080` (dùng SwitchyOmega hoặc Firefox network settings).

---

## ⚙️ CLI Reference

### `wsb server`

| Flag | Env | Default | Mô tả |
|---|---|---|---|
| `-c, --config <PATH>` | — | — | File cấu hình TOML |
| `-l, --listen <ADDR>` | `WS_BRIDGE_LISTEN` | `0.0.0.0:8766` | Địa chỉ lắng nghe WebSocket |
| `-t, --token <SECRET>` | `WS_BRIDGE_TOKEN` | `auto` (256-bit) | Token xác thực client |
| `--allow <TARGET>` | — | Tất cả | Whitelist target (lặp nhiều lần) |

### `wsb client`

| Flag | Env | Default | Mô tả |
|---|---|---|---|
| `-c, --config <PATH>` | — | — | File cấu hình TOML |
| `-w, --ws <URL>` | `WS_BRIDGE_URL` | — | URL WebSocket server (`wss://...`) |
| `-t, --token <SECRET>` | `WS_BRIDGE_TOKEN` | `12345` | Token xác thực |
| `--tcp <LISTEN=TARGET>` | — | — | TCP tunnel (vd: `127.0.0.1:13306=127.0.0.1:3306`) |
| `--udp <LISTEN=TARGET>` | — | — | UDP tunnel (vd: `127.0.0.1:10053=1.1.1.1:53`) |
| `--http <LISTEN=TARGET>` | — | — | HTTP tunnel với Host rewrite |
| `--socks5 <LISTEN>` | — | — | SOCKS5 dynamic proxy (vd: `127.0.0.1:1080`) |

### `wsb gen-token`

Sinh token ngẫu nhiên 256-bit (64 hex chars) cho server config.

---

## 📄 Cấu hình TOML

<details>
<summary><strong>config.server.toml</strong></summary>

```toml
listen = "0.0.0.0:8766"

# "auto" = tự sinh token 256-bit mỗi lần chạy
token = "auto"

# Whitelist target (bỏ comment để bật)
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

# SOCKS5 Dynamic Proxy (tùy chọn)
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

## 🗂️ Cấu trúc dự án

```
ws_bridge/
├── Cargo.toml              # Package manifest & dependencies
├── config.server.toml      # Cấu hình mẫu cho server
├── config.client.toml      # Cấu hình mẫu cho client
├── src/
│   ├── main.rs             # CLI entry point (clap)
│   ├── config.rs           # Đọc/parse TOML config & token generation
│   ├── protocol.rs         # Handshake request/response & TunnelProtocol enum
│   ├── bridge.rs           # Bidirectional TCP↔WS bridge với Ping keepalive & half-close
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
    └── test_mysql.py       # Script kiểm tra kết nối MySQL qua tunnel
```

---

## 📝 License

MIT © [conghoa004](https://github.com/conghoa004)
