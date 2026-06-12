# Production Deployment

This app can be deployed with Docker Compose using `prod.docker-compose.yaml`. The production stack runs:

- `web`: Django + Gunicorn on container port `8000`
- `worker`: Celery worker
- `scheduler`: scheduled task launcher
- `postgres`: PostgreSQL 15
- `redis`: Redis 7

The compose file builds one image, `ground-attendee:prod`, and reads runtime configuration from `.env.prod`.

## Prerequisites

- Docker Engine
- Docker Compose
- A Linux host with enough CPU and memory for Chrome, PulseAudio, Django, Celery, Postgres, and Redis
- Port `8000` available on localhost, or a reverse proxy that can reach `127.0.0.1:8000`
- Remote object storage credentials for recordings, unless you are intentionally running with local/test settings

Check Docker:

```bash
docker --version
docker compose version
```

## Environment File

Create `.env.prod` in the repository root. This file is ignored by git and must not be committed.

Generate the Django and credential encryption secrets:

```bash
python3 init_env.py
```

Use the generated `DJANGO_SECRET_KEY` and `CREDENTIALS_ENCRYPTION_KEY` values in `.env.prod`.

Important: `CREDENTIALS_ENCRYPTION_KEY` must be a valid Fernet key. It should be 44 characters and decode to 32 bytes. Do not hand-write a short password here.

Minimum local production compose example:

```dotenv
DJANGO_SECRET_KEY=<generated-django-secret>
CREDENTIALS_ENCRYPTION_KEY=<generated-fernet-key>

SITE_DOMAIN=localhost:8000
ALLOWED_HOSTS=localhost,127.0.0.1
CSRF_TRUSTED_ORIGINS=http://localhost:8000,http://127.0.0.1:8000
DISABLE_EMAIL=true

POSTGRES_DB=attendee_production
POSTGRES_USER=attendee_production_user
POSTGRES_PASSWORD=<strong-postgres-password>
DATABASE_URL=postgresql://attendee_production_user:<strong-postgres-password>@postgres:5432/attendee_production

REDIS_URL=redis://redis:6379/0

AWS_RECORDING_STORAGE_BUCKET_NAME=<recordings-bucket>
AWS_ACCESS_KEY_ID=<aws-access-key-id>
AWS_SECRET_ACCESS_KEY=<aws-secret-access-key>
AWS_DEFAULT_REGION=us-east-1
```

For HTTPS behind a reverse proxy, also set:

```dotenv
SITE_DOMAIN=your.domain.com
ALLOWED_HOSTS=your.domain.com
CSRF_TRUSTED_ORIGINS=https://your.domain.com
```

Then run compose with:

```bash
PROD_DJANGO_SSL_REQUIRE=true docker compose -f prod.docker-compose.yaml up -d
```

`PROD_DJANGO_SSL_REQUIRE` is intentionally separate from `DJANGO_SSL_REQUIRE` because Docker Compose auto-loads `.env` for interpolation. This prevents a development `.env` from silently changing production compose behavior.

## Required Environment Variables

Core:

- `DJANGO_SECRET_KEY`: required Django secret key.
- `CREDENTIALS_ENCRYPTION_KEY`: required Fernet key for stored credentials.
- `SITE_DOMAIN`: public app domain, for example `localhost:8000` or `app.example.com`.
- `ALLOWED_HOSTS`: comma-separated Django allowed hosts.
- `CSRF_TRUSTED_ORIGINS`: comma-separated trusted origins, including scheme.

Database:

- `POSTGRES_DB`: database created by the bundled Postgres container.
- `POSTGRES_USER`: Postgres username.
- `POSTGRES_PASSWORD`: Postgres password.
- `DATABASE_URL`: Django database URL. For bundled Postgres, use host `postgres`.

Redis:

- `REDIS_URL`: Redis URL. For bundled Redis, use `redis://redis:6379/0`.

Storage:

- `STORAGE_PROTOCOL`: optional, defaults to `s3`. Use `azure` for Azure Blob Storage.
- `AWS_RECORDING_STORAGE_BUCKET_NAME`: required when using S3 storage.
- `AWS_ACCESS_KEY_ID`: required when using S3 storage unless using another AWS auth mechanism.
- `AWS_SECRET_ACCESS_KEY`: required when using S3 storage unless using another AWS auth mechanism.
- `AWS_DEFAULT_REGION`: AWS region, commonly `us-east-1`.
- `AWS_ENDPOINT_URL`: optional S3-compatible endpoint.

Email:

- `DISABLE_EMAIL`: set `true` for no SMTP delivery.
- `EMAIL_HOST`, `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `DEFAULT_FROM_EMAIL`: required if email is enabled.

Optional production operations:

- `ATTENDEE_LOG_LEVEL`: defaults to `INFO`.
- `ATTENDEE_LOG_FORMAT`: set to `json` for structured logs.
- `SENTRY_DSN`: enables Sentry.
- `SENTRY_ENVIRONMENT`: defaults to `production`.
- `REQUIRE_HTTPS_WEBHOOKS`: defaults to `true`.
- `LAUNCH_BOT_METHOD`: defaults to `kubernetes`; configure according to your bot runtime.

See `docs/environment-variables.md` for the complete application variable reference.

## First Deployment

Build and start the stack:

```bash
docker compose -f prod.docker-compose.yaml build
docker compose -f prod.docker-compose.yaml up -d
```

The `web` service runs this startup command:

```bash
python manage.py migrate --noinput && python manage.py collectstatic --noinput && gunicorn attendee.wsgi:application --bind 0.0.0.0:8000 --workers 3 --timeout 120
```

Check service status:

```bash
docker compose -f prod.docker-compose.yaml ps
```

Expected:

- `postgres` is `healthy`
- `redis` is `healthy`
- `web`, `worker`, and `scheduler` are `Up`
- `web` publishes `127.0.0.1:8000->8000/tcp`

Check logs:

```bash
docker compose -f prod.docker-compose.yaml logs --tail=120 web
docker compose -f prod.docker-compose.yaml logs --tail=120 worker
docker compose -f prod.docker-compose.yaml logs --tail=120 scheduler
```

Verify HTTP:

```bash
curl -i http://127.0.0.1:8000/
```

Without an active login, the expected response is:

```text
HTTP/1.1 302 Found
Location: /accounts/login/?next=/
```

If `PROD_DJANGO_SSL_REQUIRE=true`, HTTP should redirect to HTTPS instead.

## Reverse Proxy

The compose file binds the app to localhost only:

```yaml
ports:
  - "127.0.0.1:8000:8000"
