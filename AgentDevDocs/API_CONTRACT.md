# API Contract

## Purpose

This document defines every HTTP endpoint, its request/response shape, error
codes, and authentication requirements.

Before implementing or changing any endpoint, agents must read this document.

## General Conventions

- Base URL: `https://<server>:8080/api/v1`
- Content-Type: `application/json`
- Auth: JWT Bearer Token (except `/auth/register`, `/auth/login`, `/health`,
  and `/version`)
- Error format: `{"error": "human readable message", "code": "ERROR_CODE"}`
- Success: 200 OK, body contains `{"status": "ok", ...data}`

## Validation & State Rules

### Password Policy

All password-accepting endpoints (`POST /auth/register`, `PUT /me/password`)
must enforce the same rules:

| Rule | Requirement |
|------|-------------|
| Minimum length | 10 characters |
| Maximum length | 128 characters |
| Character classes | Must contain at least 1 letter and 1 digit |
| Whitespace | Allowed, but leading/trailing whitespace is trimmed before validation |
| Reuse | `new_password` must differ from `current_password` |
| Storage | Store bcrypt hash only, cost=12 |

If validation fails, return:

```json
{
  "error": "new password does not meet policy",
  "code": "INVALID_PASSWORD"
}
```

### Contact Field Rules

Self-service profile updates use the following normalization and validation:

| Field | Required | Max Length | Validation / Normalization |
|------|----------|------------|-----------------------------|
| `display_name` | no | 64 | trim whitespace; empty string becomes null |
| `email` | no | 255 | lowercase + trim; must contain `@` if non-empty |
| `phone` | no | 32 | trim; digits, `+`, `-`, space, `(`, `)` only |
| `wechat` | no | 64 | trim; no whitespace inside |
| `telegram` | no | 64 | trim; may optionally start with `@` |

Rules:

- all contact fields are optional
- empty strings are normalized to null
- self-service endpoints may update only these fields plus `display_name`
- no contact field is globally unique at the API-contract level unless later specified by migration

If validation fails, return:

```json
{
  "error": "contact field validation failed",
  "code": "INVALID_CONTACT"
}
```

### Account Status Semantics

User `status` meanings are fixed:

| Status | Login | `GET /me` | Create/replace/disconnect own peer | Admin visible | Meaning |
|------|-------|-----------|-------------------------------------|---------------|---------|
| `active` | allowed | allowed | allowed | yes | normal account |
| `suspended` | denied | denied if token absent; existing JWT requests return 403 on state-changing operations | denied | yes | temporarily disabled |
| `deleted` | denied | denied | denied | yes | tombstoned / disabled record |

Additional rules:

- `POST /auth/login` must reject `suspended` and `deleted` users with `403 FORBIDDEN`
- `POST /peer/register`, `PUT /peer/replace-key`, and self-service profile mutation endpoints must reject non-`active` users
- admin may still inspect suspended/deleted users and their peers
- when an admin changes a user to `suspended`, future JWT-authenticated state-changing requests for that user must fail immediately based on fresh DB state

## Authentication Endpoints

### POST /auth/register

Create a new user account.

```
Auth: none
Request:
{
  "student_id": "2024001234",
  "password": "min_8_chars",
  "invite_code": "SCNET-XXXX-YYYY"
}

Response 200:
{
  "status": "ok",
  "user_id": "uuid-v4"
}

Response 400 (invalid invite):
{
  "error": "invalid or expired invite code",
  "code": "INVALID_INVITE"
}

Response 409 (duplicate):
{
  "error": "student_id already registered",
  "code": "ALREADY_EXISTS"
}
```

### POST /auth/login

Authenticate and receive a JWT.

```
Auth: none
Request:
{
  "student_id": "2024001234",
  "password": "min_8_chars"
}

Response 200:
{
  "status": "ok",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_at": "2026-04-28T15:00:00Z"
}

Response 401:
{
  "error": "invalid student_id or password",
  "code": "AUTH_FAILED"
}

Response 403:
{
  "error": "account is not active",
  "code": "FORBIDDEN"
}
```

