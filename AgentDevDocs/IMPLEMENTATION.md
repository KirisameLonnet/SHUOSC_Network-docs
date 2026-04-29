# Implementation Guide

## Purpose

This document translates the architecture into concrete implementation steps.
Agents should use this to answer:

- what file to create first
- what interface to implement against
- what dependency to import
- what counts as done for each unit

## Project Structure

```
SHUOSC_Network/
  server/                          # Go module: github.com/shuosc/scnet-server
    cmd/scnet-server/main.go
    internal/
      api/
        router.go                  # gin/chi route registration
        middleware.go              # JWT extraction + RequireAdmin
        auth_handler.go            # POST /auth/register, /auth/login
        me_handler.go              # GET /me, PUT /me, PUT /me/password,
                                   #   GET /me/peers
        peer_handler.go            # POST /peer/register, GET /peer/config,
                                   #   DELETE /peer/disconnect, PUT /peer/replace-key
        admin_handler.go           # Admin endpoints (users, peers, invites,
                                   #   summary, WG key rotation, WG toggle)
        health_handler.go          # GET /health, GET /version
      auth/
        service.go                 # AuthService interface + impl
        password.go                # bcrypt helpers
      account/
        service.go                 # AccountService (self profile, password,
                                   #   peer usage / peer list)
      peer/
        manager.go                 # PeerManager interface + impl (wgctrl)
        ipam.go                    # IPAM interface + impl
      admin/
        service.go                 # AdminService (summary aggregation,
                                   #   forced peer disconnect/revoke)
      model/
        user.go                    # User struct
        peer.go                    # Peer struct
        invite.go                  # InviteCode struct
      store/
        db.go                      # PostgreSQL connection pool
        user_store.go              # UserStore interface + impl
                                   #   (add List, CountByStatus)
        peer_store.go              # PeerStore interface + impl
                                   #   (add List, CountByStatus,
                                   #    FindByUserIDList)
        invite_store.go            # InviteStore interface + impl
                                   #   (add List, CountByState)
    admin-panel/                    # Quasar SPA build output for Go embed
      dist/spa/                     # copied build artifacts from server-panel/scnet-panel/
    migrations/
      001_init.sql
    config/
      config.go                    # Config loading from env/yaml
    go.mod
    go.sum

  cli/                             # Go module: github.com/shuosc/scnet-cli
    cmd/scnet/main.go
    internal/
      cmd/
        root.go                    # cobra root command
        connect.go                 # scnet connect
        disconnect.go              # scnet disconnect
        status.go                  # scnet status
        export.go                  # scnet export-config
        version.go                 # scnet version
      auth/
        service.go                 # AuthService (HTTP client to server API)
      tunnel/
        device.go                  # wireguard-go Device wrapper
        uapi.go                    # UAPI configuration helper
      store/
        credential.go              # ~/.scnet/credentials.json read/write
        state.go                   # ~/.scnet/state.json read/write
        config.go                  # ~/.scnet/config.json read/write
    platform/
      route_darwin.go
      route_linux.go
      route_windows.go
      route_freebsd.go
      route_openbsd.go
      dns_darwin.go
      dns_linux.go
      dns_windows.go
    go.mod
    go.sum

  server-panel/scnet-panel/        # Quasar (Vue 3) SPA for user portal + admin console
    package.json
    quasar.config.ts               # Vite + Quasar config, output -> ../../server/admin-panel/dist/spa/
    tsconfig.json
    public/
    src/
      App.vue
      boot/
        axios.ts                    # Axios setup + JWT interceptor
        auth.ts                     # Shared auth bootstrap + role redirect
      css/
        quasar.variables.sass       # Brand colors only
      layouts/
        AuthLayout.vue              # Login page shell
        UserLayout.vue              # Self-service portal shell
        AdminLayout.vue             # QLayout + QDrawer + QPageContainer
      pages/
        LoginPage.vue               # Shared login entry
        UserHomePage.vue            # Account status + peer usage
        MyPeersPage.vue             # Own peer table + disconnect/replace-key
        AccountSettingsPage.vue     # Contact info + password change
        DashboardPage.vue           # Admin summary cards + system status
        UsersPage.vue               # Admin user table + QDialog edit
        UserDetailPage.vue          # Admin user info card + peer table
        PeersPage.vue               # Admin peer table + disconnect/revoke
        InvitesPage.vue             # Admin invite table + create invite QDialog
        WGManagementPage.vue        # Admin WG key rotation + enable/disable
        NotFoundPage.vue
      components/
        shell/
          UserDrawer.vue            # User portal navigation
          AdminDrawer.vue           # QDrawer navigation
          AdminToolbar.vue          # QToolbar + QTabs
        me/
          PeerUsageCard.vue         # Current quota usage
          MyPeerTable.vue           # Self peer table wrapper
          ReplaceKeyDialog.vue      # Own peer key rotation dialog
          ContactForm.vue           # Contact info form
          PasswordForm.vue          # Password change form
        dashboard/
          SummaryCards.vue          # Admin stat cards
          SystemStatusCard.vue      # Health/version card
        users/
          UserTable.vue             # Admin user QTable wrapper
          UserEditDialog.vue        # QDialog + QForm + QInput
        peers/
          PeerTable.vue             # Admin peer QTable wrapper
          PeerActions.vue           # Admin disconnect/revoke QBtn
        invites/
          InviteTable.vue           # Admin invite QTable wrapper
          CreateInviteDialog.vue    # QDialog + QForm
        common/
          StatusBadge.vue           # QBadge wrapper
          TableToolbar.vue          # QToolbar with filters
      router/
        index.ts
        routes.ts
      stores/
        auth.ts                     # Pinia: JWT, current user identity + role
        me.ts                       # Pinia: self profile + peer usage
        myPeers.ts                  # Pinia: self peer list + actions
        dashboard.ts                # Pinia: admin summary + health
        users.ts                    # Pinia: admin user list, edit
        peers.ts                    # Pinia: admin peer list, actions
        invites.ts                  # Pinia: admin invite list, create
        wg.ts                       # Pinia: WG server status, rotate key, toggle
      services/
        me.ts                       # Typed self-service API client (Axios)
        admin.ts                    # Typed API client (Axios)
      types/
        api.ts                      # API response types
        user.ts                     # Self-service user/account types
        admin.ts                    # Admin-specific types

  docs/
    AGENTS.md
    ARCHITECTURE.md
    API_CONTRACT.md
    SECURITY.md
    WORKBOARD.md
    IMPLEMENTATION.md
```

