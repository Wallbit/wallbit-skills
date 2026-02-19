# OpenClaw CLI Reference

Detailed documentation for OpenClaw CLI commands.

---

## Global flags

- `-V`, `--version`, `-v`: print version and exit
- `--update`: shortcut for `openclaw update` (source installations only)
- `--no-color`: disable ANSI colors
- `--profile <name>`: isolate state under `~/.openclaw-<name>`
- `--dev`: isolate state under `~/.openclaw-dev` and change default ports
- `--json`: JSON output (where supported)
- `--plain`: unstyled output (where supported)

---

## Setup and onboarding

### `openclaw setup`

Initializes config and workspace.

| Option | Description |
|--------|-------------|
| `--remote-token <token>` | Remote Gateway token |
| `--remote-url <url>` | Remote Gateway URL |
| `--mode <mode>` | Wizard mode |
| `--non-interactive` | Run without prompts |
| `--wizard` | Run the onboarding wizard |
| `--workspace <path>` | Agent workspace path (default: `~/.openclaw/workspace`) |

The wizard runs automatically when wizard flags are used (`--non-interactive`, `--mode`, `--remote-url`, `--remote-token`).

### `openclaw onboard`

Interactive wizard to configure gateway, workspace and skills.

| Option | Description |
|--------|-------------|
| `--install-daemon` | Install Gateway as a service |
| `--no-install-daemon` | Do not install as a service |
| `--skip-ui` | Skip UI steps |
| `--skip-health` | Skip health check |
| `--skip-skills` | Skip skills configuration |
| `--skip-channels` | Skip channels configuration |
| `--gateway-port <port>` | Gateway port |
| `--anthropic-api-key <key>` | Anthropic API key (non-interactive) |
| `--openai-api-key <key>` | OpenAI API key (non-interactive) |
| `--auth-choice <choice>` | token \| custom-api-key \| etc. |
| `--non-interactive` | Non-interactive mode |
| `--reset` | Reset config, credentials, sessions and workspace before wizard |
| `--workspace <path>` | Workspace path |

Example:

```bash
openclaw onboard --install-daemon
openclaw onboard --non-interactive --anthropic-api-key $ANTHROPIC_API_KEY --install-daemon
```

### `openclaw configure`

Opens the interactive configuration wizard (models, channels, skills, gateway).

### `openclaw config`

Non-interactive config access. Without subcommand opens the wizard.

| Subcommand | Description |
|------------|-------------|
| `config get <path>` | Print value (dot/bracket path, e.g. `agents.defaults.model.primary`) |
| `config set <path> <value>` | Set value (JSON5 or string) |
| `config unset <path>` | Remove value |

Example:

```bash
openclaw config get gateway.port
openclaw config set gateway.http.endpoints.chatCompletions.enabled true
```

### `openclaw doctor`

Checks config, Gateway and services; applies safe fixes.

| Option | Description |
|--------|-------------|
| `--deep` | Scan system services for other Gateway installations |
| `--non-interactive` | Do not ask; apply only safe migrations |
| `--yes` | Accept defaults without prompting |

---

## Gateway

### `openclaw gateway`

Runs the WebSocket Gateway (or manages the service).

Options when running in foreground:

| Option | Description |
|--------|-------------|
| `--port <port>` | Port (default: 18789) |
| `--bind <host>` | Bind host |
| `--token <token>` | Authentication token |
| `--password <password>` | Password (when auth mode is password) |
| `--verbose` | More logs |
| `--force` | Kill existing listener on the port |
| `--dev` | Development mode |

### `openclaw gateway install|uninstall|start|stop|restart`

Manages the Gateway as a service (launchd/systemd/schtasks).

```bash
openclaw gateway install --port 18789 --token $OPENCLAW_GATEWAY_TOKEN
openclaw gateway start
openclaw gateway status
openclaw gateway stop
openclaw gateway uninstall
```

Options for `gateway install`: `--port`, `--runtime`, `--token`, `--force`, `--json`.

### `openclaw gateway status`

Shows which config the CLI uses, which config the service uses, and the probe URL. By default probes the Gateway RPC.

Options: `--no-probe`, `--deep`, `--json`.

### `openclaw gateway health`

Gets the Gateway health status via RPC.

Options: `--verbose`, `--timeout <ms>`, `--json`. Requires `--url` and `--token` when using an explicit URL.

### `openclaw gateway call [--params <json>]`

Calls a Gateway RPC. Common RPCs: `update.run`, `config.patch`, `config.apply`, `config.get`.