## Self-Service Endpoints

### GET /me

Get the current authenticated user's profile for the web portal bootstrap.

```
Auth: Bearer <jwt>
Request: none

Response 200:
{
  "status": "ok",
  "user": {
    "id": "uuid",
    "student_id": "2024001234",
    "role": "user",
    "display_name": "Alice",
    "email": "alice@example.com",
    "phone": "13800138000",
    "wechat": "alice_wechat",
    "telegram": "alice_tg",
    "max_peers": 5,
    "status": "active",
    "created_at": "2026-04-28T10:00:00Z",
    "updated_at": "2026-04-28T10:00:00Z"
  },
  "peer_usage": {
    "active": 2,
    "max_peers": 5
  }
}
```

Response 400:
{
  "error": "contact field validation failed",
  "code": "INVALID_CONTACT"
}

Server-side behavior:

- extract user_id from JWT.sub
- query users table by user_id
- count active peers for that user
- return only the current user's profile

### PUT /me

Update the current user's self-service profile fields.

```
Auth: Bearer <jwt>
Request:
{
  "display_name": "Alice",
  "email": "alice@example.com",
  "phone": "13800138000",
  "wechat": "alice_wechat",
  "telegram": "alice_tg"
}

Response 200:
{
  "status": "ok",
  "user": {
    "id": "uuid",
    "student_id": "2024001234",
    "role": "user",
    "display_name": "Alice",
    "email": "alice@example.com",
    "phone": "13800138000",
    "wechat": "alice_wechat",
    "telegram": "alice_tg",
    "max_peers": 5,
    "status": "active",
    "created_at": "2026-04-28T10:00:00Z",
    "updated_at": "2026-04-28T10:30:00Z"
  }
}
```

Server-side behavior:

- extract user_id from JWT.sub
- update only self-service fields present in the request body
- reject attempts to change `role`, `status`, or `max_peers`

### PUT /me/password

Change the current user's password.

```
Auth: Bearer <jwt>
Request:
{
  "current_password": "old-password",
  "new_password": "new-password-123"
}

Response 200:
{
  "status": "ok",
  "message": "password updated"
}

Response 401:
{
  "error": "current password is incorrect",
  "code": "AUTH_FAILED"
}

Response 400:
{
  "error": "new password does not meet policy",
  "code": "INVALID_PASSWORD"
}
```

Server-side behavior:

- extract user_id from JWT.sub
- verify current password against stored bcrypt hash
- validate the new password against minimum policy
- write the new bcrypt hash to users.password

### GET /me/peers

List only the current user's peers for the self-service portal.

```
Auth: Bearer <jwt>
Request: GET /me/peers?status=active&page=1&page_size=20

Response 200:
{
  "status": "ok",
  "items": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "public_key": "base64key...",
      "assigned_ip": "10.100.0.42",
      "status": "active",
      "last_seen": "2026-04-28T10:00:00Z",
      "created_at": "2026-04-28T10:00:00Z",
      "updated_at": "2026-04-28T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 2
  }
}
```

Query parameters:

- `status` (string, optional filter: active | disconnected | revoked)
- `page` (int, default 1)
- `page_size` (int, default 20)

Server-side behavior:

- extract user_id from JWT.sub
- list only peers owned by the current user
- support the same peer status model used by admin listings

## Peer Endpoints

### POST /peer/register

Register a WireGuard peer for the authenticated user.

```
Auth: Bearer <jwt>
Request:
{
  "public_key": "xTIB9P5E4aHBxK3HtGv0nYTdF...44 char base64..."
}

Response 200:
{
  "status": "ok",
  "peer_config": {
    "server_pubkey": "sR7xX9Y3bN2mK5jW8pQ4vF...",
    "assigned_ip": "10.100.0.42",
    "allowed_ips": ["10.100.0.0/24"],
    "endpoint": "scnet.example.com:51820",
    "persistent_keepalive": 25
  }
}

Response 409 (peer limit reached):
{
  "error": "peer limit reached (max 5 peers for this account)",
  "code": "TOO_MANY_PEERS"
}

Response 409 (already exists):
{
  "error": "user already has an active peer with this key",
  "code": "PEER_EXISTS"
}
```