## Web Portal Route Map

The embedded SPA must use explicit route guards so an LLM implementation does
not invent ad-hoc navigation.

| Route | Layout | Guard | Primary data | Redirect behavior |
|------|--------|-------|--------------|-------------------|
| `/login` | `AuthLayout` | guest-only | none | if already authenticated, call `GET /me` and redirect by role |
| `/app/` | `UserLayout` | `requiresAuth`, `requiresUser` | `GET /me` | admin users redirect to `/admin/` |
| `/app/peers` | `UserLayout` | `requiresAuth`, `requiresUser` | `GET /me/peers` | admin users redirect to `/admin/` |
| `/app/settings` | `UserLayout` | `requiresAuth`, `requiresUser` | `GET /me` | admin users redirect to `/admin/` |
| `/admin/` | `AdminLayout` | `requiresAuth`, `requiresAdmin` | `GET /admin/summary`, `/health`, `/version` | normal users redirect to `/app/` |
| `/admin/users` | `AdminLayout` | `requiresAuth`, `requiresAdmin` | `GET /admin/users` | normal users redirect to `/app/` |
| `/admin/users/:id` | `AdminLayout` | `requiresAuth`, `requiresAdmin` | `GET /admin/user/:id`, `GET /admin/user/:id/peers` | normal users redirect to `/app/` |
| `/admin/peers` | `AdminLayout` | `requiresAuth`, `requiresAdmin` | `GET /admin/peers` | normal users redirect to `/app/` |
| `/admin/invites` | `AdminLayout` | `requiresAuth`, `requiresAdmin` | `GET /admin/invites` | normal users redirect to `/app/` |
| `/admin/wg` | `AdminLayout` | `requiresAuth`, `requiresAdmin` | `GET /admin/wg` | normal users redirect to `/app/` |
| `/:catchAll(.*)*` | `AuthLayout` or none | none | none | render `NotFoundPage.vue` |

## Web Portal State Matrix

Each page must implement these UI states explicitly.

| Page | Loading | Empty | Error | Forbidden / status edge |
|------|---------|-------|-------|--------------------------|
| `LoginPage` | disable submit + show loading | n/a | inline auth error | if role mismatch after login, redirect by `GET /me` |
| `UserHomePage` | skeleton cards | no peers message | retry card + Notify | suspended/deleted user logs out and returns to `/login` |
| `MyPeersPage` | table skeleton | “no peers yet” CTA | retry table load | hide admin-only actions |
| `AccountSettingsPage` | form skeleton | n/a | inline field + Notify | role/quota/status fields are read-only / absent |
| `DashboardPage` | summary skeleton | zero counts still render cards | health/API retry | non-admin redirected away |
| `UsersPage` | table skeleton | “no users match filter” | retry table load | non-admin redirected away |
| `UserDetailPage` | metadata + table skeleton | “user has no peers” | retry detail load | 404 page if target user absent |
| `PeersPage` | table skeleton | “no peers match filter” | retry table load | non-admin redirected away |
| `InvitesPage` | table skeleton | “no invite codes yet” | retry table load | non-admin redirected away |
| `WGManagementPage` | status + form skeleton | n/a | inline error + Notify | non-admin redirected away |

