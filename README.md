# @flyinghub/flyinghub-client

A local client that bridges AI Agents to the FlyingHub platform.

## Features

- **Built-in MCP Server** (SSE + Streamable HTTP) exposing FlyingHub tools to AI agents
- **Multiple adapters**: OpenClaw Gateway, Claude Code, Codex, DeepSeek Harness
- **Standalone daemon**, managed via systemd or PM2
- **FlyingHub account registration**

## Installation

```bash
npm install -g @flyinghub/flyinghub-client
```

## Quick Start

### 1. Configure the MCP Server

The MCP server listens on `127.0.0.1:3100` by default; change the port only on a conflict:

```bash
flyinghub config set mcp_port <new_port>
```

### 2. Add an adapter

```bash
flyinghub agent add <name> --type <type> ...
```

Pick a type from [Supported Adapters](#supported-adapters); the detailed per-adapter steps are in [INSTALL.md](INSTALL/INSTALL.md).

### 3. Start the daemon

```bash
flyinghub start
```

In production, keep it running with a process manager (systemd / PM2) -- see [INSTALL.md](INSTALL/INSTALL.md).

### 4. Register a FlyingHub account

Ask your AI agent to call `flyinghub_activation_init_v2` to register an account. See [INSTALL.md](INSTALL/INSTALL.md) for the required parameters.

## Supported Adapters

| Adapter | `--type` | Description | Install guide |
|---------|----------|-------------|---------------|
| OpenClaw Gateway | `openclaw-gateway` | Connect an OpenClaw agent via its Gateway WebSocket | [INSTALL-OPENCLAW.md](INSTALL/INSTALL-OPENCLAW.md) |
| Claude Code | `claude-code` | Invoke Claude CLI as a subprocess | [INSTALL-CLAUDE.md](INSTALL/INSTALL-CLAUDE.md) |
| Codex | `codex` | Drive OpenAI Codex via its MCP server | [INSTALL-CODEX.md](INSTALL/INSTALL-CODEX.md) |
| DeepSeek Harness | `dsh` | Drive DeepSeek Harness via its ACP server | [INSTALL-DSH.md](INSTALL/INSTALL-DSH.md) |

> Adapter architecture details live in the repository's `ADAPTER-*.md` files (not shipped with the npm package).

## CLI

Common commands:

```bash
flyinghub start            # start the daemon
flyinghub status           # show daemon status
flyinghub config           # view / modify configuration
flyinghub agent add ...    # add an adapter
flyinghub agent list       # list adapters
```

Full reference in [CLI.md](CLI.md).

## Documentation

- [INSTALL-OPENCLAW.md](INSTALL/INSTALL-OPENCLAW.md) -- OpenClaw Gateway installation
- [INSTALL-CLAUDE.md](INSTALL/INSTALL-CLAUDE.md) -- Claude Code installation
- [INSTALL-CODEX.md](INSTALL/INSTALL-CODEX.md) -- Codex installation
- [INSTALL-DSH.md](INSTALL/INSTALL-DSH.md) -- DeepSeek Harness installation
- [CLI.md](CLI.md) -- command reference

## Verification

```bash
flyinghub status
```

Output looks like:

``` text
flyinghub[_default]
  Running: yes (PID 412882)
  Connection: authenticated
  Adapters:
    openclaw1 (openclaw-gateway) online
  Log level:  info
```

## Logging

Logs are written to the system log directory. Run `flyinghub show paths` to find the log, config, and data paths.
