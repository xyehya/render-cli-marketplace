---
name: render-cli
description: |
  How to deploy apps to Render.com using the Render CLI v2 and Render REST API.
  Use this skill whenever the user wants to deploy to Render, create a Render service
  or database, set environment variables on Render, trigger a Render deploy, check
  deploy status/logs, SSH into a service, connect to a database via psql,
  or troubleshoot a Render deployment — even if they just say
  "host it on Render", "deploy to Render", or "I want to put this on Render".
  Also covers render.yaml authoring and key gotchas discovered in real deployments.
---

# Render CLI Skill

## Overview

Deploy and manage apps on Render.com using the Render CLI v2 and the Render REST API.
Covers everything from first auth check to a live URL: PostgreSQL provisioning,
env-var wiring, deploy triggering, log tailing, SSH access, and psql sessions.

## Quick Reference

| Need | Command |
|------|---------|
| Auth check | `render whoami --output json` |
| List workspaces | `render workspaces --output json` |
| Set workspace | `render workspace set <id>` |
| List all services | `render services --output json --confirm` |
| List deploy history | `render deploys list <srv-id> --output json --confirm` |
| Trigger deploy (wait) | `render deploys create <srv-id> --wait --output text --confirm` |
| Deploy specific commit | `render deploys create <srv-id> --commit <sha> --confirm` |
| Deploy Docker image | `render deploys create <srv-id> --image <url> --confirm` |
| Tail logs | `render logs -r <srv-id> --output text --confirm --limit 60` |
| SSH into service | `render ssh <srv-id> --confirm` |
| Ephemeral shell | `render ssh <srv-id> --ephemeral --confirm` |
| Connect to DB (psql) | `render psql <dpg-id> --confirm` |
| Run DB query | `render psql <dpg-id> -c "SELECT version();" --confirm` |
| Validate render.yaml | `render blueprints validate --output json` |
| API key | `cat ~/.render-api` |
| DB connection string | `GET /v1/postgres/{dpg-id}/connection-info` |
| Get owner ID | `GET /v1/owners?limit=1` |

---

## Native CLI Commands

These commands work directly via the CLI without needing curl/REST calls.

### services

```bash
# List all services and datastores in the active workspace
render services --output json --confirm

# Interactive mode (lets you select a service to deploy, tail logs, or SSH)
render services
```

### deploys

```bash
# List deploy history for a service
render deploys list <srv-id> --output json --confirm

# Trigger a new deploy and block until it finishes (non-zero exit on failure)
render deploys create <srv-id> --wait --output text --confirm

# Deploy a specific git commit (Git-backed services)
render deploys create <srv-id> --commit <sha> --confirm

# Deploy a specific Docker image (image-backed services)
render deploys create <srv-id> --image <docker-image-url> --confirm
```

Deploy statuses: `build_in_progress` → `update_in_progress` → `live` ✓ or `update_failed` ✗

### psql

```bash
# Open interactive psql session to a Postgres database
render psql <dpg-id> --confirm

# Run a single query and exit
render psql <dpg-id> --command "SELECT * FROM users LIMIT 5;" --confirm

# Output query results as JSON
render psql <dpg-id> -c "SELECT * FROM users;" --output json --confirm

# Pass extra psql flags (e.g. CSV output)
render psql <dpg-id> --confirm -- --csv
```

### ssh

```bash
# SSH into a running service instance
render ssh <srv-id> --confirm

# Ephemeral shell — isolated, does NOT run the service start command
# Useful for one-off debugging without affecting the running service
render ssh <srv-id> --ephemeral --confirm
```

### skills (manage Claude Code agent skills)

```bash
render skills list
render skills install
render skills update
render skills remove
```

---

## Global Flags & Environment Variables

| Flag / Env Var | Purpose |
|----------------|---------|
| `-o` / `--output` | Output format: `json`, `yaml`, `text`, `interactive` |
| `--confirm` | Skip all confirmation prompts (required for non-interactive use) |
| `RENDER_API_KEY` | API key for auth — alternative to `render login` |
| `RENDER_OUTPUT` | Sets the default output format for all commands |
| `RENDER_CLI_CONFIG_PATH` | Path to a custom CLI config file |

