# AGENTS.md

## Purpose

This repository is intended to be developed heavily with LLM agents.

This file defines how agents should work in this codebase so that future
iterations stay aligned with the project's technical constraints.

## Project Identity

`scnet` is a campus VPN service, not a general-purpose networking toolkit.

It is:

- an invite-only WireGuard-based VPN for university students
- a cross-platform service with server (Linux) and clients (CLI + GUI)
- centrally coordinated with experimental P2P fallback
- authenticated by student ID + password, not raw WireGuard key exchange

It is not:

- a replacement for Tailscale or commercial VPN products
- a general-purpose proxy or API gateway
- an anonymity network

## Non-Negotiable Constraints

All agents must preserve these rules:

- WireGuard protocol layer must remain unmodified; changes are restricted to
  the control plane (authentication, peer management, configuration delivery)
- server must never store client private keys; key generation is always
  client-side
- server operates on Linux kernel WireGuard via wgctrl; no userspace WG on
  server
- server WireGuard key rotation and administrative toggle (enable/disable)
  are admin-only operations exposed via the HTTP API; the server never
  persists the private key beyond its in-memory copy
- backend container deployment is API-only. Admin/user web panels are deployed
  separately on Cloudflare Pages; Containerfile/Podman definitions must not
  build or serve SPA assets by default
- external communication must stay split across distinct planes:
  browser -> Cloudflare Pages over HTTPS, SPA/CLI -> backend API over HTTPS,
  client -> WireGuard endpoint over UDP, and discovery -> Cloudflare Pages
  `/api/server-info` over HTTPS
- these planes must not be merged: Cloudflare Pages must not carry WireGuard
  UDP, `wg_endpoint` must not be represented as an API URL, and `api_url` must
  not be used as a WireGuard endpoint
- client WireGuard uses wireguard-go UAPI; no dependency on kernel WG on
  desktop clients
- the authentication boundary is at the HTTP API layer, not inside the WG
  tunnel

## Primary Source Documents

Before making changes, agents should read:

1. [ARCHITECTURE.md](ARCHITECTURE.md) for system-level design
2. [IMPLEMENTATION.md](IMPLEMENTATION.md) for concrete code structure and interfaces
3. [API_CONTRACT.md](API_CONTRACT.md) for endpoint specifications
4. [WORKBOARD.md](WORKBOARD.md) for current implementation order and exit
   criteria
5. [SECURITY.md](SECURITY.md) for threat model and security invariants

If a proposed change conflicts with those documents, update the documents first
or explain the intended deviation explicitly.

## Architectural Boundary

Agents must preserve the separation of control plane and data plane.

Agents must also preserve the separation of communication planes exposed to
users and clients.

### Control plane

This includes:

- HTTP API endpoints
- student ID + password authentication
- JWT session management
- peer registration and deregistration
- IP address allocation (IPAM)
- invite code management

Rules:

- stateless where possible
- JWT used for all authenticated endpoints
- database is the authoritative record for users, peers, and invite codes

### Data plane

This includes:

- WireGuard tunnel establishment
- packet routing (direct / academic acceleration / P2P)
- DNS configuration
- traffic statistics collection

Rules:

- WireGuard protocol unchanged
- routing decisions are server-side for the standard path
- P2P path is experimental and must not break the standard path

## Communication Plane Boundary

The deployment model has four distinct communication planes:

- frontend delivery plane: browser <-> Cloudflare Pages over HTTPS
- control-plane API: SPA/CLI <-> backend API over HTTPS
- WireGuard data plane: client <-> WireGuard endpoint over UDP
- discovery plane: SPA/CLI/backend <-> Cloudflare Pages `/api/server-info`
  over HTTPS

Rules:

- do not collapse these planes into a single origin or transport unless docs
  are explicitly changed first
- do not route WireGuard UDP through Cloudflare Pages
- do not publish `wg_endpoint` as an HTTPS URL
- do not use `api_url` as a WireGuard endpoint or tunnel address

## Deployment Compatibility Policy

Core frontend and backend code should remain deployment-agnostic wherever
practical.

Rules:

- prefer preserving the existing external contract instead of hard-coding one
  exposure method into the SPA or `scnet-server`
- supported deployment modes should differ by configuration and external
  adapters first, not by forking core business logic
- public direct API exposure, public split deployment, and Lucky-based forward
  exposure should be achievable without modifying core frontend/backend logic
- reverse HTTP exposure modes may require new adapter components
  (for example Worker/relay/agent code), but should not force incompatible
  changes into `scnet-server` or the SPA unless the contract itself changes
- when adding a new deployment mode, document which parts are:
  config-only, deployment-only, adapter-only, or core-source changes

## Server Constraints

Server-specific rules:

- Go single binary, no runtime dependencies beyond PostgreSQL
- HTTP framework: gin or chi
- WG control: golang.zx2c4.com/wireguard/wgctrl (netlink)
- platform: Linux only, kernel >= 5.6 for built-in WireGuard
- database: PostgreSQL, migrations managed in code
- backend image/build path includes only the Go binary and migrations; no npm
  or frontend build step belongs in the backend container by default

## Client Constraints

Client-specific rules:

- CLI: Go + cobra + wireguard-go, cross-platform (darwin/linux/windows/freebsd/openbsd, amd64/arm64/riscv64/loong64/mips64le)
- platform-specific code must use go:build tags
- configuration stored in ~/.scnet/ with 0600 permissions for sensitive files
- GUI clients (Phase 5): Tauri for desktop, native Kotlin/Swift for mobile

## Change Discipline

When making changes, agents should prefer small, auditable steps.

Good changes:

- add one API endpoint with its handler, model, and store logic
- fill one CLI command with its service layer
- fix one protocol-level issue with documented reasoning

Risky changes:

- modifying the WireGuard protocol layer
- changing the JWT payload structure without updating all consumers
- introducing platform-specific logic without go:build tags
- adding dependencies without evaluating cross-platform compatibility
- storing client private keys on server

## Documentation Discipline

If an agent changes any of the following, it should update docs in the same
change:

- API endpoint behavior or contract
- authentication flow
- data model (schema changes)
- WireGuard peer management logic
- client credential storage format
- platform support matrix
- deployment boundary (backend container vs Cloudflare Pages frontend)

## Preferred Output Style

When reporting changes, agents should be concrete and concise.

Prefer:

- what changed
- which constraint it satisfies
- what remains placeholder

Avoid:

- vague praise
- hand-wavy claims of completion
- summarizing the entire architecture in every response

## Review Checklist

Before calling a task done, ask:

- does it preserve the server-no-private-key invariant?
- does it keep the WireGuard protocol unmodified?
- does it match the API contract?
- does it preserve platform portability for the client?
- does it handle the error path, not just the happy path?
