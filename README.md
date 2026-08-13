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
pattern (`~`) and the matching pub/sub channel pattern (`&`)**, not just the key pattern:

```
ACL SETUSER <app>_user on ><generated-password> ~<app>:* &<app>:* +@all -@admin -@dangerous
```

The channel grant is easy to miss and the failure is not obvious: found live when onboarding
n8n, whose queue-mode architecture uses Valkey pub/sub internally (not just the key-value
queue itself) to broadcast commands between its main and worker processes. An ACL user with
only `~<app>:*` connects and reads/writes queue keys fine, then crashes at startup the moment
it tries to subscribe to its own channel, with `NOPERM No permissions to access a channel` —
a Valkey ACL user has zero channel access by default (`resetchannels`) unless explicitly
granted one, independent of its key permissions.

Give the consumer app `QUEUE_BULL_PREFIX=<app>` (or equivalent key-prefix setting) so its
keys stay inside the pattern the ACL user is scoped to.
