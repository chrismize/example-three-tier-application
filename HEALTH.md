# Service Health Checks

This document describes how to verify that each service in the three-tier application is healthy and functioning correctly.

## Overview

The application consists of four main components:

1. **PostgreSQL Database** — Data persistence layer
2. **Database Migrations** — Schema management (runs once on startup)
3. **API Service** — Express REST API backend
4. **Web Service** — Next.js frontend

## Local Development (Docker Compose)

### Prerequisites

Ensure the stack is running:

```bash
docker compose up --build
```

### 1. PostgreSQL Database

**Health Check Method:** PostgreSQL's `pg_isready` utility

```bash
# Check if PostgreSQL is accepting connections
docker compose exec postgres pg_isready -U app -d app
```

**Expected Output:**
```
/var/run/postgresql:5432 - accepting connections
```

**Alternative:** Check container health status

```bash
docker compose ps postgres
```

The `STATUS` column should show `healthy`.

**What it verifies:**
- PostgreSQL server is running
- Database `app` exists and is accessible
- User `app` can connect

---

### 2. Database Migrations

**Health Check Method:** Container exit status

```bash
docker compose ps migrate
```

**Expected Output:**
The `STATUS` column should show `Exited (0)` or `exited with code 0`.

**What it verifies:**
- All migrations in `src/db/migrations/` have been applied successfully
- The `tasks` table and schema are up to date

**Manual Verification:**

```bash
# Connect to the database and list tables
docker compose exec postgres psql -U app -d app -c "\dt"
```

You should see the `tasks` table and `pgmigrations` table.

---

### 3. API Service

**Health Check Method:** HTTP GET request to `/health` endpoint

```bash
# From the host machine (requires exposing the API port)
# First, temporarily expose the API port by adding to docker-compose.yml:
#   api:
#     ports:
#       - "3001:3001"
# Then run:
curl http://localhost:3001/health

# OR check from within the web container:
docker compose exec web curl http://api:3001/health
```

**Expected Output:**
```json
{"status":"ok"}
```

**HTTP Status Code:** `200 OK`

**What it verifies:**
- Express server is running and accepting HTTP requests
- The `/health` endpoint is responding

**Additional Checks:**

Test the full API functionality:

```bash
# List all tasks
docker compose exec web curl http://api:3001/tasks

# Create a new task
docker compose exec web curl -X POST http://api:3001/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Test task"}'

# Expected: 201 Created with task JSON
```

**What it verifies:**
- Database connection pool is working
- API can read from and write to PostgreSQL
- All CRUD endpoints are functional

---

### 4. Web Service

**Health Check Method:** HTTP GET request to the homepage

```bash
# From the host machine
curl -I http://localhost:3000

# Or open in a browser
open http://localhost:3000
```

**Expected Output:**
```
HTTP/1.1 200 OK
...
```

**What it verifies:**
- Next.js server is running
- The application can render the homepage
- Static assets are being served

**Full Functionality Check:**

1. Open http://localhost:3000 in a browser
2. Verify the "To-Do List" heading is visible
3. Add a new task in the input field and click "Add"
4. Verify the task appears in the list
5. Click the checkbox to mark the task as complete
6. Verify the task shows as completed (strikethrough text)

**What it verifies:**
- Next.js server-side rendering is working
- Server actions are functional
- Web service can communicate with the API service
- End-to-end data flow (Web → API → Database) is working

---

## Production (Google Cloud Platform)

### Prerequisites

Ensure you have deployed the infrastructure:

```bash
cd src/infrastructure
terraform output web_url
terraform output api_url
```

### 1. Cloud SQL PostgreSQL

**Health Check Method:** Cloud Console or gcloud CLI

```bash
# Check instance status
gcloud sql instances describe <INSTANCE_NAME> \
  --format="value(state)"
```

**Expected Output:**
```
RUNNABLE
```

**Alternative:** Use Cloud Console

Navigate to **SQL** → Select your instance → Check that status shows **Available**.

**What it verifies:**
- Cloud SQL instance is running
- Database is accessible via private IP
- Backups are configured (for production)

---

### 2. Database Migrations

**Health Check Method:** Cloud Run Job status (if using migration.tf)

```bash
# Check the last migration job execution
gcloud run jobs executions list \
  --job=<MIGRATION_JOB_NAME> \
  --region=<REGION> \
  --limit=1
```

**Expected Output:**
The most recent execution should show `SUCCEEDED`.

**What it verifies:**
- Migrations have been applied to the Cloud SQL database
- Schema is up to date

---

### 3. API Service (Cloud Run)

**Health Check Method:** HTTP GET request to `/health` endpoint

```bash
# Get the API URL from Terraform output
API_URL=$(cd src/infrastructure && terraform output -raw api_url)

# Check health (requires authentication for internal services)
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  ${API_URL}/health
```

