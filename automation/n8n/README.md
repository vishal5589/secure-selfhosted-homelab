# n8n workflows

Event-driven notification routing (ADR-005). The engine subscribes to gateway
webhooks, switches on event type, and fans out to the correct audience.

**Exports are intentionally not committed verbatim.** Workflow JSON can embed
credential references, webhook URLs, and internal hostnames. Before any export
is published it must be scrubbed (URLs, tokens, host IPs → placeholders).

Design notes:
- Validate event `type` before routing; drop unexpected payloads (control C7).
- Batch outbound notifications to respect downstream message limits.
- Sanitise user-supplied strings (titles with non-ASCII / control chars) before
  building notification payloads to avoid malformed-request failures.
