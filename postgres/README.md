# Lumen Dashboard

Investigative dashboard for tracking harmful Telegram channels. The dashboard
is a static HTML application, the API is an Express/Node.js service, and the
data is stored in PostgreSQL.

For an explanation of Docker, the tools, and every project file, see
[PROJECT_GUIDE.md](./PROJECT_GUIDE.md).

## Architecture

```
┌─────────────────────────┐     ┌─────────────────────────┐     ┌──────────────────┐
│  lumen-dashboard │ ──▶ │   Express API (Node.js) │ ──▶ │  PostgreSQL DB   │
│        (HTML/JS)         │     │      server.js:3000    │     │  harm_tracker   │
└─────────────────────────┘     └─────────────────────────┘     └──────────────────┘
```

## Prerequisites

- Docker Desktop with Docker Compose
- The `harm_tracker_fresh.sql` file from the supplied archive

Node.js is only needed if you want to run the API outside Docker. PostgreSQL
does not need to be installed locally because Docker runs it for you.

## Run the dashboard with Docker

The normal setup uses Docker Compose for both PostgreSQL and the API. Run the
commands below from the `postgres` directory.

### Terminal 1: start the services

```powershell
cd C:\path\to\Lumen\postgres
docker compose up --build
```

This starts:

- PostgreSQL at `localhost:5433`
- The Express API at `http://localhost:3000`

The first startup imports `harm_tracker_fresh.sql`. Because the dump contains
881,898 messages, the first import can take several minutes. Leave this
terminal running and wait until PostgreSQL reports that it is ready and the API
is running.

To start in the background instead:

```powershell
docker compose up --build -d
docker compose logs -f api
```

### Terminal 2: check the services

Open a second PowerShell terminal:

```powershell
cd C:\path\to\Lumen\postgres
docker compose ps
Invoke-RestMethod http://localhost:3000/api/health
```

The health command should return a response containing:

```json
{"status":"ok"}
```

### Open the dashboard

Use one of these options:

1. Open [`lumen-dashboard.html`](./lumen-dashboard.html) directly in a browser.
2. Serve the folder over HTTP from Terminal 2:

   ```powershell
   python -m http.server 8080
   ```

   Then open <http://localhost:8080/lumen-dashboard.html>.

The dashboard reads its data from the API at `http://localhost:3000`.

### Stop the services

Run this in either terminal:

```powershell
docker compose down
```

The PostgreSQL data remains in the local `pgdata` directory.

## Database dump

The dashboard uses the supplied `harm_tracker_fresh.sql` PostgreSQL dump as its
initial dataset. The dump is local data and is intentionally ignored by Git.
Place it in this directory before starting the stack:

```text
postgres/harm_tracker_fresh.sql
```

If you received `harm_tracker_dump (1).zip`, extract
`harm_tracker_fresh.sql` from it into this directory.

The dump is ignored by Git because it is large and contains local data. It is
required only when creating a fresh PostgreSQL data directory. If `pgdata`
already contains an initialized database, PostgreSQL will not import the dump
again.

To import the dump again from scratch, stop the services and remove only the
local database volume:

```powershell
docker compose down
Remove-Item -Recurse -Force .\pgdata
docker compose up --build
```

Do this only when you intentionally want to recreate the local database.

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/stats` | Aggregate statistics |
| GET | `/api/channels` | List all channels (supports `?risk=high&search=query`) |
| GET | `/api/keywords` | Keyword frequency data |
| GET | `/api/network` | Complete graph nodes + mention/forward edges from live PostgreSQL data |
| GET | `/api/timeline` | Key events timeline |
| GET | `/api/health` | Health check |

### Example Responses

**GET /api/stats**
```json
{
  "totalChannels": 1539,
  "criticalRisk": 21,
  "totalSubscribers": 0,
  "activeChannels": 1539,
  "totalKeywords": 0
}
```

**GET /api/channels**
```json
[
  {
    "name": "@channel_username",
    "category": "channel",
    "subs": 0,
    "created": "Apr 2026",
    "lastActive": "Unknown",
    "risk": "medium",
    "status": "Active"
  }
]
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOST` | `localhost` | PostgreSQL host |
| `DB_PORT` | `5433` | PostgreSQL port |
| `DB_NAME` | `harm_tracker` | Database name |
| `DB_USER` | `tracker` | Database user |
| `DB_PASSWORD` | `tracker_pw` | Database password |
| `PORT` | `3000` | API server port |

## Database Schema

### channels
| Column | Type | Description |
|--------|------|-------------|
| channel_id | TEXT | Primary key |
| username | TEXT | Telegram username |
| title | TEXT | Channel title |
| channel_type | TEXT | Type category |
| member_count | INTEGER | Subscriber count |
| risk_level | TEXT | `critical`, `high`, `medium`, `low`, `unclassified` |
| is_active | BOOLEAN | Active status |
| discovered_at | TIMESTAMPTZ | Discovery date |

### edges
| Column | Type | Description |
|--------|------|-------------|
| edge_id | TEXT | Primary key |
| source_channel_id | TEXT | Source channel |
| target_channel_id | TEXT | Target channel |
| edge_type | TEXT | Connection type |
| weight | INTEGER | Edge weight |

### keywords
| Column | Type | Description |
|--------|------|-------------|
| keyword | TEXT | Primary key |
| channels_discovered | INTEGER | Count of channels |
| messages_matched | INTEGER | Count of messages |
| is_active | BOOLEAN | Active status |

## Development

### Project Structure

```
TGBot/
├── package.json          # Node.js dependencies
├── server.js             # Express API server
├── Dockerfile            # API container image
├── docker-compose.yml    # Docker orchestration
├── harm_tracker_fresh.sql # Local PostgreSQL schema and data dump (ignored)
├── lumen-dashboard.html  # Dashboard UI
├── PROJECT_GUIDE.md      # Architecture and file responsibilities
└── pgdata/               # Local PostgreSQL data volume (ignored)
```

### Adding New API Endpoints

1. Edit `server.js`
2. Add new route before the `app.listen()` call:
```javascript
app.get('/api/your-endpoint', async (req, res) => {
  try {
    const result = await pool.query('SELECT ...');
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

## Run the API without Docker (optional)

This is only for API development. PostgreSQL must still be running in Docker.
Use two terminals:

### Terminal 1: PostgreSQL only

```powershell
cd C:\path\to\Lumen\postgres
docker compose up postgres
```

### Terminal 2: Node.js API

```powershell
cd C:\path\to\Lumen\postgres
npm install
node server.js
```

The API will be available at `http://localhost:3000`. Do not run this option
at the same time as the Compose `api` service because both use port `3000`.

## Troubleshooting

### Database connection fails
```bash
# Check service state
docker compose ps

# Restart the complete stack
docker compose restart

# Follow API logs
docker compose logs -f api

# Follow PostgreSQL logs
docker compose logs -f postgres
```

### API returns empty data
```bash
# Verify data exists in the database
docker exec harm-tracker-postgres psql -U tracker -d harm_tracker -c "SELECT COUNT(*) FROM channels;"
```

### Port already in use
```bash
# Kill process on port 3000
netstat -ano | findstr :3000
taskkill /PID <PID> /F
```

## License

For educational and research purposes only.