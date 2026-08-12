# CLAUDE.md — valkey

Shared Valkey instance. Read `README.md` first for the standalone-vs-Coolify basics — this
file is the "don't repeat past mistakes" layer.

## This is shared infrastructure, not an app

Deployed once per tenant, reused by every app in that tenant that needs a Redis-compatible
cache or queue broker. It must **never** get a public domain or a published host port — same
convention as this fleet's shared Postgres/Qdrant. When creating the Coolify application,
explicitly suppress the auto-assigned domain: `PATCH /applications/<uuid>` with
`{"docker_compose_domains": [{"name":"valkey","domain":""}]}`.

## Version pinning

Pinned to `valkey/valkey:8-alpine` — major-version-and-alpine pin, same style already used
for the shared Postgres (`postgres:18-alpine`), since Valkey's own release cadence favors
tracking the latest patch within a major automatically rather than pinning an exact patch.
Bump the major only via a deliberate Renovate PR + this fleet's major-upgrade runbook
(dump/restore equivalent — Valkey supports `BGSAVE`/`--appendonly` for this).

## Security hardening

Both compose files carry a hardening block per the
[OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html):
healthcheck, log rotation, `cap_drop: ALL`, `no-new-privileges`, resource limits, and a
read-only root filesystem with only `/tmp` as tmpfs — `/data` (the real data, including the
`appendonly` file) is already a named volume and stays writable under `read_only: true`
regardless. Validated against a real container, including a real `ACL SETUSER` write and a
restart-preserves-data check, before landing.

Both compose files also set `user: "999:1000"` (the image's built-in `valkey:valkey`
account) explicitly, the same reason Postgres needs an explicit `cap_add` list: the image's
entrypoint normally drops privileges from root to that user itself via `setpriv`, which needs
the `SETUID`/`SETGID` capabilities — `cap_drop: ALL` removes those, so without pinning `user:`
the container restart-loops. Starting as the target user directly sidesteps the drop
entirely; `/data` is already valkey-owned and world-writable (1777) in the image, so this
works against a fresh named volume with no extra chown step.

`requirepass` is passed both as a `command:` arg and as a container `environment:` var — the
former is what `valkey-server` actually reads, the latter is so the healthcheck's own
`valkey-cli` invocation (which runs inside the container, using its own env) can authenticate
without a second hardcoded copy of the secret in the compose file. Note this means the
password is visible via `docker inspect`/`ps aux` on the host — an accepted trade-off since
this instance has no public route and only trusted deploys reach the host at all.

## Access control

Durable data (not ephemeral) — chosen so a future consumer can use this as a real cache/store,
not just a disposable queue broker. Per-consumer isolation is via Valkey ACL users scoped to
a `<app>:*` key pattern (see README's "Adding a new consumer"), not a single shared password
— mirrors the per-app-database isolation this fleet already does for the shared Postgres.
