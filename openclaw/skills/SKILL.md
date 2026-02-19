---
name: openclaw-skill
description: Integration with OpenClaw: self-hosted gateway that connects WhatsApp, Telegram, Discord and other channels to AI agents. Use when working with OpenClaw, messaging gateway, channels (WhatsApp/Telegram/Discord), AI agent, openclaw CLI, openclaw.json configuration, tools (browser, exec, messaging), or when the user mentions OpenClaw.
---

# OpenClaw

Self-hosted gateway that connects messaging apps (WhatsApp, Telegram, Discord, iMessage, etc.) to AI agents. A single Gateway process on your machine or server acts as a bridge between channels and an always-available AI assistant.

## Quick Start

**Requirements**: Node 22+, API key (Anthropic recommended).

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
openclaw channels login
openclaw gateway --port 18789
```

Default dashboard: `http://127.0.0.1:18789/`

## CLI Overview

| Category    | Main command                   | Description                          |
| ----------- | ------------------------------ | ------------------------------------ |
| Setup       | `openclaw onboard`             | Installation and configuration wizard |
| Setup       | `openclaw setup`               | Initialize config and workspace       |
| Setup       | `openclaw configure`           | Configuration wizard                  |
| Config      | `openclaw config get/set/unset` | Read/write config (dot path)         |
| Gateway     | `openclaw gateway`             | Run or manage the Gateway             |
| Gateway     | `openclaw gateway start/stop`  | Gateway service                       |
| Gateway     | `openclaw health`              | Gateway health                        |
| Channels    | `openclaw channels list`       | List configured channels              |
| Channels    | `openclaw channels login`      | Interactive login (e.g. WhatsApp)     |
| Channels    | `openclaw channels add/remove` | Add or remove a channel               |
| Models      | `openclaw models list`         | List configured models                |
| Models      | `openclaw models set <model>`  | Set primary model                     |
| Models      | `openclaw models status`       | Auth status and current model         |
| Agents      | `openclaw agents list`         | List agents                           |
| Agents      | `openclaw agents add/delete`   | Add or remove an agent                |
| Agent       | `openclaw agent --message "…"` | Run one agent turn                    |
| Sessions    | `openclaw sessions`            | List sessions                         |
| Cron        | `openclaw cron list/add/rm`    | Scheduled tasks                       |
| Browser     | `openclaw browser start/stop`  | Managed browser                       |
| Logs        | `openclaw logs --follow`       | View Gateway logs                     |
| Doctor      | `openclaw doctor`              | Diagnostics and fixes                 |
| Status      | `openclaw status --all`        | Full status (pasteable)               |

## Configuration

Main config: `~/.openclaw/openclaw.json` (JSON5).

Main key structure:

```json5
{
  channels: {
    whatsapp: { allowFrom: ["+15555550123"], groups: { "*": { requireMention: true } } },
    telegram: { /* token, etc. */ },
    discord: { /* token, etc. */ },
  },
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-5", fallbacks: [] },
      imageModel: { primary: "...", fallbacks: [] },
      models: { /* allowlist + aliases */ },
    },
    list: [
      { id: "main", workspace: "~/.openclaw/workspace", bindings: ["whatsapp:default"] },
    ],
  },
  tools: {
    profile: "full",   // full | messaging | coding | minimal
    allow: [],
    deny: [],
  },
  messages: { groupChat: { mentionPatterns: ["@openclaw"] } },
  gateway: {
    auth: { mode: "token", token: "..." },
    http: { endpoints: { chatCompletions: { enabled: true } } },
  },
}
```

- **channels**: Per-channel config (allowFrom, groups, tokens).
- **agents.defaults**: Primary model, fallbacks, model allowlist.
- **agents.list**: Isolated agents with workspace and per-channel bindings.
- **tools**: Profile (full, messaging, coding, minimal), allow/deny list for tools.
- **gateway.auth**: token or password for CLI and HTTP API.

## Channels

Supported channels: WhatsApp, Telegram, Discord, Slack, Google Chat, Mattermost (plugin), Signal, iMessage, MS Teams.

### WhatsApp

```bash
openclaw channels login --channel whatsapp
# Scan QR with WhatsApp Web
openclaw channels status --channel whatsapp
```

Restrict senders in config: `channels.whatsapp.allowFrom: ["+15555550123"]`. In groups: `requireMention: true`.

### Telegram

```bash
openclaw channels add --channel telegram --account default --token $TELEGRAM_BOT_TOKEN
openclaw channels status --channel telegram
```

### Discord

```bash
openclaw channels add --channel discord --account work --token $DISCORD_BOT_TOKEN
openclaw channels remove --channel discord --account work --delete
```

## Models

Selection order: 1) failover within provider, 2) fallbacks in order, 3) primary model.

