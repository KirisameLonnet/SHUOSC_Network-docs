# Workboard

## Purpose

This file is the concrete implementation workboard for future LLM agents.

Unlike `ARCHITECTURE.md`, this document is not trying to define the system in
full. It is trying to answer:

- what should be built next
- in what order
- what counts as done

## Working Rules

- server never stores client private keys
- WireGuard protocol layer stays unmodified
- all authenticated endpoints use JWT
- prefer small reviewable steps
- update docs whenever truth changes

## Phase 0: Repository Foundation

Status:

- done

Deliverables:

- `docs/ARCHITECTURE.md`
- `docs/AGENTS.md`
- `docs/API_CONTRACT.md`
- `docs/IMPLEMENTATION.md`
- `docs/SECURITY.md`
- `docs/WORKBOARD.md`

Exit criteria:

- the project is no longer a blank slate
- all design documents exist and are internally consistent

## Phase 1: Server MVP + Web Portal

Status:

- implemented (backend + frontend build, tests pass; pending integration verification)

Goal:

- a server that accepts authenticated peer registrations and manages WireGuard
  peers, bundled with a Quasar (Vue 3) SPA that serves both a user
  self-service portal and an admin management console

Notes:

- the web portal is merged into Phase 1 (not a separate phase) — it shares the
  same server, database, JWT auth, and error handling patterns
- frontend uses Quasar Framework components exclusively (QTable, QForm,
  QDialog, QCard, QBtn, QSelect, QInput, QLayout, QDrawer, etc.) to minimize
  raw CSS and maintain uniform design language
- frontend served from the same Go server at `/app/` and `/admin/`
  (single-origin, no CORS needed); API stays at `/api/v1/*`
- only administrators can generate invite codes or inspect other users' peers
- ordinary users can only inspect and manage resources owned by their own
  `JWT.sub`

### 1a. Server Backend

Tasks:

- initialize Go module structure under `server/`
- create database migration (users, invite_codes, peers tables)
- implement Auth Service (bcrypt password hashing, login, JWT issuance)
- implement Account Service (self profile, contact info, password change, own
  peer listing)
- implement Peer Manager (wgctrl peer add/remove, IPAM)
- implement Self-Service API (`me`, `me/password`, `me/peers`)
- implement HTTP API (auth/register, auth/login, peer/register, peer/config,
  peer/disconnect, peer/replace-key)
- implement Admin API (admin/me, admin/summary, admin/users, admin/user/:id,
  admin/user/:id/peers, admin/user/:id edit, admin/peers,
  admin/peer/:id/disconnect, admin/peer/:id/revoke, admin/invites, admin/invite)
- implement Admin middleware (role="admin" check)
- implement server startup (config loading, DB connect, wg0 init,
  reconciliation goroutine)
- serve web portal static assets under `/app/` and `/admin/` paths (Go embed.FS)

Exit criteria:

- `scnet-server` binary starts, listens on :8080
- `POST /auth/register` creates a user with valid invite code
- `POST /auth/login` returns a valid JWT
- `GET /me` returns the current user's profile and peer quota usage
- `PUT /me` updates the current user's contact information without allowing role
  or quota escalation
- `PUT /me/password` changes the current user's password after current-password
  verification
- `GET /me/peers` lists only the current user's peers
- `POST /peer/register` with JWT adds a WireGuard peer to wg0
- `POST /peer/register` returns 409 when user hits max_peers limit
- `GET /peer/config` with JWT returns the peer's configuration
- `DELETE /peer/disconnect` with JWT removes the peer from wg0
- `GET /admin/summary` returns accurate user/peer/invite counts
- `GET /admin/users` supports pagination, search, and status/role filters
- `PUT /admin/user/:id` changes per-user max_peers/status, effective immediately
- `POST /admin/peer/:id/disconnect` removes the peer from wg0
- `POST /admin/invite` generates a valid invite code
- server reconciliation goroutine runs without errors

### 1b. Web Frontend (User Portal + Admin Console)

Tasks:

- scaffold Quasar project under `server-panel/scnet-panel/` (Vite + TypeScript + Pinia)
- implement shared login flow (POST /auth/login → GET /me → role-based redirect)
- implement user home page (GET /me + quota usage)
- implement account settings page (PUT /me + PUT /me/password)
- implement My Peers page (GET /me/peers + peer disconnect/replace-key actions)
- implement admin Dashboard page (GET /admin/summary + GET /health + GET /version)
- implement admin Users page (QTable server-side pagination, search, status/role
  filters, QDialog edit form)
