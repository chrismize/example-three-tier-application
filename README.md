# example-three-tier-application

A reference implementation of a three-tier web application: a Next.js frontend, an Express REST API, and a PostgreSQL database. It runs locally with Docker Compose and deploys to Google Cloud Platform (Cloud Run + Cloud SQL) via Terraform.

## Architecture

```
Browser → Web (Next.js :3000) → API (Express :3001) → PostgreSQL
```

| Layer | Technology | Location |
|-------|-----------|----------|
| Frontend | Next.js 16, React 19, Tailwind CSS | `src/web/` |
| API | Express 5, Node.js 22 | `src/api/` |
| Database | PostgreSQL 17 | managed by Docker / Cloud SQL |
| Migrations | node-pg-migrate | `src/db/` |
| Infrastructure | Terraform (GCP) | `src/infrastructure/` |

The app is a simple task manager (to-do list) that demonstrates how the three tiers communicate.

## Running locally with Docker Compose

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine + Compose plugin)

### Start the stack

```bash
docker compose up --build
```

This starts four services in order:

1. **postgres** — PostgreSQL 17 database, waits until healthy
2. **migrate** — runs `node-pg-migrate up` to apply schema migrations, then exits
3. **api** — Express API on port 3001 (internal only)
4. **web** — Next.js frontend on port 3000 (exposed to host)

Once running, open [http://localhost:3000](http://localhost:3000).

### Stop and clean up

```bash
# Stop containers (keeps the postgres_data volume)
docker compose down

# Stop and delete all data
docker compose down -v
```

### Rebuild after code changes

```bash
docker compose up --build
```

### API endpoints

The API is not exposed directly, but you can reach it through the web container or by temporarily mapping its port:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check |
| GET | `/tasks` | List all tasks |
| POST | `/tasks` | Create a task (`{ "title": "..." }`) |
| PATCH | `/tasks/:id` | Update a task (`{ "completed": true }` or `{ "title": "..." }`) |

## Project structure

```
src/
├── api/            # Express REST API
│   ├── index.js    # Route handlers
│   ├── db.js       # PostgreSQL connection pool
│   └── Dockerfile
├── db/             # Database migrations
│   ├── migrations/ # node-pg-migrate migration files
│   └── Dockerfile
├── web/            # Next.js frontend
│   ├── app/        # App Router pages and components
│   └── Dockerfile
└── infrastructure/ # Terraform for GCP deployment
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

## Deploying to GCP

The `src/infrastructure/` directory contains Terraform that provisions:

- VPC network and subnet
- Cloud SQL PostgreSQL 17 instance (private IP)
- Cloud Run services for the API and web frontend
- Secret Manager secret for the database URL
- Service accounts and IAM bindings

### Required variables

| Variable | Description |
|----------|-------------|
| `project_id` | GCP project ID |
| `api_image` | Container image URI for the API (e.g. `gcr.io/PROJECT/api:TAG`) |
| `web_image` | Container image URI for the web frontend |
| `region` | GCP region (default: `us-central1`) |
| `environment` | `dev`, `staging`, or `prod` (default: `dev`) |

```bash
cd src/infrastructure
terraform init
terraform apply -var="project_id=my-project" \
                -var="api_image=gcr.io/my-project/api:latest" \
                -var="web_image=gcr.io/my-project/web:latest"
```

After apply, `terraform output web_url` gives the public URL.

## Database migrations

Migrations live in `src/db/migrations/` and use [node-pg-migrate](https://salsita.github.io/node-pg-migrate/).

```bash
# Apply all pending migrations (run inside the db container or with DATABASE_URL set)
cd src/db
DATABASE_URL=postgres://app:app@localhost:5432/app npx node-pg-migrate up

# Roll back the last migration
DATABASE_URL=postgres://app:app@localhost:5432/app npx node-pg-migrate down
```

When running via Docker Compose the `migrate` service handles this automatically on startup.

## Testing notes

### Linting

The web frontend includes ESLint for code quality checks:

```bash
cd src/web
npm run lint
```

### Unit tests

Currently, no unit test suite is configured. The `npm test` command in both `src/api/` and `src/db/` will report "no test specified".

To add tests in the future, consider:
- **API**: Jest or Mocha with supertest for endpoint testing
- **Web**: Jest with React Testing Library for component tests
- **Database**: Integration tests against a test database

### Integration testing

The best way to verify the full stack is to:

1. Start the application with Docker Compose:
   ```bash
   docker compose up --build
   ```

2. Open [http://localhost:3000](http://localhost:3000) and verify:
   - The page loads successfully
   - You can create new tasks
   - Tasks persist after page refresh
   - You can mark tasks as completed
   - You can update task titles

3. Check the API health endpoint:
   ```bash
   curl http://localhost:3001/health
   ```
   (Note: You may need to expose port 3001 in `docker-compose.yml` or exec into the web container to reach the API)

## Testing notes 2

### Running the test suite

This project currently has limited automated testing. Here's how to run the available tests:

#### Linting (Web Frontend)

The web frontend uses ESLint for code quality checks. To run the linter:

```bash
cd src/web
npm install  # Install dependencies if not already done
npm run lint
```

This will check all TypeScript and JavaScript files in the web application for code quality issues and Next.js best practices.

#### Unit Tests (API and Database)

The API and database packages do not currently have unit tests configured. Running `npm test` in these directories will output:

```bash
# In src/api/
cd src/api
npm test  # Outputs: "Error: no test specified"

# In src/db/
cd src/db
npm test  # Outputs: "Error: no test specified"
```

To add unit tests in the future, consider using Jest or Mocha with appropriate testing libraries for each component.

#### Full Stack Verification

The most comprehensive way to test the application is through integration testing with Docker Compose:

```bash
# Start the full stack
docker compose up --build

# In another terminal, verify the application is working
curl http://localhost:3000  # Should return the Next.js frontend HTML
curl http://localhost:3001/health  # May require port exposure or container exec
```

Then manually test the task management functionality through the web interface at [http://localhost:3000](http://localhost:3000).
