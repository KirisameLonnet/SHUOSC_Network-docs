# Security Model

## Purpose

This document defines the security invariants and threat model for scnet.

Any change that weakens these invariants must be explicitly justified.

## Core Invariants

### 1. Server Must Never Store Client Private Keys

- WireGuard keypairs are generated client-side
- Only public keys are transmitted to the server
- The `peers` table has no `private_key` column
- If a client loses its private key, it generates a new keypair and
  re-registers

### 2. WireGuard Protocol Layer Is Unmodified

- Noise Protocol Framework handshake remains unchanged
- Curve25519 key exchange is standard
- Transport encryption is standard WireGuard
- No custom fields or extensions are added to WG frames

### 3. Authentication Is Separate from WireGuard

- Student ID + password authentication happens over HTTPS before WG tunnel
  establishment
- JWT is the session credential for the HTTP API
- WireGuard tunnel establishment uses standard public key exchange
- No authentication data passes through the WG tunnel

### 4. Password Storage

- Passwords are hashed with bcrypt (cost=12) before storage
- Plaintext passwords are never logged
- No password recovery: lost passwords require invite code holder to arrange
  reset
- Passwords are independent of university credentials

### 5. Web Portal Boundaries

- The web portal may display public keys and server-side peer metadata only
- The web portal must never display, persist, or reconstruct client private keys
- Role and account-status checks must be enforced server-side, not only by SPA route guards
- Sensitive self-service mutations (`PUT /me/password`, peer rotation, peer disconnect)
  must always re-check the current DB-backed user status

## Threat Model

| Threat | Mitigation | Priority |
|--------|-----------|----------|
| Server compromise exposes client keys | Server stores no client private keys | Critical |
| Credential brute force | bcrypt(cost=12), rate limiting: 5 login attempts per student_id per minute | High |
| JWT theft | 24-hour expiry, transmitted over HTTPS only, not stored in browser localStorage | High |
| Unauthorized API access | JWT required on all endpoints except /auth/register, /auth/login, /health, and /version | High |
| Man-in-the-middle | TLS 1.3 for HTTP API, WireGuard encryption for tunnel | High |
| Peer spoofing | public_key unique constraint, user-to-peer binding enforced in DB | Medium |
| Peer state drift after server restart | Reconciliation goroutine: periodic DB-to-WG state sync | Medium |
| Invite code abuse | max_uses limit, expiry date, audit trail of who created/used | Medium |
| Stale peer accumulation | last_seen timestamp, auto-cleanup of peers inactive > 7 days | Medium |

## Data Classification

| Data | Sensitivity | Storage | Encryption |
|------|------------|---------|------------|
| Student ID | PII | DB, plaintext | DB-level encryption |
| Password hash | Auth secret | DB, bcrypt | bcrypt (cost=12) |
| JWT secret | Auth secret | Server config | File permissions 0600 |
| WireGuard private key | Tunnel secret | Client filesystem | File permissions 0600, OS keychain where available |
| WireGuard public key | Public | DB, plaintext | None needed |
| Peer IP allocation | Operational | DB | None needed |
| Traffic metadata | Operational | None (not stored) | N/A |

## HTTPS Configuration

- Minimum TLS version: 1.3
- Recommended: Let's Encrypt with auto-renewal
- Self-signed certificates acceptable for development only

## Key Rotation

- Server WireGuard keypair: manual rotation, requires all clients to fetch new
  config
- Client WireGuard keypair: `PUT /peer/replace-key` or re-register with new
  keypair
- JWT secret: manual rotation, invalidates all existing sessions

## Reporting

Security issues should be reported to the project maintainers, not filed as
public issues.
