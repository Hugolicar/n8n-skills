---
name: n8n-stack-railway
description: |
  Expert knowledge for Railway platform integration in n8n workflows. Covers PostgreSQL access,
  environment variables, healthchecks, deployment patterns, and service connections.
  Use when: workflows need to connect to Railway-hosted services or deploy to Railway.
---

# n8n Stack: Railway

## Overview

Railway is a platform for deploying applications with managed databases (PostgreSQL, Redis, etc.). This skill covers connecting to Railway PostgreSQL, environment variable management, and deployment patterns for n8n workflows.

## PostgreSQL Connection

### Environment Variables (Required)

| Variable | Source | Usage |
|----------|--------|-------|
| `DATABASE_URL` | Railway Variables | Full connection string (preferred) |
| `PGHOST` | Railway Variables | Hostname |
| `PGPORT` | Railway Variables | Port (usually 5432) |
| `PGDATABASE` | Railway Variables | Database name |
| `PGUSER` | Railway Variables | Username |
| `PGPASSWORD` | Railway Variables | Password |

### Connection String Format

```
postgresql://{PGUSER}:{PGPASSWORD}@{PGHOST}:{PGPORT}/{PGDATABASE}
```

### n8n Postgres Node Configuration

```json
{
  "node": "n8n-nodes-base.postgres",
  "parameters": {
    "operation": "executeQuery",
    "connectionString": "={{ $env.DATABASE_URL }}",
    "options": {
      "rejectUnauthorized": true,
      "ssl": true
    }
  }
}
```

**Critical:** Always use `={{ $env.DATABASE_URL }}` — never hardcode credentials.

## SSL/TLS Requirements

Railway PostgreSQL requires SSL. In n8n Postgres node:
- Enable SSL: **Yes**
- Reject Unauthorized: **Yes** (production) / **No** (only for debugging)

If you get `self-signed certificate` errors:
- Download Railway CA certificate
- Add to `ssl.ca` parameter in Postgres node
- Or set `rejectUnauthorized: false` temporarily (not for production)

## Environment Variables in n8n

### Accessing in Workflows

| Method | Syntax | Use Case |
|--------|--------|----------|
| Expression | `{{ $env.VAR_NAME }}` | Read env vars in nodes |
| Code Node | `$env.VAR_NAME` | Access in JavaScript/Python |
| Set Node | Value: `{{ $env.VAR_NAME }}` | Store for reuse |

### Setting in Railway

1. Railway Dashboard → Project → Variables
2. Add variable: `N8N_BASIC_AUTH_ACTIVE=true`
3. Restart service to apply

### Common n8n Variables for Railway

```bash
# Security
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secure_password_here

# Database (managed by Railway if using Railway Postgres)
DB_TYPE=postgresdb
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_HOST=${{PGHOST}}
DB_POSTGRESDB_PORT=${{PGPORT}}
DB_POSTGRESDB_USER=${{PGUSER}}
DB_POSTGRESDB_PASSWORD=${{PGPASSWORD}}

# Encryption
N8N_ENCRYPTION_KEY=random_32_char_string

# Webhook
WEBHOOK_URL=https://your-railway-app.up.railway.app/

# Timezone
GENERIC_TIMEZONE=America/Sao_Paulo
TZ=America/Sao_Paulo
```

## Healthchecks

### Railway Healthcheck URL

Configure in Railway service settings:
```
Healthcheck Path: /healthz
```

n8n responds with `200 OK` at `/healthz` when healthy.

### Custom Healthcheck Workflow

```
Webhook (GET /custom-health)
  → Postgres (SELECT 1)
  → IF (success)
    → Respond: { status: "ok", db: "connected" }
  → ELSE
    → Respond: { status: "error", db: "disconnected" }
    → HTTP 503
```

## Deployment Patterns

### N8N on Railway (Docker)

```dockerfile
FROM n8nio/n8n:latest

ENV N8N_PORT=5678
ENV N8N_BASIC_AUTH_ACTIVE=true
ENV DB_TYPE=postgresdb

EXPOSE 5678
CMD ["n8n", "start"]
```

