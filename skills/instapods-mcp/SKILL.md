---
name: instapods-mcp
description: >
  Deploy and debug applications on InstaPods using MCP tools. Handles first
  deploy, redeployment, failed deploy debugging, log analysis, and pod management.
  Use when deploying apps, checking pod status, viewing logs, or debugging issues
  via the InstaPods MCP server connected to Claude Desktop or Claude.ai.
---

# InstaPods MCP — Deploy & Debug

Deploy apps to InstaPods and debug issues using the InstaPods MCP server tools.

## Prerequisites

The InstaPods MCP server must be connected and authenticated. If tools like `list_pods` or `get_pod` are not available, the user needs to add the MCP server first:

**Claude Desktop** — add to config (`~/Library/Application Support/Claude/claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "instapods": {
      "url": "https://app.instapods.com/api/mcp"
    }
  }
}
```

**Claude.ai** — go to Settings > Integrations > Add MCP Server, enter `https://app.instapods.com/api/mcp`.

## Available Tools

| Tool | Description |
|------|-------------|
| `create_pod` | Create a new pod (name, preset, plan, region) |
| `get_pod` | Get pod details (status, domain, preset, plan) |
| `list_pods` | List all pods |
| `manage_pod` | Start, stop, restart, reload, or delete a pod |
| `exec_command` | Run shell commands inside a pod (as instapod user) |
| `write_file` | Write or update files inside a pod |
| `read_file` | Read file contents from a pod |
| `list_files` | List files in a pod directory |
| `get_logs` | Get application logs from a pod |
| `list_presets` | List available presets |
| `list_plans` | List pricing plans |
| `list_regions` | List deployment regions |

---

## Deploy Workflows

### First Deploy (New App)

Use when the user wants to deploy an app that doesn't have a pod yet.

**Step 1: Check if pod exists**

Use `get_pod` with the desired name. If it returns an error (not found), proceed to create it.

**Step 2: Create the pod**

Use `create_pod`:
- **name**: User's chosen name (DNS-safe, lowercase, max 63 chars)
- **preset**: Match the project type:
  - `nodejs` — package.json, tsconfig.json, next.config.js
  - `python` — requirements.txt, pyproject.toml, app.py
  - `php` — composer.json, *.php, artisan
  - `static` — index.html, *.html
- **plan**: Default `launch` ($3/mo). Use `list_plans` to show options.

**Step 3: Upload application files**

Use `write_file` to create each file in `/home/instapod/app/`. Write the most important files first:

For **nodejs**: package.json, then source files (index.js, app.js, etc.)
For **python**: requirements.txt, then source files (app.py, wsgi.py, etc.)
For **php**: composer.json (if any), then PHP files
For **static**: index.html, then CSS/JS assets

**Step 4: Install dependencies and start**

Use `manage_pod` with action `reload`. This:
- Installs dependencies (npm install / pip install / composer install)
- Detects entry points and configures the service
- Restarts the application service

**Step 5: Verify deployment**

1. Use `get_pod` — confirm status is `running`, note the domain URL
2. Use `get_logs` — check for startup errors
3. Use `exec_command` with `systemctl is-active app` (nodejs/python) or `systemctl is-active nginx` (php/static)
4. Use `exec_command` with `curl -s http://localhost:PORT/` to test locally
   - nodejs: PORT=3000, python: PORT=8000, php/static: PORT=80

Report the pod URL to the user.

### Redeploy (Existing App)

Use when the pod already exists and the user has updated code.

**Step 1**: Use `get_pod` to confirm the pod exists and is running
**Step 2**: Use `write_file` to update changed files
**Step 3**: Use `manage_pod` with action `reload`
**Step 4**: Use `get_logs` to verify no errors

### File-Only Deploy

Use when the user wants to update specific files without a full reload.

1. Use `write_file` for each file to update
2. For static sites, changes are live immediately
3. For nodejs/python/php, use `manage_pod` with `reload` or `restart` to pick up changes

---

## Debug Workflows

### 502 / Site Not Responding

**Step 1: Check pod status**

Use `get_pod` with the pod name.
- If status is `stopped`: use `manage_pod` with action `start`
- Note the preset — it determines which services and ports to check

**Step 2: Check application logs**

Use `get_logs` with the pod name. Look for:
- `EADDRINUSE` (Node.js): Port already in use
- `MODULE_NOT_FOUND` / `Cannot find module` (Node.js): Missing dependency
- `ModuleNotFoundError` (Python): Missing package
- `502 Bad Gateway` (nginx): Backend service not running
- `connect() failed` (nginx): php-fpm or app service is down
- `SIGKILL` / `OOMKilled`: Out of memory

**Step 3: Check service status**

Use `exec_command` on the pod:

For **nodejs** or **python** pods:
```
systemctl status app --no-pager
```

For **php** pods:
```
systemctl status php8.3-fpm --no-pager
```
```
systemctl status nginx --no-pager
```

For **static** pods:
```
systemctl status nginx --no-pager
```

**Step 4: Check listening ports**

Use `exec_command` with: `ss -tlnp`

Expected ports:
| Preset | Port |
|--------|------|
| nodejs | 3000 |
| python | 8000 |
| php | 80 |
| static | 80 |

**Step 5: Test locally**

Use `exec_command` with:
```
curl -s -o /dev/null -w "%{http_code}" http://localhost:PORT/
```

Replace PORT with the expected port for the preset.
- HTTP 200: App works locally, proxy issue — use `manage_pod` with `reload`
- Connection refused: App not running — check logs

**Step 6: Preset-specific fixes**

**Node.js not listening:**
- App MUST bind to `0.0.0.0`, not `127.0.0.1`
- App MUST use `process.env.PORT` (default: 3000)
- Fix: Use `write_file` to update the code, then `manage_pod` with `reload`
- Check env: `exec_command` with `env | grep -i port`

**Python not responding:**
- Must use gunicorn/uvicorn (not `python app.py`)
- Must bind to `0.0.0.0:8000`
- Fix: Add gunicorn to requirements.txt via `write_file`, then `manage_pod` with `reload`
- Check gunicorn: `exec_command` with `/home/instapod/app/venv/bin/pip list | grep gunicorn`

**PHP returning 502:**
- Check php-fpm: `exec_command` with `systemctl status php8.3-fpm`
- Check nginx errors: `exec_command` with `tail -20 /var/log/nginx/error.log`
- Laravel: Ensure `public/` exists — reload auto-detects it as webroot

**Static site showing default page:**
- Check files: use `list_files` to verify index.html is in the app directory

### Crash After Reload

**Step 1**: Use `get_logs` — check for crash messages
**Step 2**: Use `exec_command` with `systemctl status app` (or nginx for static/php)
**Step 3**: Check dependencies:
  - Node.js: `exec_command` with `ls /home/instapod/app/node_modules/ | head -5`
  - Python: `exec_command` with `/home/instapod/app/venv/bin/pip list`
  - PHP: `exec_command` with `ls /home/instapod/app/vendor/ | head -5`
**Step 4**: If deps missing, use `manage_pod` with `reload`
**Step 5**: Test entry point:
  - Node.js: `exec_command` with `node -e "require('/home/instapod/app/index.js')"`
  - Python: `exec_command` with `bash -c "source /home/instapod/app/venv/bin/activate && python -c 'import app'"`

### Resource Issues

**Step 1**: Check usage with `exec_command`:
  - Memory: `free -h`
  - Disk: `df -h /`
  - Load: `uptime`

**Step 2**: Use `get_pod` to see current plan limits

**Step 3**: If resources are exhausted, tell the user to upgrade their plan via the Dashboard or CLI:
  - `launch`: 512MB RAM, 10GB disk ($3/mo)
  - `build`: 1GB RAM, 25GB disk ($7/mo)
  - `grow`: 2GB RAM, 50GB disk ($15/mo)
  - `scale`: 4GB RAM, 100GB disk ($25/mo)
  - `turbo`: 8GB RAM, 150GB disk ($49/mo)

---

## Database Setup

Database services (MySQL, PostgreSQL, Redis) are managed through the Dashboard or CLI — the MCP server does not have service install tools.

To help the user set up a database:

1. Tell them to install via Dashboard or CLI: `instapods services add NAME --service postgresql --wait`
2. Once installed, help configure the app to connect:
   - Use `exec_command` to check the service is running: `systemctl status postgresql`
   - Use `write_file` to create/update `/home/instapod/app/.env` with the connection string
   - Use `manage_pod` with `reload` to pick up the new env vars

---

## Quick Reference

### Preset Details

| Preset | Runtime | App Port | Web Server | Dep File | Dep Tool |
|--------|---------|----------|------------|----------|----------|
| nodejs | Node 20 | 3000 | - | package.json | npm |
| python | Python 3.12 | 8000 | - | requirements.txt | pip |
| php | PHP 8.3 | 80 | nginx + fpm | composer.json | composer |
| static | - | 80 | nginx | - | - |

### Tool Quick Reference

| Task | Tool | Key Parameters |
|------|------|----------------|
| Create pod | `create_pod` | name, preset, plan |
| Check status | `get_pod` | name |
| Deploy files | `write_file` | name, path, content |
| Install deps + restart | `manage_pod` | name, action: "reload" |
| View logs | `get_logs` | name, lines, service |
| Run command | `exec_command` | name, command |
| List files | `list_files` | name, path |
| Read file | `read_file` | name, path |
| Start/stop/restart | `manage_pod` | name, action |
| Delete pod | `manage_pod` | name, action: "delete" |

---

## Important Notes

- **App root**: All app files go in `/home/instapod/app/`.
- **User context**: `exec_command` runs as the `instapod` user (not root).
- **File paths**: Must be within `/home/instapod/`, `/var/www/`, or `/tmp/`.
- **Port binding**: Node.js and Python apps MUST bind to `0.0.0.0` (not `127.0.0.1`). Use `process.env.PORT` for Node.js (default 3000), `0.0.0.0:8000` for Python.
- **Reload vs restart**: `reload` reinstalls deps and restarts services. `restart` does a quick container restart without dep installation.
- **Pod names**: DNS-safe, lowercase, max 63 characters. Letters, numbers, and hyphens only.
- **No git or SSH tools**: Git deploys and SSH are available only via the CLI or Dashboard, not MCP.
- **No service install tools**: Database services (mysql, postgresql, redis) must be installed via Dashboard or CLI.
