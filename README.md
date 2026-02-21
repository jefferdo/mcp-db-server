# mcp-db-server

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)

An MCP (Model Context Protocol) server that exposes relational databases (PostgreSQL, MySQL, SQLite) to AI agents. Supports natural language queries, full CRUD operations, and dynamic database switching — all over the modern **streamable HTTP** MCP transport.

## Features

- **Multi-Database Support**: PostgreSQL, MySQL, SQLite
- **Natural Language to SQL**: Convert plain English to SQL (rule-based with optional HuggingFace transformers)
- **Full CRUD via MCP Tools**: Query, insert, update, delete, create tables
- **Dynamic DB Switching**: Change the connected database at runtime via an MCP tool
- **REST API**: FastAPI endpoints for direct HTTP access (health, schema, NL query)
- **Streamable HTTP Transport**: Modern MCP transport, compatible with VS Code, Claude Desktop, Google Antigravity, and any MCP client
- **Reverse Proxy**: nginx handles all traffic on a single port (8000)
- **DNS Rebinding Protection**: Per-container `MCP_ALLOWED_HOSTS` allowlist
- **Docker Watch Mode**: Live sync/restart during development without full rebuilds
- **Safety First**: Read-only `execute_sql`, dangerous ops gated behind `execute_unsafe_sql`

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  AI Client (VS Code / Claude Desktop / Antigravity)      │
│  MCP endpoint: http://<host>:8000/mcp                    │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP :8000
              ┌────────▼────────┐
              │   mcp-nginx     │  nginx reverse proxy
              └──┬──────────┬───┘
     /mcp        │          │  /health, /mcp/*, /docs
    ┌────────────▼──┐  ┌────▼────────────────┐
    │   mcp-http    │  │     mcp-api          │
    │  (port 8002)  │  │   (port 8000)        │
    │  FastMCP      │  │   FastAPI REST       │
    │  streamable   │  │                      │
    │  HTTP         │  │                      │
    └───────┬───────┘  └────────┬─────────────┘
            └─────────┬─────────┘
                      │
              ┌───────▼────────┐
              │    Database     │
              │ MySQL/PG/SQLite │
              └────────────────┘
```

### Containers

| Container   | Image                       | Role                                    | Internal Port |
| ----------- | --------------------------- | --------------------------------------- | ------------- |
| `mcp-nginx` | `nginx:alpine`              | Reverse proxy, single public port       | 8000          |
| `mcp-http`  | `mcp-database-server:latest`| MCP server (streamable HTTP transport)  | 8002          |
| `mcp-api`   | `mcp-database-server:latest`| FastAPI REST API                        | 8000          |

## Quick Start

### Prerequisites

- Docker + Docker Compose
- An existing MySQL, PostgreSQL or SQLite database

### Start

```bash
git clone https://github.com/Souhar-dya/mcp-db-server.git
cd mcp-db-server

# Set your database URL in docker-compose.yml (DATABASE_URL env var)
docker compose up --build
```

All traffic is served on **port 8000**:
- `http://localhost:8000/mcp` → MCP streamable HTTP endpoint
- `http://localhost:8000/health` → health check
- `http://localhost:8000/docs` → FastAPI Swagger UI

### Development (live reload)

```bash
docker compose watch
```

File changes are synced into running containers automatically:
- `app/` → synced to `mcp-api` (uvicorn `--reload` picks it up) and `mcp-http` (container restarts)
- `mcp_server.py` → synced to `mcp-http` (container restarts)
- `nginx/nginx.conf` → synced to `mcp-nginx` (container restarts)
- `requirements.txt` → triggers a full image rebuild

## Connecting AI Clients

### VS Code

In `.vscode/mcp.json`:

```json
{
  "servers": {
    "mcp-db-server": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

### Google Antigravity

Open the MCP store → **Manage MCP Servers** → **View raw config**, then add:

```json
{
  "mcpServers": {
    "mcp-db-server": {
      "url": "http://mcp-nginx:8000/mcp"
    }
  }
}
```

> Use `mcp-nginx` (Docker service name) when Antigravity itself runs inside a container on the same Docker network. Use `localhost` when it runs on the host machine.

### Claude Desktop

In `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "mcp-db-server": {
      "command": "python",
      "args": ["mcp_server.py", "--transport", "stdio", "--database-url", "sqlite+aiosqlite:///mydb.db"]
    }
  }
}
```

> See `claude_desktop_config_example.json` in the repo for a full example.

## MCP Tools

These tools are available to any connected AI agent:

| Tool | Description |
|---|---|
| `query_database` | Convert a natural language question to SQL and execute it |
| `list_tables` | List all tables with column counts |
| `describe_table` | Get schema (columns, types, nullability) for a table |
| `execute_sql` | Run a raw SELECT query (safe — blocks DROP/DELETE/UPDATE) |
| `execute_unsafe_sql` | Run any SQL including DDL and DML — use with caution |
| `create_table` | Create a new table |
| `insert_data` | Insert a row into a table |
| `update_data` | Update rows in a table |
| `delete_data` | Delete rows from a table |
| `connect_to_database` | Switch to a different database at runtime |
| `get_current_database_info` | Show current connection URL, type, and table stats |
| `get_connection_examples` | Get example connection strings for all supported databases |

## MCP Resources

| Resource URI | Description |
|---|---|
| `database://tables` | Live list of all tables and column counts |
| `database://schema` | Full schema of all tables as JSON |

## REST API Endpoints

Served by `mcp-api` via nginx at `http://localhost:8000`:

| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Service health and version |
| `/mcp/list_tables` | GET | All tables with column counts |
| `/mcp/describe/{table_name}` | GET | Schema for a specific table |
| `/mcp/query` | POST | Execute a natural language query |
| `/mcp/tables/{table_name}/sample` | GET | Sample rows from a table (default 5) |
| `/docs` | GET | Swagger UI |

### Example REST calls

```bash
# Health check
curl http://localhost:8000/health

# List tables
curl http://localhost:8000/mcp/list_tables

# Describe a table
curl http://localhost:8000/mcp/describe/orders

# Natural language query
curl -X POST http://localhost:8000/mcp/query \
  -H "Content-Type: application/json" \
  -d '{"nl_query": "show top 5 customers by total orders"}'

# Sample data
curl "http://localhost:8000/mcp/tables/orders/sample?limit=10"
```

## Configuration

### Environment Variables

| Variable | Description | Default |
|---|---|---|
| `DATABASE_URL` | Full database connection URL | *(required)* |
| `MCP_ALLOWED_HOSTS` | Comma-separated allowed `Host` header values for DNS rebinding protection | Auto-set from `--host`/`--port` |
| `PYTHONUNBUFFERED` | Disable Python output buffering | `1` |

### `DATABASE_URL` formats

```bash
# PostgreSQL
postgresql+asyncpg://user:password@host:5432/dbname

# MySQL
mysql+aiomysql://user:password@host:3306/dbname

# SQLite
sqlite+aiosqlite:///path/to/database.db

# PostgreSQL with SSL (cloud: Neon, Supabase, Aiven)
postgresql+asyncpg://user:password@host:5432/dbname?sslmode=require

# MySQL cloud (Aiven, PlanetScale)
mysql+aiomysql://user:password@host:3306/dbname
```

### `MCP_ALLOWED_HOSTS`

The MCP server enforces DNS rebinding protection — requests are rejected if the `Host` header is not in the allowlist. The `docker-compose.yml` sets this automatically for the reverse-proxy setup. Override it if you expose the server under a custom domain:

```yaml
- MCP_ALLOWED_HOSTS=mycompany.internal,mycompany.internal:8000,localhost,localhost:8000
```

## Transport Modes

`mcp_server.py` supports three transport modes via `--transport`:

| Mode | Flag | Use case |
|---|---|---|
| `stdio` | `--transport stdio` | Claude Desktop (local subprocess) |
| `sse` | `--transport sse` | Legacy SSE clients |
| `streamable-http` | `--transport streamable-http` | VS Code, Antigravity, modern clients (default in Docker) |

```bash
# stdio (Claude Desktop)
python mcp_server.py --transport stdio --database-url sqlite+aiosqlite:///mydb.db

# Streamable HTTP (standalone, no nginx)
python mcp_server.py --transport streamable-http --host 0.0.0.0 --port 8002
```

## Security

- **DNS rebinding protection**: `Host` header validated against `MCP_ALLOWED_HOSTS`
- **Read-only by default**: `execute_sql` blocks `DROP`, `DELETE`, `TRUNCATE`, `ALTER`, `UPDATE`
- **Unsafe operations explicit**: Destructive SQL requires the separate `execute_unsafe_sql` tool — the AI agent must consciously choose it
- **Result limits**: NL queries capped at 50 rows; raw queries display the first 20
- **Input sanitization**: Queries run through SQLAlchemy with parameterised execution

## Project Structure

```
mcp-db-server/
├── app/
│   ├── __init__.py
│   ├── server.py          # FastAPI REST API (mcp-api container)
│   ├── db.py              # Database connection + query execution
│   └── nl_to_sql.py       # Natural language → SQL conversion
├── nginx/
│   └── nginx.conf         # Reverse proxy config (dynamic upstream resolution)
├── config/                # Optional JSON config files for database URLs
├── data/                  # Mounted data directory
├── mcp_server.py          # FastMCP server (mcp-http container)
├── docker-compose.yml
├── Dockerfile
└── requirements.txt
```

## Changelog

### v2.0.0 (2026-02-21)

- **Breaking**: Replaced SSE transport with **streamable HTTP** (`/mcp`) — single endpoint works for all modern MCP clients
- **Added**: Google Antigravity support via streamable HTTP
- **Added**: `mcp-http` container running FastMCP with streamable HTTP transport
- **Added**: nginx reverse proxy with Docker DNS-based dynamic upstream resolution (`resolver 127.0.0.11`)
- **Added**: Docker Compose `watch` mode for live development sync
- **Added**: `MCP_ALLOWED_HOSTS` environment variable for configurable DNS rebinding protection
- **Added**: `--transport streamable-http` flag to `mcp_server.py`
- **Removed**: `mcp-sse` container (SSE transport still available as a flag but not deployed by default)
- **Fixed**: nginx startup failure when upstream containers aren't ready (replaced static `upstream` blocks with dynamic `set $var` + runtime resolver)

### v1.3.0 (2025-12-24)

- Fixed import path issues in Docker container
- Robust path resolution in `mcp_server.py` for both local and Docker environments

### v1.2.0 (2025-11-03)

- Fixed `Could not locate column in row` error with MySQL `describe_table`
- Switched to index-based row access for cross-database compatibility

### v1.1.0 (2025-09-28)

- Fixed `str can't be used in 'await' expression` in MCP server
- NL query processing now works correctly with Claude Desktop

### v1.0.0 (2025-09-25)

- Initial release: FastAPI REST API, NL→SQL, Docker, multi-database support

## License

Apache License 2.0 — see [LICENSE](LICENSE).

