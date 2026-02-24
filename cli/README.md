# InstaPods CLI

Command-line tool for deploying and managing apps on InstaPods.

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/instapods/sdk/main/cli/install.sh | bash
```

Or download the binary from [GitHub Releases](https://github.com/instapods/instapods/releases).

## Authentication

```bash
instapods auth login
```

Opens a browser to sign in and stores the token locally.

## Quick Start

```bash
# Deploy current directory
instapods deploy my-app

# Check status
instapods pods get my-app

# View logs
instapods logs my-app

# Open in browser
instapods open my-app
```

## Commands

### Pods

| Command | Description |
|---------|-------------|
| `instapods pods list` | List all pods |
| `instapods pods create NAME --preset nodejs` | Create a pod |
| `instapods pods get NAME` | Get pod details |
| `instapods pods start NAME` | Start a stopped pod |
| `instapods pods stop NAME` | Stop a pod |
| `instapods pods restart NAME` | Restart a pod |
| `instapods pods reload NAME` | Reinstall deps and restart services |
| `instapods pods delete NAME` | Delete a pod |
| `instapods pods resize NAME --plan build` | Change plan |

### Deploy

| Command | Description |
|---------|-------------|
| `instapods deploy NAME` | Full deploy (create + sync + reload) |
| `instapods deploy NAME --preset nodejs` | Deploy with explicit preset |
| `instapods deploy NAME --local ./src` | Deploy from specific directory |
| `instapods deploy NAME --no-reload` | Sync files only, skip reload |

### Files

| Command | Description |
|---------|-------------|
| `instapods files ls NAME` | List files in pod |
| `instapods files cat NAME /path` | Read a file |
| `instapods files write NAME /path` | Write file (stdin or --content) |
| `instapods files upload NAME ./file` | Upload a local file |
| `instapods files sync NAME` | Sync current directory to pod |

### Logs & Exec

| Command | Description |
|---------|-------------|
| `instapods logs NAME` | View app logs |
| `instapods logs NAME -n 50` | Last 50 lines |
| `instapods logs NAME -s nginx` | Filter by service |
| `instapods exec NAME -- ls -la` | Run command in pod |
| `instapods ssh NAME` | SSH into pod |

### Services (Databases)

| Command | Description |
|---------|-------------|
| `instapods services list NAME` | List installed services |
| `instapods services add NAME -s mysql` | Install MySQL |
| `instapods services add NAME -s postgresql --wait` | Install PostgreSQL (wait for ready) |
| `instapods services creds NAME -s mysql` | Get credentials |
| `instapods services remove NAME -s mysql` | Remove service |

### Git Integration

| Command | Description |
|---------|-------------|
| `instapods git connect NAME --repo URL` | Connect GitHub repo |
| `instapods git disconnect NAME` | Disconnect repo |
| `instapods git status NAME` | Check git config |
| `instapods git deploy NAME` | Trigger deployment |
| `instapods git deployments NAME` | List deployments |
| `instapods git logs NAME` | View build log |
| `instapods git rollback NAME` | Rollback to previous |

### SSH Keys

| Command | Description |
|---------|-------------|
| `instapods ssh-keys list` | List account keys |
| `instapods ssh-keys list NAME` | List pod keys |
| `instapods ssh-keys add NAME` | Add local key to pod |

## Presets

| Preset | Runtime | Detected From |
|--------|---------|---------------|
| `nodejs` | Node.js 20 | package.json, tsconfig.json, next.config.js |
| `python` | Python 3.12 | requirements.txt, pyproject.toml, app.py |
| `php` | PHP 8.3 | composer.json, *.php, artisan |
| `static` | nginx | index.html, *.html |

## Plans

| Plan | Price | CPU | RAM | Disk |
|------|-------|-----|-----|------|
| launch | $3/mo | 1 vCPU | 512MB | 10GB |
| build | $7/mo | 1 vCPU | 1GB | 25GB |
| grow | $15/mo | 2 vCPU | 2GB | 50GB |
| scale | $25/mo | 2 vCPU | 4GB | 100GB |
| turbo | $49/mo | 4 vCPU | 8GB | 150GB |

## Global Flags

| Flag | Description |
|------|-------------|
| `--json` | Output in JSON format |
| `--api URL` | Override API endpoint |
| `--token TOKEN` | Use specific auth token |

## Full Documentation

[docs.instapods.com](https://docs.instapods.com)
