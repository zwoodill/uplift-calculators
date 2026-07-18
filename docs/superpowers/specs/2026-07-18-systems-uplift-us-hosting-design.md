# Hosting the systems calculators at systems.uplift.us

**Date:** 2026-07-18
**Status:** Approved (serving mode and root URL confirmed by Trevor)

## Goal

Serve the static calculators in this repo (`water-systems-planner.html`,
`wall-stack-up.html`) at `https://systems.uplift.us`, using the same pattern as
`pcb.uplift.us` (schematic-tools) and `power.uplift.us` (power-architecture):
a Docker Compose stack in the app repo that joins the shared `proxy` network
and self-registers with the company-infra Traefik instance via labels.

Explicit non-goal: no build step, no SPA/React conversion, no changes to the
HTML files themselves.

## Design

- **Server:** stock `nginx:1.27-alpine` container — no Dockerfile.
- **Content:** the repo directory is bind-mounted read-only at
  `/usr/share/nginx/html`. Deploying an update is just `git pull` on
  `atx-srv-01`; no rebuild, no restart. (A whole-directory mount is used
  instead of per-file mounts because single-file bind mounts pin the inode and
  go stale after `git pull`.)
- **Root URL:** nginx `index water-systems-planner.html;` so
  `https://systems.uplift.us/` opens the planner directly. Deep links to
  `/water-systems-planner.html` and `/wall-stack-up.html` keep working.
- **Hygiene:** nginx denies dotfile paths (`location ~ /\.`) so the
  bind-mounted `.git/` directory is not served. Markdown docs in the repo
  root remain reachable; they are internal documentation and the service is
  LAN/Tailscale-only, so that is acceptable.
- **Routing/TLS:** standard label set on the container —
  `Host(`systems.uplift.us`)`, `entrypoints=websecure`,
  `tls.certresolver=letsencrypt`, `server.port=80`. Traefik issues a
  per-service Let's Encrypt cert via Cloudflare DNS-01 on first request.
- **DNS:** one new grey-cloud A record `systems.uplift.us → 192.168.8.159`,
  per the hard rules in company-infra (never a wildcard, never touching
  apex/mail records).

## Files added to this repo

- `nginx.conf` — server block: root, index directive, dotfile deny.
- `docker-compose.yml` — the `web` service with the Traefik labels above,
  external `proxy` network.

## Error handling / testing

- Verify with `curl -sI https://systems.uplift.us/` from the LAN: expect
  `200`, a valid LE cert, and the planner HTML at `/`.
- Verify `/.git/config` returns `403`.
- If the cert doesn't issue, check `docker logs traefik` for ACME errors
  (known gotchas are documented in company-infra/CLAUDE.md).

## Alternatives considered

- **Baked image (Dockerfile COPY):** immutable deploys but every content
  update needs `docker compose up -d --build`. Rejected — the bind-mount's
  `git pull`-only deploy fits a static page better.
- **Hand-written index page at `/`:** deferred until this repo actually hosts
  more than one live calculator.
