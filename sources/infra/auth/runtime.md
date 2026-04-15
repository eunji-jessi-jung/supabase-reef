# Runtime Topology -- Auth

> Extracted from docker-compose-dev.yml

## Services

| Service | Image/Build | Ports | Depends On | Purpose |
|---------|------------|-------|------------|---------|
| auth | Build from Dockerfile.dev | 9999:9999, 9100:9100 | postgres | Go auth server with hot-reload (CompileDaemon) |
| postgres | Build from Dockerfile.postgres.dev | 5432:5432 | - | PostgreSQL database with auth schema |

## Environment Variables

| Variable | Source | Purpose | Secret? |
|----------|--------|---------|---------|
| GOTRUE_JWT_SECRET | example.env | JWT signing secret | yes |
| GOTRUE_JWT_EXP | example.env | JWT expiration (seconds) | no |
| GOTRUE_JWT_AUD | example.env | JWT audience claim | no |
| GOTRUE_DB_DRIVER | example.env | Database driver (postgres) | no |
| DATABASE_URL | example.env | Postgres connection string | yes |
| API_EXTERNAL_URL | example.env | Public-facing URL for callbacks | no |
| PORT | example.env | HTTP listen port | no |
| GOTRUE_SMTP_HOST | example.env | SMTP server for emails | no |
| GOTRUE_SMTP_PASS | example.env | SMTP password | yes |
| GOTRUE_MAILER_AUTOCONFIRM | example.env | Skip email confirmation | no |
| DB_NAMESPACE | example.env | PostgreSQL schema name (auth) | no |
| GOTRUE_DB_MIGRATIONS_PATH | docker-compose | Path to migration files | no |

## Volumes / Storage Mounts

| Mount | Host Path / Volume | Container Path | Purpose |
|-------|-------------------|----------------|---------|
| source code | ./ | /go/src/github.com/supabase/auth | Live reload dev mount |
| postgres_data | named volume | /var/lib/postgresql/data | Persistent DB storage |