Railway `railway.json`:
```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "DOCKERFILE"
  },
  "deploy": {
    "healthcheckPath": "/healthz",
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 3
  }
}
```

### Connecting External n8n to Railway Postgres

If n8n runs elsewhere (local, VPS, etc.):

1. In Railway Dashboard → PostgreSQL → Connect
2. Copy connection string (includes SSL params)
3. Add to n8n environment variables
4. Use `{{ $env.DATABASE_URL }}` in Postgres node

**Security:** Enable "Private Networking" in Railway if possible to avoid exposing DB publicly.

## Database Operations

### Query Patterns

| Operation | n8n Node | Example |
|-----------|----------|---------|
| Select | Postgres | `SELECT * FROM leads WHERE status = $1` |
| Insert | Postgres | `INSERT INTO logs (event, data) VALUES ($1, $2)` |
| Update | Postgres | `UPDATE leads SET status = $1 WHERE id = $2` |
| Upsert | Postgres | `INSERT ... ON CONFLICT (id) DO UPDATE ...` |

**Always use parameterized queries** — never concatenate `$json` values directly into SQL.

### Logging Table Pattern

Standard team table for workflow logging:

```sql
CREATE TABLE workflow_logs (
  id SERIAL PRIMARY KEY,
  workflow_name VARCHAR(255),
  node_name VARCHAR(255),
  execution_id VARCHAR(255),
  status VARCHAR(50),
  payload JSONB,
  error_message TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index for queries
CREATE INDEX idx_workflow_logs_name ON workflow_logs(workflow_name);
CREATE INDEX idx_workflow_logs_created ON workflow_logs(created_at DESC);
```

### Idempotency Pattern

Store processed IDs to avoid duplicate processing:

```sql
CREATE TABLE processed_events (
  event_id VARCHAR(255) PRIMARY KEY,
  source VARCHAR(100),
  processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

In workflow:
```
Webhook (incoming event)
  → Postgres (SELECT * FROM processed_events WHERE event_id = $json.id)
  → IF (found)
    → Stop (already processed)
  → ELSE
    → Process event
    → Postgres (INSERT INTO processed_events ...)
```

## Connection Pooling

Railway PostgreSQL has connection limits. Use pooling in n8n:

```json
{
  "parameters": {
    "options": {
      "pool": {
        "max": 10,
        "min": 2,
        "idleTimeoutMillis": 30000
      }
    }
  }
}
```

If you hit connection limits:
- Reduce pool size in n8n
- Enable Railway connection pooling (PgBouncer)
- Use separate read replicas for heavy queries

## Team Naming Convention

| Prefix | Use |
|--------|-----|
| `railway-deploy-{service}` | Deployment workflows |
| `railway-db-{operation}` | Database maintenance |
| `railway-health-{check}` | Health monitoring |

## Common Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| SSL connection failed | Missing SSL config | Enable SSL in Postgres node |
| Connection timeout | Network/firewall | Check Railway networking settings |
| Too many connections | Pool exhausted | Reduce pool size; add PgBouncer |
| Variable not found | Not set in Railway | Add in Railway Dashboard → Variables |
| Encoding issues | UTF-8 mismatch | Ensure `client_encoding=utf8` |
| Self-signed cert error | Railway CA not trusted | Download CA or `rejectUnauthorized: false` |

## Monitoring

### Railway Dashboard Metrics
- CPU/Memory usage
- Database connections
- Query performance
- Disk usage

### n8n + Railway Alerts
```
Cron (every 5 min)
  → Postgres (SELECT COUNT(*) FROM workflow_logs WHERE status = 'error' AND created_at > NOW() - INTERVAL '5 min')
  → IF (count > threshold)
    → HTTP (send alert to Slack/Discord)
```

## Backup & Recovery

Railway provides automated backups for PostgreSQL:
- Daily backups retained for 7 days
- Point-in-time recovery available
- Export via Railway CLI: `railway db backup`

For critical data, also export periodically:
```bash
railway db export --format=sql > backup_$(date +%Y%m%d).sql
```
