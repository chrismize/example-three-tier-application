# Services

This document lists all services in this repository and the ports they listen on.

## Service Overview

| Service | Port | Description | Exposed to Host |
|---------|------|-------------|-----------------|
| **web** | 3000 | Next.js frontend application | ✅ Yes (3000:3000) |
| **api** | 3001 | Express REST API backend | ❌ No (internal only) |
| **postgres** | 5432 | PostgreSQL 17 database | ❌ No (internal only) |
| **migrate** | N/A | Database migration runner (exits after completion) | N/A |

## Service Details

### web (Frontend)
- **Technology**: Next.js 16, React 19, Tailwind CSS
- **Port**: 3000
- **Location**: `src/web/`
- **Environment Variables**:
  - `PORT`: 3000 (default)
  - `API_URL`: http://api:3001
- **Access**: http://localhost:3000
- **Description**: The user-facing web application that provides the task manager interface.

### api (Backend API)
- **Technology**: Express 5, Node.js 22
- **Port**: 3001
- **Location**: `src/api/`
- **Environment Variables**:
  - `PORT`: 3001 (default)
  - `DATABASE_URL`: postgres://app:app@postgres:5432/app
- **Access**: Internal only (not exposed to host)
- **Description**: REST API that handles task operations and communicates with the database.
- **Endpoints**:
  - `GET /health` - Health check
  - `GET /tasks` - List all tasks
  - `POST /tasks` - Create a task
  - `PATCH /tasks/:id` - Update a task

### postgres (Database)
- **Technology**: PostgreSQL 17 (Alpine)
- **Port**: 5432 (standard PostgreSQL port)
- **Access**: Internal only (not exposed to host)
- **Description**: PostgreSQL database that stores application data.
- **Credentials** (local development):
  - Database: `app`
  - User: `app`
  - Password: `app`

### migrate (Database Migrations)
- **Technology**: node-pg-migrate
- **Port**: N/A (not a listening service)
- **Location**: `src/db/`
- **Description**: Runs database migrations on startup and exits. Not a long-running service.

## Port Configuration

Ports are configured in the following locations:

- **docker-compose.yml**: Defines port mappings and environment variables
- **src/api/index.js**: API port with fallback `const PORT = process.env.PORT || 3001;`
- **src/web/**: Next.js uses PORT environment variable (default 3000)

## Network Architecture

```
Browser → web:3000 → api:3001 → postgres:5432
                ↑
          (exposed to host)
```

Only the `web` service is exposed to the host machine. The `api` and `postgres` services communicate over Docker's internal network and are not directly accessible from the host.
