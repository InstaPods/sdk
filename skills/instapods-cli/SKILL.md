---
name: instapods-cli
description: >
  Deploy and debug applications on InstaPods using the instapods CLI.
  Handles first deploy, redeployment, failed deploy debugging, log analysis,
  and database setup. Use when deploying apps, checking pod status, viewing
  logs, or debugging issues via the command line.
allowed-tools: Bash(instapods *)
---

# InstaPods CLI — Deploy & Debug

Deploy apps to InstaPods and debug issues using the `instapods` CLI.

## Prerequisites

Before starting any workflow, verify the CLI is installed and authenticated:

```bash
instapods --version
```

If not installed, install with:

```bash
curl -fsSL https://raw.githubusercontent.com/instapods/sdk/main/cli/install.sh | bash
```

Verify authentication:

```bash
instapods pods list
```

If authentication fails, run:

```bash
instapods auth login
```

---

## Deploy Workflows

### First Deploy (New App)

Use when the user wants to deploy an app that doesn't have a pod yet.

**Step 1: Deploy in one command**

The `deploy` command handles everything: creates the pod, syncs files, installs deps, and reloads.

```bash
instapods deploy NAME
```

This auto-detects the preset from project files:
- `package.json`, `tsconfig.json`, `next.config.js` -> `nodejs`
- `composer.json`, `*.php`, `artisan` -> `php`
- `requirements.txt`, `pyproject.toml`, `app.py` -> `python`
- `index.html`, `*.html` -> `static`

If auto-detection fails or the user wants a specific preset:

```bash
instapods deploy NAME --preset nodejs
```

To deploy from a specific directory:

```bash
instapods deploy NAME --local ./src
```

**Step 2: Verify deployment**

```bash
instapods pods get NAME
```

Check that `Status: running` and note the `Domain: https://...` URL.

**Step 3: Check logs for errors**

```bash
instapods logs NAME
```

Look for startup errors, missing modules, port binding issues.

**Step 4: Open in browser**

```bash
instapods open NAME
```

### Redeploy (Existing App)

Use when the pod already exists and the user has updated code.

```bash
instapods deploy NAME
```

The deploy command detects the existing pod, skips creation, syncs changed files, and reloads. Output shows "Pod exists (nodejs · running)".

If only syncing files without reloading:

```bash
instapods deploy NAME --no-reload
```

### Git Deploy

Use when the user wants to connect a GitHub repo for automatic deployments.

**Step 1: Connect repository**

```bash
instapods git connect NAME --repo https://github.com/user/repo
```

For private repos, add a personal access token:

```bash
instapods git connect NAME --repo https://github.com/user/repo --token ghp_xxx
```

**Step 2: Trigger a deploy**

```bash
instapods git deploy NAME
```

**Step 3: Check deployment status**

```bash
instapods git deployments NAME
```

**Step 4: View build logs if deploy failed**

```bash
instapods git logs NAME
```

### File-Only Deploy

Use when the user wants to upload specific files without a full deploy cycle.

Upload a single file:

```bash
instapods files upload NAME ./index.html
```

Sync an entire directory:

```bash
instapods files sync NAME --local ./dist
```

Then reload the service:

```bash
instapods pods reload NAME
```

---

## Debug Workflows

### 502 / Site Not Responding

**Step 1: Check pod status**

```bash
instapods pods get NAME
```

If status is `stopped`, start it:

```bash
instapods pods start NAME
```

**Step 2: Check application logs**

```bash
instapods logs NAME
```

Look for crash messages, uncaught exceptions, or startup failures.

**Step 3: Check if the app is listening on the correct port**

```bash
instapods exec NAME -- ss -tlnp
```

Expected listening ports by preset:

| Preset | Expected Port | Service |
|--------|--------------|---------|
| nodejs | 3000 | app (node) |
| python | 8000 | app (gunicorn/uvicorn) |
| php | 80 | nginx + php8.3-fpm |
| static | 80 | nginx |

**Step 4: Test the app locally inside the pod**

```bash
instapods exec NAME -- curl -s http://localhost:3000/    # nodejs
instapods exec NAME -- curl -s http://localhost:8000/    # python
instapods exec NAME -- curl -s http://localhost:80/      # php/static
```

If the local curl works but the public URL doesn't, it's a proxy issue. Reload:

```bash
instapods pods reload NAME
```

**Step 5: Preset-specific fixes**

**Node.js not listening:**
- App MUST bind to `0.0.0.0`, not `127.0.0.1` or `localhost`
- App MUST use `process.env.PORT` (default: 3000)
- Check for `EADDRINUSE` — another process using port 3000
- Check for `MODULE_NOT_FOUND` — run reload to install deps

```bash
instapods exec NAME -- env | grep -i port
instapods pods reload NAME
```

**Python not responding:**
- Use gunicorn or uvicorn for production (not `python app.py`)
- Must bind to `0.0.0.0:8000`
- Add gunicorn to `requirements.txt`
- Check for import errors in logs

```bash
instapods exec NAME -- /home/instapod/app/venv/bin/pip list | grep gunicorn
instapods exec NAME -- ls /home/instapod/app/
```

**PHP returning 502:**
- Check if php8.3-fpm is running
- Check nginx error log
- For Laravel: ensure `public/` is the webroot

```bash
instapods exec NAME -- systemctl status php8.3-fpm
instapods exec NAME -- cat /var/log/nginx/error.log | tail -20
```

