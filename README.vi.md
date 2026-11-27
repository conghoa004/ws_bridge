<p align="center">
  <h1 align="center">🌉 WSB — WebSocket Bridge</h1>
  <p align="center">
    <strong>Công cụ tạo đường hầm đa giao thức qua WebSocket, hiệu năng cao, viết bằng Rust.</strong>
  </p>
  <p align="center">
    <img alt="Platform" src="https://img.shields.io/badge/Platform-Linux%20x86__64-FCC624?logo=linux&logoColor=black">
    <img alt="Tokio" src="https://img.shields.io/badge/async-Tokio-blue?logo=rust">
    <img alt="License" src="https://img.shields.io/badge/license-MIT-green">
    <img alt="Version" src="https://img.shields.io/badge/version-0.2.0-blueviolet">
  </p>
  <p align="center">
    <a href="README.md">English</a> ｜ <a href="README.zh-CN.md">中文</a> ｜ <strong>Tiếng Việt</strong>
  </p>
</p>

---

**WSB (`wsb` v0.2.0)** đóng gói và truyền tải toàn bộ lưu lượng **TCP**, **UDP**, **HTTP** và **SOCKS5** thông qua **WebSocket** (`ws://` / `wss://`), giúp truy cập an toàn tới bất kỳ dịch vụ nội bộ nào trên máy chủ từ xa (như **Kaggle Notebooks**, Google Colab, Docker container hoặc Linux VPS/VM) qua các web tunnel như **Cloudflare Quick Tunnel** — **không cần mở cổng TCP/UDP thô hay có IP tĩnh**.

Thư mục dự án hiện cung cấp sẵn bản dựng **Linux x86_64** [`wsb`](wsb) kèm theo các file cấu hình mẫu TOML cho server và client.

## Mục lục

