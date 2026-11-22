# Deployment notes

Setup tips for the `docker-compose.*.yml` files in this folder when deployed with Dokploy.

## 1. Create the network first

Every stack joins the external network `production-network`. Create it once on the Dokploy host:

```bash
docker network create production-network
docker network inspect production-network
```

Keep `external: true` in the compose files. If you remove it, each stack claims ownership of the network: a `docker compose down` in one stack can delete it for all the others, and another stack may fail with `network already exists but was not created by Docker Compose`.

Dokploy's Traefik runs on its own `dokploy-network` (created by Dokploy) and only routes to containers attached to it. Every service in these templates therefore joins both networks: `production-network` for app-to-service traffic, and `dokploy-network` with `traefik.docker.network: dokploy-network` for Traefik routing. The label alone does not publish anything; Dokploy adds the router and TLS labels when you assign a domain.

## 2. Create storage / drives

Each stack uses an explicitly named volume set by `*_VOLUME_NAME`: `valkey_data`, `redis_data`, `postgres_data`, `opensearch_data`, `minio_data`, `rabbitmq_data`, `poste-data`, `moodle_data`, and `moodledata`. Mail defaults to `${MAIL_NAME:-dev-mail}_poste-data` so an existing deployment reuses its data. Docker creates these automatically on the first deploy. `docker compose down` keeps them; only `down -v` or deleting the volume removes the data.

Explicit volume names are not prefixed by the Compose project, so set a unique `*_VOLUME_NAME` for each deployment to avoid sharing data. If you previously used a host bind mount such as `/storage/minio`, copy that data into the new named volume before switching.

For a dedicated disk or shared storage:

1. Mount the drive on the host, for example `/mnt/data`.
2. Create directories, one per service and deployment:
   ```bash
   sudo mkdir -p /mnt/data/{valkey,redis,postgres,opensearch,minio,rabbitmq,mail,moodle}
   ```
3. Replace the named volume in the compose file with a bind mount:
   ```yaml
   volumes:
     - /mnt/data/postgres:/var/lib/postgresql
   ```
4. Make sure the directory is writable by the container user. Some images run as non-root.

Postgres 18 note: mount at `/var/lib/postgresql`, not `/var/lib/postgresql/data`. Since 18, `PGDATA` is `/var/lib/postgresql/18/docker` and the declared volume moved to the parent directory.

Back up the volumes or the mounted drive on a schedule. Test a restore before relying on it.

## 3. Set environment variables in Dokploy

Set these in each service's Environment section instead of editing the compose files. Defaults exist for local use only; replace all passwords before exposing anything.

| Service | Variables |
| --- | --- |
| Valkey | `VALKEY_NAME`, `VALKEY_IMAGE`, `VALKEY_ALIAS`, `VALKEY_PORT`, `VALKEY_VOLUME_NAME` |
| Redis | `REDIS_NAME`, `REDIS_IMAGE`, `REDIS_ALIAS`, `REDIS_VOLUME_NAME` |
| Postgres | `POSTGRES_NAME`, `POSTGRES_IMAGE`, `POSTGRES_ALIAS`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_PORT`, `POSTGRES_VOLUME_NAME` |
| OpenSearch | `OPENSEARCH_NAME`, `OPENSEARCH_IMAGE`, `OPENSEARCH_ALIAS`, `OPENSEARCH_CLUSTER_NAME`, `OPENSEARCH_NODE_NAME`, `OPENSEARCH_JAVA_OPTS`, `OPENSEARCH_INITIAL_ADMIN_PASSWORD`, `OPENSEARCH_PORT`, `OPENSEARCH_PERFORMANCE_PORT`, `OPENSEARCH_VOLUME_NAME` |
| MinIO | `MINIO_NAME`, `MINIO_IMAGE`, `MINIO_ALIAS`, `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, `MINIO_PORT`, `MINIO_CONSOLE_PORT`, `MINIO_VOLUME_NAME` |
| RabbitMQ | `RABBITMQ_NAME`, `RABBITMQ_IMAGE`, `RABBITMQ_ALIAS`, `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`, `RABBITMQ_PORT`, `RABBITMQ_MANAGEMENT_PORT`, `RABBITMQ_VOLUME_NAME` |
| Mail | `MAIL_NAME`, `MAIL_IMAGE`, `MAIL_HOSTNAME`, `MAIL_ALIAS`, `MAIL_TZ`, `MAIL_HTTPS`, `MAIL_WEB_PORT`, `MAIL_SMTP_PORT`, `MAIL_SUBMISSION_PORT`, `MAIL_SMTPS_PORT`, `MAIL_IMAP_PORT`, `MAIL_IMAPS_PORT`, `MAIL_POP3_PORT`, `MAIL_POPS_PORT`, `MAIL_SIEVE_PORT`, `DOMAIN`, `MAIL_VOLUME_NAME` |
| Moodle | `MOODLE_NAME`, `MOODLE_IMAGE`, `MOODLE_ALIAS`, `MOODLE_PORT`, `MOODLE_HTTPS_PORT`, `DB_TYPE`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `MOODLE_USERNAME`, `MOODLE_PASSWORD`, `MOODLE_EMAIL`, `MOODLE_SITE_NAME`, `MOODLE_PHP_MEMORY_LIMIT`, `MOODLE_PHP_MAX_EXECUTION_TIME`, `MOODLE_VOLUME_NAME`, `MOODLE_DATA_VOLUME_NAME` |

