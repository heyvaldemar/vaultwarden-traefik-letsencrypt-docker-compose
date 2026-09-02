# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [1.1.0] - 2026-09-02

### Added

- **`update.sh`** — unattended updates to the newest tagged release,
  and nothing else: a tag is cut only after CI has booted the pinned
  images and passed the smoke tests, so "update to the latest tag" means
  "update to a combination a machine has already run". It refuses to
  cross a major version on its own (`--allow-major` after reading the
  notes), refuses a checkout with local modifications, and supports
  `--dry-run`. Put it on a cron timer for hands-off minor/patch updates.

## [1.0.1] - 2026-09-02

### Added

- `.env.example` — the README told you to copy it, and it did not exist.
  Every variable the compose file reads is documented with its default
  and a generation command where one applies.
- This changelog, which the v1.0.0 release notes already linked to.
- MIT `LICENSE` and `SECURITY.md`.

## [1.0.0] - 2026-08-31

First semver release. Brings this template to the fleet standard established
in [keycloak-traefik-letsencrypt-docker-compose](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose).

### Changed

- **Vaultwarden updated to 1.37.2** (was 1.29.1 — two years of upstream
  security fixes for a password manager) and **Traefik to v3.7** (was
  3.2, whose Docker client cannot talk to Docker Engine 29), both pinned
  by `tag@sha256:digest` in the compose `x-images` block. `git pull`
  delivers the tested combination; `VAULTWARDEN_IMAGE_TAG` /
  `TRAEFIK_IMAGE_TAG` in `.env` override deliberately.
- Required variables fail fast with `${VAR:?…}` guards;
  `VAULTWARDEN_SIGNUPS_ALLOWED` and `TRAEFIK_LOG_LEVEL` have defaults.

### Added

- **Deployment Verification workflow**: actionlint; a Trivy scan of each
  pinned image; `check-pin-freshness` (digest drift + Vaultwarden/Traefik
  release lag); and a deploy-and-test job that boots the stack and
  requires `/alive` and the web vault to answer through Traefik.

[Unreleased]: https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose/compare/v1.1.0...HEAD
[1.1.0]: https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose/compare/v1.0.1...v1.1.0
[1.0.1]: https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/heyvaldemar/vaultwarden-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
