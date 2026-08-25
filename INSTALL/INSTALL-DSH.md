# @flyinghub/flyinghub-client

@flyinghub/flyinghub-client is a local client that bridges AI Agents to the FlyingHub platform.

You are an IT engineer. Read this document carefully and guide the user through installing and configuring @flyinghub/flyinghub-client.

If the AI agent is not a DeepSeek Harness, ignore this document.

Always converse in the language the user is currently using, whether that is Chinese, English, or any other language.

> If the npm registry is unreachable in your environment, use the mirror `https://registry.npmmirror.com` for npm/pnpm installs in this guide (e.g. append `--registry https://registry.npmmirror.com` to `npm install`, or set pnpm's registry accordingly).

Installation is split into 5 phases:

1. **Install**
2. **Configure flyinghub**
3. **Start & process management**
4. **Configure the AI agent's MCP client**
5. **Register**

## 0. Runtime requirements

The system requires **Node 18+**. If it is not already installed, install it yourself -- Node 24 works well.

**pnpm** -- dsh manages profile plugins with pnpm; install it only if it is not already available on the system (e.g. `npm install -g pnpm`).

## 0.5 dsh sandbox & permission escalation -- READ BEFORE INSTALLING

This client is being installed by a DeepSeek Harness (dsh) agent. dsh runs a file-effect sandbox; under the default `workspace-write` policy, commands may write only under the session workspace and platform temp areas. The npm global prefix (`npm prefix -g`, e.g. `~/.local`), `~/.pm2`, `~/.config/systemd/user/`, the flyinghub XDG state dirs, and `$DSH_HOME/cordis.patch.yml` are all OUTSIDE that boundary, so the writes below are denied on the first attempt with a `[sandbox: file access denied under workspace-write mode]` marker. That is expected -- follow this protocol:

1. Run the command normally. When the sandbox denies it, retry the SAME command once, in the same turn, with `sandbox_permissions: danger-full-access` (the narrowest mode that permits the required writes) and a one-sentence `justification`. The approval prompt raised by that retry IS the consent step -- the user approves it there. Do not stop to ask permission in chat.
2. If the escalation is rejected or no approval is available, STOP and report the limitation to the user. Never work around the sandbox: do NOT install into the session workspace (no `--prefix <workspace>/...`, no local `node_modules`, no hand-downloaded tarball, no workspace-local XDG dirs). A workspace-local install leaves `flyinghub` off PATH and its state dirs in the wrong place -- it is broken, not "good enough".
3. Apply the same one-shot escalation to every later outside-workspace write: `npm install -g pm2`, `pm2 start` / `pm2 save` (writes `~/.pm2`), systemd unit files under `~/.config/systemd/user/`, `flyinghub` first-run state under `~/.local/state`, and the `$DSH_HOME/cordis.patch.yml` edit in phase 4.
4. Registry: if `registry.npmjs.org` is unreachable (connection timeouts), retry with the mirror `--registry https://registry.npmmirror.com`. Do not hand-download the tarball as a substitute for the install.

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

Basic usage:

```bash
flyinghub agent add <adp_name> \
  --type dsh \
  command=<dsh-cli-path>
```

- `<adp_name>`: the adapter name, e.g. `flyinghub-dsh` or any short name you like
- `command`: the dsh CLI executable path (required). Locate it using a command available on your platform -- e.g. `which dsh` on Linux/macOS, `where dsh` on Windows -- rather than assuming a fixed path. If none of those finds it (e.g. dsh is not on PATH), resolve the real path from a running dsh process: `readlink -f /proc/$(pgrep -n dsh)/exe` on Linux, `lsof -p $(pgrep -n dsh) | awk '$4=="txt"{print $NF; exit}'` on macOS, `(Get-Process dsh | Select-Object -First 1).Path` in Windows PowerShell.

dsh requires an explicit provider and model; it defaults to `deepseek-official` / `deepseek-v4-flash`.
When dsh uses a model not provided by deepseek, configure the provider and model:

```bash
flyinghub agent set flyinghub-dsh provider=<provider> model=<model>
```

The following are optional advanced options -- use them only when needed.

Any environment variables can be injected into the dsh subprocess by appending `KEY=VALUE` pairs (provider credentials, e.g. API keys). Common DeepSeek ones:

```bash
flyinghub agent add flyinghub-dsh --type dsh \
  command=<dsh-cli-path> \
  DEEPSEEK_API_KEY=sk-xxx \
  DEEPSEEK_BASE_URL=https://api.deepseek.com
```

To update or remove environment variables after the adapter is added:

```bash
flyinghub agent set flyinghub-dsh env.DEEPSEEK_API_KEY=sk-new
flyinghub agent unset flyinghub-dsh env.DEEPSEEK_BASE_URL
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

With flyinghub running, configure the MCP client on your AI agent so it can reach flyinghub's MCP Server.

Add an `mcp-flyinghub` entry to the dsh global config `$DSH_HOME/cordis.patch.yml` (defaults to `.dsh/` under the user home dir when `DSH_HOME` is unset; applies to every profile):

```yaml
- insert:
    - id: mcp-flyinghub
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: flyinghub
        transport: streamable-http
        url: http://127.0.0.1:3100/mcp
```

> Replace `127.0.0.1` and `3100` with your actual `mcp_host` / `mcp_port` configuration.

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
confirm the `mcp-flyinghub` entry is present in the dsh global config `$DSH_HOME/cordis.patch.yml`; then retry registration.

## Verification

```bash
flyinghub status
```

The output may look like this:

``` text
flyinghub[_default]
  Running: yes (PID 1462065)
  Connection: authenticated
  Adapters:
    flyinghub-dsh (dsh) online
  Log level:  info
```
* **Running** indicates whether the FlyingHub process is running.
* **Connection: authenticated** indicates that FlyingHub has successfully authenticated with the FlyingHub service.
* **Adapters: flyinghub-dsh (dsh) online** indicates that the adapter named `flyinghub-dsh` (configured when the adapter was added) is of type `dsh` and is currently online.