```

Put Nginx, Caddy, Traefik, or another proxy in front of it for public access. The proxy should:

- Terminate TLS.
- Forward traffic to `http://127.0.0.1:8000`.
- Set `X-Forwarded-Proto: https` when serving HTTPS.
- Preserve the original `Host` header.

When behind HTTPS, start the stack with:

```bash
PROD_DJANGO_SSL_REQUIRE=true docker compose -f prod.docker-compose.yaml up -d
```

## Updates

Pull or deploy the new code, then rebuild and recreate app containers:

```bash
docker compose -f prod.docker-compose.yaml build
docker compose -f prod.docker-compose.yaml up -d
```

The `web` startup command applies pending migrations before starting Gunicorn.

Watch startup:

```bash
docker compose -f prod.docker-compose.yaml logs -f web
```

## Operational Commands

Run migrations manually:

```bash
docker compose -f prod.docker-compose.yaml exec -T web python manage.py migrate --noinput
```

Check migrations:

```bash
docker compose -f prod.docker-compose.yaml exec -T web python manage.py showmigrations --plan
```

Run Django deploy checks:

```bash
docker compose -f prod.docker-compose.yaml exec -T web python manage.py check --deploy
```

Open a Django shell:

```bash
docker compose -f prod.docker-compose.yaml exec web python manage.py shell
```

Restart app services:

```bash
docker compose -f prod.docker-compose.yaml restart web worker scheduler
```

Recreate app services:

```bash
docker compose -f prod.docker-compose.yaml up -d --force-recreate web worker scheduler
```

Stop the stack:

```bash
docker compose -f prod.docker-compose.yaml down
```

Do not remove volumes unless you intentionally want to delete Postgres and Redis data.

## Data Volumes

The production compose stack uses named volumes:

- `postgres_data`: PostgreSQL data
- `redis_data`: Redis append-only data

Back up `postgres_data` before upgrades or host maintenance.

Example database dump:

```bash
docker compose -f prod.docker-compose.yaml exec -T postgres sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' > attendee-production.sql
```

Restore example:

```bash
docker compose -f prod.docker-compose.yaml exec -T postgres sh -c 'psql -U "$POSTGRES_USER" "$POSTGRES_DB"' < attendee-production.sql
```

## Troubleshooting

Check all containers:

```bash
docker compose -f prod.docker-compose.yaml ps
```

Check whether `web` can resolve Postgres:

```bash
docker compose -f prod.docker-compose.yaml exec -T web getent hosts postgres
```

Expected output includes an IP and `postgres`.

Check whether the app has the expected HTTPS setting:

```bash
docker compose -f prod.docker-compose.yaml exec -T web sh -c 'echo DJANGO_SSL_REQUIRE=$DJANGO_SSL_REQUIRE'
```

Check parsed database and Redis hosts without printing secrets:

```bash
docker compose -f prod.docker-compose.yaml exec -T web python - <<'PY'
import os
import urllib.parse

print("DATABASE_URL host:", urllib.parse.urlparse(os.environ["DATABASE_URL"]).hostname)
print("REDIS_URL host:", urllib.parse.urlparse(os.environ["REDIS_URL"]).hostname)
PY
```

Validate `CREDENTIALS_ENCRYPTION_KEY` shape without printing it:

```bash
docker compose -f prod.docker-compose.yaml exec -T web python - <<'PY'
import os
import base64

value = os.environ.get("CREDENTIALS_ENCRYPTION_KEY", "")
print("length:", len(value))
print("decoded_length:", len(base64.urlsafe_b64decode(value.encode())))
PY
```

Expected:

```text
length: 44
decoded_length: 32
```

If migrations fail with `Fernet key must be 32 url-safe base64-encoded bytes`, replace `CREDENTIALS_ENCRYPTION_KEY` with a value generated by `python3 init_env.py`. If the app already has encrypted credentials in the database, changing this key will make those existing credentials unreadable.

If Docker Compose prints a warning like:

```text
The "y" variable is not set. Defaulting to a blank string.
```

check the ignored `.env` file. Docker Compose auto-loads `.env` for interpolation even when services use `.env.prod`. A value containing `$y` can trigger this warning. Keep production values in `.env.prod`, and avoid relying on `.env` for production compose interpolation.

If `curl http://127.0.0.1:8000/` redirects to HTTPS unexpectedly, check:

```bash
docker compose -f prod.docker-compose.yaml exec -T web sh -c 'echo DJANGO_SSL_REQUIRE=$DJANGO_SSL_REQUIRE'
```

For local HTTP access, it should be `false`. For reverse-proxied HTTPS, run compose with `PROD_DJANGO_SSL_REQUIRE=true`.
