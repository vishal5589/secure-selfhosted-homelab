# Scheduled reporting jobs

PowerShell jobs (run via Task Scheduler on the Windows host) that pull usage
analytics and publish periodic digests to the community.

Security notes:
- Read-only against the monitoring datastore.
- No credentials inline — pull from a local, git-ignored config.
- Scripts are published as sanitised references, not with live endpoints/keys.