### `openclaw logs`

Shows Gateway logs (by file via RPC).

| Option | Description |
|--------|-------------|
| `--follow` | Follow in real time |
| `--limit <n>` | Number of lines (default: 200) |
| `--plain` | Plain text |
| `--json` | One JSON line per event |

Example:

```bash
openclaw logs --follow
openclaw logs --limit 200 --json
```

---

## Channels

### `openclaw channels list`

Lists configured channels and auth status.

Options: `--json`, `--no-usage` (omit provider usage/quota).

### `openclaw channels status`

Checks Gateway reach and channel health. Prints warnings when misconfiguration is detected.

Options: `--probe` (additional checks), `--channel <name>`, `--account <id>`.

### `openclaw channels add`

Adds a channel. Without flags opens the wizard; with flags runs non-interactively.

| Option | Description |
|--------|-------------|
| `--channel <name>` | whatsapp \| telegram \| discord \| googlechat \| slack \| mattermost \| signal \| imessage \| msteams |
| `--account <id>` | Account ID (default: default) |
| `--name <name>` | Display name |
| `--token <token>` | Bot token (Telegram, Discord, etc.) |

Example:

```bash
openclaw channels add --channel telegram --account alerts --name "Alerts Bot" --token $TELEGRAM_BOT_TOKEN
openclaw channels add --channel discord --account work --name "Work Bot" --token $DISCORD_BOT_TOKEN
```

### `openclaw channels remove`

Deactivates a channel. Use `--delete` to remove config entries without prompting.

Options: `--channel`, `--account`, `--delete`.

### `openclaw channels login`

Interactive login for the channel (WhatsApp Web, etc.).

Options: `--channel` (default: whatsapp), `--account`, `--verbose`.

### `openclaw channels logout`

Logs out of the channel (if the channel supports it).

Options: `--channel`, `--account`.

### `openclaw channels logs`

Shows recent channel logs from the Gateway log file.

Options: `--channel` (default: all), `--lines <n>`, `--json`.

---

## Models

### `openclaw models` (no subcommand)

Shortcut for `openclaw models status`.

### `openclaw models list`

Lists configured models.

| Option | Description |
|--------|-------------|
| `--json` | Machine-readable output |
| `--plain` | One model per line |
| `--provider <name>` | Filter by provider |
| `--local` | Local providers only |
| `--all` | Full catalog |

### `openclaw models status`

Shows the resolved primary model, fallbacks, image model and auth summary. Includes OAuth expiry status.

| Option | Description |
|--------|-------------|
| `--check` | Exit 1 if auth is missing/expired, 2 if about to expire |
| `--plain` | Resolved primary model only |
| `--json` | Full JSON |
| `--probe` | Live probe of configured profiles (may consume tokens) |

### `openclaw models set <provider/model>`

Sets `agents.defaults.model.primary`.

### `openclaw models set-image <provider/model>`

Sets `agents.defaults.imageModel.primary`.

### `openclaw models aliases list|add|remove`

Manages model aliases.

```bash
openclaw models aliases add Sonnet anthropic/claude-sonnet-4-5
openclaw models aliases remove Sonnet
openclaw models aliases list --json
```

### `openclaw models fallbacks list|add|remove|clear`

Manages the chat model fallback list.

```bash
openclaw models fallbacks add openai/gpt-4o
openclaw models fallbacks remove openai/gpt-4o
openclaw models fallbacks clear
```

### `openclaw models image-fallbacks list|add|remove|clear`

Manages image model fallbacks.

### `openclaw models scan`

Inspects the OpenRouter free model catalog and optionally probes (tool/image support). Requires OpenRouter API key in auth or `OPENROUTER_API_KEY`.

| Option | Description |
|--------|-------------|
| `--set-default` | Set primary model with the first selection |
| `--set-image` | Set image model with the first selection |
| `--no-probe` | Metadata only, no probe |
| `--yes` | Accept defaults in non-interactive mode |
| `--provider <prefix>` | Filter by provider |
| `--max-candidates <n>` | Fallback list size |
| `--max-age-days <n>` | Ignore models older than N days |
| `--min-params <n>` | Minimum parameters (billions) |

### `openclaw models auth add|setup-token|paste-token`

Manages provider credentials.

- `setup-token`: recommended flow for Anthropic (`openclaw models auth setup-token --provider anthropic`)
- `paste-token`: paste token manually with `--provider`, `--profile-id`, `--expires-in`
- `add`: interactive helper

