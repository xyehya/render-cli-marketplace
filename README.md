# xyehya-skills — Claude Code Plugin Marketplace

A personal Claude Code skill marketplace by [Yehya Karout](https://github.com/xyehya).

## Install

```
/plugin marketplace add xyehya/render-cli-marketplace
```

## Available Skills

### `render-cli`

Deploy apps to [Render.com](https://render.com) using the CLI v2 and REST API.

Covers the full workflow:
- Auth and workspace setup
- PostgreSQL database provisioning
- Web service creation
- Getting DB connection strings
- Setting environment variables
- Triggering and monitoring deploys
- Real-world gotchas (silent failures, correct API endpoints, server startup patterns)

**Install:**
```
/plugin install render-cli@xyehya-skills
```

**Triggers when you say things like:**
- "deploy this to Render"
- "host it on Render"
- "set up a Render service"
- "create a Render database"