**Static site showing default page:**
- Ensure `index.html` is in `/home/instapod/app/`

```bash
instapods exec NAME -- ls -la /home/instapod/app/
```

### Crash After Reload

Use when the app was working but crashes after a code update.

**Step 1: Check service status**

```bash
instapods exec NAME -- systemctl status app    # nodejs/python
instapods exec NAME -- systemctl status nginx   # static/php
```

**Step 2: View recent logs**

```bash
instapods logs NAME -n 50
```

**Step 3: Check if dependencies are missing**

```bash
# Node.js
instapods exec NAME -- ls /home/instapod/app/node_modules/ | head -5

# Python
instapods exec NAME -- /home/instapod/app/venv/bin/pip list

# PHP
instapods exec NAME -- ls /home/instapod/app/vendor/ | head -5
```

If deps are missing, reload reinstalls them:

```bash
instapods pods reload NAME
```

**Step 4: Check for syntax errors or bad imports**

```bash
# Node.js — test if the entry point loads
instapods exec NAME -- node -e "require('/home/instapod/app/index.js')"

# Python — test if the app module imports
instapods exec NAME -- bash -c "source /home/instapod/app/venv/bin/activate && python -c 'import app'"
```

### Git Deploy Failed

**Step 1: Check deployment history**

```bash
instapods git deployments NAME
```

**Step 2: View build log of the failed deploy**

```bash
instapods git logs NAME
```

Common failures:
- **Clone failed**: Check repo URL and access token
- **npm install failed**: Check package.json for invalid deps
- **Build failed**: Check build script in package.json
- **Health check failed**: App didn't start after deploy

**Step 3: Rollback to previous deployment**

```bash
instapods git rollback NAME
```

### Resource Issues

Use when the app is slow or running out of memory/disk.

**Step 1: Check current resource usage**

```bash
instapods exec NAME -- free -h         # memory
instapods exec NAME -- df -h /         # disk
instapods exec NAME -- uptime          # load average
```

**Step 2: Check current plan limits**

```bash
instapods pods get NAME
```

Note the Plan, CPU, Memory, and Disk values.

**Step 3: Upgrade plan if needed**

```bash
instapods pods resize NAME --plan build
```

Available plans: `launch` ($3), `build` ($7), `grow` ($15), `scale` ($25), `turbo` ($49).

---

## Database Setup

### Add a Database Service

Available services: `mysql`, `postgresql`, `redis`.

```bash
instapods services add NAME --service postgresql --wait
```

The `--wait` flag blocks until installation completes (takes 30-60s).

### Get Credentials

```bash
instapods services creds NAME --service postgresql
```

This prints host, port, username, password, database, and connection string.

### Set Environment Variables

Write the database URL to the app's `.env` file:

```bash
# Get the credentials first
instapods services creds NAME --service postgresql

# Then write .env with the connection string
echo "DATABASE_URL=postgresql://USER:PASS@localhost:5432/DB" | instapods files write NAME /home/instapod/app/.env
```

Then reload:

```bash
instapods pods reload NAME
```

---

## Quick Reference

### Preset Details

| Preset | Runtime | App Port | Web Server | Dep File | Dep Tool |
|--------|---------|----------|------------|----------|----------|
| nodejs | Node 20 | 3000 | - | package.json | npm |
| python | Python 3.12 | 8000 | - | requirements.txt | pip |
| php | PHP 8.3 | 80 | nginx + fpm | composer.json | composer |
| static | - | 80 | nginx | - | - |

### Common Commands

| Task | Command |
|------|---------|
| Deploy app | `instapods deploy NAME` |
| Check status | `instapods pods get NAME` |
| View logs | `instapods logs NAME` |
| Run command | `instapods exec NAME -- COMMAND` |
| List files | `instapods files ls NAME` |
| Read a file | `instapods files cat NAME /path/to/file` |
| Upload file | `instapods files upload NAME ./file` |
| Sync directory | `instapods files sync NAME --local ./dir` |
| Reload service | `instapods pods reload NAME` |
| Restart pod | `instapods pods restart NAME` |
| SSH into pod | `instapods ssh NAME` |
| Open in browser | `instapods open NAME` |
| Add database | `instapods services add NAME --service mysql --wait` |
| View DB creds | `instapods services creds NAME --service mysql` |
| Connect git repo | `instapods git connect NAME --repo URL` |
| Trigger git deploy | `instapods git deploy NAME` |

### CLI Flags

| Flag | Effect |
|------|--------|
| `--json` | Machine-readable JSON output |
| `--wait` / `-w` | Block until operation completes |

---

## Important Notes

- **App root**: All app files live in `/home/instapod/app/`.
- **User context**: Commands via `instapods exec` run as the `instapod` user (not root).
- **`.env` handling**: Deploy excludes `.env` by default. Upload separately with `instapods files write`.
- **Port binding**: Node.js and Python apps MUST bind to `0.0.0.0` (not `127.0.0.1`). Use `process.env.PORT` for Node.js, `0.0.0.0:8000` for Python.
- **Reload vs restart**: `reload` reinstalls deps and restarts services. `restart` does a quick container restart without dep installation.
- **Excluded files**: Deploy excludes `node_modules`, `vendor`, `.git`, `.env`, `__pycache__`, `.DS_Store`, `.venv`, `venv` by default. Customize with `--exclude`.
- **Pod names**: DNS-safe, lowercase, max 63 characters. Letters, numbers, and hyphens only.
