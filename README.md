# InstaPods SDK

Tools and integrations for deploying and managing apps on [InstaPods](https://instapods.com) with AI assistants.

InstaPods is container hosting for developers — deploy Node.js, Python, PHP, and static apps with a single command. This SDK provides two ways to integrate InstaPods with Claude:

## Integration Paths

### Claude Code (CLI Skill)

A skill that teaches Claude Code how to deploy and debug apps using the `instapods` CLI.

**Setup:**

1. Install the CLI:
   ```bash
   curl -fsSL https://raw.githubusercontent.com/instapods/sdk/main/cli/install.sh | bash
   instapods auth login
   ```

2. Copy the skill to your project:
   ```bash
   mkdir -p .claude/skills/instapods-cli
   curl -fsSL https://raw.githubusercontent.com/instapods/sdk/main/skills/instapods-cli/SKILL.md \
     -o .claude/skills/instapods-cli/SKILL.md
   ```

3. Ask Claude Code to deploy:
   ```
   > deploy my app to instapods
   > my app is returning 502, help me debug
   > add a PostgreSQL database to my pod
   ```

### Claude Desktop / Claude.ai (MCP Skill)

Two options — use the MCP skill for guided workflows, or connect directly via MCP.

**Option A: MCP Skill (recommended)**

Copy the skill to your project:
```bash
mkdir -p .claude/skills/instapods-mcp
curl -fsSL https://raw.githubusercontent.com/instapods/sdk/main/skills/instapods-mcp/SKILL.md \
  -o .claude/skills/instapods-mcp/SKILL.md
```

**Option B: Direct MCP connection**

Add to your Claude Desktop config (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "instapods": {
      "url": "https://app.instapods.com/api/mcp"
    }
  }
}
```

On first use, Claude will walk you through OAuth authentication. See [mcp/README.md](mcp/README.md) for the full setup guide.

The MCP server includes built-in prompts:
- **deploy-app** — Guided deployment workflow
- **debug-pod** — Systematic pod diagnosis
- **getting-started** — Overview of available tools and resources

## What's in This Repo

```
sdk/
├── README.md                              # This file
├── skills/
│   ├── instapods-cli/
│   │   └── SKILL.md                       # Skill for Claude Code (uses CLI)
│   └── instapods-mcp/
│       └── SKILL.md                       # Skill for Claude Desktop/Claude.ai (uses MCP tools)
├── mcp/
│   └── README.md                          # MCP server setup guide
└── cli/
    └── README.md                          # CLI quick reference
```

## Links

- [InstaPods Dashboard](https://app.instapods.com)
- [Documentation](https://docs.instapods.com)
- [CLI Releases](https://github.com/instapods/instapods/releases)

## License

MIT
