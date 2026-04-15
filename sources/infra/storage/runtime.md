# Runtime Topology -- Storage

> Extracted from docker-compose.yml

## Services

| Service | Image/Build | Ports | Depends On | Purpose |
|---------|------------|-------|------------|---------|
| storage | supabase/storage-api:latest | 5000:5000 | tenant_db, pg_bouncer, minio_setup | Storage API server |
| tenant_db | (from infra compose) | 5432 | - | PostgreSQL for metadata |
| pg_bouncer | (from infra compose) | 6432 | - | Connection pooler |
| minio | (from infra compose) | 9000 | - | S3-compatible object store (dev) |
| imgproxy | (from infra compose) | 8080 | - | Image transformation proxy |

## Environment Variables

| Variable | Source | Purpose | Secret? |
|----------|--------|---------|---------|
| SERVER_PORT | docker-compose | HTTP listen port (5000) | no |
| AUTH_JWT_SECRET | docker-compose | JWT validation secret | yes |
| DATABASE_URL | docker-compose | Direct Postgres connection | yes |
| DATABASE_POOL_URL | docker-compose | PgBouncer pooled connection | yes |
| STORAGE_BACKEND | docker-compose | Backend type (s3/file) | no |
| STORAGE_S3_BUCKET | docker-compose | S3 bucket name | no |
| STORAGE_S3_ENDPOINT | docker-compose | S3-compatible endpoint URL | no |
| STORAGE_S3_REGION | docker-compose | S3 region | no |
| AWS_ACCESS_KEY_ID | docker-compose | S3 access key | yes |
| AWS_SECRET_ACCESS_KEY | docker-compose | S3 secret key | yes |
| UPLOAD_FILE_SIZE_LIMIT | docker-compose | Max upload size (bytes) | no |
| IMAGE_TRANSFORMATION_ENABLED | docker-compose | Enable imgproxy transforms | no |
| IMGPROXY_URL | docker-compose | Image proxy service URL | no |
| S3_PROTOCOL_ACCESS_KEY_ID | docker-compose | S3-protocol auth key | yes |
| S3_PROTOCOL_ACCESS_KEY_SECRET | docker-compose | S3-protocol auth secret | yes |
| TUS_URL_PATH | docker-compose | TUS resumable upload path prefix | no |
| ICEBERG_CATALOG_URL | docker-compose | Iceberg REST catalog URL | no |
| ICEBERG_CATALOG_AUTH_TOKEN | docker-compose | Iceberg auth token | yes |

## Storage Patterns

Storage uses S3-compatible backends (MinIO in dev, AWS S3 or compatible in production). Object metadata is stored in PostgreSQL, while actual binary objects live in the S3 bucket. The service supports multiple protocols: HTTP REST, TUS resumable uploads, S3-compatible API, Iceberg REST catalog, and vector storage.
