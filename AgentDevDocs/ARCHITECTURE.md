# Architecture

## Purpose

This document defines the system architecture of scnet (SHUOSC_Network).

Before making changes, agents should also read:

- [AGENTS.md](AGENTS.md) for working rules
- [API_CONTRACT.md](API_CONTRACT.md) for endpoint specifications
- [SECURITY.md](SECURITY.md) for threat model and invariants
- [WORKBOARD.md](WORKBOARD.md) for implementation order

## Project Summary

scnet is an invite-only WireGuard-based campus VPN for Shanghai University of
Finance and Economics students. It provides internal network access and
optional academic website acceleration. A web portal hosted on Cloudflare Pages
serves both ordinary users and administrators. Client keys are always generated
client-side. The server never stores private keys.

## System Overview

```
                        Cloudflare Pages
                    ┌──────────────────────────┐
                    │  Quasar SPA (Vue 3)       │
                    │  Pages Function:           │
                    │    GET  /api/server-info   │  ← CLI/SPA 拉取后端地址
                    │    POST /api/server-info   │  ← scnet-server 上报地址
                    └──────────────────────────┘
                         ▲     │
                    上报  │     │ 拉取
                         │     ▼
                    ┌────┴────────────────────────────────┐
                    │       宿主机 (内网, 无公网 IP)         │
                    │                                      │
                    │  Lucky (STUN UDP 穿透)                │
                    │  ├─ TCP :18080 → 容器 :8080 (API)     │
                    │  └─ UDP :51820 → 容器 :51820 (WG)     │
                    │           │                          │
                    │  ┌────────▼────────────────────┐     │
                    │  │  Podman 容器                  │     │
                    │  │  scnet-server                 │     │
                    │  │  :8080  REST API (CORS 开)    │     │
                    │  │  wg_scnet    WireGuard 接口        │     │
                    │  │  策略路由 / 代理池             │     │
                    │  │  上报协程 → CF /api/server-info│     │
                    │  └──────────────────────────────┘     │
                    └──────────────────────────────────────┘

Client (CLI/GUI) ──HTTPS──▶ Lucky 穿透地址 (API)
        │                         │
 wireguard-go UAPI                │
        │                         │
 WireGuard Tunnel ──UDP──▶ Lucky 穿透地址 (WG)
```

**部署要点：**
- 前端 SPA 托管在 Cloudflare Pages，不再嵌入 Go server
- 后端 API 通过 Lucky STUN 穿透暴露公网地址
- 地址发现：server 上报到 CF Pages Function → CLI/SPA 拉取
- 如果没有公网 IP，Lucky 同时暴露 TCP(API) 和 UDP(WireGuard)
- 如果有公网 IP，Lucky 可跳过，直接用宿主机 IP

## Data Flow

Standard internal traffic:

```
User -> WG Tunnel -> Server tun0 -> Direct Route -> Internal Target
```

Academic acceleration traffic:

```
User -> WG Tunnel -> Server tun0 -> Classifier -> SOCKS5 LB -> Proxy Pool -> Egress
```

Authentication flow:

```
Client -> POST /auth/login -> JWT
Client -> POST /peer/register (JWT + public_key) -> Server -> DB -> wgctrl -> WG kernel
```

## Technology Stack

| Layer | Component | Constraint |
|-------|-----------|------------|
| Server | Go, gin, wgctrl, Podman 容器化 | Linux only, `--cap-add=NET_ADMIN` |
| Client CLI | Go, cobra, wireguard-go | Cross-platform |
| Client GUI | Tauri (desktop), native (mobile) | Phase 5 |
| Web Portal | Quasar (Vue 3), Vite, Pinia, Axios | Cloudflare Pages 托管 |
| Database | PostgreSQL | 宿主机或独立容器 |
| WireGuard Server | Kernel module | >= Linux 5.6 (容器内) |
| WireGuard Client | wireguard-go UAPI | Userspace |
| NAT 穿透 | Lucky (STUN) | 宿主机运行，TCP+UDP 穿透 |
| 地址发现 | Cloudflare Pages Functions | 共享密钥鉴权 |

## Server Design

### Module Structure

```
server/
  cmd/scnet-server/main.go
  internal/
    api/        router, middleware, handlers
    auth/       service, password (bcrypt)
    account/    service (self profile, password, self peer listing)
    peer/       manager (wgctrl), ipam
    admin/      service (summary, force operations)
    discovery/  reporter (上报地址到 CF Pages Function)
    model/      user, peer, invite
    store/      db, user_store, peer_store, invite_store
  migrations/   001_init.sql
  config/       config.go

server-panel/
  scnet-panel/  Quasar source project → build 部署到 Cloudflare Pages

cloudflare/
  functions/
    api/
      server-info.ts   CF Pages Function: GET(拉取) + POST(上报)
```

### Component Responsibilities

