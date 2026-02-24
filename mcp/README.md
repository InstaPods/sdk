# InstaPods MCP Server

Connect Claude Desktop or Claude.ai to InstaPods via the [Model Context Protocol](https://modelcontextprotocol.io/).

## Setup

### Claude Desktop

Add the InstaPods MCP server to your config file:

**macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "instapods": {
      "url": "https://app.instapods.com/api/mcp"
    }
  }
}
```

Restart Claude Desktop after saving.

### Claude.ai

1. Go to [Claude.ai Settings](https://claude.ai/settings) > Integrations
2. Click "Add MCP Server"
3. Enter the URL: `https://app.instapods.com/api/mcp`
4. Complete the OAuth flow when prompted

## Authentication

The MCP server uses OAuth 2.0 with PKCE for authentication.

On first connection, Claude will redirect you to the InstaPods login page. Sign in with your InstaPods account, select your team, and authorize access. The token is cached automatically — you only need to authenticate once.

If you need to re-authenticate, remove the server from your config and re-add it.

## Available Tools

| Tool | Description |
|------|-------------|
| `list_pods` | List all your pods |
| `get_pod` | Get details of a specific pod |
| `create_pod` | Create a new pod |
| `manage_pod` | Start, stop, restart, reload, or delete a pod |
| `exec_command` | Run a shell command inside a pod |
| `write_file` | Write or update files inside a pod |
| `list_files` | List files in a pod directory |
| `read_file` | Read file contents from a pod |
| `get_logs` | Get application logs from a pod |
| `list_presets` | List available presets (static, php, nodejs, python) |
| `list_plans` | List pricing plans |
| `list_regions` | List deployment regions |

## Built-in Prompts

The MCP server includes guided prompts for common workflows:

### deploy-app

Guided workflow for deploying an application. Walks through preset selection, pod creation, file deployment, and verification.

**Usage:** Select the `deploy-app` prompt, provide the preset and app name.

**Example conversation:**
> Use the deploy-app prompt with preset "nodejs" and app name "my-api"

Claude will guide you through creating the pod, uploading files, and verifying the deployment.

### debug-pod

Systematic diagnosis of a pod that isn't working correctly. Checks status, logs, services, files, and network configuration.

**Usage:** Select the `debug-pod` prompt with the pod name.

**Example conversation:**
> Use the debug-pod prompt for pod "my-api"

Claude will walk through a step-by-step diagnosis, identify the issue, and suggest fixes.

### getting-started

Overview of InstaPods — what's available, how to create your first pod, and tips for effective use.

## Example Conversations

**Deploy a Node.js app:**
```
You: Create a nodejs pod called "my-api" and deploy my Express server
Claude: [Uses create_pod, write_file, manage_pod tools to set up the app]
```

**Debug a failing app:**
```
You: My pod "my-api" is returning 502 errors
Claude: [Uses get_pod, get_logs, exec_command to diagnose and fix]
```

**Add a database:**
```
You: I need a PostgreSQL database for my pod
Claude: [Note: Database services are managed through the Dashboard or CLI —
        MCP can help check if services exist and configure environment variables]
```

## Notes

- The MCP server runs on `https://app.instapods.com/api/mcp`
- All operations are scoped to your authenticated team
- Commands run inside pods execute as the `instapod` user (not root)
- File writes are restricted to `/home/instapod/`, `/var/www/`, and `/tmp/`
- Database services (MySQL, PostgreSQL, Redis) are managed through the Dashboard or CLI

## Troubleshooting

**"Authentication required" errors:**
Remove the MCP server from your config and re-add it to trigger a fresh OAuth flow.

**Tools not appearing:**
Ensure the URL is exactly `https://app.instapods.com/api/mcp` (no trailing slash).

**Connection timeouts:**
The MCP server uses Streamable HTTP transport. Ensure your network allows HTTPS connections to `app.instapods.com`.

## Full Documentation

For complete API docs, CLI reference, and guides: [docs.instapods.com](https://docs.instapods.com)
