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

Each stack uses a named volume (`valkey_data`, `redis_data`, `postgres_data`, `opensearch_data`, `minio_data`, `rabbitmq_data`). Docker creates these automatically on the first deploy. `docker compose down` keeps them; only `down -v` or deleting the volume removes the data.

For a dedicated disk or shared storage:

1. Mount the drive on the host, for example `/mnt/data`.
2. Create directories, one per service:
   ```bash
   sudo mkdir -p /mnt/data/{valkey,redis,postgres,opensearch,minio,rabbitmq}
   ```
3. Replace the named volume in the compose file with a bind mount:
   ```yaml
   volumes:
     - /mnt/data/postgres:/var/lib/postgresql
   ```
4. Make sure the directory is writable by the container user. Some images (including the Chainguard MinIO image) run as non-root.

Postgres 18 note: mount at `/var/lib/postgresql`, not `/var/lib/postgresql/data`. Since 18, `PGDATA` is `/var/lib/postgresql/18/docker` and the declared volume moved to the parent directory.

Back up the volumes or the mounted drive on a schedule. Test a restore before relying on it.

## 3. Set environment variables in Dokploy

Set these in each service's Environment section instead of editing the compose files. Defaults exist for local use only; replace all passwords before exposing anything.

| Service | Variables |
| --- | --- |
| Valkey | `VALKEY_NAME`, `VALKEY_IMAGE`, `VALKEY_PORT` |
| Redis | `REDIS_NAME`, `REDIS_IMAGE` |
| Postgres | `POSTGRES_NAME`, `POSTGRES_IMAGE`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_PORT` |
| OpenSearch | `OPENSEARCH_NAME`, `OPENSEARCH_IMAGE`, `OPENSEARCH_CLUSTER_NAME`, `OPENSEARCH_NODE_NAME`, `OPENSEARCH_JAVA_OPTS`, `OPENSEARCH_INITIAL_ADMIN_PASSWORD`, `OPENSEARCH_PORT`, `OPENSEARCH_PERFORMANCE_PORT` |
| MinIO | `MINIO_NAME`, `MINIO_IMAGE`, `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, `MINIO_PORT`, `MINIO_CONSOLE_PORT` |
| RabbitMQ | `RABBITMQ_NAME`, `RABBITMQ_IMAGE`, `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`, `RABBITMQ_PORT`, `RABBITMQ_MANAGEMENT_PORT` |


Compose file	Canonical host	Also resolves	Internal port(s)
docker-compose.valkey.yml	dokploy-valkey:6379	valkey	6379
docker-compose.redis.yml	dokploy-redis:6379	redis	6379
docker-compose.postgres.yml	dokploy-postgres:5432	postgres	5432
docker-compose.opensearch.yml	dokploy-opensearch:9200	opensearch	9200 API, 9600 perf
docker-compose.minio.yml	dokploy-minio:9000	minio	9000 S3 API, 9001 console
docker-compose.rabbitmq.yml	dokploy-rabbitmq:5672	rabbitmq	5672 AMQP, 15672 management

Each `*_IMAGE` variable overrides the default image tag, so you can pin a version or switch registry without changing the file. Each `*_NAME` variable sets both the Compose project name and the container name; its default is the current `prod-*` value.

Change these defaults in particular: `POSTGRES_PASSWORD`, `MINIO_ROOT_PASSWORD` / `MINIO_ROOT_USER`, `RABBITMQ_DEFAULT_PASS` / `RABBITMQ_DEFAULT_USER`, and `OPENSEARCH_INITIAL_ADMIN_PASSWORD`.

## 4. How other apps reach the services

All services are on `production-network`. Use the Compose service name as the hostname:

- `postgres:5432`
- `redis:6379` or `valkey:6379`
- `rabbitmq:5672`
- `minio:9000`
- `opensearch:9200`

The old `dokploy-*` names are kept as network aliases, so existing connections using `dokploy-postgres`, `dokploy-redis`, etc. keep working.

Redis is internal-only: it has no host port and is reached at `redis:6379` on `production-network`. Valkey maps host port 6379 by default, so change `VALKEY_PORT` if both run on the same host.

## 5. Service-specific tips

- **Postgres** — `pg_isready` healthcheck. Back up with `pg_dump` from another container on `production-network`.
- **Valkey / Redis** — AOF persistence is enabled (`--appendonly yes`). Redis has no host port by design.
- **RabbitMQ** — management UI on host port 15672. `RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS` disables the guest-only-localhost rule, so set real credentials and restrict who can reach the ports.
- **OpenSearch** — security plugin is disabled (`plugins.security.disabled: "true"`), so keep it on the private network. Set `vm.max_map_count=262144` on the host:
  ```bash
  sudo sysctl -w vm.max_map_count=262144
  echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-opensearch.conf
  ```
  Default heap is 512 MB via `OPENSEARCH_JAVA_OPTS`; raise it for larger datasets.
- **MinIO** — uses `cgr.dev/chainguard/minio:latest` because the old community images are no longer published. API is on 9000, console on 9001. Healthcheck calls `/minio/health/live`; create buckets from the console or with an S3 client. When adding a domain in Dokploy, set the target container port to **9001** for the web console; port 9000 is the S3 API.

## 6. Deploy and troubleshoot

1. Create `production-network` (Dokploy creates `dokploy-network` itself).
2. Set the environment variables in Dokploy.
3. Deploy one stack at a time and check its health status in Dokploy.

Common problems:

- `network production-network not found` — run `docker network create production-network`.
- Domain shows `404 page not found` — the service must be on `dokploy-network`, the `traefik.docker.network` label must say `dokploy-network`, and the domain's target port must match the app (9001 for the MinIO console).
- `port is already allocated` — change the matching `*_PORT` variable.
- OpenSearch exits immediately — check `vm.max_map_count`, memory, and heap settings.
- MinIO is unhealthy — confirm the image can be pulled and that `/minio/health/live` responds on port 9000.
