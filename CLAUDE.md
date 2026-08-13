# CLAUDE.md — browser

Shared Chrome/CDP sidecar. Read `README.md` first for the standalone-vs-Coolify basics —
this file is the "don't repeat past mistakes" layer.

## This is shared infrastructure, not an app

Deployed once per tenant, reused by every app that needs browser automation. It must
**never** get a public domain or a published host port — same convention as this fleet's
shared Postgres/Qdrant/Valkey. When creating the Coolify application, explicitly suppress
the auto-assigned domain: `PATCH /applications/<uuid>` with
`{"docker_compose_domains": [{"name":"browser","domain":""}]}`.

## The real CDP port is 9223, not 9222

Chrome's own `--remote-debugging-port` (set to `9222` via `CHROME_CLI`) binds to
`127.0.0.1` inside the container by the image's own design — it is not reachable from
another container directly. The image's own nginx
(`/etc/nginx/cdp-proxy.conf`, baked in, not configured by this repo) proxies `9222` onto
`9223` bound to `0.0.0.0`, and *that* is the port every consumer must use
(`http://browser:9223`). Confirmed live via a real CDP `Page.navigate` call, not just a
port-open check — see the Hub's `docs/superpowers/plans/2026-08-13-openclaw-qwen-deployment.md`
for the full verification.

## Security hardening — one real correction to generic "Chrome in Docker" advice

Generic advice for running Chromium in Docker says it needs `cap_add: SYS_ADMIN` (or
`security_opt: seccomp=unconfined`) for the browser's own internal sandboxing. **Live-tested
false for this specific image**: zero extra capabilities, real page navigation over CDP
worked cleanly, no sandbox errors in the logs. What this image actually needs is
`cap_add: [CHOWN, FOWNER, DAC_OVERRIDE, SETUID, SETGID]` — the exact same set this fleet's
`postgresql` already documents, for the same underlying reason (an s6-overlay
root→PUID/PGID privilege-drop entrypoint dance that bare `cap_drop: ALL` breaks:
`s6-applyuidgid: fatal: unable to set supplementary group list: Operation not permitted`
and nginx `chown(...) failed`, both confirmed live and both gone once those five caps are
restored). `no-new-privileges:true` stays enabled — also confirmed compatible.

**Deliberately no `read_only: true`** — this image runs a full desktop environment (X
server, nginx runtime state, browser cache/downloads) with a write surface too broad to
safely enumerate ahead of time. Same posture and same reasoning as `n8n`/`metamcp`'s own
documented exception.

## Known gap: the CDP port has no authentication

`/etc/nginx/cdp-proxy.conf` (baked into the image) is a bare reverse-proxy passthrough onto
Chrome's debug port — no password, no token, nothing. The image *does* support a
`SELKIES_MASTER_TOKEN` env var, but that only protects the separate web-desktop UI
(ports 3000/3001), not CDP. Once this service is on the shared `coolify` network, *any*
other container already on that network can drive this browser, not just its intended
consumer(s) — Docker network membership is the only boundary today. Accepted as a bounded
risk: every container in this fleet is first-party and curated, not arbitrary/untrusted
code. Revisit if that ever changes.
