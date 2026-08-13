# valkey

Shared Valkey (Redis-compatible) cache/queue broker. Standalone-usable with plain Docker
Compose, or deployed as shared tenant infrastructure on Coolify. First consumer: `n8n`
(queue mode).

## Standalone

```bash
cp .env.example .env
# set VALKEY_REQUIREPASS in .env
docker compose up -d
```

## On Coolify (as shared tenant infra)

Deployed with `docker_compose_location` set to `/docker-compose.coolify.yaml`, which joins
the `coolify` external Docker network so other apps in the same tenant can reach it by the
service name `valkey`. No public domain/FQDN.

## Adding a new consumer

Don't hand out the instance-level `VALKEY_REQUIREPASS` to a consumer app directly. Provision
a dedicated ACL user scoped to that app's own key prefix instead — **grant both the key
pattern (`~`) and the matching pub/sub channel pattern (`&`)**, not just the key pattern,
**and persist it**, not just apply it in-memory:

```
docker exec <container> valkey-cli --no-auth-warning -a '<instance requirepass>' \
  acl setuser <app>_user on '>'<generated-password>'' '~<app>:*' '&<app>:*' +@all -@admin -@dangerous
docker exec <container> valkey-cli --no-auth-warning -a '<instance requirepass>' acl save
```

Both steps matter, and both were found live during n8n's onboarding, from two separate
incidents:

- **The channel grant** (`&<app>:*`) is easy to miss and the failure is not obvious: n8n's
  queue-mode architecture uses Valkey pub/sub internally (not just the key-value queue
  itself) to broadcast commands between its main and worker processes. An ACL user with only
  `~<app>:*` connects and reads/writes queue keys fine, then crashes at startup the moment it
  tries to subscribe to its own channel, with `NOPERM No permissions to access a channel` —
  a Valkey ACL user has zero channel access by default (`resetchannels`) unless explicitly
  granted one, independent of its key permissions.
- **`ACL SAVE`** is required because this instance now runs with `--aclfile /data/users.acl`
  (see `docker-compose.coolify.yaml`'s comments) specifically so `ACL SETUSER` grants survive
  a container recreation. Skipping `ACL SAVE` leaves a grant live in memory only — it works
  until the next deploy of this repo (even an unrelated docs-only push, which triggers Coolify's
  auto-deploy webhook the same as a real change), at which point the container is recreated
  from the last-saved aclfile and any un-saved grant is silently gone. This actually happened
  once in production before `--aclfile` was added at all.

Give the consumer app `QUEUE_BULL_PREFIX=<app>` (or equivalent key-prefix setting) so its
keys stay inside the pattern the ACL user is scoped to.