The response intentionally excludes any client private key. The server never
stores client private keys, so the web portal cannot regenerate a full
WireGuard client config after key loss.

Server-side behavior:

- extract user_id from JWT.sub
- read user.max_peers from users table
- count active peers: SELECT COUNT(*) FROM peers WHERE user_id = $1 AND status = 'active'
- if count >= user.max_peers, return 409 TOO_MANY_PEERS
- allocate IP from 10.100.0.0/24 pool
- add peer to kernel WireGuard interface via wgctrl
- insert row into peers table

### GET /peer/config

Retrieve the authenticated user's peer configuration. Uses `public_key` query
parameter to select which peer when user has multiple devices.

```
Auth: Bearer <jwt>
Request: GET /peer/config?public_key=xTIB9P5E...

Response 200:
{
  "status": "ok",
  "peer_config": {
    "public_key": "xTIB9P5E...",
    "assigned_ip": "10.100.0.42",
    "server_pubkey": "sR7xX9Y3bN2mK5jW8pQ4vF...",
    "allowed_ips": ["10.100.0.0/24"],
    "endpoint": "scnet.example.com:51820",
    "connected": true
  }
}

Response 404:
{
  "error": "no active peer found for this user",
  "code": "NO_ACTIVE_PEER"
}
```

Server-side behavior:

- extract user_id from JWT.sub
- query peers table for public_key + user_id match
- if no match, return 404
- return peer configuration

### DELETE /peer/disconnect

Remove the authenticated user's peer. Uses `public_key` query parameter to
select which peer.

```
Auth: Bearer <jwt>
Request: DELETE /peer/disconnect?public_key=xTIB9P5E...

Response 200:
{
  "status": "ok",
  "message": "peer removed from server"
}
```

Server-side behavior:

- extract user_id from JWT.sub
- find peer by user_id + public_key
- remove peer from kernel WireGuard interface via wgctrl
- update peers table status to disconnected

### PUT /peer/replace-key

Replace a peer's public key. Uses `public_key` query parameter to select the
existing peer to replace.

```
Auth: Bearer <jwt>
Request: PUT /peer/replace-key?public_key=OLD_KEY...
{
  "public_key": "NEW_KEY..."
}

Response 200:
{
  "status": "ok",
  "message": "peer public key updated"
}
```

Server-side behavior:

- extract user_id from JWT.sub
- find peer by user_id + old public_key from query param
- update the peer's public_key in peers table
- update the peer in kernel WireGuard interface via wgctrl

## Admin Endpoints

### PUT /admin/user/:id

Modify per-user settings (peer limit, status).

```
Auth: Bearer <admin-jwt>
Request:
{
  "max_peers": 10,         // optional, new peer limit
  "status": "suspended"     // optional, active / suspended
}

Response 200:
{
  "status": "ok",
  "user": {
    "id": "uuid",
    "student_id": "2024001234",
    "max_peers": 10,
    "status": "suspended"
  }
}
```

Server-side behavior:

- extract target user_id from URL param
- update only the fields present in request body
- effective immediately, no server restart needed
- reads from DB on next peer/register call for that user

### POST /admin/invite

Generate a new invite code.

```
Auth: Bearer <admin-jwt>
Request:
{
  "max_uses": 3,
  "expires_days": 30
}

Response 200:
{
  "status": "ok",
  "invite_code": "SCNET-K3M9-W7P2",
  "expires_at": "2026-05-27T00:00:00Z"
}
```

Server-side behavior:

- generate a unique invite code
- insert into invite_codes table with admin's user_id as created_by
- return the code and expiry time

### GET /admin/me

Verify that the current JWT belongs to an admin. Used by SPA bootstrap.

