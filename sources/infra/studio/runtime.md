# Runtime Topology -- Studio (Full Supabase Stack)

> Extracted from docker/docker-compose.yml in the supabase monorepo. This describes the complete self-hosted Supabase stack.

## Services

| Service | Image | Ports | Depends On | Purpose |
|---------|-------|-------|------------|---------|
| studio | supabase/studio | 3000 | analytics | Next.js admin dashboard |
| kong | kong:3.9.1 | 8000 (HTTP), 8443 (HTTPS) | auth, rest, realtime, storage, pg_meta, functions | API gateway / reverse proxy |
| auth (gotrue) | supabase/gotrue | 9999 | db | Authentication server |
| rest (PostgREST) | postgrest/postgrest | 3000 | db | Auto-generated REST API from DB |
| realtime | supabase/realtime | 4000 | db | WebSocket realtime engine |
| storage | supabase/storage-api | 5000 | db, imgproxy | Object storage API |
| imgproxy | darthsim/imgproxy | 5001 | - | Image transformation service |
| meta (pg-meta) | supabase/postgres-meta | 8080 | db | Postgres introspection API |
| functions | supabase/edge-runtime | 54321 | - | Deno Edge Functions runtime |
| analytics | supabase/logflare | 4000 | db | Log analytics and querying |
| db | supabase/postgres | 5432 | - | PostgreSQL database |
| vector | timberio/vector | - | - | Log collection/routing |

## Key Environment Variables

| Variable | Source | Purpose | Secret? |
|----------|--------|---------|---------|
| JWT_SECRET | .env | Shared JWT signing secret | yes |
| ANON_KEY | .env | Anonymous role JWT | no |
| SERVICE_ROLE_KEY | .env | Service role JWT | yes |
| POSTGRES_PASSWORD | .env | Database password | yes |
| SUPABASE_PUBLIC_URL | .env | Public-facing base URL | no |
| STUDIO_PG_META_URL | docker-compose | pg-meta introspection URL | no |
| LOGFLARE_PUBLIC_ACCESS_TOKEN | .env | Analytics access token | yes |