- implement admin User Detail page (metadata card + associated peers table)
- implement admin Peers page (QTable, filters, force-disconnect/revoke actions)
- implement admin Invites page (QTable, QDialog create form)
- implement Axios boot file (JWT interceptor, error handling)
- implement Pinia stores (auth, me, myPeers, dashboard, users, peers, invites)
- implement Vue Router route guards (`requiresAuth`, `requiresUser`,
  `requiresAdmin`)
- wire Quasar Notify/Dialog/Loading plugins for UX feedback
- configure Quasar build to output to `server/admin-panel/dist/spa/` for Go embed

Exit criteria:

- `POST /auth/login` from the shared login page returns JWT
- `GET /me` determines whether the browser enters `/app/` or `/admin/`
- user home page shows account status and peer quota usage
- account settings page updates contact information and password
- My Peers page lists only the current user's peers
- My Peers page can disconnect a peer and rotate its public key
- admin Dashboard shows live summary counts and system health status
- admin Users page lists all users with working server-side pagination and filters
- editing a user's max_peers or status via QDialog saves and reflects immediately
- admin User Detail page shows associated peers with disconnect/revoke actions
- admin Peers page lists all peers with status filters
- admin force-disconnecting/revoking a peer removes it from wg0
- admin Invites page shows all invite codes; creating a new invite works
- All API errors display via Quasar Notify toasts
- Logout clears JWT and redirects to login page
- the user portal never attempts to recover or display a client private key
- `quasar build` produces a valid SPA under `server/admin-panel/dist/spa/`

### Phase 1 Minimum E2E Matrix

The following scenarios define MVP completion. LLM agents should not claim
Phase 1 complete without verifying them.

| ID | Actor | Scenario | Expected result |
|----|-------|----------|-----------------|
| E2E-01 | guest | register with valid invite | account created, status `active`, role `user` |
| E2E-02 | guest | register with invalid/expired invite | `400 INVALID_INVITE` |
| E2E-03 | user | login from shared portal | receives JWT, `GET /me` returns profile, browser routes to `/app/` |
| E2E-04 | user | update contact info in settings | `PUT /me` persists only allowed fields; role/quota unchanged |
| E2E-05 | user | change password with wrong current password | `401 AUTH_FAILED` |
| E2E-06 | user | create peer until quota reached | peers added until limit; next add returns `409 TOO_MANY_PEERS` |
| E2E-07 | user | view My Peers and disconnect one peer | `GET /me/peers` shows only own peers; disconnect updates wg0 and DB |
| E2E-08 | user | rotate own peer public key | old key invalidated, new key active, no cross-user effect |
| E2E-09 | admin | login from shared portal | receives JWT, `GET /me` or role bootstrap routes to `/admin/` |
| E2E-10 | admin | list users, edit one user's `max_peers` and `status` | change is persisted and takes effect on next state-changing request |
| E2E-11 | admin | force-disconnect and revoke peer from admin peers view | wg0 and DB state updated correctly |
| E2E-12 | admin | create invite code and verify it can register a new user | invite appears in admin list and works for signup |
| E2E-13 | suspended user | attempt login and peer mutation | login denied; existing authenticated mutation requests fail |
| E2E-14 | portal boundary | user attempts to recover private key from web UI | impossible by design; no API returns private key |

Verification guidance:

- run API-level checks first (`curl` / integration tests)
- then run browser-level checks for routing, guards, and user/admin UI visibility
- record which scenarios remain blocked by environment rather than silently skipping them

### Phase 1 Security Deferrals

Recorded: 2026-04-28

Status:

- accepted for development stage
- do not treat as fixed
- must be revisited before production rollout

Deferred items:

- `SEC-03` CORS remains broader than ideal.
  Current meaning:
  the browser is allowed to make cross-origin API reads more freely than a production deployment should allow.
  Development decision on 2026-04-28:
  defer tightening the allowlist until deployment topology and final frontend origin are stable.
  Production revisit:
  replace wildcard/default-open CORS behavior with an explicit origin allowlist.

- `SEC-04` CLI stores WireGuard private key and JWT in local files under `~/.scnet/`.
  Current meaning:
  secrets are protected by local filesystem permissions (`0700` directory, `0600` files), but not yet moved into an OS keychain/credential vault.
  Development decision on 2026-04-28:
  accept filesystem-backed storage during development.
  Production revisit:
  evaluate moving JWT and/or private key material into platform keychain facilities where practical.

- `SEC-05` login rate limiting is process-local in-memory state.
  Current meaning:
  brute-force protection works only per running process; counters reset on restart and are not shared across multiple server instances.
  Development decision on 2026-04-28:
  keep the current lightweight limiter for single-instance development.
  Production revisit:
  move rate limiting to a shared store or upstream gateway/WAF so limits survive restarts and scale across instances.