- [Tính năng](#-tính-năng)
- [Kiến trúc & Luồng dữ liệu](#-kiến-trúc--luồng-dữ-liệu)
- [Chuẩn bị](#-chuẩn-bị)
- [Khởi chạy nhanh](#-khởi-chạy-nhanh)
  - [1. Server (Kaggle / Máy chủ Linux từ xa)](#1-server-kaggle--máy-chủ-linux-từ-xa)
  - [2. Client (Máy Linux cục bộ)](#2-client-máy-linux-cục-bộ)
- [Ví dụ sử dụng](#-ví-dụ-sử-dụng)
- [CLI Reference](#%EF%B8%8F-cli-reference)
- [Cấu hình TOML](#-cấu-hình-toml)
- [Cấu trúc dự án](#%EF%B8%8F-cấu-trúc-dự-án)
- [License](#-license)

---

## ✨ Tính năng

| Tính năng | Mô tả |
|---|---|
| ⚡ **Bản dựng Linux dựng sẵn** | Cung cấp sẵn file thực thi độc lập cho Linux `x86_64` (`wsb` v0.2.0) viết bằng Rust + Tokio (`tokio-tungstenite`) — độ trễ thấp, ít RAM, không cần cài Rust toolchain. |
| 🔌 **TCP Tunnel** | Chuyển tiếp mọi dịch vụ TCP: MySQL, PostgreSQL, Redis, SSH, MongoDB, Kafka, v.v. |
| 📡 **UDP Tunnel** | Truyền tải datagram UDP (tối đa 64 KB/gói, vd: DNS, Syslog, game server) với session tracking theo từng peer (hàng đợi 100 gói) và idle timeout 60 giây. |
| 🌐 **HTTP Tunnel** | Truy cập ứng dụng web (phpMyAdmin, REST API, dashboard) với **tự động viết lại `Host` header** sang địa chỉ đích, hỗ trợ HTTP/1.1 Keep-Alive, `Content-Length`, `Transfer-Encoding: chunked` và passthrough cho `Upgrade`/WebSocket. |
| 🧦 **SOCKS5 Dynamic Proxy** | Proxy cục bộ SOCKS5 (`CONNECT`, không xác thực) hỗ trợ IPv4, IPv6 và tên miền — cho phép trình duyệt, `curl`, database GUI và SSH truy cập **mọi cổng/dịch vụ** trên máy chủ từ xa qua một cổng duy nhất. |
| 🔒 **Bảo mật** | So khớp token theo thời gian hằng (constant-time) khi handshake, tự sinh token ngẫu nhiên 256-bit (64 ký tự hex) hoặc dùng token tĩnh, kèm whitelist `allowed_targets` phía server. |
| ⚙️ **Cấu hình linh hoạt** | Kết hợp file TOML (`config.server.toml`, `config.client.toml`), cờ CLI và biến môi trường (`WS_BRIDGE_LISTEN`, `WS_BRIDGE_URL`, `WS_BRIDGE_TOKEN`, `RUST_LOG`). |
| ☁️ **Tương thích Cloudflare Tunnel** | Hoạt động mượt mà với `cloudflared tunnel --url http://127.0.0.1:8766`. |
| 🔐 **TLS & Ưu tiên IPv4** | Bản Linux dùng `rustls` (WebPKI roots), không phụ thuộc OpenSSL runtime bên ngoài. Client phân giải DNS và thử **IPv4 trước IPv6** (timeout 10 giây mỗi địa chỉ), bật `TCP_NODELAY`. |
| 🫀 **Keepalive & Half-Close** | Gửi WebSocket `Ping` mỗi 25 giây để giữ kết nối sống qua proxy/CDN, đồng thời bảo toàn ngữ nghĩa TCP half-close qua thông điệp điều khiển `wsb:eof`. |

---

## 📐 Kiến trúc & Luồng dữ liệu

```text
  Máy cục bộ (Client)                              Máy chủ Linux từ xa (Server)
  ===================                              ============================

  [ MySQL / DBeaver   ] ─ TCP :13306 ─┐
  [ Redis CLI         ] ─ TCP :16379 ─┤
  [ Trình duyệt web   ] ─ HTTP :18080 ┤
  [ DNS Client        ] ─ UDP :10053 ─┤
  [ Ứng dụng SOCKS5   ] ─ SOCKS5 :1080┤
                                      │
                              ┌───────▼───────────────┐
                              │     wsb client        │
                              └───────┬───────────────┘
                                      │  WS Binary Frames (ws:// hoặc wss://)
                              ┌───────▼───────────────┐
                              │  Cloudflare Tunnel /   │
                              │  Reverse Proxy (tùy chọn)
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

**Luồng hoạt động:**

1. **Client** (`wsb client`) lắng nghe trên các cổng cục bộ theo cấu hình TCP, UDP, HTTP và/hoặc SOCKS5.
2. Khi có kết nối đến, client mở một kết nối WebSocket (`ws://` hoặc `wss://`) theo cơ chế ưu tiên IPv4 và gửi **handshake JSON** gồm `token`, `protocol` (`tcp`, `udp`, hoặc `http`; SOCKS5 dùng `tcp`) và `target`.
3. **Server** (`wsb server`) xác thực token (constant-time), kiểm tra `allowed_targets` (nếu có), **kết nối tới dịch vụ đích trước** (timeout 10 giây) rồi mới trả về `HandshakeResponse` JSON.
4. Dữ liệu được truyền hai chiều qua các binary frame của WebSocket, kèm `Ping` mỗi 25 giây và đồng bộ trạng thái đóng một chiều (`EOF`) cho TCP.

---

## 📦 Chuẩn bị

Trên Linux `x86_64` không cần cài Rust hay trình biên dịch. Cấp quyền thực thi cho file [`wsb`](wsb):

```bash
chmod 755 ./wsb
```

Kiểm tra nhanh:

```bash
./wsb --version
# Kết quả: wsb 0.2.0
```

> [!TIP]
> Copy vào `PATH` để gọi `wsb` từ bất kỳ đâu:
> ```bash
> cp wsb ~/.local/bin/   # hoặc: sudo cp wsb /usr/local/bin/
> ```

> [!NOTE]
> File nhị phân Linux `wsb` hỗ trợ cả hai chế độ `server` và `client`. Nếu máy tính cá nhân của bạn dùng Windows, hãy dùng bản phát hành `wsb.exe` trên Windows làm client để kết nối tới `./wsb server` đang chạy trên máy chủ Linux (như Kaggle, Google Colab, Ubuntu VPS).

---

## 📖 Khởi chạy nhanh

### 1. Server (Kaggle / Máy chủ Linux từ xa)

**Dùng file cấu hình mẫu ([`config.server.toml`](config.server.toml)):**
```bash
./wsb server -c config.server.toml
```

**Dùng cờ CLI:**
```bash
# Tự sinh token ngẫu nhiên 256-bit mỗi lần khởi chạy (khuyên dùng):
./wsb server --listen 0.0.0.0:8766

# Hoặc chỉ định token cố định và whitelist target:
./wsb server --listen 0.0.0.0:8766 --token "my-secret-token" --allow 127.0.0.1:3306 --allow 127.0.0.1:80

# Sinh sẵn token 256-bit (64 ký tự hex) để dán vào file cấu hình:
./wsb gen-token
```

**Tùy chọn — Public server qua Cloudflare Quick Tunnel:**
```bash
cloudflared tunnel --url http://127.0.0.1:8766 --no-autoupdate
# → https://xxx-xxx-xxx.trycloudflare.com
```
*(Điền `wss://xxx-xxx-xxx.trycloudflare.com` vào `ws_url` ở client, hoặc dùng `ws://<ip-server>:8766` nếu kết nối trực tiếp.)*

### 2. Client (Máy Linux cục bộ)

**Dùng file cấu hình mẫu ([`config.client.toml`](config.client.toml)):**

Sửa `ws_url` và `token` trong `config.client.toml`, sau đó chạy:
```bash
./wsb client -c config.client.toml
```

**Dùng cờ CLI:**
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
> Bạn có thể kết hợp `-c config.client.toml` với `--ws`, `--token`, `--socks5` hoặc bổ sung thêm các cờ `--tcp` / `--udp` / `--http`.

---

## 🎯 Ví dụ sử dụng

### MySQL qua TCP Tunnel

Kết nối bằng DBeaver, Navicat, MySQL Workbench hoặc `mysql` CLI:
```bash
mysql -h 127.0.0.1 -P 13306 -u <your_user> -p
```
- **Host:** `127.0.0.1` &nbsp;|&nbsp; **Port:** `13306`

### phpMyAdmin / Web qua HTTP Tunnel

Mở trình duyệt tại:
```text
http://127.0.0.1:18080/phpmyadmin/
```

> [!NOTE]
> `wsb` tự động viết lại header `Host` thành địa chỉ `target` từ xa (ví dụ `127.0.0.1:80`) để Apache/Nginx nhận diện đúng virtual host và tránh lỗi redirect.

### DNS qua UDP Tunnel

```bash
dig @127.0.0.1 -p 10053 google.com
# hoặc: nslookup -port=10053 google.com 127.0.0.1
```

### SOCKS5 Dynamic Proxy

Chỉ cần **1 cổng** `127.0.0.1:1080` để chuyển tiếp kết nối TCP tới bất kỳ địa chỉ và cổng nào trên máy chủ từ xa:

```bash
# Gọi HTTP qua SOCKS5 (phân giải DNS ở phía remote)
curl --socks5-hostname 127.0.0.1:1080 http://127.0.0.1:80/phpmyadmin/

# SSH qua SOCKS5
ssh -o "ProxyCommand=nc -X 5 -x 127.0.0.1:1080 %h %p" root@127.0.0.1
```

**Trình duyệt:** Cấu hình SOCKS5 proxy là `127.0.0.1:1080` (dùng SwitchyOmega trên Chrome hoặc cài đặt mạng trong Firefox).  
*(Giới hạn: SOCKS5 tích hợp chỉ hỗ trợ lệnh `CONNECT` với chế độ không xác thực; chưa hỗ trợ `BIND` và `UDP ASSOCIATE`.)*

---

## ⚙️ CLI Reference

### `wsb server`

| Cờ | Biến môi trường | Mặc định | Mô tả |
|---|---|---|---|
| `-c, --config <CONFIG>` | — | — | Đường dẫn tới file cấu hình TOML cho server |
| `-l, --listen <LISTEN>` | `WS_BRIDGE_LISTEN` | `0.0.0.0:8766` | Địa chỉ lắng nghe kết nối WebSocket |
| `-t, --token <TOKEN>` | `WS_BRIDGE_TOKEN` | `auto` | Token xác thực client (`auto` / `generate` / rỗng sẽ tự sinh token 256-bit khi khởi chạy) |
| `--allow <ALLOWED>` | — | Tất cả | Whitelist `host:port` được phép kết nối (có thể lặp lại; ghi đè `allowed_targets` trong config) |

### `wsb client`

| Cờ | Biến môi trường | Mặc định | Mô tả |
|---|---|---|---|
| `-c, --config <CONFIG>` | — | — | Đường dẫn tới file cấu hình TOML cho client |
| `-w, --ws <WS_URL>` | `WS_BRIDGE_URL` | *Bắt buộc* | URL WebSocket server từ xa (`wss://...` hoặc `ws://...`) |
| `-t, --token <TOKEN>` | `WS_BRIDGE_TOKEN` | `12345` | Token bí mật khớp với server |
| `--tcp <LISTEN=TARGET>` | — | — | Thêm rule TCP tunnel (lặp lại được, vd: `127.0.0.1:13306=127.0.0.1:3306`) |
| `--udp <LISTEN=TARGET>` | — | — | Thêm rule UDP tunnel (lặp lại được, vd: `127.0.0.1:10053=1.1.1.1:53`) |
| `--http <LISTEN=TARGET>` | — | — | Thêm rule HTTP tunnel kèm rewrite `Host` (lặp lại được) |
| `--socks5 <LISTEN_ADDR>` | — | — | Bật SOCKS5 dynamic proxy (vd: `127.0.0.1:1080`; ghi đè `socks5` trong config) |

> [!TIP]
> Chạy với `RUST_LOG=debug ./wsb ...` để xem log chi tiết từng kết nối và handshake. Mặc định là `wsb=info,info`.

### `wsb gen-token`

In ra stdout một token ngẫu nhiên 256-bit an toàn mật mã (64 ký tự hex viết thường).

---

## 📄 Cấu hình TOML

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

## 🗂️ Cấu trúc dự án

```text
ws_bridge/
├── wsb                     # Bản dựng Linux x86_64 dựng sẵn (v0.2.0)
├── config.server.toml      # Cấu hình mẫu cho server
├── config.client.toml      # Cấu hình mẫu cho client
├── README.md               # Tài liệu hướng dẫn (English)
├── README.vi.md            # Tài liệu hướng dẫn (Tiếng Việt)
└── README.zh-CN.md         # Tài liệu hướng dẫn (简体中文)
```

---

## 📝 License

MIT © [conghoa004](https://github.com/conghoa004)
