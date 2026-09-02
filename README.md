# Vaultwarden + Traefik + Let's Encrypt — Docker Compose

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

This repository deploys **Vaultwarden** (the lightweight Bitwarden-compatible server) behind **Traefik** with automatic **Let's Encrypt TLS**. One `docker compose up` away from a self-hosted password manager at `https://your-domain`, compatible with all official Bitwarden clients.

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

- **A Linux server** with a public IP. Vaultwarden is light — the smallest VPS works.
- **Docker Engine 24+ and Docker Compose 2.20+.**
- **A domain you control,** with two `A` records pointing at your server's public IP — one for Vaultwarden, one for the Traefik dashboard. DNS must propagate before deploy. Bitwarden clients require HTTPS, which this stack provides out of the box.
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

Within a minute `https://${VAULTWARDEN_HOSTNAME}` serves the web vault with a fresh Let's Encrypt certificate. **Register your account(s), then disable sign-ups** (see the checklist).

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
- **Mobile app refuses to connect.** The client requires a valid HTTPS URL — use the public hostname, never an IP.
- **`network vaultwarden-network not found`.** Step 2 was skipped.

### Apply `.env` or compose-file changes

```bash
docker compose -f vaultwarden-traefik-letsencrypt-docker-compose.yml -p vaultwarden up -d --force-recreate
```

## Features

- **Vaultwarden** latest stable (1.37.2) — the Rust reimplementation of the Bitwarden server API; works with all official clients, browser extensions, and apps.
- **Traefik v3** with automatic HTTP→HTTPS redirect and Let's Encrypt TLS-ALPN certificate issuance.
- **Basic-auth protected Traefik dashboard** on a separate hostname.
- **Sign-ups togglable** via `VAULTWARDEN_SIGNUPS_ALLOWED`.
- **SQLite storage in a named volume** — one directory to back up.

## Supply chain trust

This repository is a **deployment template**, not a custom Docker image. It orchestrates two upstream images:

- [`traefik`](https://hub.docker.com/_/traefik) — reverse proxy, Docker Hub official image
- [`vaultwarden/server`](https://hub.docker.com/r/vaultwarden/server) — Vaultwarden upstream

Both are pinned to `tag@sha256:<digest>` as interpolation defaults in the compose file's `x-images` block — `git pull` alone delivers the version combination this repository has tested; an `*_IMAGE_TAG` variable in `.env` overrides deliberately.

The daily `check-pin-freshness` CI job re-resolves both pinned tags against their registries and compares the pinned Vaultwarden and Traefik versions against the latest upstream releases. CI runs on every push, pull request, and every day at 06:00 UTC. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Register your accounts, then set `VAULTWARDEN_SIGNUPS_ALLOWED=false`** and recreate the stack — a password manager should not accept strangers.
- [ ] **Strong Traefik dashboard hash** — regenerate per deployment.
- [ ] **Back up the `vaultwarden-data` volume** off-host on a schedule — it holds every vault. `sqlite3`-consistent snapshots or stopping the container briefly are both fine at this scale.
- [ ] **Verify Let's Encrypt cert issuance** in the Traefik logs on first start.
- [ ] **Consider fail2ban or Traefik rate-limiting** on the admin and login endpoints for internet-exposed instances.

## Backups

Vault data (SQLite database, attachments, keys) lives in the `vaultwarden-data` named volume. Simplest reliable backup:

```bash
docker compose -p vaultwarden exec vaultwarden sqlite3 /data/db.sqlite3 ".backup /data/db-backup.sqlite3"
docker cp "$(docker compose -p vaultwarden ps -q vaultwarden)":/data/db-backup.sqlite3 ./
```

Ship the copy (plus `/data/attachments` and `/data/rsa_key*` if present) to off-host storage on a schedule.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC:

1. **Lint** — actionlint on the workflow.
2. **Trivy scans** of both pinned images (CRITICAL/HIGH, SARIF to the Security tab).
3. **Pin freshness** (daily/manual) — digest drift plus release-lag checks for Vaultwarden and Traefik.
4. **Deploy-and-test** — boots the stack and requires `/alive` to answer through Traefik plus a 200 web vault page.

A green run is the authoritative proof that the template deploys end-to-end.

## Security Notes

- No credentials ship in this repository; `.env` is gitignored and compose fails fast on missing required variables.
- The admin panel (`/admin`) is disabled unless you set `ADMIN_TOKEN` — leave it disabled unless you need it, and protect it if you enable it.
- Upstream image digests are pinned; the daily freshness job flags drift loudly.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
