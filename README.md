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
a dedicated ACL user scoped to that app's own key prefix instead:

```
ACL SETUSER <app>_user on ><generated-password> ~<app>:* +@all -@admin -@dangerous
```

Give the consumer app `QUEUE_BULL_PREFIX=<app>` (or equivalent key-prefix setting) so its
keys stay inside the pattern the ACL user is scoped to.