> When scripting, always pass `--output json --confirm` to avoid interactive prompts.

---

## Deployment Workflow

### 1. Auth and Workspace Setup

```bash
# Verify CLI is logged in
render whoami --output json

# List workspaces — grab the id (pattern: tea-xxxxxxxxxxxxxxxxxxxx)
render workspaces --output json

# Set the default workspace — REQUIRED before any CLI commands work
render workspace set <workspace-id>
```

Without `workspace set`, nearly all CLI commands fail with:
`"no workspace specified and no default workspace set"`

### 2. Read the API Key

```bash
RENDER_API_KEY=$(cat ~/.render-api)
```

All REST calls use `Authorization: Bearer $RENDER_API_KEY`.
Alternatively, set `export RENDER_API_KEY=$(cat ~/.render-api)` so the CLI picks it up automatically.

### 3. Push Code to GitHub

Render deploys from a Git remote — it can't pull from a local directory.

```bash
cd /path/to/project
git init && git add . && git commit -m "Initial commit"
git branch -m master main
gh repo create my-app --public --source=. --remote=origin --push
```

### 4. Get Owner ID

```bash
OWNER_ID=$(curl -s "https://api.render.com/v1/owners?limit=1" \
  -H "Authorization: Bearer $RENDER_API_KEY" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)[0]['owner']['id'])")
```

### 5. Create the PostgreSQL Database

```bash
curl -s -X POST "https://api.render.com/v1/postgres" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"my-app-db\",
    \"databaseName\": \"myapp\",
    \"ownerId\": \"$OWNER_ID\",
    \"plan\": \"free\",
    \"region\": \"oregon\",
    \"version\": \"16\"
  }"
```

Save the `id` field from the response (pattern: `dpg-xxxxxxxxxxxxxxxxxxxx`).

> `version` is required — omitting it returns `{"message":"version is required"}`.

### 6. Wait for the Database

```bash
DB_ID="dpg-xxxxxxxxxxxxxxxxxxxx"
for i in $(seq 1 18); do
  STATUS=$(curl -s "https://api.render.com/v1/postgres/$DB_ID" \
    -H "Authorization: Bearer $RENDER_API_KEY" | \
    python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  echo "DB status: $STATUS"
  if [ "$STATUS" = "available" ]; then break; fi
  sleep 10
done
```

### 7. Get the Connection String

The endpoint is `/connection-info` — `/connection-string` returns 404.

```bash
curl -s "https://api.render.com/v1/postgres/$DB_ID/connection-info" \
  -H "Authorization: Bearer $RENDER_API_KEY"
```

Response fields:
- `internalConnectionString` — use this for services running on Render
- `externalConnectionString` — use for local dev / external tools
- `password` — the generated password
- `psqlCommand` — ready-to-paste psql invocation

Or connect directly via CLI:

```bash
render psql $DB_ID --confirm
```

### 8. Create the Web Service

```bash
SERVICE_RESPONSE=$(curl -s -X POST "https://api.render.com/v1/services" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"type\": \"web_service\",
    \"name\": \"my-app\",
    \"ownerId\": \"$OWNER_ID\",
    \"repo\": \"https://github.com/USERNAME/my-app\",
    \"autoDeploy\": \"yes\",
    \"branch\": \"main\",
    \"serviceDetails\": {
      \"runtime\": \"node\",
      \"buildCommand\": \"npm install\",
      \"startCommand\": \"node server.js\",
      \"plan\": \"free\",
      \"region\": \"oregon\"
    },
    \"envVars\": [
      {\"key\": \"NODE_ENV\", \"value\": \"production\"}
    ]
  }")

SERVICE_ID=$(echo "$SERVICE_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['service']['id'])")
APP_URL=$(echo "$SERVICE_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['service']['serviceDetails']['url'])")
echo "Service: $SERVICE_ID  URL: $APP_URL"
```

### 9. Set Environment Variables

```bash
INTERNAL_DB_URL="<internalConnectionString from step 7>"

curl -s -X PUT "https://api.render.com/v1/services/$SERVICE_ID/env-vars" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "[
    {\"key\": \"NODE_ENV\", \"value\": \"production\"},
    {\"key\": \"DATABASE_URL\", \"value\": \"$INTERNAL_DB_URL\"}
  ]"
```