| Component | Responsibility | Constraint |
|-----------|---------------|------------|
| HTTP API | REST endpoints | All except `/auth/*`, `/health`, `/version` require JWT; CORS 允许 CF Pages 域名 |
| Auth Middleware | JWT validation, user_id extraction | Stateless, no session storage |
| Admin Middleware | JWT role check (role="admin") | Returns 403 if role != admin |
| Auth Service | Registration, login, password hashing | Never generates WG keys |
| Account Service | Current-user profile, contact info, password change, self peer listing | Can only operate on JWT.sub |
| Peer Manager | Peer CRUD, wgctrl calls | Only operates on authenticated peers |
| IPAM | IP allocation (10.100.0.0/24) | Unique per peer |
| Admin Service | Summary aggregation, forced peer ops, WG key rotation, WG toggle | Read-only counts + wgctrl actions |
| Discovery Reporter | 定时上报 api_url + wg_endpoint 到 CF | Shared secret 鉴权，30s 间隔 |
| Store | PostgreSQL access | DB is authoritative record |
| Web Portal (SPA) | Quasar Vue 3 user/admin UI | Cloudflare Pages 托管，API 调 Lucky 穿透地址 |
| CF Pages Function | 地址发现：GET 返回 server 地址，POST 接收上报 | Shared secret 鉴权 |

### Startup Sequence

1. Load config (port, DB, WG, discovery settings)
2. Connect PostgreSQL, run migrations
3. Initialize Peer Manager (ensure wg_scnet exists)
4. Start HTTP server on :8080 (API only, 带 CORS)
5. Start discovery reporter goroutine (定时上报地址到 CF)
6. Background: reconciliation goroutine (DB-WG state alignment, every 5 min)

### Web Portal Roles

SPA 托管在 Cloudflare Pages。两个角色共享同一构建产物：

- `user`: self-service portal for account profile, contact information,
  password changes, and management of the user's own peers and public keys
- `admin`: global control console for all users, peers, invite codes, and
  system summary views

The login flow is shared. After `POST /auth/login`, the SPA calls `GET /me`
and redirects by `role`:

- `role == "user"` → `/app/`
- `role == "admin"` → `/admin/`

Only administrators may create invite codes or inspect other users' peers.
Ordinary users may only act on resources resolved from their own `JWT.sub`.

### Address Discovery Flow

```
Server 启动
  │
  ├── Lucky 穿透成功 → 获得 api_url + wg_endpoint
  │
  ├── Discovery Reporter (每 30s):
  │     POST https://<cf-pages>/api/server-info
  │     Authorization: Bearer <DISCOVERY_SECRET>
  │     Body: { "api_url": "...", "wg_endpoint": "..." }
  │     → CF Pages Function 存入内存缓存 (TTL 60s)
  │
  ▼
SPA / CLI 客户端:
  │
  ├── GET https://<cf-pages>/api/server-info
  │     → { "api_url": "...", "wg_endpoint": "..." }
  │
  ├── SPA: 设置 baseUrl = api_url, 后续所有 fetch 用 baseUrl + path
  └── CLI: 用 api_url 做登录, 用 wg_endpoint 建立 WireGuard 隧道
```

**鉴权规则：**
- `GET /api/server-info` — 公开，无需鉴权（返回的是公网地址，无所谓谁看到）
- `POST /api/server-info` — 需要 `Authorization: Bearer <DISCOVERY_SECRET>`，只有 server 可以上报

**客户端手动覆盖：**
- CLI 的 `--server-url` 参数优先级高于自动发现
- SPA 的 `?api=` URL 参数可覆盖 auto-discovered baseUrl（便于测试）

### User Portal Constraints

The user portal can show:

- current account status and peer quota usage
- contact information stored on the server
- the user's own peers, public keys, assigned IPs, and server-side connection
  parameters

The user portal cannot recover a full client configuration after private-key
loss, because the server never stores client private keys.

## Client Design

### CLI Architecture

```
cli/
  cmd/scnet/main.go
  internal/
    cmd/       root, connect, disconnect, status, export, version
    auth/      login, peer register, config fetch
    tunnel/    wireguard-go device, UAPI configuration
    store/     credential persistence, state management
  platform/    route_*.go, dns_*.go (go:build tags per OS)
```

### Commands

| Command | Function |
|---------|----------|
| `scnet connect` | Login, generate keypair, register peer, establish tunnel |
| `scnet disconnect` | Close tunnel, deregister peer |
| `scnet status` | Display connection state, peer info, traffic |
| `scnet export-config` | Export standard wg-quick configuration |
| `scnet version` | Version information |

### Authentication Flow (All Platforms)

First connection:

1. User enters student_id + password
2. `POST /auth/login` -> JWT
3. `wgtypes.GeneratePrivateKey()` -> local keypair
4. `POST /peer/register` (JWT + public_key) -> peer_config
5. Establish WG tunnel with local private key + returned peer_config
6. Persist credentials to `~/.scnet/credentials.json` (jwt included)

Reconnection:

1. Read `~/.scnet/credentials.json`
2. If JWT valid: `GET /peer/config?public_key=...` -> rebuild tunnel
3. If JWT expired: `POST /auth/login` -> new JWT -> `GET /peer/config?public_key=...`
4. If credentials lost: full first-connection flow with new keypair

### Platform Support

| Platform | Architecture | Status |
|----------|-------------|--------|
| macOS | amd64, arm64 | CLI Phase 2, GUI Phase 5 |
| Linux | amd64, arm64, riscv64, loong64, mips64le | CLI Phase 2 |
| Windows | amd64, arm64 | CLI Phase 2, GUI Phase 5 |
| FreeBSD | amd64, arm64 | CLI Phase 2 |
| OpenBSD | amd64, arm64 | CLI Phase 2 |
| Android | Kotlin + wireguard-android | GUI Phase 5 |
| iOS | Swift + NetworkExtension | GUI Phase 5 |

### Client Storage

```
~/.scnet/
  config.json         Server URL, version
  credentials.json    student_id, public_key, private_key, jwt (0600)
  state.json          Connection state, peer config, traffic stats
```

## Data Model

### PostgreSQL Schema

```sql
CREATE TABLE users (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id VARCHAR(20) UNIQUE NOT NULL,
    password   VARCHAR(255) NOT NULL,            -- bcrypt hash
    role       VARCHAR(8) DEFAULT 'user',           -- 'user' | 'admin'
    invite_id  UUID REFERENCES invite_codes(id),
    display_name VARCHAR(64),
    email      VARCHAR(255),
    phone      VARCHAR(32),
    wechat     VARCHAR(64),
    telegram   VARCHAR(64),
    max_peers  INT DEFAULT 5,                    -- max active peers per user, adjustable per-user
    status     VARCHAR(16) DEFAULT 'active',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE invite_codes (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code       VARCHAR(32) UNIQUE NOT NULL,
    created_by UUID,                               -- nullable for bootstrap
    used_by    UUID REFERENCES users(id),
    max_uses   INT DEFAULT 1,
    use_count  INT DEFAULT 0,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE peers (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    UUID REFERENCES users(id) NOT NULL,
    public_key VARCHAR(64) UNIQUE NOT NULL,      -- WG public key, base64
    assigned_ip INET NOT NULL,
    status     VARCHAR(16) DEFAULT 'active',      -- active / disconnected / revoked
    last_seen  TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
-- Peer limit enforced in application layer per user.max_peers
```

### Peer Lifecycle

```
Created -> active -> disconnected -> active (reconnect)
                  -> revoked (admin ban)
                  -> expired (> 7 days no traffic)
```

Heartbeat: PersistentKeepalive = 25s (WireGuard built-in)
Reconciliation: Server goroutine ensures DB and WG interface state match.

## Academic Acceleration

Optional server-side feature. Routes academic website traffic through a proxy
pool instead of direct egress.

```
WG Client -> Server tun0 -> Classifier
                              |
                     Academic IP -> SOCKS5 LB -> Proxy Pool (100+ nodes) -> Egress
                              |
                     Internal IP -> Direct route
```

Components:

- Classifier: matches destination IP against academic CIDR list
- SOCKS5 LB: round-robin with health checks (TCP probe every 30s)
- Proxy Pool: external nodes provided by third-party proxy services

DNS: client-side, no server-side DNS interception.

## Experimental P2P (Phase 6)

Goal: reduce latency via direct peer-to-peer connections.

```
Phase 1: Central coordination + fallback
  - Nodes register with central server (IP, port, public_key, NAT type)
  - Client queries central server for candidate nodes
  - TCP STUN hole punching attempted
  - Punch fails -> fallback to server relay

Phase 2: DHT-based discovery, optimized routing
```

Toggle: client-side switch, off by default.

## Deployment

### Server (Podman 容器)

| Requirement | Value |
|-------------|-------|
| OS | Linux, kernel >= 5.6 |
| CPU | 2+ cores |
| RAM | 2+ GB |
| Storage | 20 GB |
| Container runtime | Podman, `--cap-add=NET_ADMIN` |
| Database | PostgreSQL >= 14 (宿主机或独立容器) |
| Build | Go >= 1.22 |

**网络：**
- 有公网 IP → Lucky 可跳过，直接暴露 TCP 8080 + UDP 51820
- 无公网 IP → Lucky (STUN) 暴露 TCP(API) + UDP(WireGuard)，地址通过 CF Pages Function 发现

### Frontend (Cloudflare Pages)

| Requirement | Value |
|-------------|-------|
| Build | `quasar build` (Vite) |
| Output | `server-panel/scnet-panel/dist/spa/` |
| Deploy | Cloudflare Pages (直接上传或 Git 集成) |
| Functions | `cloudflare/functions/api/server-info.ts` |

### Operations

- Logging: stdout, systemd journal
- Monitoring: `/health` endpoint (Prometheus metrics in Phase 2)
- Backup: PostgreSQL pg_dump + WAL archiving