Shared behavior rules:

- all mutation actions use optimistic button-disabled state plus Quasar `Loading`
- all API failures emit Quasar `Notify`
- all destructive actions (`disconnect`, `revoke`) require a confirmation dialog
- a `401 AUTH_FAILED` response clears JWT and redirects to `/login`
- a `403 FORBIDDEN` response triggers role/status re-check through `GET /me` when safe

## Go Module Configuration

### server/go.mod

```
module github.com/shuosc/scnet-server

go 1.22

require (
    github.com/gin-gonic/gin          v1.x  // or go-chi/chi
    github.com/golang-jwt/jwt/v5      v5.x
    golang.zx2c4.com/wireguard/wgctrl  v0.x
    golang.zx2c4.com/wireguard          v0.x
    golang.org/x/crypto               v0.x  // bcrypt
    github.com/jackc/pgx/v5           v5.x  // PostgreSQL driver
)
```

### cli/go.mod

```
module github.com/shuosc/scnet-cli

go 1.22

require (
    github.com/spf13/cobra            v1.x
    golang.zx2c4.com/wireguard        v0.x  // wireguard-go, UAPI
    golang.zx2c4.com/wireguard/wgctrl  v0.x  // for wgtypes key generation
)
```

## Core Interface Definitions

### Server: AuthService

```go
// internal/auth/service.go

type AuthService interface {
    // Register creates a new user. inviteCode must be valid and unexpired.
    Register(ctx context.Context, studentID, password, inviteCode string) (*model.User, error)

    // Login verifies credentials and returns a JWT.
    Login(ctx context.Context, studentID, password string) (token string, expiresAt time.Time, err error)

    // ValidateToken parses and validates a JWT, returning user identity.
    ValidateToken(tokenString string) (*Claims, error)
}

type Claims struct {
    jwt.RegisteredClaims
    StudentID string `json:"student_id"`
    Role      string `json:"role"` // "user" | "admin"
}

var (
    ErrInvalidCredentials = errors.New("invalid student_id or password")
    ErrStudentIDTaken     = errors.New("student_id already registered")
    ErrInvalidInvite      = errors.New("invalid or expired invite code")
    ErrTokenExpired       = errors.New("token expired")
)
```

### Server: AccountService

```go
// internal/account/service.go

type AccountService interface {
    // GetMe returns the current user's profile plus peer quota usage.
    GetMe(ctx context.Context, userID string) (*SelfProfile, error)

    // UpdateMe updates self-service profile/contact fields only.
    UpdateMe(ctx context.Context, userID string, fields SelfUpdateFields) (*model.User, error)

    // ChangePassword verifies the current password and writes a new bcrypt hash.
    ChangePassword(ctx context.Context, userID, currentPassword, newPassword string) error

    // ListMyPeers returns the current user's peers for the web portal.
    ListMyPeers(ctx context.Context, userID string, params SelfPeerListParams) (*store.PeerListResult, error)
}

type SelfProfile struct {
    User       *model.User
    ActivePeers int
    MaxPeers    int
}

type SelfUpdateFields struct {
    DisplayName *string
    Email       *string
    Phone       *string
    Wechat      *string
    Telegram    *string
}

type SelfPeerListParams struct {
    Page     int
    PageSize int
    Status   string // exact filter
}
```

### Server: PeerManager

```go
// internal/peer/manager.go

type PeerManager interface {
    // AddPeer registers a peer with the kernel WireGuard interface.
    AddPeer(ctx context.Context, userID string, pubKey wgtypes.Key) (*PeerRegistration, error)

    // RemovePeer removes a peer by its public key from the kernel WG interface.
    RemovePeer(ctx context.Context, userID, pubKey string) error

    // GetPeerConfig returns the configuration for a specific public key.
    GetPeerConfig(ctx context.Context, userID, pubKey string) (*PeerRegistration, error)

    // ReplaceKey updates a peer's public key (key rotation).
    ReplaceKey(ctx context.Context, userID, oldPubKey, newPubKey string) error

    // RotateServerKey replaces the server WG keypair, preserving all peers.
    RotateServerKey(newKey wgtypes.Key) (wgtypes.Key, error)

    // WGEnabled reports whether the WG interface is up.
    WGEnabled() (bool, error)

    // SetWGEnabled enables or disables the WG interface.
    SetWGEnabled(enabled bool) error
}

type PeerRegistration struct {
    PublicKey  string   `json:"public_key"`
    AssignedIP string   `json:"assigned_ip"`
    AllowedIPs []string `json:"allowed_ips"`
    Endpoint   string   `json:"endpoint"`
    ServerKey  string   `json:"server_pubkey"`
}

var (
    ErrTooManyPeers   = errors.New("peer limit reached")
    ErrPeerExists     = errors.New("peer with this key already exists")
    ErrNoActivePeer   = errors.New("no active peer found")
)
```