**Expected Output:**
```json
{"status":"ok"}
```

**HTTP Status Code:** `200 OK`

**Alternative:** Check Cloud Run service status

```bash
gcloud run services describe <API_SERVICE_NAME> \
  --region=<REGION> \
  --format="value(status.conditions[0].status)"
```

**Expected Output:**
```
True
```

**What it verifies:**
- Cloud Run API service is deployed and running
- Container startup probe passed (`/health` endpoint responded)
- Liveness probe is passing (service is healthy)
- VPC connector is working (API can reach Cloud SQL)
- Secret Manager integration is working (DATABASE_URL is accessible)

---

### 4. Web Service (Cloud Run)

**Health Check Method:** HTTP GET request to the public URL

```bash
# Get the web URL from Terraform output
WEB_URL=$(cd src/infrastructure && terraform output -raw web_url)

# Check the homepage
curl -I ${WEB_URL}

# Or open in a browser
open ${WEB_URL}
```

**Expected Output:**
```
HTTP/2 200
...
```

**What it verifies:**
- Cloud Run web service is deployed and running
- Container startup probe passed (homepage responded)
- Public ingress is configured correctly
- Service is accessible from the internet

**Full Functionality Check:**

1. Open the web URL in a browser
2. Verify the "To-Do List" page loads
3. Add a new task
4. Mark a task as complete
5. Verify the counter updates correctly

**What it verifies:**
- Next.js is rendering pages correctly
- Web service can communicate with API service (via VPC connector)
- API service can communicate with Cloud SQL
- End-to-end production data flow is working

---

## Monitoring and Observability

### Docker Compose

**View logs for all services:**

```bash
docker compose logs -f
```

**View logs for a specific service:**

```bash
docker compose logs -f api
docker compose logs -f web
docker compose logs -f postgres
```

**Check resource usage:**

```bash
docker stats
```

---

### Google Cloud Platform

**View Cloud Run logs:**

```bash
# API service logs
gcloud run services logs read <API_SERVICE_NAME> \
  --region=<REGION> \
  --limit=50

# Web service logs
gcloud run services logs read <WEB_SERVICE_NAME> \
  --region=<REGION> \
  --limit=50
```

**View Cloud SQL logs:**

```bash
gcloud logging read "resource.type=cloudsql_database" \
  --limit=50 \
  --format=json
```

**Cloud Console Monitoring:**

- **Cloud Run:** Navigate to Cloud Run → Select service → Metrics tab
- **Cloud SQL:** Navigate to SQL → Select instance → Monitoring tab
- **Logs Explorer:** Use the Logs Explorer for advanced log filtering and analysis

---

## Troubleshooting

### PostgreSQL not healthy

- Check if the container is running: `docker compose ps postgres`
- View logs: `docker compose logs postgres`
- Verify the volume is mounted correctly
- Ensure port 5432 is not already in use on the host

### Migrations failed

- Check migration logs: `docker compose logs migrate`
- Verify PostgreSQL is healthy before migrations run
- Check migration files in `src/db/migrations/` for syntax errors
- Manually connect to the database and check the `pgmigrations` table

### API service not responding

- Check if the container is running: `docker compose ps api`
- View logs: `docker compose logs api`
- Verify `DATABASE_URL` environment variable is set correctly
- Test database connectivity from the API container:
  ```bash
  docker compose exec api node -e "const db = require('./db'); db.query('SELECT 1').then(() => console.log('OK')).catch(console.error)"
  ```

### Web service not loading

- Check if the container is running: `docker compose ps web`
- View logs: `docker compose logs web`
- Verify `API_URL` environment variable is set correctly
- Test API connectivity from the web container:
  ```bash
  docker compose exec web curl http://api:3001/health
  ```

### Cloud Run service unhealthy

- Check service status: `gcloud run services describe <SERVICE_NAME> --region=<REGION>`
- View recent logs: `gcloud run services logs read <SERVICE_NAME> --region=<REGION>`
- Verify the container image is built correctly and pushed to the registry
- Check VPC connector status and configuration
- Verify Secret Manager permissions for the service account
- Check Cloud SQL private IP connectivity

---

## Health Check Summary

| Service | Local Check | Production Check | Endpoint/Command |
|---------|-------------|------------------|------------------|
| **PostgreSQL** | `docker compose exec postgres pg_isready -U app -d app` | `gcloud sql instances describe <INSTANCE>` | N/A |
| **Migrations** | `docker compose ps migrate` (exit 0) | `gcloud run jobs executions list` | N/A |
| **API** | `curl http://localhost:3001/health` (via web container) | `curl ${API_URL}/health` (with auth) | `GET /health` |
| **Web** | `curl http://localhost:3000` | `curl ${WEB_URL}` | `GET /` |

All services should return successful status codes and expected responses when healthy.