---

## Agents

### `openclaw agents list`

Lists configured agents.

Options: `--bindings`, `--json`.

### `openclaw agents add [name]`

Adds an isolated agent. Without flags runs the wizard; with `--non-interactive` requires `--workspace`.

| Option | Description |
|--------|-------------|
| `--workspace <path>` | Workspace path (required in non-interactive) |
| `--model <provider/model>` | Agent model |
| `--bind <spec>` | Binding (repeatable), e.g. `whatsapp:default`, `telegram:alerts` |
| `--agent-dir <path>` | Agent directory |
| `--non-interactive` | No wizard |
| `--json` | JSON output |

### `openclaw agents delete <id>`

Removes an agent and cleans up its workspace and state.

Options: `--force`, `--json`.

### `openclaw agent`

Runs one agent turn via Gateway (or `--local` embedded).

| Option | Description |
|--------|-------------|
| `--message <text>` | User message (required) |
| `--session-id <id>` | Session ID |
| `--to <target>` | Session key and optional delivery |
| `--channel <name>` | Channel |
| `--local` | Run locally without Gateway |
| `--deliver` | Deliver response to the channel |
| `--timeout <ms>` | Timeout |
| `--verbose <level>` | Verbosity |
| `--thinking <level>` | Thinking level (compatible models) |
| `--json` | JSON output |

Example:

```bash
openclaw agent --message "Summarize this paragraph in one line"
openclaw agent --message "Hello" --session-id my-session --json
```

---

## Sessions

### `openclaw sessions`

Lists stored conversation sessions.

Options: `--active <min>`, `--store <path>`, `--verbose`, `--json`.

---

## Cron

All `cron` commands accept `--url`, `--token`, `--timeout`, `--expect-final` to target a specific Gateway.

### `openclaw cron status`

Cron job status.

Options: `--json`.

### `openclaw cron list`

Lists cron jobs.

Options: `--all`, `--json`.

### `openclaw cron add`

Creates a cron job. Requires `--name` and exactly one of `--at` \| `--every` \| `--cron`, and exactly one payload: `--system-event` \| `--message`.

### `openclaw cron edit <id>`

Edits a job (field patch).

### `openclaw cron rm <id>`

Removes a cron job (alias: remove, delete).

### `openclaw cron enable|disable <id>`

Enables or disables a job.

### `openclaw cron run [--force]`

Runs the job (or all jobs depending on context).

### `openclaw cron runs --id <id> [--limit <n>]`

Lists executions of a job.

---

## Browser

Commands for the OpenClaw managed browser. Common options: `--browser-profile <name>`, `--url`, `--token`, `--timeout`, `--json`.

### Management

- `openclaw browser status` — Browser status
- `openclaw browser start` — Start
- `openclaw browser stop` — Stop
- `openclaw browser tabs` — List tabs
- `openclaw browser open <url>` — Open URL
- `openclaw browser focus <targetId>` — Focus tab
- `openclaw browser close [targetId]` — Close tab
- `openclaw browser profiles` — List profiles
- `openclaw browser create-profile --name <name> [--color] [--cdp-url]` — Create profile
- `openclaw browser delete-profile --name <name>` — Delete profile
- `openclaw browser reset-profile` — Reset default profile

### Inspection

- `openclaw browser snapshot [--format aria|ai] [--target-id] [--limit] [--interactive] [--compact] [--depth] [--selector] [--out]` — DOM/accessibility snapshot
- `openclaw browser screenshot [targetId] [--full-page] [--ref] [--element] [--type png|jpeg]` — Screenshot

### Actions

- `openclaw browser navigate <url> [--target-id]` — Navigate
- `openclaw browser click [--ref] [--double] [--button] [--modifiers] [--target-id]` — Click
- `openclaw browser type <text> [--submit] [--slowly] [--target-id]` — Type
- `openclaw browser press <key> [--target-id]` — Key press
- `openclaw browser hover [--ref] [--target-id]` — Hover
- `openclaw browser fill [--fields] [--fields-file] [--target-id]` — Fill form
- `openclaw browser select [--ref] [--values] [--target-id]` — Select option
- `openclaw browser drag [--source-ref] [--target-ref] [--target-id]` — Drag
- `openclaw browser wait [--time <s>] [--text] [--text-gone] [--target-id]` — Wait
- `openclaw browser dialog --accept|--dismiss [--prompt] [--target-id]` — Dialogs
- `openclaw browser evaluate --fn <code> [--ref] [--target-id]` — Evaluate JS
- `openclaw browser console [--level] [--target-id]` — Console messages
- `openclaw browser pdf [--target-id]` — Export PDF
- `openclaw browser resize [--width] [--height] [--target-id]` — Resize window
- `openclaw browser upload [--ref] [--input-ref] [--element] [--target-id]` — Upload file