```
Auth: Bearer <jwt>
Request: none

Response 200:
{
  "status": "ok",
  "admin": {
    "id": "uuid",
    "student_id": "admin",
    "role": "admin",
    "status": "active"
  }
}

Response 401:
{
  "error": "invalid or expired token",
  "code": "AUTH_FAILED"
}

Response 403:
{
  "error": "admin role required",
  "code": "FORBIDDEN"
}
```

Server-side behavior:

- extract user_id from JWT.sub
- query users table, verify role == "admin"
- return 403 if role is not admin

### GET /admin/summary

Dashboard overview with aggregate counts.

```
Auth: Bearer <admin-jwt>
Request: none

Response 200:
{
  "status": "ok",
  "summary": {
    "users_total": 120,
    "users_active": 112,
    "users_suspended": 8,
    "peers_active": 54,
    "peers_disconnected": 17,
    "peers_revoked": 3,
    "invites_total": 40,
    "invites_available": 12,
    "invites_expired": 6
  }
}
```

Server-side behavior:

- count users grouped by status
- count peers grouped by status (active / disconnected / revoked)
- count invites with derived state (available = max_uses > use_count AND NOT expired, expired = expires_at < now)
- all counts are database-level aggregates

### GET /admin/users

List all users with server-side pagination, search, and filters.

```
Auth: Bearer <admin-jwt>
Request: GET /admin/users?page=1&page_size=20&student_id=2024&status=active&role=user&sort_by=created_at&sort_order=desc

Response 200:
{
  "status": "ok",
  "items": [
    {
      "id": "uuid",
      "student_id": "2024001234",
      "role": "user",
      "display_name": "Alice",
      "invite_id": "uuid",
      "max_peers": 5,
      "status": "active",
      "created_at": "2026-04-28T10:00:00Z",
      "updated_at": "2026-04-28T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 120
  }
}
```

Query parameters:

- `page` (int, default 1)
- `page_size` (int, default 20, max 100)
- `student_id` (string, partial match / LIKE)
- `status` (string, exact: active | suspended | deleted)
- `role` (string, exact: user | admin)
- `sort_by` (string: created_at | student_id | status, default created_at)
- `sort_order` (string: asc | desc, default desc)

Server-side behavior:

- build WHERE clauses from query filters
- ORDER BY sort_by sort_order
- LIMIT page_size OFFSET (page-1)*page_size
- also COUNT total matching rows for pagination
- password field MUST NOT be included in response

### GET /admin/user/:id

Get a single user's full details (preload for edit dialog or detail page).

```
Auth: Bearer <admin-jwt>
Request: GET /admin/user/:id

Response 200:
{
  "status": "ok",
  "user": {
    "id": "uuid",
    "student_id": "2024001234",
    "role": "user",
    "display_name": "Alice",
    "email": "alice@example.com",
    "phone": "13800138000",
    "wechat": "alice_wechat",
    "telegram": "alice_tg",
    "invite_id": "uuid",
    "max_peers": 5,
    "status": "active",
    "created_at": "2026-04-28T10:00:00Z",
    "updated_at": "2026-04-28T10:00:00Z"
  }
}

Response 404:
{
  "error": "user not found",
  "code": "USER_NOT_FOUND"
}
```

Server-side behavior:

- extract user ID from URL param
- query users table by primary key
- return 404 if not found
- password field MUST NOT be included in response

### GET /admin/user/:id/peers

List all peers belonging to a specific user.

```
Auth: Bearer <admin-jwt>
Request: GET /admin/user/:id/peers?status=active

Response 200:
{
  "status": "ok",
  "items": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "public_key": "base64key...",
      "assigned_ip": "10.100.0.42",
      "status": "active",
      "last_seen": "2026-04-28T10:00:00Z",
      "created_at": "2026-04-28T10:00:00Z",
      "updated_at": "2026-04-28T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 1
  }
}
```

Query parameters:

- `status` (string, optional filter: active | disconnected | revoked)
- `page` (int, default 1)
- `page_size` (int, default 20)

### GET /admin/peers

List all peers across all users with server-side pagination and filters.

