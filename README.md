# Software Marketplace - QA environment

Runs the complete backend (API, database, file storage, email catcher) with one command.
You only need Docker. No Java, no Maven, no source code.

> Everything here is **for QA only**. The passwords and keys are public. Never use them for anything real.

## Quick start

Requirements: Docker Desktop (Windows/macOS) or Docker Engine with Compose v2 (Linux), with at least **4 GB RAM** given to Docker.

```bash
git clone https://github.com/NaviGamage/Software-marketplace-deployment.git
cd Software-marketplace-deployment
docker compose up -d
```

The first start downloads the images and prepares the database; allow 1 to 3 minutes.

```bash
docker compose ps             # wait until "app" shows (healthy)
docker compose logs -f app    # Ctrl+C to stop following
```

The `seed-admin` container exits by itself after creating the admin account. That is normal.

## What is inside

| Service | Purpose | Image |
|---|---|---|
| `app` | The marketplace backend (REST API) | `navindu2001/software-marketplace-backend:qa` |
| `postgres` | Database (schema is created by the app on start) | `postgres:16-alpine` |
| `minio` + `minio-init` | S3-compatible file storage for product files, with the bucket created automatically | `minio/minio`, `minio/mc` |
| `mailpit` | Catches every email the system sends so you can read it | `axllent/mailpit` |
| `seed-admin` | Runs once: creates the QA admin account, then exits | `postgres:16-alpine` |
| `clamav` | Virus scanner (**off by default**, see Optional) | `clamav/clamav` |

## Repository contents

| Path | What it is |
|---|---|
| `docker-compose.yml` | The whole environment |
| `.env.example` | Optional settings you can override (copy to `.env`) |
| `postman/` | Postman collection (183 automated tests) and its environment |
| `docs/QA-TEST-GUIDE.md` | How to run the tests, known findings, how to report bugs |
| `docs/MAINTAINERS.md` | For developers: how the backend image is built and published |
| `seed/seed-admin-qa.sql` | The admin account SQL (for reference; compose already contains it) |

## Addresses

| What | URL |
|---|---|
| API | http://localhost:8080 |
| API docs (Swagger UI) | http://localhost:8080/swagger-ui.html |
| Emails sent by the system | http://localhost:8025 (Mailpit) |
| File storage console (MinIO) | http://localhost:9001, user `qaminio`, password `qaminio-secret-123` |
| Database | `localhost:5433`, db `marketplace_db`, user `marketplace_user`, password `marketplace_pass` |

## Test accounts

| Role | Email | Password |
|---|---|---|
| Admin | `qa.admin@example.com` | `Passw0rd!QA` |
| Vendor / Buyer | created by the Postman collection | `Passw0rd!QA` |

## Run the API tests

1. Import both files from [`postman/`](postman) into Postman and select the environment **QA (local)**.
2. Follow [`docs/QA-TEST-GUIDE.md`](docs/QA-TEST-GUIDE.md): run order, files to attach, the payout step, known findings, how to report bugs.

Use a **800 ms delay between requests** in the Postman Runner; the API allows 100 requests per minute per client.

## Everyday commands

| Task | Command |
|---|---|
| Stop (keep data) | `docker compose stop` |
| Start again | `docker compose start` |
| **Reset everything** (wipe database and files) | `docker compose down -v` then `docker compose up -d` |
| Get the newest backend build | `docker compose pull app` then `docker compose up -d` |
| Logs of one service | `docker compose logs -f app` |
| Run SQL (the payout prep step) | `docker exec -it marketplace_postgres psql -U marketplace_user -d marketplace_db -c "<SQL>"` |

Reset to a clean database before each full test run: the collection creates its own users and data.

## Optional

Copy `.env.example` to `.env`, change what you need, then run `docker compose up -d` again.

- **Virus scanning (ClamAV)**: set `COMPOSE_PROFILES=scan` and `VIRUS_SCAN_ENABLED=true`. Needs about 2 GB extra RAM; the first start downloads signatures and takes several minutes (`docker compose logs -f clamav`).
- **Own S3 / Supabase bucket instead of MinIO**: set the `AWS_*` values. Use a bucket made for QA and get the keys from the developer. Never commit them.
- **Shared QA server**: set `BIND_ADDR=0.0.0.0` and use the server address as `baseUrl` in Postman.
- **Pin a build**: set `APP_IMAGE` to a specific tag so the build cannot change during a test cycle.

## Checking a file download by hand

The API returns a signed download link. With the bundled MinIO the link host is `minio`, which your computer does not know. Add this line once to your hosts file:

```
127.0.0.1 minio
```

Windows: `C:\Windows\System32\drivers\etc\hosts` (edit as Administrator). Linux/macOS: `/etc/hosts`. The link then opens in a browser, and after it expires it must be refused.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `pull access denied` for the backend image | The backend image `navindu2001/software-marketplace-backend:qa` is not published yet (or is private on Docker Hub). Ask the developer. |
| `port is already allocated` | Another program uses 8080, 5433, 8025 or 9000. Change the port in `.env` or stop that program. |
| `app` stays `starting` or becomes `unhealthy` | `docker compose logs app`. Send the last 50 lines to the developer. |
| Upload requests return 500 | File storage is unreachable. Check `docker compose ps` shows `minio` running. |
| Many `429 Too Many Requests` | Use the 800 ms delay in the Postman Runner; after a login burst wait a minute. |
| Apple Silicon (ARM) Mac, the image fails to start | Tell the developer; the image may need an ARM build. |

## Maintainers

How the backend image is published and how this repo is updated: [`docs/MAINTAINERS.md`](docs/MAINTAINERS.md).
