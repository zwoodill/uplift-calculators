# uplift-calculators
Micro-home build planning tools (static HTML calculators).

## What it is
A stock nginx container serving this repo as static files. The default page is
`water-systems-planner.html`.

## Hosting
Served at **https://systems.uplift.us** from this workspace VM. nginx binds
host port **8101**, and the company **registrar** routes the public URL to
this host:port over the tailnet (see `docker-compose.yml`). There is no auth —
the tailnet is the security boundary.

## Run locally
```
docker compose up -d --build
curl -sS http://127.0.0.1:8101/
```

## Data
None. The site is fully static; nginx serves the repo read-only.

## Deploys
Registered in `~/apps/watch.list`, so the workspace deploy-watcher pulls
`origin/main` and rebuilds within ~60s of a push from outside this VM. If you
change files on the VM directly, run `docker compose up -d --build` here and
push so GitHub matches what's running.
