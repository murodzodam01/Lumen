# Lumen Dashboard Project Guide

This document explains the purpose of the tools and files in the Lumen
dashboard. For startup commands, see [README.md](./README.md).

## What the system does

Lumen displays investigative information about Telegram channels, messages,
keywords, and relationships between channels:

1. The browser loads the dashboard UI.
2. The UI requests data from the Express API.
3. The API queries PostgreSQL.
4. PostgreSQL returns the imported harm-tracker dataset.

```text
Browser
  │
  │ HTTP requests to /api/*
  ▼
Express API (Node.js, port 3000)
  │
  │ SQL queries
  ▼
PostgreSQL (host port 5433, container port 5432)
```

## What Docker is for

Docker packages and runs services in isolated containers so the project does
not require a locally installed PostgreSQL server or a manually configured
Node.js runtime.

Docker Compose coordinates the two containers:

- `postgres`: runs PostgreSQL and imports the SQL dump the first time its data
  directory is created.
- `api`: builds the Node.js API image, installs the dependencies, and starts
  `server.js`.

The `pgdata` directory is a Docker bind-mounted data directory. After the
initial import, PostgreSQL uses this directory and does not import the dump on
every restart.

## Tools and what they are for

| Tool | Purpose |
|---|---|
| Docker Desktop | Runs the PostgreSQL and API containers |
| Docker Compose | Starts, stops, and connects the containers |
| PostgreSQL | Stores channels, messages, edges, actors, and keywords |
| Node.js | Runs the Express API |
| npm | Installs the API packages listed in `package.json` |
| PowerShell | Runs the Windows commands in the setup instructions |
| Browser | Displays the dashboard HTML |
| Python `http.server` | Optional static web server for the dashboard UI |
| Git | Tracks source code and documentation |

## File-by-file explanation

### `docker-compose.yml`

Defines the local two-service environment:

- PostgreSQL uses the `postgres:16-alpine` image.
- The API is built from `Dockerfile`.
- PostgreSQL is exposed on host port `5433`.
- The API is exposed on host port `3000`.
- The SQL dump is mounted as PostgreSQL's first-start initialization script.
- The API waits for PostgreSQL's health check before starting in Compose.

### `Dockerfile`

Builds the API container:

1. Starts with Node.js 20 Alpine.
2. Sets `/app` as the working directory.
3. Copies `package.json`.
4. Installs production dependencies.
5. Copies `server.js`.
6. Starts the API on port `3000`.

### `package.json`

Declares the API's Node.js dependencies:

- `express`: HTTP server and routing.
- `pg`: PostgreSQL client.
- `cors`: allows the browser dashboard to call the API.

It also defines the `npm start` command.

### `package-lock.json`

Records the exact dependency tree selected by npm. This makes dependency
installation repeatable and is used when building the Docker image.

### `server.js`

Implements the Express REST API. It:

- Connects to PostgreSQL using the Compose environment variables.
- Provides statistics, channel, message, keyword, network, actor, timeline,
  export, and health endpoints.
- Converts database records into JSON for the dashboard.
- Logs connection and request errors.

### `lumen-dashboard.html`

Contains the browser dashboard UI, including the layout, styles, charts,
navigation, filters, tables, network visualization, and browser-side API
requests. It is a single static HTML file and does not require a frontend
build step.

### `harm_tracker_fresh.sql`

The supplied PostgreSQL schema-and-data dump. It creates the database tables,
indexes, constraints, and imported records. It is approximately 789 MB, is
ignored by Git, and must be supplied locally before a fresh database can be
created.

### `pgdata/`

The local PostgreSQL data directory created by Docker. It contains the
initialized database after the first import. It is ignored by Git and should
not be edited manually.

### `README.md`

The quick-start and operations manual: prerequisites, terminal commands,
ports, database setup, optional local development, and troubleshooting.

### `.gitignore` at the repository root

Prevents generated files, dependency directories, local database data, and the
large SQL dump from being committed accidentally.

## Important ports

| Address | Component | Use |
|---|---|---|
| `localhost:3000` | Express API | Dashboard data and health endpoint |
| `localhost:5433` | PostgreSQL from the host | Optional direct database access |
| `localhost:5432` | PostgreSQL inside Docker | Container-to-container API connection |
| `localhost:8080` | Optional Python static server | Dashboard browser page |

## Typical command ownership

### Terminal 1: services

```powershell
cd C:\path\to\Lumen\postgres
docker compose up --build
```

### Terminal 2: checks and browser server

```powershell
cd C:\path\to\Lumen\postgres
Invoke-RestMethod http://localhost:3000/api/health
python -m http.server 8080
```

Then open <http://localhost:8080/lumen-dashboard.html>.