### Server: IPAM

```go
// internal/peer/ipam.go

type IPAM interface {
    // Allocate assigns the next available IP from the pool.
    Allocate(ctx context.Context) (string, error)

    // Release returns an IP to the pool.
    Release(ctx context.Context, ip string) error
}
```

### Server: Store Interfaces

```go
// internal/store/user_store.go

type UserStore interface {
    Create(ctx context.Context, user *model.User) error
    FindByStudentID(ctx context.Context, studentID string) (*model.User, error)
    FindByID(ctx context.Context, id uuid.UUID) (*model.User, error)
    UpdatePassword(ctx context.Context, userID uuid.UUID, passwordHash string) error
    Update(ctx context.Context, userID uuid.UUID, fields UserUpdateFields) error
    UpdateStatus(ctx context.Context, userID uuid.UUID, status string) error
    List(ctx context.Context, params UserListParams) (*UserListResult, error)
    Count(ctx context.Context) (int, error)
    CountByStatus(ctx context.Context) (map[string]int, error)
}

type UserUpdateFields struct {
    DisplayName *string // nil = don't update
    Email       *string
    Phone       *string
    Wechat      *string
    Telegram    *string
    MaxPeers    *int    // nil = don't update
    Status      *string // nil = don't update
}

type UserListParams struct {
    Page      int
    PageSize  int
    StudentID string // LIKE filter
    Status    string // exact filter
    Role      string // exact filter
    SortBy    string // created_at | student_id | status
    SortOrder string // asc | desc
}

type UserListResult struct {
    Items      []*model.User
    Total      int
}
```

```go
// internal/store/peer_store.go

type PeerStore interface {
    Create(ctx context.Context, peer *model.Peer) error
    FindActiveByUserID(ctx context.Context, userID uuid.UUID) ([]*model.Peer, error)
    FindByID(ctx context.Context, id uuid.UUID) (*model.Peer, error)
    CountActiveByUserID(ctx context.Context, userID uuid.UUID) (int, error)
    UpdateStatus(ctx context.Context, peerID uuid.UUID, status string) error
    UpdatePublicKey(ctx context.Context, peerID uuid.UUID, newPubKey string) error
    List(ctx context.Context, params PeerListParams) (*PeerListResult, error)
    CountByStatus(ctx context.Context) (map[string]int, error)
    ListByUserID(ctx context.Context, userID uuid.UUID, params PeerListByUserParams) (*PeerListResult, error)
}

type PeerListParams struct {
    Page      int
    PageSize  int
    Status    string
    StudentID string // via JOIN
    PublicKey string // LIKE filter
    SortBy    string // created_at | last_seen | assigned_ip
    SortOrder string // asc | desc
}

type PeerListByUserParams struct {
    Page     int
    PageSize int
    Status   string // exact filter
}

type PeerListResult struct {
    Items      []*PeerWithStudent // includes student_id from join
    Total      int
}

type PeerWithStudent struct {
    *model.Peer
    StudentID string
}
```

```go
// internal/store/invite_store.go

type InviteStore interface {
    Create(ctx context.Context, invite *model.InviteCode) error
    FindByCode(ctx context.Context, code string) (*model.InviteCode, error)
    FindByID(ctx context.Context, id uuid.UUID) (*model.InviteCode, error)
    MarkUsed(ctx context.Context, inviteID uuid.UUID, usedBy uuid.UUID) error
    List(ctx context.Context, params InviteListParams) (*InviteListResult, error)
    CountByState(ctx context.Context) (map[string]int, error)
}

type InviteListParams struct {
    Page     int
    PageSize int
    Code     string // LIKE filter
    State    string // derived: available | used_up | expired
}

type InviteListResult struct {
    Items []*model.InviteCode
    Total int
}
```

### Model Structs

```go
// internal/model/user.go

type User struct {
    ID         uuid.UUID
    StudentID  string
    Password   string      // bcrypt hash
    Role       string      // "user" | "admin"
    InviteID   uuid.UUID
    DisplayName string
    Email      string
    Phone      string
    Wechat     string
    Telegram   string
    MaxPeers   int
    Status     string      // "active" | "suspended" | "deleted"
    CreatedAt  time.Time
    UpdatedAt  time.Time
}
```

```go
// internal/model/peer.go

type Peer struct {
    ID         uuid.UUID
    UserID     uuid.UUID
    PublicKey  string
    AssignedIP string
    Status     string      // "active" | "disconnected" | "revoked"
    LastSeen   *time.Time
    CreatedAt  time.Time
    UpdatedAt  time.Time
}
```

