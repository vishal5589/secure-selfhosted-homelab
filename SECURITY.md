# Security Model

This document is the core of the project. It describes the threat model for a self-hosted service stack and the controls put in place to mitigate the risks identified. It is written the way I would write an internal hardening record at work: assets, threats, controls, and residual risk.

---

## 1. Scope & assets

**What I'm defending:**

| Asset | Sensitivity | Notes |
|---|---|---|
| Host machines (Docker host + Windows app host) | High | Compromise = full lab control |
| Service API keys (per-service) | High | Enable control of automation components |
| Bot / webhook tokens | High | Enable impersonation and message injection |
| User roster & access tiers | Medium | Personal data of a small trusted group |
| Media server account | Medium | Sharing surface; account takeover risk |
| Home network | High | Lab shares a LAN with personal devices |

**Trust boundaries:**

1. **Internet ↔ home lab** — the most hostile boundary. Default posture: nothing inbound from the public internet.
2. **Container ↔ container** — containers from different Compose projects are treated as semi-trusted neighbours, not one flat trusted zone.
3. **Community user ↔ control plane** — users can *request*, but cannot *act*. The approval gate is the boundary.

---

## 2. Threat model

Threats considered, roughly in STRIDE order:

| # | Threat | Vector | Likelihood | Impact |
|---|---|---|---|---|
| T1 | Internet-facing service exploited | Exposed/port-forwarded listener | Med | High |
| T2 | Credential / token leak | Secret committed to git, or reused across services | Med | High |
| T3 | Egress / IP leak from download traffic | VPN tunnel drops, client keeps running | Med | Med |
| T4 | Lateral movement after container compromise | Flat Docker network, shared creds | Low | High |
| T5 | Malicious add-on / "free IPTV" supply chain | Untrusted source bundling malware or credential harvesters | Med | High |
| T6 | Unauthorised control-plane action by a community user | Request bot abused to trigger admin actions | Low | Med |
| T7 | Message-injection / spoofed webhooks | Forged events to the workflow engine | Low | Med |
| T8 | Stale shared access after a user leaves | Orphaned media-server share | Med | Low |

---

## 3. Controls

### C1 — Zero public attack surface *(mitigates T1)*
Remote access is provided through a **private mesh VPN (Tailscale)**, not port-forwarding. There are **no inbound ports exposed to the internet**, so there is no public listener to scan, fingerprint, or brute-force. Access requires being an authenticated node on the mesh.

*Residual risk:* mesh-account compromise. Mitigated by MFA on the identity provider and device authorisation.

### C2 — Secrets never touch the repo *(mitigates T2)*
- All secrets live in `.env` files that are **git-ignored**; only `.env.example` templates with placeholder values are committed.
- **No shared credentials** — each service gets its own narrowly-scoped API key, so one leak doesn't cascade.
- A **gitleaks GitHub Action** (`.github/workflows/secrets-scan.yml`) scans every push and pull request. A committed secret fails CI before it can merge.
- Local pre-commit scanning recommended for defence-in-depth (catch before push, not just before merge).

*This is why this repository can safely be public.*

### C3 — Fail-closed egress *(mitigates T3)*
The download client's traffic is routed through a **VPN container with a kill-switch (gluetun)**. The design is **deny-by-default**: if the tunnel drops, egress is blocked rather than silently falling back to the host's real connection. This is the network-egress equivalent of "fail secure" — the failure mode is *no traffic*, not *leaked traffic*. See [`docker/gluetun/`](./docker/gluetun/).

### C4 — Network segmentation *(mitigates T4)*
Services run in **separate Docker Compose projects on isolated networks**. Cross-project communication is done over explicit host LAN routing rather than implicit shared bridges. A container compromise is contained to its own network segment instead of having flat reachability to the entire stack. This also surfaced a real operational lesson (see §5).

### C5 — Supply-chain discipline *(mitigates T5)*
I deliberately **reject untrusted "free IPTV" / unvetted add-on sources.** These are a well-documented vector for malware, credential harvesting, and hostile binaries — convenience bundled with risk. Only vetted, maintained, pinned images are used. This is a direct application of supply-chain threat awareness to a home context, and it's a decision I can defend with reasoning rather than just policy.

### C6 — Human-in-the-loop control plane *(mitigates T6)*
Community users interact only through a request bot that can **request, not act**. Every request passes an **approval gate** before any automation runs. Users have no path to trigger privileged operations directly — the boundary between "ask" and "do" is explicit and enforced.

### C7 — Webhook trust *(mitigates T7)*
Internal webhooks between the gateway and the workflow engine stay on the internal/LAN path and are not publicly reachable (reinforced by C1). Event payloads are validated by type before routing, so malformed or unexpected events are dropped rather than processed.

### C8 — Access lifecycle hygiene *(mitigates T8)*
Membership in the community group and access to the media server should stay in sync. The plan (and partial implementation) reconciles the user roster against active shares so that **access is revoked when a user leaves** — closing the orphaned-share gap rather than letting access accumulate indefinitely.

---

## 4. Residual risks (honest accounting)

No system is risk-free; pretending otherwise is itself a red flag. Known residual risks:

- **Mesh-identity compromise** would grant network access (C1). Mitigated by MFA, not eliminated.
- **Host-OS compromise** is out of scope for the container controls — the Windows app host is a single point of failure for several native services and is hardened at the OS level separately.
- **Manual approval is a human control** — it depends on the admin exercising judgment. It gates automation but isn't a technical guarantee against social engineering.
- **Access-lifecycle automation (C8)** is partially implemented; until complete, orphaned-share cleanup is periodic rather than event-driven.

---

## 5. Operational lessons that shaped the design

Real findings from running this, which I keep as a record because the *reasoning* is the reusable part:

- **Container networking is not flat by default — and that's a feature.** Discovering that cross-Compose-project containers can't resolve each other by name pushed the design toward explicit, intentional routing instead of implicit trust. Segmentation became a deliberate property, not an accident.
- **"Convenient" media sources are a supply-chain decision.** The temptation to bolt on an untrusted source was reframed as: *would I run this unknown binary on a machine that shares a LAN with my personal devices?* The answer drove C5.
- **Secrets sprawl is the default failure mode of home labs.** Per-service `.env` files plus CI scanning turn "I'll remember to remove that key later" into a control that can't be forgotten.

---

## Reporting

This is a personal lab, but if you spot something in the *published configuration templates or documentation* that leaks information or models a control poorly, open an issue — I treat that as a valid finding.