Each `*_IMAGE` variable overrides the default image tag, so you can pin a version or switch registry without changing the file. Each `*_NAME` variable sets both the Compose project name and the container name; its default is the current `dev-*` value. Each `*_ALIAS` variable sets the stable internal DNS name on `production-network` and `dokploy-network`; its default is the old `dokploy-*` value.

For multiple deployments on one host, set a unique `*_NAME`, `*_ALIAS`, and `*_VOLUME_NAME` for every stack. `*_PORT` values set the container port; leave them at their defaults unless the service is also configured to listen on the new port.

All `ports` entries use container-port-only syntax, so Docker assigns a free host port for each deployment instead of binding a fixed host port. To find the generated host port:

```bash
docker compose -p dev-postgres -f docker-compose.postgres.yml port postgres 5432
```

`MAIL_HOSTNAME` defaults to `mail.${DOMAIN:-example.com}`. Set it explicitly for each mail deployment so Poste.io generates the correct certificate and URLs.

Change these defaults in particular: `POSTGRES_PASSWORD`, `MINIO_ROOT_PASSWORD` / `MINIO_ROOT_USER`, `RABBITMQ_DEFAULT_PASS` / `RABBITMQ_DEFAULT_USER`, `OPENSEARCH_INITIAL_ADMIN_PASSWORD`, `MOODLE_PASSWORD` / `DB_PASSWORD`.

## 4. How other apps reach the services

All services are on both `production-network` and `dokploy-network`. Use the `*_ALIAS` value (or the unique `*_NAME` value) as the hostname:

- `${POSTGRES_ALIAS:-dokploy-postgres}:5432`
- `${REDIS_ALIAS:-dokploy-redis}:6379` or `${VALKEY_ALIAS:-dokploy-valkey}:6379`
- `${RABBITMQ_ALIAS:-dokploy-rabbitmq}:5672`
- `${MINIO_ALIAS:-dokploy-minio}:9000`
- `${OPENSEARCH_ALIAS:-dokploy-opensearch}:9200`
- `${MAIL_ALIAS:-dokploy-mail}:25`
- `${MOODLE_ALIAS:-dokploy-moodle}:8080`

The `dokploy-*` defaults keep existing connections working. For multiple deployments, set unique aliases (for example `POSTGRES_ALIAS=postgres-blog` and `POSTGRES_ALIAS=postgres-moodle`) so DNS never points at the wrong instance.

Redis is internal-only: it has no published port and is reached over `production-network`. The other stacks publish their container ports to random host ports, which avoids host port conflicts.

## 5. Service-specific tips

- **Postgres** — `pg_isready` healthcheck. Back up with `pg_dump` from another container on `production-network`.
- **Valkey / Redis** — AOF persistence is enabled (`--appendonly yes`). Redis has no published port by design.
- **RabbitMQ** — management UI listens on container port 15672. Use `docker compose -p <name> -f docker-compose.rabbitmq.yml port rabbitmq 15672` to find its host port. `RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS` disables the guest-only-localhost rule, so set real credentials.
- **OpenSearch** — security plugin is disabled (`plugins.security.disabled: "true"`), so keep it on the private network. Set `vm.max_map_count=262144` on the host:
  ```bash
  sudo sysctl -w vm.max_map_count=262144
  echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-opensearch.conf
  ```
  Default heap is 512 MB via `OPENSEARCH_JAVA_OPTS`; raise it for larger datasets.
- **MinIO** — the default image is `elestio/minio:latest`. API is on container port 9000, console on 9001. Healthcheck calls `mc ready local`. When adding a domain in Dokploy, set the target container port to **9001** for the web console; port 9000 is the S3 API.
- **Mail** — set `MAIL_HOSTNAME` (or `DOMAIN`) before the first deploy. `HTTPS=OFF` expects TLS termination in Dokploy's Traefik. Publish container port **80** (`MAIL_WEB_PORT`) for the Poste.io admin UI and set the Dokploy domain target port to 80. Mail ports are published to random host ports; keep the `*_PORT` defaults unless Poste.io is reconfigured to match.
- **Moodle** — set `DB_HOST` to the unique Postgres alias for this deployment, not `localhost`. HTTP and HTTPS listen on container ports 8080 and 8443.

## 6. Deploy and troubleshoot

1. Create `production-network` (Dokploy creates `dokploy-network` itself).
2. Set unique `*_NAME` and `*_ALIAS` values plus real credentials in Dokploy.
3. Deploy one stack at a time and check its health status in Dokploy.

Common problems:

- `network production-network not found` — run `docker network create production-network`.
- Domain shows `404 page not found` — the service must be on `dokploy-network`, the `traefik.docker.network` label must say `dokploy-network`, and the domain's target port must match the app (9001 for the MinIO console).
- `container name is already in use` — two deployments are using the same `*_NAME`; set a unique value for each.
- App connects to the wrong database/cache — two deployments are sharing a `*_ALIAS`; set a unique alias for each.
- OpenSearch exits immediately — check `vm.max_map_count`, memory, and heap settings.
- MinIO is unhealthy — confirm the image can be pulled and that `mc ready local` responds inside the container.
- Mail cannot receive connections — confirm `MAIL_HOSTNAME` and the Traefik domain, and check the generated host ports with `docker compose -p <name> -f docker-compose.mail.yml port mailserver 25`.