```go
// internal/model/invite.go

type InviteCode struct {
    ID        uuid.UUID
    Code      string
    CreatedBy uuid.UUID
    UsedBy    *uuid.UUID
    MaxUses   int
    UseCount  int
    ExpiresAt *time.Time
    CreatedAt time.Time
}
```

## Configuration Schema

```yaml
# config.yaml

server:
  port: 8080

database:
  host: localhost
  port: 5432
  user: scnet
  password: ${SCNET_DB_PASSWORD}
  name: scnet
  sslmode: disable

wireguard:
  interface: wg_scnet
  listen_port: 51820
  private_key: ${SCNET_WG_PRIVATE_KEY}   # server's own WG private key
  subnet: 10.100.0.0/24

jwt:
  secret: ${SCNET_JWT_SECRET}
  expiry_hours: 24

invite:
  default_max_uses: 1
  default_expiry_days: 30

admin:
  student_id: admin
  password: ${SCNET_ADMIN_PASSWORD}
```

```go
// server/config/config.go

type Config struct {
    Server    ServerConfig    `yaml:"server"`
    Database  DatabaseConfig  `yaml:"database"`
    WireGuard WireGuardConfig `yaml:"wireguard"`
    JWT       JWTConfig       `yaml:"jwt"`
    Invite    InviteConfig    `yaml:"invite"`
    Admin     AdminConfig     `yaml:"admin"`
}

type ServerConfig struct {
    Port int `yaml:"port"` // default: 8080
}

type DatabaseConfig struct {
    Host     string `yaml:"host"`
    Port     int    `yaml:"port"`
    User     string `yaml:"user"`
    Password string `yaml:"password"`
    Name     string `yaml:"name"`
    SSLMode  string `yaml:"sslmode"`
}

type WireGuardConfig struct {
    Interface  string `yaml:"interface"`   // default: "wg_scnet"
    ListenPort int    `yaml:"listen_port"` // default: 51820
    PrivateKey string `yaml:"private_key"` // server WG private key
    Subnet     string `yaml:"subnet"`       // default: "10.100.0.0/24"
}

type JWTConfig struct {
    Secret      string `yaml:"secret"`
    ExpiryHours int    `yaml:"expiry_hours"` // default: 24
}

type InviteConfig struct {
    DefaultMaxUses    int `yaml:"default_max_uses"`    // default: 1
    DefaultExpiryDays int `yaml:"default_expiry_days"` // default: 30
}

type AdminConfig struct {
    StudentID string `yaml:"student_id"` // default: "admin"
    Password  string `yaml:"password"`   // required for bootstrap
}
```

Environment variable substitution: `${VAR_NAME}` in YAML values resolved at load time.

## Database Migration

```sql
-- migrations/001_init.sql

-- +migrate Up
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE users (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id VARCHAR(20) UNIQUE NOT NULL,
    password   VARCHAR(255) NOT NULL,
    role       VARCHAR(8) DEFAULT 'user',
    invite_id  UUID,
    display_name VARCHAR(64),
    email      VARCHAR(255),
    phone      VARCHAR(32),
    wechat     VARCHAR(64),
    telegram   VARCHAR(64),
    max_peers  INT DEFAULT 5,
    status     VARCHAR(16) DEFAULT 'active',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE invite_codes (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code       VARCHAR(32) UNIQUE NOT NULL,
    created_by UUID,                              -- nullable for bootstrap
    used_by    UUID,
    max_uses   INT DEFAULT 1,
    use_count  INT DEFAULT 0,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE peers (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    UUID REFERENCES users(id) NOT NULL,
    public_key VARCHAR(64) UNIQUE NOT NULL,
    assigned_ip INET NOT NULL,
    status     VARCHAR(16) DEFAULT 'active',
    last_seen  TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_student_id ON users(student_id);
CREATE INDEX idx_peers_user_id ON peers(user_id);
CREATE INDEX idx_peers_public_key ON peers(public_key);
CREATE INDEX idx_peers_user_status ON peers(user_id, status);
CREATE INDEX idx_invite_codes_code ON invite_codes(code);

-- +migrate Down
DROP TABLE IF EXISTS peers;
DROP TABLE IF EXISTS invite_codes;
DROP TABLE IF EXISTS users;
```

Use `golang-migrate/migrate` or `pressly/goose` for migration execution.

### Migration Plan For Phase 1 Web Portal

Use one of these two strategies. Do not mix them.

#### Strategy A: Greenfield repo, no schema applied yet

- keep all currently documented fields in `001_init.sql`
- create all three tables plus contact/profile columns in the first migration
- preferred for an LLM doing first-time repository scaffolding

#### Strategy B: Existing Phase 1 schema already applied