```
Auth: Bearer <admin-jwt>
Request: GET /admin/peers?page=1&page_size=20&status=active&student_id=2024&public_key=xTIB

Response 200:
{
  "status": "ok",
  "items": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "student_id": "2024001234",
      "public_key": "base64key...",
      "assigned_ip": "10.100.0.42",
      "status": "active",
      "last_seen": "2026-04-28T10:00:00Z",
      "created_at": "2026-04-28T10:00:00Z",
      "updated_at": "2026-04-28T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 54
  }
}
```

Query parameters:

- `page` (int, default 1)
- `page_size` (int, default 20, max 100)
- `status` (string, exact: active | disconnected | revoked)
- `student_id` (string, partial match via JOIN with users table)
- `public_key` (string, partial match / LIKE)
- `sort_by` (string: created_at | last_seen | assigned_ip, default created_at)
- `sort_order` (string: asc | desc, default desc)

Server-side behavior:

- JOIN peers with users to include student_id in response
- build WHERE clauses from filters
- ORDER BY sort_by sort_order
- LIMIT page_size OFFSET (page-1)*page_size
- COUNT total matching rows

### POST /admin/peer/:id/disconnect

Force-disconnect a peer: remove from wg0 and mark status as disconnected.

```
Auth: Bearer <admin-jwt>
Request: POST /admin/peer/:id/disconnect

Response 200:
{
  "status": "ok",
  "message": "peer disconnected"
}

Response 404:
{
  "error": "peer not found",
  "code": "PEER_NOT_FOUND"
}
```

Server-side behavior:

- find peer by ID
- remove peer from kernel WireGuard interface via wgctrl
- update peers.status to "disconnected"
- return 404 if peer not found

### POST /admin/peer/:id/revoke

Revoke a peer: remove from wg0 and mark status as revoked (permanent).

```
Auth: Bearer <admin-jwt>
Request: POST /admin/peer/:id/revoke

Response 200:
{
  "status": "ok",
  "message": "peer revoked"
}

Response 404:
{
  "error": "peer not found",
  "code": "PEER_NOT_FOUND"
}
```

Server-side behavior:

- find peer by ID
- remove peer from kernel WireGuard interface via wgctrl
- update peers.status to "revoked"
- return 404 if peer not found

### GET /admin/invites

List all invite codes with server-side pagination and filters.

```
Auth: Bearer <admin-jwt>
Request: GET /admin/invites?page=1&page_size=20&code=SCNET&state=available

Response 200:
{
  "status": "ok",
  "items": [
    {
      "id": "uuid",
      "code": "SCNET-K3M9-W7P2",
      "created_by": "uuid",
      "used_by": "uuid",
      "max_uses": 3,
      "use_count": 1,
      "expires_at": "2026-05-27T00:00:00Z",
      "created_at": "2026-04-28T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 40
  }
}
```

Query parameters:

- `page` (int, default 1)
- `page_size` (int, default 20, max 100)
- `code` (string, partial match / LIKE)
- `state` (string, derived filter: available | used_up | expired)

Server-side behavior:

- `state` is not a stored column; it is derived at query time:
  - `available`: max_uses > use_count AND (expires_at IS NULL OR expires_at > NOW())
  - `used_up`: max_uses <= use_count
  - `expired`: expires_at IS NOT NULL AND expires_at <= NOW() AND max_uses > use_count
- `used_by` is a foreign key to users(id); return as-is (null if unused)

---

## Admin Error Codes

Additional error codes applicable to admin endpoints:

| Code | HTTP Status | Meaning |
|------|------------|---------|
| FORBIDDEN | 403 | JWT role is not "admin" |
| USER_NOT_FOUND | 404 | User does not exist |
| PEER_NOT_FOUND | 404 | Peer does not exist |
| INVALID_STATUS | 400 | Invalid user status value (must be active or suspended) |
| INVALID_CONTACT | 400 | Contact field validation failed |
| INVALID_PASSWORD | 400 | Password change request does not meet policy |

---

## System Endpoints

### GET /health

Health check.

```
Auth: none
Request: none

Response 200:
{
  "status": "ok",
  "db": "connected",
  "wg": "active",
  "peers": 42
}
```

