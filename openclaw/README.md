# OpenClaw Skills

Cursor skill for integration with OpenClaw: self-hosted gateway that connects WhatsApp, Telegram, Discord and other channels to AI agents.

## Description

This skill gives Cursor the knowledge needed to work with OpenClaw, enabling it to generate correct code and configuration for:

- Installing and configuring the Gateway (CLI, onboarding, service)
- Managing channels (WhatsApp, Telegram, Discord, Slack, etc.)
- Configuring models and authentication (Anthropic, OpenAI, OpenRouter)
- Using agent tools (browser, exec, messaging, web_search, cron, etc.)
- Multi-agent routing and isolated workspaces
- Integrating with the HTTP API (OpenAI-compatible Chat Completions)
- Webhooks and scripting (Node.js, PHP/Laravel)

## Installation

To use this skill in Cursor, copy the `openclaw` folder (or just the `skills` subfolder) to your project or configure the path in Cursor preferences:

```
~/.cursor/skills/openclaw-skill/SKILL.md
```

Make sure Cursor has access to the folder containing `SKILL.md`, `cli-reference.md` and `examples.md` (for example, by copying `openclaw/skills` and `openclaw/examples` under `~/.cursor/skills/openclaw-skill/`).

## Usage

Once installed, Cursor will automatically detect when you are working with OpenClaw and use this skill to:

1. **Generate code with the correct structure** — Docstrings, error handling, camelCase naming
2. **Use the correct CLI commands** — Knows `openclaw onboard`, `channels`, `models`, `gateway`, `agent`, etc.
3. **Configure openclaw.json** — Structure of `channels`, `agents`, `tools`, `gateway`
4. **Implement authentication** — Gateway token/password, environment variables
5. **Handle errors** — 401, 429 with Retry-After, diagnostics with `doctor` and `status`

## Main CLI Commands

| Category    | Command                          | Description                          |
| ----------- | -------------------------------- | ------------------------------------ |
| Setup       | `openclaw onboard`               | Installation wizard                   |
| Setup       | `openclaw configure`             | Configuration wizard                  |
| Config      | `openclaw config get/set/unset`  | Read or write config                  |
| Gateway     | `openclaw gateway`               | Run Gateway                           |
| Gateway     | `openclaw gateway start/stop`    | Gateway service                       |
| Gateway     | `openclaw health`                | Gateway health                        |
| Channels    | `openclaw channels list/login/add/remove` | Channels                   |
| Models      | `openclaw models list/set/status` | Models and auth                      |
| Agents      | `openclaw agents list/add/delete` | Isolated agents                      |
| Agent       | `openclaw agent --message "…"`   | One agent turn                        |
| Sessions    | `openclaw sessions`              | List sessions                         |
| Cron        | `openclaw cron list/add/rm`      | Scheduled tasks                       |
| Browser     | `openclaw browser start/snapshot/screenshot` | Browser              |
| Logs        | `openclaw logs --follow`         | Gateway logs                          |
| Doctor      | `openclaw doctor`                | Diagnostics and fixes                 |
| Status      | `openclaw status --all`          | Full status                           |

## Supported Languages

The skill includes examples and guides for:

- **JavaScript/Node.js** — HTTP Chat Completions client, CLI scripts, webhooks, programmatic config
- **PHP/Laravel** — Gateway service, webhooks, Artisan commands

## Configuration

### Environment Variables

```bash
# Gateway (for HTTP API and scripts)
OPENCLAW_GATEWAY_URL=http://127.0.0.1:18789
OPENCLAW_GATEWAY_TOKEN=your_gateway_token
OPENCLAW_AGENT_ID=main

# Optional: config path (for commands that modify openclaw.json)
OPENCLAW_CONFIG_PATH=~/.openclaw/openclaw.json
```

### Laravel

In `config/services.php`:

```php
'openclaw' => [
    'gateway_url' => env('OPENCLAW_GATEWAY_URL', 'http://127.0.0.1:18789'),
    'gateway_token' => env('OPENCLAW_GATEWAY_TOKEN'),
    'agent_id' => env('OPENCLAW_AGENT_ID', 'main'),
],
```

### Enable HTTP API (Chat Completions)

In `~/.openclaw/openclaw.json`:

```json5
{
  "gateway": {
    "http": {
      "endpoints": {
        "chatCompletions": { "enabled": true }
      }
    }
  }
}
```

Restart the Gateway after changing the config.

## Skill Structure

```
openclaw/
├── skills/
│   └── SKILL.md           # Main skill definition
├── examples/
│   ├── cli-reference.md   # Detailed CLI reference
│   └── examples.md        # JavaScript and PHP/Laravel examples
└── README.md              # This file
```

## Code Conventions

This skill follows these conventions:

- **Naming**: camelCase for functions and variables
- **Documentation**: Docstrings (PHPDoc / JSDoc) on all functions
- **Security**: Tokens and passwords in environment variables, never hardcoded
- **Error handling**: Check 401, 429 (and Retry-After); use `openclaw doctor` and `status` for diagnostics

## Integration with Claude

### Claude Desktop (MCP)

You can use an MCP server that talks to your Gateway (for example, an HTTP proxy) and configure `claude_desktop_config.json` with the OpenClaw URL and token. The skill itself is documentation; include it as context or in Project knowledge.

### Claude API (System Prompt)

Include the skill content in the system prompt:

```python
from pathlib import Path

skill_content = Path("openclaw/skills/SKILL.md").read_text()
cli_reference = Path("openclaw/examples/cli-reference.md").read_text()

# In system:
# f"You are an expert OpenClaw assistant.\n\n{skill_content}\n\n## CLI Reference\n{cli_reference}"
```

### Claude Projects

1. Create a project in Claude
2. In "Project knowledge", upload:
   - `skills/SKILL.md`
   - `examples/cli-reference.md`
   - `examples/examples.md`
3. Claude will use this knowledge in project conversations

### Example prompt with context

```
Context: I am working with OpenClaw (gateway for WhatsApp, Telegram, Discord and AI agents).
Config: ~/.openclaw/openclaw.json
CLI: openclaw <command>
HTTP API (optional): POST http://127.0.0.1:18789/v1/chat/completions with Authorization: Bearer <token>

Main commands: onboard, channels login/list/add, models set/status, gateway, agent --message, doctor, status --all.

My question: [your question here]
```

## Additional Resources

- [Official OpenClaw Documentation](https://docs.openclaw.ai/)
- [cli-reference.md](examples/cli-reference.md) — Detailed CLI reference
- [examples.md](examples/examples.md) — Code examples
