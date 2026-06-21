# Design Decisions

Short architecture decision records (ADRs). Each captures a real trade-off, the options considered, and why I chose what I chose. These exist because *the reasoning* is what's portable to a new context — not the specific tools.

---

## ADR-001 — Remote access via mesh VPN, not port-forwarding

**Context.** Users need to reach the stack from outside the home network.

**Options.**
- (a) Port-forward services through the home router with a reverse proxy + TLS.
- (b) Private mesh VPN (Tailscale); no inbound ports.

**Decision.** (b).

**Why.** Port-forwarding creates a permanent, internet-exposed listener — something to scan, fingerprint, and brute-force, plus the ongoing burden of patching that exposed surface. A mesh VPN reduces the public attack surface to **zero open ports**. Access becomes an authentication problem (who's on the mesh) rather than a network-exposure problem.

**Trade-off accepted.** Every remote user must enrol a device on the mesh — slightly higher onboarding friction in exchange for removing an entire class of internet-facing risk. Worth it.

---

## ADR-002 — Separate Compose projects per service

**Context.** A dozen services need to run on one Docker host.

**Options.**
- (a) One monolithic `docker-compose.yml` on a shared default network.
- (b) Separate `docker-compose.yml` per service, isolated networks.

**Decision.** (b).

**Why.** A single flat network gives every container implicit reachability to every other — convenient, but it means a compromise anywhere is a compromise everywhere (T4). Per-project isolation makes the blast radius of a container compromise its own segment. Cross-service traffic then has to be *explicit*, which is exactly the property I want in a security boundary.

**Trade-off accepted.** Cross-project services can't resolve each other by container name; they communicate over explicit host LAN routing. More verbose, but intentional — and it forced clarity about which services actually need to talk to each other.

---

## ADR-003 — Fail-closed egress for the download client

**Context.** One component generates outbound traffic that should never use the host's real connection unprotected.

**Options.**
- (a) Run the client directly; rely on a system VPN being up.
- (b) Route the client through a dedicated VPN container with a kill-switch.

**Decision.** (b) — gluetun sidecar.

**Why.** Option (a) fails *open*: if the VPN drops, the client keeps running and leaks. Option (b) fails *closed*: if the tunnel drops, egress is blocked. "Deny by default on failure" is the same principle as fail-secure access control, applied to the network layer.

**Trade-off accepted.** Slightly more complex container wiring; occasional reconnect handling. Acceptable for a guaranteed no-leak failure mode.

---

## ADR-004 — Hybrid hosting: containers + native Windows services

**Context.** Some services run great in Docker; others have first-class native Windows builds with better hardware/filesystem integration.

**Decision.** Containerise the network-facing, easily-isolated services (bot, gateway, workflow engine, indexer manager, VPN). Run the media server, orchestrators, and monitoring natively on Windows where they integrate best with storage and GPU.

**Why.** Dogmatically containerising everything would have meant fighting the platform for the media server and download/storage paths. The pragmatic split keeps the *security-sensitive, network-facing* tier in cheap-to-isolate containers while letting the heavy I/O tier run where it performs.

**Trade-off accepted.** Two operational surfaces to maintain (Docker + Windows). The Windows host is hardened separately at the OS level and is acknowledged as a single point of failure in `SECURITY.md`.

---

## ADR-005 — Event-driven notifications via a workflow engine

**Context.** Multiple events (approved, denied, available, issue reported) need to fan out to different audiences without manual admin work.

**Decision.** Route events through a workflow engine (n8n) that switches on event type and dispatches to the right destination.

**Why.** Hard-coding notification logic into each component would scatter it and make changes brittle. A central router decouples *what happened* from *who needs to know*, so adding a new notification path is a workflow change, not a code change across services. It also gives one place to validate event payloads before acting on them (supports C7).

**Trade-off accepted.** The workflow engine becomes a dependency in the path; mitigated by keeping it internal-only and its logic version-controlled.

---

## ADR-006 — Approval gate as a hard control-plane boundary

**Context.** Untrusted-ish community users need to drive automation that consumes resources and writes to shared storage.

**Decision.** Users can submit requests; **no automation runs until a request is explicitly approved.**

**Why.** This is the "ask vs. do" boundary. Letting requests auto-fulfil would hand resource-consuming, storage-writing actions to people outside the trust boundary. The gate keeps a human decision between an outside request and any privileged action (supports C6).

**Trade-off accepted.** Approval is manual — the one intentional manual step in an otherwise fully automated pipeline. That's by design: it's the control I *don't* want to automate away.