Create a new migration `002_user_profile_fields.sql`:

```sql
-- +migrate Up
ALTER TABLE users ADD COLUMN display_name VARCHAR(64);
ALTER TABLE users ADD COLUMN email VARCHAR(255);
ALTER TABLE users ADD COLUMN phone VARCHAR(32);
ALTER TABLE users ADD COLUMN wechat VARCHAR(64);
ALTER TABLE users ADD COLUMN telegram VARCHAR(64);

-- optional check constraints for basic hygiene
ALTER TABLE users ADD CONSTRAINT users_phone_len CHECK (phone IS NULL OR char_length(phone) <= 32);
ALTER TABLE users ADD CONSTRAINT users_display_name_len CHECK (display_name IS NULL OR char_length(display_name) <= 64);

-- +migrate Down
ALTER TABLE users DROP CONSTRAINT IF EXISTS users_phone_len;
ALTER TABLE users DROP CONSTRAINT IF EXISTS users_display_name_len;
ALTER TABLE users DROP COLUMN IF EXISTS telegram;
ALTER TABLE users DROP COLUMN IF EXISTS wechat;
ALTER TABLE users DROP COLUMN IF EXISTS phone;
ALTER TABLE users DROP COLUMN IF EXISTS email;
ALTER TABLE users DROP COLUMN IF EXISTS display_name;
```

Migration execution order:

1. run schema migration
2. deploy code that reads nullable contact fields safely
3. enable `/me` and portal settings UI
4. backfill optional contact data manually if needed

LLM rule:

- if the workspace has no applied migrations, implement Strategy A only
- if `001_init.sql` is already considered deployed, implement Strategy B only

## Error Handling Pattern

### Layer Responsibilities

```
HTTP Handler:
  - parse request, validate input format
  - call Service
  - map Service errors to HTTP status codes + error format
  - do NOT access database directly

Service:
  - execute business logic
  - call Store for persistence
  - return domain errors (not HTTP errors)

Store:
  - execute SQL queries
  - return sql.ErrNoRows -> wrap as domain error
  - do NOT return HTTP status codes
```

### Handler error mapping

```go
// internal/api/peer_handler.go

func (h *PeerHandler) RegisterPeer(c *gin.Context) {
    userID := c.GetString("user_id") // from JWT middleware

    var req struct {
        PublicKey string `json:"public_key"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, apiError("INVALID_KEY", "malformed public key"))
        return
    }

    result, err := h.peerManager.AddPeer(c.Request.Context(), userID, pubKey)
    switch {
    case errors.Is(err, peer.ErrTooManyPeers):
        c.JSON(409, apiError("TOO_MANY_PEERS", err.Error()))
    case errors.Is(err, peer.ErrPeerExists):
        c.JSON(409, apiError("PEER_EXISTS", err.Error()))
    case err != nil:
        log.Printf("peer register error: %v", err)
        c.JSON(500, apiError("INTERNAL_ERROR", "internal server error"))
    default:
        c.JSON(200, gin.H{"status": "ok", "peer_config": result})
    }
}

