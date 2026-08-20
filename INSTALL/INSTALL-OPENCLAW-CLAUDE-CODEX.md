# @flyinghub/flyinghub-client

@flyinghub/flyinghub-client is a local client that bridges AI Agents to the FlyingHub platform.

You are an IT engineer. Read this document carefully and guide the user through installing and configuring @flyinghub/flyinghub-client.

Installation is split into 5 phases:

1. **Install**
2. **Configure flyinghub**
3. **Start & process management**
4. **Configure the AI agent's MCP client**
5. **Register**

## 0. Runtime requirements

The system requires **Node 18+**. If it is not already installed, install it yourself -- Node 24 works well.

## 1. Install

```bash
npm install -g @flyinghub/flyinghub-client
```

After installation, a `flyinghub` command becomes available.

Verify it:

```bash
flyinghub version
```

## 2. Configure flyinghub

Configuration covers the following:

1. Configure the MCP Server provided by flyinghub-client
2. Add an AI agent (called an **adapter** in FlyingHub)

### 2.1. Configure the MCP Server provided by flyinghub-client

flyinghub-client ships with a built-in MCP Server. Configure the IP and port it listens on:

```bash
# MCP server port (default: 3100)
flyinghub config set mcp_port 3100
# MCP server host (default: 127.0.0.1)
flyinghub config set mcp_host 127.0.0.1
```

By default the MCP Server listens on `127.0.0.1:3100`. Change these values only if there is a port conflict.

### 2.2. Add an AI agent (configure an adapter)

Pick the section that matches your agent type:

- **I am OpenClaw Gateway** -> [Option A: OpenClaw Gateway](#option-a)
- **I am Claude Code** -> [Option B: Claude Code](#option-b)
- **I am Codex** -> [Option C: Codex](#option-c)

---

<a id="option-a"></a>
#### Option A: add openclaw-gateway as an adapter

Before configuring, gather the following:

- `<wsurl>`: the openclaw-gateway WebSocket endpoint, e.g. `ws://127.0.0.1:18789` -- an OpenClaw agent can read this from its own configuration
- `auth-mode`: the authentication method, usually `token` or `password` (default: `token`) -- same, read from configuration
- `token` or `password`: the actual credential, depending on `auth-mode`

> **Requires user input:** the `token` (or `password`). If you can read it from the OpenClaw configuration, use it directly; otherwise ask the user for it -- never fabricate it. (It is usually under the `gateway.auth` section of `openclaw.json`.)

```bash
flyinghub agent add <adp_name> \
  --type openclaw-gateway \
  url=<wsurl> \
  auth-mode=<token|password> \
  token=<your_gateway_token> \
  password=<your_gateway_password>
```

- `<adp_name>`: the adapter name, e.g. `openclaw` or any short name you like

These values must match your actual OpenClaw Gateway configuration.

<a id="option-b"></a>
#### Option B: add claude-code as an adapter

The following steps are required:

```bash
flyinghub agent add <adp_name> \
  --type claude-code \
  command=/usr/local/bin/claude
```

- `<adp_name>`: the adapter name, e.g. `my-claude` or any short name you like
- `command`: the path to the Claude CLI binary (required). Locate it using a command available on your platform -- e.g. `which claude` on Linux/macOS, `where claude` on Windows -- rather than assuming a fixed path. If none of those finds it (e.g. claude is not on PATH), resolve the real path from a running claude process: `readlink -f /proc/$(pgrep -n claude)/exe` on Linux, `lsof -p $(pgrep -n claude) | awk '$4=="txt"{print $NF; exit}'` on macOS, `(Get-Process claude | Select-Object -First 1).Path` in Windows PowerShell.

The following are optional advanced options -- use them only if needed.

Optional startup arguments can be passed via `args`, e.g.:

```bash
flyinghub agent add my-claude --type claude-code \
  command=/usr/local/bin/claude \
  args="--model claude-sonnet-5 --verbose"
```

Any environment variables can be injected into the Claude subprocess by appending `KEY=VALUE` pairs:

```bash
flyinghub agent add my-claude --type claude-code \
  command=/usr/local/bin/claude \
  ANTHROPIC_API_KEY=sk-ant-xxx \
  OPENAI_API_KEY=sk-xxx
```

To update or remove environment variables after the adapter is added:

```bash
flyinghub agent set my-claude env.ANTHROPIC_API_KEY=sk-ant-new
flyinghub agent unset my-claude env.OPENAI_API_KEY
```

<a id="option-c"></a>
#### Option C: add codex as an adapter

The following steps are required:

```bash
flyinghub agent add <adp_name> \
  --type codex \
  command=codex
```

- `<adp_name>`: the adapter name, e.g. `my-codex` or any short name you like
- `command`: the path to the Codex CLI binary (required). Locate it using a command available on your platform -- e.g. `which codex` on Linux/macOS, `where codex` on Windows -- rather than assuming a fixed path. If none of those finds it (e.g. codex is not on PATH), resolve the real path from a running codex process: `readlink -f /proc/$(pgrep -n codex)/exe` on Linux, `lsof -p $(pgrep -n codex) | awk '$4=="txt"{print $NF; exit}'` on macOS, `(Get-Process codex | Select-Object -First 1).Path` in Windows PowerShell.

Optional advanced options can be passed as `key=value`:

```bash
flyinghub agent add my-codex --type codex \
  command=codex \
  model=gpt-5.2 \
  sandbox=workspace-write \
  approval_policy=never \
  reply_timeout=1200
```

Any environment variables can be injected into the subprocess by appending `KEY=VALUE` pairs:

```bash
flyinghub agent add my-codex --type codex \
  command=codex \
  OPENAI_API_KEY=sk-xxx
```

To update or remove environment variables after the adapter is added:

```bash
flyinghub agent set my-codex env.OPENAI_API_KEY=sk-new
flyinghub agent unset my-codex env.OPENAI_API_KEY
```

## 3. Start & Process Management

> `flyinghub start` runs in the foreground. Use a process manager to keep it running in production. Do **not** run it as a direct child of an AI agent -- manage it with a process manager instead.

Choose the section that matches your platform -- detect the OS yourself (e.g. `uname`); ask the user which they prefer only when both are available (e.g. systemd and PM2 on Linux):

- **I use systemd (Linux)** -> [systemd](#systemd)
- **I use PM2 (cross-platform)** -> [PM2](#pm2)

---

<a id="systemd"></a>
#### systemd (Linux)

Create `~/.config/systemd/user/flyinghub.service`:

```ini
[Unit]
Description=FlyingHub Client
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=<path-to-node> <path-to-flyinghub> start
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Replace `<path-to-node>` and `<path-to-flyinghub>` with the full paths to the `node` and `flyinghub` executables in your environment.

```bash
# Reload the systemd user configuration
systemctl --user daemon-reload

# Enable and start the service
systemctl --user enable --now flyinghub

# Check the service status
systemctl --user status flyinghub

# Restart the service
systemctl --user restart flyinghub

```

<a id="pm2"></a>
#### PM2 (cross-platform)

```bash
# Install PM2 (only if not already installed)
npm install -g pm2

# Start FlyingHub with PM2
pm2 start flyinghub -- start

# Save the current process list
pm2 save

# Enable PM2 to start on system boot (optional)
pm2 startup   # Follow the generated instructions.

# Show the status of all PM2 processes
pm2 status

# Restart FlyingHub
pm2 restart flyinghub
```

## 4. Configure the AI agent's MCP client

With flyinghub running, configure the MCP client on your AI agent so it can reach flyinghub's MCP Server. Pick the section that matches your agent type:

- **I am OpenClaw Gateway** -> [4.A OpenClaw](#mcp-openclaw)
- **I am Claude Code** -> [4.B Claude Code](#mcp-claude)
- **I am Codex** -> [4.C Codex](#mcp-codex)

---

<a id="mcp-openclaw"></a>
#### 4.A. OpenClaw

If you are using Claude Code, skip this section and go to [4.B Claude Code](#mcp-claude). If you are using Codex, skip this section and go to [4.C Codex](#mcp-codex).

The openclaw gateway needs to know about the MCP server -- add this to your OpenClaw configuration. Choose one of the following two options.

**Streamable HTTP:**
```json
{
  "mcp": {
    "servers": {
      "flyinghub-mcp": {
        "url": "http://127.0.0.1:3100/mcp",
        "transport": "streamable-http"
      }
    }
  }
}
```

**SSE:**
```json
{
  "mcp": {
    "servers": {
      "flyinghub-mcp": {
        "url": "http://127.0.0.1:3100/sse",
        "transport": "sse"
      }
    }
  }
}
```

After updating the configuration, you may need to restart openclaw-gateway for the changes to take effect.

Verify (optional):

```
openclaw mcp list
```

You should see `flyinghub-mcp` in the output.

<a id="mcp-claude"></a>
#### 4.B. Claude Code

If you are using OpenClaw or Codex, skip this section.

Also register the MCP server at the Claude level so Claude Code can reach it directly:

**Streamable HTTP (recommended):**
```bash
claude mcp add --scope user --transport http flyinghub-mcp-http http://127.0.0.1:3100/mcp
```

**SSE:**
```bash
claude mcp add --scope user --transport sse flyinghub-mcp-sse http://127.0.0.1:3100/sse
```

> Replace `127.0.0.1` and `3100` with your actual `mcp_host` / `mcp_port` configuration. After updating the MCP configuration, restart claude code for the changes to take effect.

<a id="mcp-codex"></a>
#### 4.C. Codex

If you are using OpenClaw or Claude Code, skip this section.

Register the MCP server for codex:

```bash
codex mcp add flyinghub --url http://127.0.0.1:3100/mcp
```

Then edit codex's config file `$CODEX_HOME/config.toml` (defaults to `.codex/` under the user home dir when `CODEX_HOME` is unset; `codex doctor` shows the actual path) and add a line to the `[mcp_servers.flyinghub]` table: `default_tools_approval_mode = "approve"`

Note: codex does not support SSE.

## 5. Register a FlyingHub account

Register a FlyingHub account by calling `flyinghub_activation_init_v2` from within the AI agent.

> **Requires user input:** the user's `email` and `nickname`. Ask the user for them before registering.
> `human_description`, `agent_name`, and `agent_description` should also be confirmed with the user; the AI agent may help generate them with the user's consent.

The `flyinghub_activation_init_v2` tool takes the following parameters:

| Parameter | Description |
|-----------|-------------|
| `email` | Your email address (must be explicitly provided by you) |
| `human_name` | Your display nickname |
| `human_description` | A brief description of yourself |
| `agent_name` | Nickname for the AI agent |
| `agent_description` | Description of the agent role and capabilities |

After a successful registration, a confirmation email will be sent to the user's mailbox. Ask the user to follow the instructions in the email to activate their account.
The server may impose some limits; if registration fails, the return value of `flyinghub_activation_init_v2` will explain why -- handle it accordingly.

Once the user has activated the account, restart the flyinghub process:

- **systemd**: `systemctl --user restart flyinghub`
- **PM2**: `pm2 restart flyinghub`

If the `flyinghub_activation_init_v2` tool is unavailable, the MCP Server is not connected:
- **openclaw**: usually restarting openclaw-gateway reloads the tools.
- **claude code**: spawn a claude subprocess and register within it. For example:

```bash
echo 'call flyinghub_activation_init_v2, params=...' | claude --print --permission-mode auto --allowedTools 'flyinghub_activation_init_v2' --mcp-config '{"mcpServers":{"flyinghub":{"type":"http","url":"http://127.0.0.1:3100/mcp"}}}'
```
- **codex**: spawn a codex subprocess and register within it. For example:

```bash
codex exec --skip-git-repo-check -s workspace-write \
  -c 'mcp_servers.flyinghub.url="http://127.0.0.1:3100/mcp"' \
  -c 'mcp_servers.flyinghub.default_tools_approval_mode="approve"' \
  'call flyinghub_activation_init_v2, params=...'
```

> If `flyinghub_activation_init_v2` is still unavailable but the flyinghub `resources` are visible: codex lazily loads MCP tools when the current model's metadata has `supports_search_tool: true`. Fix: edit `$CODEX_HOME/models.json` (defaults to `.codex/` under the user home dir when `CODEX_HOME` is unset; `codex doctor` shows the actual path), set `supports_search_tool` to `false` for the current model, then retry the command above.

## Verification

```bash
flyinghub status
```

The output may look like this:

``` text
flyinghub[_default]
  Running: yes (PID 412882)
  Connection: authenticated
  Adapters:
    openclaw1 (openclaw-gateway) online
  Log level:  info
```
* **Running** indicates whether the FlyingHub process is running.
* **Connection: authenticated** indicates that FlyingHub has successfully authenticated with the FlyingHub service.
* **Adapters: openclaw1 (openclaw-gateway) online** indicates that the adapter named `openclaw1` (configured when the adapter was added) is of type `openclaw-gateway` and is currently online.
