# CLI Reference

## Usage

```
flyinghub [global flags] <command> [command flags...]
```

The build artifact is `dist/index.cjs`, which can be run directly:

```
node dist/index.cjs <command> [flags...]
```

---

## Global Flags

| Flag | Description |
|------|-------------|
| `--instance <name>` | Specify instance name (default `_default`) |
| `--all` | Execute operation on all instances |
| `--log-level <level>` | Log level: `debug` / `info` / `warn` / `error` / `none` (daemon default info, CLI default info) |

---

## Commands

### `help`

Display help information.

### `start [options]`

Start the daemon. Runs in the foreground -- use a process manager in production (see [README.md](./README.md#3-start-the-daemon)).

**Options:**

| Flag | Description |
|------|-------------|
| `--port <number>` | MCP service port (default 3100) |
| `--host <address>` | MCP bind address (default 127.0.0.1) |
| `--agent-key <key>` | FlyingHub agent key |
| `--stdio` | Start as a child process of Claude, serving MCP via stdin/stdout; also listens on a random 127.0.0.1 port for child processes |

**Examples:**

```bash
# Start with default config
flyinghub start
```

### `stop`

Stop the daemon.

**Examples:**

```bash
flyinghub stop
```

### `status`

View daemon runtime status and connection information.

**Examples:**

```bash
flyinghub status
```

### `config [<sub>]`

View or modify configuration.

**Subcommands:**

| Subcommand | Description |
|------------|-------------|
| `config` | Display configured values |
| `config dump` | Display all configuration items (including defaults) |
| `config get <key>` | View a single configuration item |
| `config set <key> <value>` | Set a configuration item |
| `config help` / `--help` | Display help |

**Configuration Items:**

| Key | Description | Default |
|-----|-------------|---------|
| `agent_key` | FlyingHub agent key | -- |
| `api_base_url` | API base URL | `https://api.flyinghub.app/api/v1` |
| `ws_url` | WebSocket URL | `wss://ws.flyinghub.app/` |
| `mcp_host` | MCP Server listen address within the daemon | `127.0.0.1` |
| `mcp_port` | MCP Server listen port within the daemon | `3100` |
| `mcp_disable_sse` | Disable SSE transport | `false` |
| `mcp_disable_streamable` | Disable Streamable HTTP | `false` |
| `log_level` | Log level | `info` |
| `admin_token` | Admin API token | Auto-generated |
| `onStartAll` | Whether to start with `--all` | -- |

**Examples:**

```bash
# View current config
flyinghub config

# View all config (including defaults)
flyinghub config dump

# View a single config item
flyinghub config get mcp_port

# Modify config
flyinghub config set log_level debug
```

### `showconfigpath`

Display the config file path.

### `show paths`

Display all runtime paths (config, data, cache, log, temp).

**Examples:**

```bash
flyinghub show paths
flyinghub showpaths   # alias
```

### `debug <on|off|status> [permanent]`

Manage daemon log level.

| Subcommand | Description |
|------------|-------------|
| `debug on [permanent]` | Enable debug logging (+permanent writes to config permanently) |
| `debug off [permanent]` | Restore info logging |
| `debug status` | View current log level |

**Examples:**

```bash
flyinghub debug on
flyinghub debug on permanent
flyinghub debug status
```

### `version`

Display version number.

### `agent <sub> [options]`

Manage adapter configuration.

**Subcommands:**

| Subcommand | Description |
|------------|-------------|
| `agent list` | List configured adapters |
| `agent types` | List supported adapter types |
| `agent add <name> --type <type>` | Add an adapter |
| `agent remove <name>` | Remove an adapter |
| `agent enable <name>` | Enable an adapter |
| `agent disable <name>` | Disable an adapter |
| `agent set <name> [options]` | Modify adapter configuration |
| `agent show <name>` | View adapter configuration |
| `agent get <name> <key>` | View a single adapter config item |
| `agent validate <name>` | Validate adapter configuration |
| `agent unset <name> <key>` | Remove an adapter config item |
| `agent help` | Display help (also supports `--help`, `-h`) |

### `agent op <name> <action>`

Send an operation command to an adapter.

| Operation | Description |
|-----------|-------------|
| `init` | Pair with Gateway. Requires `url`, `auth-mode`, `token` / `password` configured via `agent add` or `agent set` first |

### `agent op <name> session <action>`

Directly operate sessions on the Gateway. No `init` pairing required; the adapter only needs `token`/`password` configured. Does not require the daemon to be running.

**Available operations:**

| Operation | Description |
|-----------|-------------|
| `list [terse]` | List sessions (with `terse`, shows only key and id) |
| `create <session-key>` | Create a session |
| `send <session> <message>` | Send a message |
| `steer <session> <message>` | Send a message and interrupt the current reply |
| `ssend <session> <message>` | Send a message and wait for a reply |
| `ssteer <session> <message>` | Send, interrupt, and wait for a reply |
| `abort <session>` | Abort a running reply |
| `reset <session>` | Reset session state |
| `delete <session>` | Delete a session |
| `usage <session>` | View session usage statistics |
| `subscribe event` | Subscribe to session events |
| `subscribe message <session>` | Subscribe to session messages |

**Examples:**

```bash
# List sessions
flyinghub agent op openclaw session list

# Send a message
flyinghub agent op openclaw session send agent:main:main "Hello"

# View usage
flyinghub agent op openclaw session usage agent:main:main
```

**Supported adapter types:**

| Type | `--type` value | Description |
|------|----------------|-------------|
| OpenClaw Gateway | `openclaw-gateway` | Connect OpenClaw Agent via Gateway protocol |
| Claude Code | `claude-code` | Invoke Claude CLI as a child process |
| Codex | `codex` | Drive OpenAI Codex via its MCP server |
| DeepSeek Harness | `dsh` | Drive DeepSeek Harness via its ACP server |

**Options (add / set):**

| Flag | Description |
|------|-------------|
| `--type <type>` | Adapter type (see table above) |
| `--priority <N>` | Priority (0 highest, default 0) |
| `--disabled` | Do not enable after creation |
| `<key>=<value>` | Adapter-specific config (multiple allowed, see table below) |

**Available keys for `openclaw-gateway` (also applies to `agent get` / `agent set`):**

| key | Description | Default |
|-----|-------------|---------|
| `url` | Gateway WebSocket URL | `ws://127.0.0.1:18789` |
| `auth-mode` | Auth mode: `token` / `password` / `none` | `token` |
| `token` | Token authentication credential | -- |
| `password` | Password authentication credential | -- |

**Available keys for `claude-code`:**

| key | Description | Default |
|-----|-------------|---------|
| `command` | Claude CLI path | -- |
| `args` | Additional startup arguments | `""` |
| `env.*` | Environment variables (`KEY=VALUE`) | -- |

**Available keys for `codex`:**

| key | Description | Default |
|-----|-------------|---------|
| `command` | Codex CLI path | -- |
| `model` | Model override (e.g. `gpt-5.2`) | `""` |
| `sandbox` | Sandbox mode: `read-only` / `workspace-write` / `danger-full-access` | `workspace-write` |
| `approval_policy` | Approval policy: `never` / `on-request` / `untrusted` | `never` |
| `reply_timeout` | Single-turn reply timeout (seconds) | `1200` |
| `args` | Additional CLI arguments | `""` |
| `env.*` | Environment variables (`KEY=VALUE`) | -- |

**Available keys for `dsh`:**

| key | Description | Default |
|-----|-------------|---------|
| `command` | dsh CLI path | -- |
| `provider` | Provider route | `deepseek-official` |
| `model` | Model id | `deepseek-v4-flash` |
| `reply_timeout` | Single-turn reply timeout (seconds) | `1200` |
| `npm_registry` | pnpm registry mirror (default https://registry.npmmirror.com) | -- |
| `env.*` | Environment variables (`KEY=VALUE`) | -- |

**Examples:**

```bash
# List adapters
flyinghub agent list

# Add a gateway adapter (token auth)
flyinghub agent add my-gateway --type openclaw-gateway url=ws://192.168.1.100:18789 token=my_token

# Modify an adapter's url
flyinghub agent set my-gateway url=ws://192.168.1.101:18789

# View adapter full config
flyinghub agent show my-gateway

# Set environment variables for claude-code (nested fields use dot notation)
flyinghub agent set my-claude env.ANTHROPIC_API_KEY=sk-xxx
flyinghub agent unset my-claude env.OPENAI_API_KEY
```

---

## Deployment

The build artifact `dist/index.cjs` is a single file with all dependencies inlined:

```bash
npm run build
cp dist/index.cjs /deploy/path/
node /deploy/path/index.cjs start
```

No `node_modules` required.
