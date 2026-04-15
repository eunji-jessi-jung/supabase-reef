# Runtime Topology -- Realtime

> Extracted from docker-compose.yml

## Services

| Service | Image/Build | Ports | Depends On | Purpose |
|---------|------------|-------|------------|---------|
| realtime | Build from Dockerfile | 4000:4000 | db | Elixir/Phoenix realtime server |
| db | supabase/postgres:17.6.1.074 | 5432:5432 | - | Primary PostgreSQL (management DB) |
| tenant_db | supabase/postgres:17.6.1.074 | 5433:5432 | - | Tenant PostgreSQL (per-tenant replication) |

## Environment Variables

| Variable | Source | Purpose | Secret? |
|----------|--------|---------|---------|
| PORT | docker-compose | HTTP listen port (4000) | no |
| DB_HOST | docker-compose | PostgreSQL host | no |
| DB_PORT | docker-compose | PostgreSQL port | no |
| DB_USER | docker-compose | PostgreSQL user | no |
| DB_PASSWORD | docker-compose | PostgreSQL password | yes |
| DB_NAME | docker-compose | PostgreSQL database | no |
| DB_ENC_KEY | docker-compose | Encryption key for tenant secrets | yes |
| DB_AFTER_CONNECT_QUERY | docker-compose | search_path for _realtime schema | no |
| API_JWT_SECRET | docker-compose | JWT validation secret for API auth | yes |
| SECRET_KEY_BASE | docker-compose | Phoenix secret key base | yes |
| METRICS_JWT_SECRET | docker-compose | JWT for metrics endpoint | yes |
| APP_NAME | docker-compose | OTP application name | no |
| RUN_JANITOR | docker-compose | Enable background cleanup jobs | no |
| JANITOR_INTERVAL | docker-compose | Cleanup interval (ms) | no |
| SEED_SELF_HOST | docker-compose | Auto-create default tenant | no |
| DASHBOARD_USER | docker-compose | Dashboard basic auth user | no |
| DASHBOARD_PASSWORD | docker-compose | Dashboard basic auth password | yes |
