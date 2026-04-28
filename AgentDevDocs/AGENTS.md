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

## Server Constraints

Server-specific rules:

- Go single binary, no runtime dependencies beyond PostgreSQL
- HTTP framework: gin or chi
- WG control: golang.zx2c4.com/wireguard/wgctrl (netlink)
- platform: Linux only, kernel >= 5.6 for built-in WireGuard
- database: PostgreSQL, migrations managed in code

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
