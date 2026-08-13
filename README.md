# browser

Shared Chrome/CDP browser sidecar, based on
[`coollabsio/openclaw-browser`](https://github.com/coollabsio/openclaw). Standalone-usable
with plain Docker Compose, or deployed as a shared fleet service on Coolify for any tenant
app that needs browser automation — currently `openclaw`.

## Standalone

```bash
docker compose up -d
```

Web desktop UI at http://localhost:3000, CDP at http://localhost:9223/json/version.

## On Coolify

Deployed with `docker_compose_location` set to `/docker-compose.coolify.yaml`, which joins
the `coolify` external network and never gets a public domain (see the Hub's README
"Adding a tenant app" runbook, `docker_compose_domains` step — this service is
"shared infrastructure with no public route", same as `postgresql`/`qdrant`/`valkey`).

## Using this from another app

Point `BROWSER_CDP_URL` at `http://browser:9223` (the `openclaw` repo does this — see its
`docker-compose.coolify.yaml`). No authentication on this port — see this repo's
`CLAUDE.md` for why, and the accepted-risk reasoning.