---

## Message (send from CLI)

### `openclaw message send --target <target> --message "<text>"`

Sends a message to the target (e.g. phone number, channel id).

### `openclaw message poll --channel discord --target channel:123 --poll-question "Snack?" --poll-option Pizza --poll-option Sushi`

Creates a poll (Discord, WhatsApp, MS Teams depending on channel).

Other subcommands: `message react`, `message read`, `message edit`, `message delete`, `message pin`, `message thread`, etc. See [message CLI documentation](https://docs.openclaw.ai/cli/message).

---

## System

### `openclaw system event --text "<text>"`

Queues a system event and optionally triggers a heartbeat (Gateway RPC).

Options: `--url`, `--token`, `--timeout`, `--expect-final`, `--json`, `--mode`.

### `openclaw system heartbeat last|enable|disable`

Controls the heartbeat (Gateway RPC).

### `openclaw system presence`

Lists system presence entries (RPC).

---

## Status and health

### `openclaw status`

Status of linked sessions and recent recipients.

| Option | Description |
|--------|-------------|
| `--all` | Full diagnostic (read-only, pasteable) |
| `--deep` | Include channel probe |
| `--usage` | Show provider usage/quota |
| `--verbose` | More detail |
| `--json` | JSON output |
| `--timeout <ms>` | Probe timeout |

### `openclaw health`

Gets Gateway health (RPC).

Options: `--verbose`, `--timeout <ms>`, `--json`.

---

## Reset and uninstall

### `openclaw reset`

Resets local config/state (keeps the CLI installed).

Options: `--dry-run`, `--non-interactive`, `--yes`, `--scope <scope>`. In non-interactive mode `--scope` and `--yes` are required.

### `openclaw uninstall`

Uninstalls the Gateway service and local data (the CLI remains installed).

Options: `--dry-run`, `--non-interactive`, `--yes`, `--all`, `--app`, `--workspace`, `--state`, `--service`. In non-interactive mode `--yes` and explicit scopes (or `--all`) are required.

---

## Skills, plugins, pairing

### `openclaw skills list`

Lists available skills.

Options: `-v`/`--verbose`, `--json`, `--eligible` (ready skills only).

### `openclaw skills info <name>`

Details of a skill.

### `openclaw skills check`

Summary of ready skills vs missing requirements.

### `openclaw plugins list|info|install|enable|disable|doctor`

Manages plugins. E.g.: `openclaw plugins install <name>`, `openclaw plugins enable <name>`.

### `openclaw pairing list [--json]`

Lists pairing requests.

### `openclaw pairing approve [--notify]`

Approves pending pairing requests.

---

## Memory

### `openclaw memory search "<query>"`

Semantic search over MEMORY.md and memory/*.md.

### `openclaw memory index`

Re-indexes memory files.

### `openclaw memory status`

Memory index statistics.

---

## Nodes

For controlling paired nodes (iOS/Android, macOS companion, headless node host).

Common options: `--url`, `--token`, `--timeout`, `--json`.

- `openclaw nodes list [--connected] [--last-connected]` — List nodes
- `openclaw nodes describe --node <id>` — Node details
- `openclaw nodes status` — Status
- `openclaw nodes approve|reject <id>` — Approve/reject pairing
- `openclaw nodes pending` — Pending approvals
- `openclaw nodes notify --node <id> [--title] [--body] [--sound]` — Notification (mac)
- `openclaw nodes run --node <id> <command...>` — Run command on node
- `openclaw nodes invoke --node <id> --command <name> [--params <json>]` — Invoke command
- `openclaw nodes camera snap|clip --node <id> [--facing front|back]` — Camera
- `openclaw nodes screen record --node <id> [--duration] [--out]` — Record screen
- `openclaw nodes canvas snapshot|present|a2ui push ... --node <id>` — Canvas
- `openclaw nodes location get --node <id>` — Location

---

## Security

### `openclaw security audit`

Audits config and local state for common security issues.

- `openclaw security audit --fix` — Apply safe default values and correct chmod
- `openclaw security audit --deep` — Live probe to the Gateway (best-effort)