- `SEC-06` PostgreSQL transport encryption is not enforced by default.
  Current meaning:
  the default database config still uses `sslmode=disable`, which is acceptable for local development but not for remote/shared production deployments.
  Development decision on 2026-04-28:
  defer DB TLS enforcement while the stack remains in development.
  Production revisit:
  require TLS-enabled PostgreSQL connectivity and document the production connection policy.

## Phase 2: CLI Client MVP

Status:

- implemented (all platform DNS/route helpers done; end-to-end blocked on server)

Notes:

- CLI foundation work can proceed against the documented API contract before
  the server MVP is complete.
- End-to-end verification for `scnet connect` remains blocked until the server
  control plane exists.

Goal:

- a cross-platform CLI binary that authenticates, registers a peer, and
  establishes a WireGuard tunnel

Tasks:

- initialize Go module structure under `cli/`
- implement local credential store (~/.scnet/credentials.json, config.json,
  state.json)
- implement auth service (login, peer register, config fetch)
- implement tunnel layer (wireguard-go device creation, UAPI configuration)
- implement platform-specific route/DNS helpers (go:build tags per OS)
- implement CLI commands:
  - `scnet connect` (login + keygen + register + tunnel up)
  - `scnet disconnect` (tunnel down + peer deregister)
  - `scnet status` (tunnel state, traffic, latency)
  - `scnet export-config` (dump wg-quick compatible config)
  - `scnet version`

Exit criteria:

- `scnet connect` completes end-to-end: login, key generation, peer
  registration, tunnel establishment
- `scnet status` shows live WireGuard peer statistics
- `scnet disconnect` cleanly removes the tunnel and notifies server
- `scnet export-config` produces a valid wg-quick configuration
- built and verified on at least: darwin/amd64, darwin/arm64, linux/amd64,
  windows/amd64

## Phase 3: Cross-Platform Build & CI

Status:

- pending

Goal:

- make building for all target platforms deterministic and automated

Tasks:

- create Makefile with targets for all platform/architecture combinations
- create GitHub Actions workflow for cross-platform builds
- verify builds on: linux/riscv64, linux/loong64, linux/mips64le,
  freebsd/amd64, freebsd/arm64, openbsd/amd64, openbsd/arm64
- add artifact upload for releases

Exit criteria:

- `make build-all` produces binaries for all target platforms
- CI pipeline runs on push and PR
- release artifacts are downloadable from GitHub

## Phase 4: Academic Acceleration

Status:

- pending

Goal:

- route academic website traffic through a proxy pool with load balancing

Tasks:

- implement traffic classification (academic IP/CIDR list vs internal)
- implement SOCKS5 load balancer (round-robin with failover)
- implement proxy pool health checker (TCP probe every 30s)
- wire into server data plane (tun0 -> classifier -> direct or proxy pool)

Exit criteria:

- traffic to academic IP ranges (GitHub, Google Scholar) is routed through
  proxy pool
- traffic to internal IP ranges is directly routed
- proxy node failure is detected and traffic shifts to healthy nodes
- no DNS modification required on client side

## Phase 5: GUI Clients

Status:

- pending

Goal:

- graphical clients for major platforms

Sub-phases:

### 5a: macOS GUI (Tauri/SwiftUI)

- login UI
- one-click connect/disconnect
- status display (IP, traffic, latency)
- academic acceleration toggle
- experimental P2P toggle

### 5b: Windows GUI (Tauri)

- same feature set as macOS

### 5c: Android (Kotlin + wireguard-android)

- native Android VPN Service integration
- Jetpack Compose UI
- background connection persistence

### 5d: iOS (Swift + NetworkExtension)

- NEPacketTunnelProvider integration
- SwiftUI interface
- system VPN toggle integration

Exit criteria for each:

- user can log in, connect, and access internal network without CLI
- connection status is visible in system tray / notification area
- disconnect is one action

## Phase 6: Experimental P2P

Status:

- pending

Goal:

- reduce latency via direct peer-to-peer connections when possible

Tasks:

- implement TCP STUN hole punching on client
- implement central P2P coordination server (node registration, NAT type
  detection, candidate exchange)
- implement P2P fallback: punch fails -> route through server
- add client-side toggle (off by default)

Exit criteria:

- two clients behind cone NAT can establish direct P2P connection
- P2P failure falls back to server relay without connection loss
- toggle correctly switches between P2P and server-relay modes

## Review Questions For Any Future Task

Before starting a task:

- does this belong to server, client, or shared protocol work?
- does it require a doc update first?
- is there already a schema or invariant that should guide it?

Before finishing a task:

- did the server-no-private-key invariant remain intact?
- did the WireGuard protocol layer remain unmodified?
- did platform portability remain intact?
- did the change move the project toward a working VPN service?
