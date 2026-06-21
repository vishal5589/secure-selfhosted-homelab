# Containerised services

Each service is its own Compose project on an isolated network (ADR-002).
Cross-project calls use explicit host LAN routing, not container-name DNS.

| Folder | Service | Security relevance |
|---|---|---|
| `gluetun/` | Fail-closed VPN egress | Control C3 — deny-by-default egress |
| `seerr/` | Approval gateway | Control C6 — control-plane boundary |
| `n8n/` | Workflow / notification router | Control C7 — typed-event validation |

**Usage:** in any folder, `cp ../../.env.example .env`, fill in real values, then
`docker compose up -d`. `.env` and `config/` are git-ignored — no secrets are tracked.

> Images are tagged `:latest` here for readability; pin to digests in production.