func apiError(code, message string) gin.H {
    return gin.H{"error": message, "code": code}
}
```

## Logging Convention

- Use `log/slog` (Go 1.21+ structured logging)
- INFO: request/response, peer add/remove, config changes
- WARN: rate limit hits, auth failures, reconciliation drift
- ERROR: database failures, wgctrl failures, unexpected panics

Example:

```go
slog.Info("peer registered",
    "user_id", userID,
    "public_key", pubKey[:8]+"...",
    "assigned_ip", ip,
)
slog.Warn("peer limit reached", "user_id", userID, "max", maxPeers)
slog.Error("wgctrl add peer failed", "error", err, "public_key", pubKey[:8]+"...")
```

## Phase 1 Detailed Route

### Step 1.0: Server WG Keypair Initialization

Files: `server/cmd/scnet-server/main.go`, `server/config/config.go`
Depends on: nothing

What to do:

1. Create `go.mod`, install dependencies
2. Implement `config.go`: load YAML, resolve env vars
3. In `main.go`: load config, generate/load server WG private key, write to `/etc/scnet/server.key`

Done when: `scnet-server --config config.yaml` starts and prints loaded config.

### Step 1.1: Database & Migration

Files: `server/internal/store/db.go`, `server/migrations/001_init.sql`
Depends on: config loading

What to do:

1. Implement `store/db.go`: pgx connection pool
2. Add migration runner in `main.go`
3. Run `001_init.sql` to create tables

Done when: all three tables exist, indexes confirmed, migration is idempotent.

### Step 1.2: Model & Store Layer

Files: `server/internal/model/*.go`, `server/internal/store/*_store.go`
Depends on: database

What to do:

1. Implement model structs (user.go, peer.go, invite.go)
2. Implement UserStore (Create, FindByStudentID, FindByID, Update,
   UpdatePassword, CountByStatus)
3. Implement PeerStore (Create, FindActiveByUserID, CountActiveByUserID,
   UpdateStatus, UpdatePublicKey, ListByUserID)
4. Implement InviteStore (Create, FindByCode, MarkUsed, List, CountByState)

Done when: each store method is unit-tested against a real PostgreSQL instance (testcontainers or local).

### Step 1.3: Auth Service + Account Service

Files: `server/internal/auth/service.go`, `server/internal/auth/password.go`,
`server/internal/account/service.go`
Depends on: UserStore, PeerStore

What to do:

1. Implement bcrypt hashing (cost=12)
2. Implement Register: validate invite code, hash password, create user (role=user)
3. Implement Login: find user, compare bcrypt, issue JWT (HMAC-SHA256, include role in claims)
4. Implement ValidateToken: parse JWT, return Claims
5. Implement GetMe: return self profile + active peer count / max_peers
6. Implement UpdateMe: allow self-service updates to display_name/contact fields only
7. Implement ChangePassword: verify current password, write new bcrypt hash
8. Implement ListMyPeers: list only peers owned by JWT.sub

Done when: Register returns ErrInvalidInvite for bad invite codes, Login returns JWT for valid credentials, JWT validates correctly, and self-service profile/password APIs operate only on the current user.

**Multi-peer note**: `GET /peer/config`, `DELETE /peer/disconnect`, and
`PUT /peer/replace-key` all require a `public_key` query parameter to select
which peer to operate on. The client stores its own `public_key` in
`credentials.json` and passes it on reconnect/disconnect.

**Web-portal note**: The server can show the user's public key and server-side
peer parameters, but it cannot recover a lost client private key or generate a
complete client config containing that private key.

**Admin bootstrap**: On first run with empty `users` table, create admin directly via
UserStore (bypasses Register since admin needs no invite code):

```go
// main.go startup
if count, _ := userStore.Count(ctx); count == 0 {
    hash, _ := bcrypt.GenerateFromPassword([]byte(cfg.Admin.Password), 12)
    admin := &model.User{
        StudentID: cfg.Admin.StudentID,
        Password:  string(hash),
        Role:      "admin",
        MaxPeers:  1,
    }
    userStore.Create(ctx, admin)
    slog.Info("admin user bootstrapped", "student_id", admin.StudentID)
}
```

The first invite code is created by admin via `POST /admin/invite`. The
`invite_codes.created_by` column must be nullable for bootstrap.

### Step 1.4: Peer Manager & IPAM

Files: `server/internal/peer/manager.go`, `server/internal/peer/ipam.go`
Depends on: PeerStore, UserStore, wgctrl

What to do:

1. Implement IPAM: allocate from 10.100.0.0/24 pool, track used IPs
2. Implement AddPeer: check max_peers, allocate IP, wgctrl add peer, insert DB row
3. Implement RemovePeer: wgctrl remove peer, update DB status
4. Implement GetPeerConfig: query active peer, build PeerRegistration
5. Implement ReplaceKey: update DB, wgctrl update

Done when: AddPeer succeeds for first peer, returns TOO_MANY_PEERS at limit, RemovePeer persists status change.

### Step 1.5: HTTP Handlers & Middleware

Files: `server/internal/api/`
Depends on: AuthService, AccountService, PeerManager

What to do:

1. Implement JWT middleware (extract token, validate, set user_id + role in context)
2. Implement admin-only middleware (reject if ctx role != "admin")
3. Implement auth_handler.go (Register, Login)
4. Implement me_handler.go (GetMe, UpdateMe, ChangePassword, ListMyPeers)
5. Implement peer_handler.go (RegisterPeer, GetConfig, Disconnect, ReplaceKey)
6. Implement admin_handler.go (`/admin/me`, `/admin/summary`, `/admin/users`,
   `/admin/user/:id`, `/admin/user/:id/peers`, `/admin/peers`,
   `/admin/peer/:id/disconnect`, `/admin/peer/:id/revoke`, `PUT /admin/user/:id`,
   `GET /admin/invites`, `POST /admin/invite`) — routes wrapped in admin middleware
7. Implement health_handler.go (Health, Version)
8. Wire routes in router.go, including SPA fallbacks for `/app/*` and `/admin/*`

Done when: all API_CONTRACT.md endpoints return documented responses for both success and error cases.

### Step 1.6: Reconciliation Goroutine

Files: `server/internal/peer/manager.go` (or separate reconciler.go)
Depends on: PeerManager, PeerStore, wgctrl

What to do:

1. Every 5 minutes: list peers from wgctrl, list active peers from DB
2. Peers in DB but not in wg_scnet: log warning, attempt re-add to wg_scnet
3. Peers in wg_scnet but not in DB: log warning, remove from wg_scnet
4. Update last_seen for peers with recent traffic

Done when: reconciliation runs without errors in a long-running test.

## CLI Client: Key Design Notes

### Tun Device Lifecycle

```
wireguard-go creates the TUN device when Device.Up() is called.
The TUN device is destroyed when the Device is closed (via Close() or process exit).
No manual TUN cleanup needed on Linux/macOS/Windows.

Platform-specific route helpers only need to add/remove routes before/after Device.Up()/Close().
```

### Credential Storage

```go
// internal/store/credential.go

type Credential struct {
    StudentID  string `json:"student_id"`
    PublicKey  string `json:"public_key"`
    PrivateKey string `json:"private_key"` // WireGuard base64
    JWT        string `json:"jwt"`
    ServerURL  string `json:"server_url"`
    ExpiresAt  time.Time `json:"expires_at"` // JWT expiry, cached for quick check
}

func SaveCredential(path string, cred *Credential) error
func LoadCredential(path string) (*Credential, error)
func DeleteCredential(path string) error
```

File permissions: `credentials.json` must be written with `0600`.

### JWT Expiry Check

```go
func (c *Credential) JWTExpired() bool {
    return time.Now().After(c.ExpiresAt)
}
```

Client checks before calling `GET /peer/config?public_key=...`. If expired,
calls `POST /auth/login` first.

### Connect Flow (Pseudocode)

```go
func connect(serverURL, studentID, password string) error {
    cred, err := store.LoadCredential()
    if err == nil && !cred.JWTExpired() {
        // Reconnect path
        config, err := auth.FetchConfig(cred.JWT, cred.PublicKey, serverURL)
        if err != nil {
            // Config fetch failed, might need full re-connect
            cred = nil
        } else {
            return tunnel.Connect(cred.PrivateKey, config)
        }
    }

    // Full first-connection flow
    jwt, err := auth.Login(serverURL, studentID, password)
    if err != nil {
        return fmt.Errorf("login failed: %w", err)
    }

    privKey, _ := wgtypes.GeneratePrivateKey()
    config, err := auth.RegisterPeer(serverURL, jwt, privKey.PublicKey().String())
    if err != nil {
        return fmt.Errorf("peer register failed: %w", err)
    }

    cred = &store.Credential{
        StudentID:  studentID,
        PublicKey:  privKey.PublicKey().String(),
        PrivateKey: privKey.String(),
        JWT:        jwt,
        ServerURL:  serverURL,
        ExpiresAt:  time.Now().Add(24 * time.Hour),
    }
    store.SaveCredential(cred)

    return tunnel.Connect(privKey, config)
}
```

### Export Config

```go
func exportConfig(cred *Credential, config *tunnel.PeerConfig) string {
    return fmt.Sprintf(`[Interface]
PrivateKey = %s
Address = %s/24

[Peer]
PublicKey = %s
Endpoint = %s
AllowedIPs = %s
PersistentKeepalive = 25
`,
        cred.PrivateKey,
        config.AssignedIP,
        config.ServerKey,
        config.Endpoint,
        strings.Join(config.AllowedIPs, ", "),
    )
}
```

## Cross-Platform Build

## Phase 1 Scope Boundary

Agents implementing Phase 1 MUST NOT implement:

- P2P hole punching or STUN (Phase 6)
- Academic acceleration proxy pool (Phase 4)
- GUI clients (Phase 5)
- Cross-platform CI (Phase 3)
- Client CLI (Phase 2)
- Any endpoint not listed in API_CONTRACT.md

Phase 1 deliverables are server control plane plus the embedded web portal:
- Server starts, accepts auth, manages WireGuard peers, serves self-service and
  admin endpoints, and serves the Quasar SPA under `/app/` and `/admin/`.
- Test backend endpoints by curl and verify both user portal and admin console
  against the same server. No client CLI needed in Phase 1.

```makefile
# Makefile (project root)
.PHONY: build-server build-cli build-cli-all

build-server:
    cd server && GOOS=linux GOARCH=amd64 go build -o ../bin/scnet-server ./cmd/scnet-server

build-cli:
    cd cli && go build -o ../bin/scnet ./cmd/scnet

CLI_TARGETS = \
    darwin/amd64   darwin/arm64   \
    linux/amd64    linux/arm64    linux/riscv64 linux/loong64 \
    windows/amd64  windows/arm64  \
    freebsd/amd64  freebsd/arm64  \
    openbsd/amd64  openbsd/arm64

build-cli-all:
    @mkdir -p bin
    @for t in $(CLI_TARGETS); do \
        GOOS=$${t%/*} GOARCH=$${t#*/} go build -o bin/scnet-$${t%/*}-$${t#*/} ./cli/cmd/scnet; \
    done
```
