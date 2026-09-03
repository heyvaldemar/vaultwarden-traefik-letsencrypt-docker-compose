# Vaultwarden + Traefik + Let's Encrypt on Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Contents

- [Why this stack?](#why-this-stack)
- [Prerequisites](#prerequisites)
- [Getting started](#getting-started)
- [Features](#features)
- [Supply chain trust](#supply-chain-trust)
- [Production checklist](#production-checklist)
- [Backups](#backups)
- [Testing](#testing)
- [Security Notes](#security-notes)
- [About the maintainer](#about-the-maintainer)

This repository deploys Vaultwarden (the lightweight Bitwarden-compatible server) behind Traefik with automatic Let's Encrypt TLS. One `docker compose up` away from a self-hosted password manager at `https://your-domain`, compatible with all official Bitwarden clients.

📙 Full narrative installation guide on the blog: [heyvaldemar.com/install-vaultwarden-using-docker-compose/](https://www.heyvaldemar.com/install-vaultwarden-using-docker-compose/).

## Why this stack?

| Need | This stack | Manual install | Bitwarden official | Other compose examples |
|------|-----------|----------------|--------------------|-----------------------|
| Ready to deploy in <5 min | ✅ | ❌ | ❌ heavy (MSSQL etc.) | Often |
| TLS via Let's Encrypt, auto-renewed | ✅ Traefik ACME built-in | Manual certbot | Manual | Rare |
| Tiny footprint (~50 MB RAM) | ✅ | ✅ | ❌ | ✅ |
| Works with official Bitwarden apps | ✅ | ✅ | ✅ | ✅ |
| Upstream images pinned by `sha256` digest | ✅ | N/A | N/A | Rare |
| Weekly pin-freshness check in CI | ✅ | N/A | N/A | Rare |
| CI-verified deployment on every push | ✅ /alive answers | N/A | N/A | Rare |

Two moving parts (Traefik + Vaultwarden). No Kubernetes prerequisites, no manual certificate management.

## Prerequisites

- **A Linux server** with a public IP. Vaultwarden is light: the smallest VPS works.
- **Docker Engine 24+ and Docker Compose 2.20+.**
- **A domain you control,** with two `A` records pointing at your server's public IP: one for Vaultwarden, one for the Traefik dashboard. DNS must propagate before deploy. Bitwarden clients require HTTPS, which this stack provides out of the box.
- **Ports 80 and 443 open** on the server's firewall.

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose
cd vaultwarden-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create vaultwarden-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: VAULTWARDEN_HOSTNAME, TRAEFIK_HOSTNAME, TRAEFIK_ACME_EMAIL,
#   TRAEFIK_BASIC_AUTH.

# 4. Deploy
docker compose -f vaultwarden-traefik-letsencrypt-docker-compose.yml -p vaultwarden up -d
```

Within a minute `https://${VAULTWARDEN_HOSTNAME}` serves the web vault with a fresh Let's Encrypt certificate. Register your account(s), then disable sign-ups (see the checklist).

### What success looks like

```bash
# Services healthy:
docker compose -f vaultwarden-traefik-letsencrypt-docker-compose.yml -p vaultwarden ps

# Liveness endpoint answers with a timestamp:
curl -fsS "https://${VAULTWARDEN_HOSTNAME}/alive"

# Traefik issued a certificate:
docker compose -p vaultwarden logs traefik | grep -i "adding certificate"
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet.
- **Mobile app refuses to connect.** The client requires a valid HTTPS URL: use the public hostname, never an IP.
- **`network vaultwarden-network not found`.** Step 2 was skipped.

### Apply `.env` or compose-file changes

```bash
docker compose -f vaultwarden-traefik-letsencrypt-docker-compose.yml -p vaultwarden up -d --force-recreate
```

## Features

- **Vaultwarden** latest stable (1.37.2), the Rust reimplementation of the Bitwarden server API; works with all official clients, browser extensions, and apps.
- **Traefik v3** with automatic HTTP→HTTPS redirect and Let's Encrypt TLS-ALPN certificate issuance.
- **Basic-auth protected Traefik dashboard** on a separate hostname.
- **Sign-ups togglable** via `VAULTWARDEN_SIGNUPS_ALLOWED`.
- **SQLite storage in a named volume**: one directory to back up.

## Supply chain trust

This repository is a deployment template, not a custom Docker image. It orchestrates two upstream images:

- [`traefik`](https://hub.docker.com/_/traefik): reverse proxy, Docker Hub official image
- [`vaultwarden/server`](https://hub.docker.com/r/vaultwarden/server): Vaultwarden upstream

Both are pinned to `tag@sha256:<digest>` as interpolation defaults in the compose file's `x-images` block: `git pull` alone delivers the version combination this repository has tested; an `*_IMAGE_TAG` variable in `.env` overrides deliberately.

Two override levels exist per image. `<PREFIX>_IMAGE_VERSION` in `.env` swaps only the version of that image (Compose then pulls the tag, without a digest) and leaves every other pin as tested; `<PREFIX>_IMAGE_TAG` replaces the whole reference, digest included. The variable names are listed in `.env.example`. Nested defaults need Docker Compose v2.5 or newer (2022); v2.0 to v2.4 leave the inner `${...}` unexpanded and `docker compose up` fails with an invalid reference instead of deploying something unexpected.

The daily `check-pin-freshness` CI job re-resolves both pinned tags against their registries and compares the pinned Vaultwarden and Traefik versions against the latest upstream releases. CI runs on every push, pull request, and every day at 06:00 UTC. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Register your accounts, then set `VAULTWARDEN_SIGNUPS_ALLOWED=false`** and recreate the stack: a password manager should not accept strangers.
- [ ] **Strong Traefik dashboard hash**: regenerate per deployment.
- [ ] **Back up the `vaultwarden-data` volume** off-host on a schedule: it holds every vault. `sqlite3`-consistent snapshots or stopping the container briefly are both fine at this scale.
- [ ] **Verify Let's Encrypt cert issuance** in the Traefik logs on first start.
- [ ] **Consider fail2ban or Traefik rate-limiting** on the admin and login endpoints for internet-exposed instances.

## Backups

The `backups` container runs on a loop: an initial delay (`VAULTWARDEN_BACKUP_INIT_SLEEP`, default 30m), then every `VAULTWARDEN_BACKUP_INTERVAL` (default 24h) it takes a consistent copy of each SQLite database (`db.sqlite3`) through Python's `sqlite3` backup API - no application stop - and a `tar.gz` of the rest of the data directory (live database files excluded), into the `vaultwarden-backups` volume; files older than `VAULTWARDEN_BACKUP_PRUNE_DAYS` (default 7) are pruned. Each artefact logs `... backup OK: <file> (<bytes> bytes)` or `FAILED` (kept as `<file>.failed`): grep the log for `FAILED` from your monitoring.

**Verify backups are running:**

```bash
docker compose -p vaultwarden logs backups | tail -5
docker compose -p vaultwarden exec backups ls -la /srv/vaultwarden/backups/
```

**Restore** a backup set with the interactive script (`chmod +x vaultwarden-restore-data.sh` once): it stops vaultwarden, unpacks the data archive over the data directory, restores each database from its consistent copy, and starts vaultwarden again.

```bash
./vaultwarden-restore-data.sh
```

**Off-host replication.** Backups live in a named volume on the same host: bind-mount `VAULTWARDEN_BACKUPS_PATH` to a directory covered by your off-host backup solution (restic, rclone, Borg, S3 sync).

## Unattended updates

Releases are the update channel: a tag is cut only after CI has built the pinned images, booted the full stack, and passed the smoke tests. `update.sh` moves a deployment to the newest tag and nothing else:

```bash
./update.sh --dry-run   # show what would be applied
./update.sh             # update within the current major and redeploy
```

Put it on a timer for hands-off minor/patch updates:

```bash
# crontab -e
17 5 * * *  /opt/vaultwarden-traefik-letsencrypt-docker-compose/update.sh >> /var/log/vaultwarden-update.log 2>&1
```

The script refuses to cross a MAJOR template version on its own. Majors are breaking by definition and their release notes exist to be read. After reading them, `./update.sh --allow-major` performs the jump. It also refuses to touch a checkout with local modifications: your customization belongs in `.env`, which updates never overwrite.

This is deliberately a host-side script and not a container in the stack: an in-stack updater needs the Docker socket (root on the host) and turns "someone pushed to a repo" into "someone deployed to your machine" with no operator in the loop. A cron job under your own user updates only to tagged, CI-verified states and leaves the trust boundary where it was.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults, the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`, e.g. `TRAEFIK_MEMORY_LIMIT=512m`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> --format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Container hardening

Every service runs with `security_opt: no-new-privileges:true`, so a process cannot gain privileges through setuid binaries even if it escapes its initial capability set. Infrastructure containers (the reverse proxy, databases, caches, backups) run with `cap_drop: [ALL]` and add back only what their entrypoints need: `NET_BIND_SERVICE` for Traefik to bind :80/:443, `CHOWN`/`SETUID`/`SETGID` (and friends) for database images to own their data directory and drop to their service user. Application containers keep the default capability set on purpose: upstream images assume it, and a wrong guess there is a boot loop in production rather than a hardening win. CI boots the stack under exactly these settings on every push, so what ships is what was tested.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC:

1. **Lint**: actionlint on the workflow.
2. **Trivy scans** of both pinned images (CRITICAL/HIGH, SARIF to the Security tab).
3. **Pin freshness** (daily/manual): digest drift plus release-lag checks for Vaultwarden and Traefik.
4. **Deploy-and-test**: boots the stack and requires `/alive` to answer through Traefik plus a 200 web vault page.

A green run is the authoritative proof that the template deploys end-to-end.

### Backup and restore, proven

`tests/e2e-backup-restore.sh` runs against the live stack and is what CI executes after the smoke test. The scenario that matters most is the restore roundtrip: the application is stopped, the baseline database copy is put back, and a row inserted after the baseline is gone. The tests stop the application briefly and write into its data directory: run them on a staging copy with short intervals in `.env` (`VAULTWARDEN_BACKUP_INIT_SLEEP=15s`, `VAULTWARDEN_BACKUP_INTERVAL=60s`), never on production.

```bash
chmod +x tests/e2e-backup-restore.sh
./tests/e2e-backup-restore.sh
```

## Security notes

- No credentials ship in this repository; `.env` is gitignored and compose fails fast on missing required variables.
- The admin panel (`/admin`) is disabled unless you set `ADMIN_TOKEN`: leave it disabled unless you need it, and protect it if you enable it.
- Upstream image digests are pinned; the daily freshness job flags drift loudly.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** · Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