### GET /version

Server version.

```
Auth: none
Request: none

Response 200:
{
  "version": "1.0.0-dev",
  "commit": "abc1234"
}
```

## JWT Specification

### Payload

```json
{
  "sub": "user-uuid",
  "student_id": "2024001234",
  "role": "user",
  "iat": 1714233600,
  "exp": 1714319999
}
```

- Algorithm: HMAC-SHA256
- Expiry: 24 hours
- Secret: server-side key, never exposed
- `role`: `"user"` | `"admin"` — admin endpoints require `role: "admin"`
- Peer binding: server resolves user -> peer by querying peers table on
  user_id, not from JWT claims

### JWT Flow

```
1. POST /auth/login -> receives JWT
2. All subsequent requests include Authorization: Bearer <jwt>
3. Middleware extracts user_id from sub claim
4. If JWT expires, client re-authenticates via POST /auth/login
```

## Error Codes

| Code | HTTP Status | Meaning |
|------|------------|---------|
| AUTH_FAILED | 401 | Wrong student_id or password |
| FORBIDDEN | 403 | Account or role is not permitted for this operation |
| INVALID_INVITE | 400 | Invite code invalid or expired |
| INVALID_CONTACT | 400 | Contact field validation failed |
| ALREADY_EXISTS | 409 | Student ID already registered |
| PEER_EXISTS | 409 | User already has an active peer with this key |
| TOO_MANY_PEERS | 409 | User has reached max_peers limit |
| NO_ACTIVE_PEER | 404 | No active peer found for user |
| INVALID_KEY | 400 | Malformed WireGuard public key |
| INVALID_PASSWORD | 400 | Password change request does not meet policy |
| RATE_LIMITED | 429 | Too many requests |
| INTERNAL_ERROR | 500 | Unexpected server error |

## Rate Limiting

- Login: 5 attempts per student_id per minute
- Register: 10 attempts per IP per hour
- Peer operations: 30 requests per JWT per minute

## Address Discovery Endpoint

The `/api/server-info` endpoint lives on **Cloudflare Pages** (not the Go server).
It serves as an address-discovery rendezvous so that the CLI and SPA can locate
the backend server when it is behind dynamic NAT (e.g., Lucky STUN).

Base URL: `https://<cf-pages-domain>/api/server-info`

### GET /api/server-info

Retrieve the current server address. Public, no authentication required.

```
Auth: none
Request: none

Response 200:
{
  "api_url": "https://xxx.lucky.example.com:18080/api/v1",
  "wg_endpoint": "xxx.lucky.example.com:51820",
  "updated_at": "2026-04-28T12:00:00Z"
}

Response 503 (no server has reported yet):
{
  "error": "server not available",
  "code": "SERVER_UNAVAILABLE"
}
```

### POST /api/server-info

Server-side address report. Requires shared secret authentication.

```
Auth: Bearer <DISCOVERY_SECRET>
Request:
{
  "api_url": "https://xxx.lucky.example.com:18080/api/v1",
  "wg_endpoint": "xxx.lucky.example.com:51820"
}

Response 200:
{
  "status": "ok"
}

Response 401:
{
  "error": "unauthorized",
  "code": "AUTH_FAILED"
}
```

**Client consumption rules:**

- CLI: fetches `GET /api/server-info` → uses `api_url` for API calls, `wg_endpoint` for WireGuard tunnel. `--server-url` flag overrides both.
- SPA: fetches `GET /api/server-info` at boot → sets `api_url` as `baseUrl` for all `apiFetch` calls. `?api=` URL parameter overrides.

**Server reporting rules:**

- `scnet-server` POSTs its current address to this endpoint every 30 seconds.
- The CF Pages Function stores the latest report in memory (no durable storage needed — if server restarts it re-reports).
- `DISCOVERY_SECRET` is a shared secret configured identically on the server and in the CF Pages Function environment.

## Versioning

- Current: v1
- URL prefix: `/api/v1/`
- Breaking changes require new version prefix
- Backward-compatible additions do not bump version