Config keys: `agents.defaults.model.primary`, `agents.defaults.model.fallbacks`, `agents.defaults.imageModel.primary`, `agents.defaults.models` (allowlist).

```bash
openclaw models set anthropic/claude-sonnet-4-5
openclaw models fallbacks add openai/gpt-4o
openclaw models status --check
```

Recommended auth for Anthropic: `claude setup-token` then `openclaw models auth setup-token --provider anthropic`. If `agents.defaults.models` is defined, only those models are allowed; if a user picks another, OpenClaw responds "Model ... is not allowed".

## Tools

Tools exposed to the agent (configurable with `tools.profile`, `tools.allow`, `tools.deny`):

| Tool / Group    | Description                                                   |
| --------------- | ------------------------------------------------------------- |
| `browser`       | Managed browser: navigate, snapshot, click, type, screenshot  |
| `exec`          | Run commands in workspace/sandbox/gateway/node                |
| `process`       | Manage background exec sessions                               |
| `message`       | Send messages and actions in Discord/Telegram/WhatsApp/etc.   |
| `web_search`    | Web search (Brave API)                                        |
| `web_fetch`     | Fetch URL and extract content (markdown/text)                 |
| `canvas`        | Canvas on nodes: present, snapshot, a2ui_push                 |
| `nodes`         | Notifications, run, camera, screen on nodes                   |
| `cron`          | Manage Gateway cron jobs                                      |
| `gateway`       | config.get/patch/apply, restart                               |
| `sessions_*`    | sessions_list, sessions_history, sessions_send, sessions_spawn, session_status |
| `group:fs`      | read, write, edit, apply_patch                                |
| `group:runtime` | exec, bash, process                                           |

Profiles: `full` (no restriction), `messaging`, `coding`, `minimal`.

## Agents

Multi-agent routing: each agent can have its own workspace, model and per-channel bindings.

```bash
openclaw agents list
openclaw agents add support --workspace ~/support-workspace --bind "slack:default"
openclaw agents delete support --force
```

Bindings: `channel[:accountId]` (e.g. `whatsapp:default`, `telegram:alerts`). The `main` agent is the default.

Run one turn from CLI: `openclaw agent --message "Summarize this text" --session-id my-session`.

## Error Handling / Troubleshooting

| Command / Situation  | Use |
| -------------------- | --- |
| `openclaw doctor`    | Check config, Gateway and services; apply safe fixes |
| `openclaw doctor --deep` | Also search for other Gateways in the system |
| `openclaw status --all` | Full diagnostic as pasteable text |
| `openclaw status --deep` | Include channel probe |
| `openclaw health`    | Gateway health (RPC) |
| `openclaw gateway status` | Service status and port |
| `openclaw logs --follow` | View logs in real time |
| `openclaw security audit` | Review permissions and security config |

If the model does not respond: verify the model is in `agents.defaults.models` (if an allowlist exists) and that provider auth is configured (`openclaw models status`).

## HTTP API (Chat Completions)

The Gateway can expose an OpenAI-compatible endpoint. Enable in config:

```json5
{
  gateway: {
    http: {
      endpoints: { chatCompletions: { enabled: true } },
    },
  },
}
```

- **URL**: `POST http://<host>:<port>/v1/chat/completions` (same port as the Gateway, e.g. 18789).
- **Auth**: `Authorization: Bearer <token>` (uses `gateway.auth.token` or `OPENCLAW_GATEWAY_TOKEN`).
- **Agent**: header `x-openclaw-agent-id: main` or in the body `model: "openclaw:main"`.
- **Stable session**: send `user` in the body to derive a session key and maintain context.

See [cli-reference.md](../examples/cli-reference.md) for all commands and [examples.md](../examples/examples.md) for code.

## Code Generation Guidelines

When generating code that interacts with OpenClaw:

1. **Naming**: camelCase for functions and variables; docstrings on all functions.
2. **Config**: Do not hardcode tokens or passwords; use environment variables (e.g. `OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_GATEWAY_URL`).
3. **HTTP API**: When using `/v1/chat/completions`, always send `Authorization: Bearer` and optionally `x-openclaw-agent-id`.
4. **Webhooks**: Validate origin and signature if OpenClaw calls your server; document in docstrings.
5. **CLI**: For scripts, use `openclaw --json` where available for machine-readable output; `--non-interactive` and explicit flags in automated flows.
6. **Error handling**: Check HTTP codes and error body; for the Gateway, 429 with `Retry-After` on rate limit.

## Additional Resources

- Official documentation: https://docs.openclaw.ai/
- Detailed CLI reference: [cli-reference.md](../examples/cli-reference.md)
- Code examples: [examples.md](../examples/examples.md)
