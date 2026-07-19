# Infrastructure: Voite

## Provider

- Hetzner Cloud
- Single VM, no load balancer, no multi-node
- OS: Ubuntu LTS (containers absorb most host dependencies; AppArmor is lower-friction with Docker bind mounts than AlmaLinux's default SELinux enforcing)

## VM Specs

- CPX line (dedicated vCPU, x86), matching the local development architecture
- Chosen over CAX (ARM) or CX (shared vCPU) despite CPX's June 2026 price increase: the VM only runs during short, high-stakes windows rather than continuously, so predictable performance during a live event outweighs the standing cost difference, and x86 avoids any architecture-specific build or debugging surprises versus local dev
- Target: 4 vCPU / 8GB RAM, NVMe storage (standard across tiers)
- Local disk mostly holds the OS, Docker images, and rotated logs; Postgres data lives on a separate Volume, so disk size isn't a sizing driver
- Check Hetzner's current pricing page for the exact CPX tier matching 4 vCPU / 8GB before locking in, given the recent price adjustment

## Provisioning: Terraform + Cloud-Init

- Terraform (`hcloud` provider) creates the VM and the external Cloud Firewall
- Cloud-Init runs once, at first boot: OS updates, Docker install, SSH hardening (key-only, no password login)
- Terraform state kept local, excluded from git; periodically backed up to the same Hetzner Object Storage bucket used for Postgres backups
- Hetzner API token scoped to this project, kept out of git

## Configuration management: Ansible

- Handles everything after first boot: Docker Compose file, Caddy config, re-running the security toolkit
- Cloud-Init isn't re-runnable, so ongoing changes go through Ansible instead
- Secrets encrypted at rest via Ansible Vault; decrypted to local files on deploy and mounted into containers via Docker Compose's file-based `secrets:`, not passed as plain env vars

## Runtime: Docker Compose

- Three services: app, Postgres, Caddy
- Only Caddy exposes ports to the host; app and Postgres stay internal
- Containers run with `restart: unless-stopped`
- ulimits raised in Compose for the app container; Docker's default file descriptor limit is too low for thousands of idle SSE connections
- App image built by GitHub Actions and pushed to GitHub Container Registry (`ghcr.io`); Ansible pulls by tag and recreates the container, previous tag kept on the box for rollback
- A lightweight GitHub Actions pipeline gates the build: tests must pass before an image is pushed to `ghcr.io`
- Docker's `json-file` logging driver capped (`max-size`/`max-file`) so logs rotate instead of filling the disk
- Deploy freeze: no image rollouts, Ansible runs, or host changes while an event is Live. Recreating the app container drops every SSE connection and can interrupt a grace-period tally (the app recovers it on restart, but spectators see a stall). Deploys happen between events and are verified against the health endpoint

## Container specs

- App (Spring Boot): memory limit ~3.5GB, 2 CPUs. JVM heap set explicitly (`-XX:MaxRAMPercentage=70` or an explicit `-Xmx`), not left at Java's default 25%, since the app deliberately holds the live snapshot and connection state in memory. Gets the most CPU headroom of the three, to absorb the reconnect-herd spike
- Postgres: memory limit ~1.5GB, 1 CPU. `shared_buffers` set explicitly (~25% of the container's memory, ~384-512MB) rather than left at Postgres's much smaller out-of-the-box default
- Caddy: memory limit ~512MB, 1.5 CPUs. Memory stays light, but the reconnect herd lands on Caddy first: 10,000 TLS handshakes in a burst are the most CPU-intensive moment on the box, so Caddy gets real CPU headroom rather than a token share. Caddy's ulimits are raised like the app's; as the proxy it holds two sockets per spectator (client plus upstream), roughly twice the descriptors
- Memory limits (~5.5GB combined) stay well under the 8GB VM, since memory limits are hard. CPU limits intentionally sum past 4 vCPU: they are ceilings, not reservations, and the two peaks (Caddy's handshake burst, the app's fan-out) do not coincide
- Set via `deploy.resources.limits` in the Compose file; confirm the Compose CLI version in use actually honors this outside Swarm mode
- Each service defines a `healthcheck:` (app: hit `/actuator/health`; Postgres: readiness check; Caddy: basic reachability), so Docker can detect and restart a hung-but-not-crashed container automatically, rather than relying solely on StatusCake noticing from outside

## Reverse proxy: Caddy

- Automatic HTTPS via Let's Encrypt
- Streams SSE responses, no buffering
- Forwards real client IPs to the app

## Edge / CDN: Cloudflare

- Proxied for static spectator assets only, on a separate subdomain
- API/SSE subdomain bypasses Cloudflare entirely: DNS-only, direct to the Hetzner IP
- Reason: Cloudflare's free/Pro tiers cap idle connections around 100s and can buffer dynamic responses, both risky for a long-lived SSE stream

## Database: Postgres

- Self-managed, runs as a Docker Compose service
- Data directory on a dedicated Hetzner Volume, separate from the OS disk
- Backups via periodic `pg_dump`, compressed, pushed to Hetzner Object Storage; more frequent during a live event
- No point-in-time recovery, restores go to the last dump; acceptable given event data is largely disposable
- Restore procedure tested periodically, not just written down
- Major-version upgrades by dump-and-restore between events (the persistent dataset is one roster plus at most a pending card, so this stays trivial); minor versions by bumping the image tag. Never during a live window

## Failure & recovery

- Decision: losing the VM mid-event must not end the night. Target: spectators reconnected within ~15 minutes, accepting loss of votes since the last dump (the RPO is the dump interval, raised in frequency during events per Backups)
- Warm standby on event nights only: Hetzner bills hourly, so on the day of an event Terraform brings up a second VM (same Ansible run, images pre-pulled, latest dump restored, app container stopped). Failover: restore the newest dump, start the app, flip DNS. Torn down after the event; standing cost is near zero
- DNS: 60-second TTL on both subdomains, so the flip lands within the reconnect window; spectator `EventSource` retries find the standby with no user action, and the snapshot-on-connect contract makes the switch invisible beyond the gap itself
- The failover is a written runbook and rehearsed like the restore procedure, not just provisioned

## Load testing

- The scenario is the reconnect herd (see [TECHNICAL_SPEC.md](./TECHNICAL_SPEC.md)), not steady state: 10,000 `EventSource` clients connecting within ~10 seconds, each receiving the snapshot, followed by a vote burst inside one voting window
- Generator: a throwaway high-core Hetzner VM (e.g. 16 vCPU) created by Terraform and destroyed after; cost measured in cents. It must speak TLS to the real Caddy, since the handshakes are the point
- Tooling: k6, or a small custom harness; SSE is just a long-lived GET, so no exotic client is needed
- Pass criteria tied to the functional targets: p95 EM-action-to-phone under 2s and tally-to-phone under 5s during the herd, zero container restarts, no descriptor or memory alerts
- Run before the first live event and after any change to the push path, Caddy config, or container limits

## Security verification toolkit

- Nmap: confirm the firewall works and no backend port has leaked to the public
- Lynis: host hardening audit (SSH config, kernel params, file permissions)
- Docker Bench for Security: Docker daemon and container config audit
- testssl.sh: run against the direct-to-origin subdomain, not the Cloudflare one
- Run after provisioning and after major changes, not continuously

## Patch strategy

- Unattended-upgrades: non-reboot security patches only
- Reboot-requiring patches applied manually via Ansible, on a schedule the operator controls (e.g. a few days before an event), rebooting at a safe time and verifying the app comes back up

## DDoS

- No protection beyond Hetzner's built-in L3/4 protection and the Cloud Firewall
- Realistic risk is a legitimate traffic spike (QR-scan stampede, reconnect herd), already handled by the app's in-memory snapshot design
- Application-layer abuse (fake votes) is a separate, accepted risk

## Monitoring

- External uptime check via StatusCake free tier (permits commercial use, unlike UptimeRobot's free tier), alerting via email/webhook
- Checks the app's health endpoint
- Spring Boot Actuator `/actuator/health` and `/actuator/metrics` for on-demand connection count, heap, and thread visibility during a live event, no extra infrastructure
- Exposed via Caddy but credential-protected (not IP-restricted), so it's reachable on the go rather than only from a fixed location
- Netdata on the host: per-second visibility into CPU, file descriptors, sockets, and per-container memory during a live event, with threshold alerts to the same webhook as StatusCake. Chosen over node_exporter plus a hosted Grafana to keep it one moving part on one box
- The app exposes its SSE connection count as a Micrometer gauge; Netdata's Prometheus collector scrapes `/actuator/prometheus`, putting connection count on the same dashboard and alert path as host metrics
- Three alerts that matter during a live window: open descriptors above 80% of the limit (app or Caddy), app container memory above 90% of its limit, and connection count collapsing while the health check still passes (the failure external uptime checks cannot see)
- Rotated logs are tarred and pushed to the same Object Storage bucket as the dumps after each event, so a dead VM does not take the evidence with it