> The `fromDatabase` reference format (`{"fromDatabase": {...}}`) does NOT work via
> the API — it returns `{"message":"missing environment variable value"}`.
> Always use a plain `value` string.

### 10. Trigger a Deploy

```bash
# CLI — blocks until live, non-zero exit on failure (preferred)
render deploys create $SERVICE_ID --wait --output text --confirm

# REST API alternative
curl -s -X POST "https://api.render.com/v1/services/$SERVICE_ID/deploys" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"clearCache": "do_not_clear"}' | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print('Deploy:', d['id'], '| Status:', d['status'])"
```

### 11. Watch Logs Until Live

```bash
# CLI log tail (best for watching in real-time)
render logs -r $SERVICE_ID --output text --confirm --limit 60

# CLI deploy history
render deploys list $SERVICE_ID --output json --confirm

# API poll for deploy status
curl -s "https://api.render.com/v1/services/$SERVICE_ID/deploys?limit=1" \
  -H "Authorization: Bearer $RENDER_API_KEY" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)[0]['deploy']['status'])"
```

---

## Common Failures and Fixes

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `"no workspace specified"` | Workspace not set | `render workspace set <id>` |
| `"DB init failed:"` in logs | `DATABASE_URL` not configured | Set env var + redeploy |
| Deploy fails, build succeeds | Server crashes before port bind | Start HTTP listener before async DB init (see below) |
| `"version is required"` | Missing version on DB create | Add `"version": "16"` |
| `404` on `/connection-string` | Wrong endpoint | Use `/connection-info` |
| `"missing environment variable value"` | `fromDatabase` format | Use literal value string |
| IP allowlist PUT seems to work but doesn't apply | Silent no-op | Verify with a GET after; the endpoint may not support all fields |
| CLI prompts hang in scripts | Interactive mode triggered | Always pass `--confirm` and `--output json/text` |

---

## Node.js Server Pattern for Render

Start the HTTP server **before** connecting to the database. Render's health check
hits the port immediately after the process starts — if the server crashes during
DB init, Render marks the deploy as failed.

```javascript
// Good: HTTP server starts immediately, DB connects async
app.listen(PORT, () => console.log(`Running on port ${PORT}`));
initDB().catch(err => console.error('DB init failed:', err.message));

// Bad: if DB init throws, the server never starts
initDB()
  .then(() => app.listen(PORT))
  .catch(err => { process.exit(1); });
```

If `DATABASE_URL` might not be set (e.g., during first deploy before env vars are wired),
guard the DB init so the server still comes up:

```javascript
async function initDB() {
  if (!process.env.DATABASE_URL) {
    console.warn('DATABASE_URL not set — DB features disabled');
    return;
  }
  // ... connect and create tables ...
}
```

---

## render.yaml Reference

```yaml
services:
  - type: web
    name: my-app
    env: node            # use 'env', not 'runtime'
    plan: free
    buildCommand: npm install
    startCommand: node server.js
    envVars:
      - key: NODE_ENV
        value: production
      - key: DATABASE_URL
        fromDatabase:
          name: my-app-db
          property: connectionString

databases:
  - name: my-app-db
    databaseName: myapp
    plan: free
```

`render blueprints validate` validates this YAML correctly. However, in CLI v2.10
there is no `blueprint apply` command — use the REST API for programmatic creation.

---

## ID Patterns

| Resource | Prefix | Example |
|----------|--------|---------|
| Workspace / Team | `tea-` | `tea-d3qeqbili9vc73c9sngg` |
| Web Service | `srv-` | `srv-d6eqnf4r85hc73fspidg` |
| PostgreSQL DB | `dpg-` | `dpg-d6eqn7tm5p6s73bk7ju0-a` |
| Deploy | `dep-` | `dep-d6eqnfcr85hc73fspijg` |

---

## Free Tier Notes

- **Web service**: Spins down after 15 min idle; cold start is ~30–60 s
- **PostgreSQL**: Expires after **90 days** (Render emails a reminder)
- To keep the service always-on, upgrade to Starter ($7/mo)
