# Docker Compose Self-Hosting Guide

Deploy Z3rno on a single machine using Docker Compose. This guide covers everything from initial setup to ongoing maintenance.

## Prerequisites

| Requirement | Minimum |
|---|---|
| Docker Engine + Compose v2 | 24+ |
| RAM | 2 GB (4 GB recommended) |
| Disk | 10 GB for PostgreSQL data and embeddings |

Confirm your Docker version:

```bash
docker --version     # Docker version 24.x or higher
docker compose version  # Docker Compose version v2.x
```

## Step-by-step setup

### 1. Clone the repository

```bash
git clone https://github.com/the-ai-project-co/z3rno-server.git
cd z3rno-server
```

### 2. Configure environment variables

Copy the example environment file and edit it:

```bash
cp .env.example .env
```

At minimum, set these values in `.env`:

```bash
# Required
OPENAI_API_KEY=sk-your-openai-api-key

# Recommended to change for production
POSTGRES_PASSWORD=a-strong-random-password
Z3RNO_API_KEY=z3rno_sk_prod_your-secure-key
```

See the full [environment variable reference](#environment-variable-reference) below for all options.

### 3. Start all services

```bash
docker compose -f docker-compose.dev.yml up -d
```

This starts four containers:

| Service | Host Port | Description |
|---|---|---|
| `z3rno-server` | 8000 | FastAPI application server |
| `z3rno-worker` | -- | Celery background worker |
| `postgres` | 5432 | PostgreSQL 17 with pgvector, pgvectorscale, Apache AGE, pg_cron, pgaudit |
| `valkey` | 6379 | Valkey 8 (Redis-compatible) for cache, sessions, rate limiting, Celery broker |

### 4. Verify health

Wait about 30 seconds for services to start, then check:

```bash
# Quick check
curl http://localhost:8000/v1/health

# Readiness (checks DB and cache connectivity)
curl http://localhost:8000/v1/ready

# Full automated health check
./scripts/healthcheck.sh
```

You should see `{"status": "ok"}` from the health endpoint.

Check container status:

```bash
docker compose -f docker-compose.dev.yml ps
```

All containers should show `healthy` or `running`.

## Environment variable reference

### PostgreSQL

| Variable | Default | Description |
|---|---|---|
| `POSTGRES_DB` | `z3rno` | Database name |
| `POSTGRES_USER` | `z3rno` | Database user |
| `POSTGRES_PASSWORD` | `z3rno_dev_password` | Database password |
| `POSTGRES_HOST_PORT` | `5432` | Host port mapping for PostgreSQL |

### Server

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | Auto-constructed from PG vars | Full PostgreSQL connection string (asyncpg) |
| `REDIS_URL` | `redis://valkey:6379/0` | Valkey/Redis connection URL |
| `LOG_LEVEL` | `INFO` | Log level: DEBUG, INFO, WARNING, ERROR |
| `Z3RNO_API_KEY` | `z3rno_sk_test_localdev` | API key for authenticating requests |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | OpenAI embedding model |
| `OPENAI_API_KEY` | (empty) | OpenAI API key for embeddings |
| `CORS_ORIGINS` | `http://localhost:3000,http://localhost:8000` | Comma-separated allowed CORS origins |
| `SERVER_HOST_PORT` | `8000` | Host port mapping for the API server |

### Valkey

| Variable | Default | Description |
|---|---|---|
| `VALKEY_HOST_PORT` | `6379` | Host port mapping for Valkey |

### Worker

| Variable | Default | Description |
|---|---|---|
| `CELERY_WORKER_CONCURRENCY` | `4` | Number of concurrent Celery worker processes |

## Backup procedures

### Create a backup

Back up the PostgreSQL database using `pg_dump` in custom format (compressed, supports selective restore):

```bash
docker compose -f docker-compose.dev.yml exec postgres \
  pg_dump -U z3rno -Fc z3rno > backup_$(date +%Y%m%d_%H%M%S).dump
```

### Automated daily backups

Add a cron job for scheduled backups:

```bash
# crontab -e
0 3 * * * cd /opt/z3rno-server && \
  docker compose -f docker-compose.dev.yml exec -T postgres \
  pg_dump -U z3rno -Fc z3rno > /backups/z3rno_$(date +\%Y\%m\%d).dump 2>&1
```

### Back up Valkey data

Valkey persists data to its volume with AOF. To create a point-in-time backup:

```bash
docker compose -f docker-compose.dev.yml exec valkey valkey-cli BGSAVE
docker cp z3rno-valkey:/data/dump.rdb ./valkey_backup_$(date +%Y%m%d).rdb
```

## Restore procedures

### Restore PostgreSQL from backup

```bash
# Stop the server and worker to prevent writes during restore
docker compose -f docker-compose.dev.yml stop server worker

# Restore the database (--clean drops existing objects first)
docker compose -f docker-compose.dev.yml exec -T postgres \
  pg_restore -U z3rno -d z3rno --clean --if-exists < backup_20260420.dump

# Restart all services
docker compose -f docker-compose.dev.yml up -d
```

### Restore from a full volume backup

If you backed up the entire PostgreSQL data volume:

```bash
docker compose -f docker-compose.dev.yml down
docker volume rm z3rno_postgres_data
docker volume create z3rno_postgres_data
docker run --rm -v z3rno_postgres_data:/data -v $(pwd):/backup alpine \
  tar xzf /backup/postgres_data.tar.gz -C /data
docker compose -f docker-compose.dev.yml up -d
```

## Upgrading

### Standard upgrade

```bash
# Pull the latest images
docker compose -f docker-compose.dev.yml pull

# Or rebuild if using a local Dockerfile
docker compose -f docker-compose.dev.yml build --no-cache

# Run database migrations
docker compose -f docker-compose.dev.yml run --rm server alembic upgrade head

# Restart all services with the new images
docker compose -f docker-compose.dev.yml up -d
```

### Upgrade checklist

1. **Back up the database** before upgrading (see backup procedures above).
2. **Read the release notes** for any breaking changes or required migration steps.
3. **Pull or rebuild images** to get the latest code.
4. **Run migrations** -- never skip this step.
5. **Restart services** and verify health.
6. **Monitor logs** for errors after the upgrade:

```bash
docker compose -f docker-compose.dev.yml logs -f --tail=100
```

## Monitoring (optional overlay)

Add Prometheus and Grafana to monitor request rate, latency percentiles, error rate, and memory operations:

```bash
docker compose \
  -f docker-compose.dev.yml \
  -f docker-compose.monitoring.yml \
  up -d
```

This adds two more containers:

| Service | Host Port | Description |
|---|---|---|
| `prometheus` | 9090 | Scrapes z3rno-server /metrics every 15s |
| `grafana` | 3001 | Pre-configured dashboards (login: admin / admin) |

The Z3rno Overview dashboard includes panels for:

- Request rate (req/s) by method
- Error rate (4xx and 5xx)
- Latency percentiles (p50, p95, p99)
- Memory operations count by operation type

### Monitoring environment variables

| Variable | Default | Description |
|---|---|---|
| `PROMETHEUS_HOST_PORT` | `9090` | Host port for Prometheus |
| `GRAFANA_HOST_PORT` | `3001` | Host port for Grafana |
| `GRAFANA_ADMIN_USER` | `admin` | Grafana admin username |
| `GRAFANA_ADMIN_PASSWORD` | `admin` | Grafana admin password |

## Stopping and removing

```bash
# Stop all containers (data is preserved in volumes)
docker compose -f docker-compose.dev.yml down

# Stop and remove all data volumes (DESTRUCTIVE)
docker compose -f docker-compose.dev.yml down -v
```

## Troubleshooting

### Container keeps restarting

Check container logs:

```bash
docker compose -f docker-compose.dev.yml logs server --tail=50
```

### PostgreSQL connection refused

The server waits for PostgreSQL to be healthy before starting. If PostgreSQL is slow to initialize (first run with extensions), increase the start period:

```bash
docker compose -f docker-compose.dev.yml logs postgres --tail=50
```

### Port conflicts

If ports 5432, 6379, or 8000 are already in use on your host, override them in `.env`:

```bash
POSTGRES_HOST_PORT=5433
VALKEY_HOST_PORT=6380
SERVER_HOST_PORT=8001
```
