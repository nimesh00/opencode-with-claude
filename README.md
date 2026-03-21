# opencode-with-claude

Use [OpenCode](https://opencode.ai) with your [Claude Max](https://claude.ai) subscription — no API credits needed.

One command to install. One command to run.

## How It Works

```
┌─────────────┐       ┌──────────────────┐       ┌───────────────┐
│   OpenCode   │──────▶│  Claude Max Proxy │──────▶│   Claude Max   │
│   (TUI/Web)  │ :3456 │  (local server)   │  SDK  │  (your $100/mo │
│              │◀──────│                    │◀──────│   subscription)│
└─────────────┘       └──────────────────┘       └───────────────┘
```

[OpenCode](https://opencode.ai) speaks the Anthropic REST API. Claude Max provides access via the [Claude Agent SDK](https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk) (not the REST API). The [opencode-claude-max-proxy](https://github.com/rynfar/opencode-claude-max-proxy) bridges the gap — it accepts API requests from OpenCode and translates them into Agent SDK calls using your Claude Max session.

## Quick Start

### One-liner install

```bash
curl -fsSL https://raw.githubusercontent.com/ianjwhite99/opencode-with-claude/main/install.sh | bash
```

This installs everything you need:
- [Claude Code CLI](https://www.npmjs.com/package/@anthropic-ai/claude-code) — authentication with Claude
- [OpenCode](https://www.npmjs.com/package/opencode-ai) — the coding assistant
- [opencode-claude-max-proxy](https://www.npmjs.com/package/opencode-claude-max-proxy) — bridges OpenCode to Claude Max
- **`oc`** — launcher that ties it all together

Then run:

```bash
cd your-project
oc
```

That's it. The `oc` command starts the proxy in the background, waits for it to be ready, and launches OpenCode.

## Prerequisites

- **Node.js >= 18** — [nodejs.org](https://nodejs.org)
- **Claude Max subscription** — the $100/mo plan on [claude.ai](https://claude.ai)
- **curl** — for proxy health checks (pre-installed on most systems)

## Install Methods

### Option 1: curl one-liner (recommended)

```bash
curl -fsSL https://raw.githubusercontent.com/ianjwhite99/opencode-with-claude/main/install.sh | bash
```

Installer options:

```bash
# Skip the Claude login prompt
curl -fsSL ... | bash -s -- --no-auth

# Don't modify shell PATH
curl -fsSL ... | bash -s -- --no-modify-path

# Show help
curl -fsSL ... | bash -s -- --help
```

### Option 2: git clone

```bash
git clone https://github.com/ianjwhite99/opencode-with-claude.git
cd opencode-with-claude
./bin/setup.sh
```

### Option 3: Docker

```bash
git clone https://github.com/ianjwhite99/opencode-with-claude.git
cd opencode-with-claude

# Build and start
docker compose up -d

# Authenticate (first time only)
docker exec -it opencode-with-claude claude login

# Access web UI
open http://localhost:4096
```

### Option 4: Manual

Install the three components yourself:

```bash
npm install -g @anthropic-ai/claude-code opencode-ai opencode-claude-max-proxy

# Authenticate
claude login

# Clear any existing OpenCode Anthropic auth
opencode auth logout
```

Then either use the `oc` launcher from this repo, or start manually:

```bash
# Terminal 1: start the proxy
CLAUDE_PROXY_PASSTHROUGH=1 claude-max-proxy

# Terminal 2: start OpenCode pointed at the proxy
ANTHROPIC_API_KEY=dummy ANTHROPIC_BASE_URL=http://127.0.0.1:3456 opencode
```

## Usage

### `oc` command

The `oc` launcher handles everything — starts the proxy, waits for health, launches OpenCode, and cleans up on exit:

```bash
oc              # Start OpenCode TUI in current directory
oc web          # Start OpenCode web UI on port 4096
oc --help       # Show help
oc --version    # Show component versions
```

All arguments are passed through to `opencode`, so anything that works with `opencode` works with `oc`.

### Docker

```bash
# Start the container
docker compose up -d

# Access the web UI
open http://localhost:4096

# Run claude commands inside the container
docker exec -it opencode-with-claude claude login
docker exec -it opencode-with-claude claude auth status

# Run opencode commands inside the container
docker exec -it opencode-with-claude opencode auth list

# Or use the convenience wrappers (symlink to your PATH)
ln -s "$(pwd)/bin/claude" ~/.local/bin/claude
ln -s "$(pwd)/bin/opencode" ~/.local/bin/opencode
```

## Configuration

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_PROXY_PORT` | random | Port for the proxy server |
| `CLAUDE_PROXY_WORKDIR` | `$PWD` | Working directory for the proxy |
| `OC_SKIP_AUTH_CHECK` | unset | Set to `1` to skip Claude auth check on launch |

### OpenCode config

The `oc` launcher sets `ANTHROPIC_API_KEY` and `ANTHROPIC_BASE_URL` as environment variables, so no config file changes are needed.

If you prefer a config file (e.g., for running OpenCode without the `oc` launcher), copy the template:

```bash
cp config/opencode.json ~/.config/opencode/opencode.json
```

This sets OpenCode to use the proxy at `http://127.0.0.1:3456` with a dummy API key.

### Docker volumes

The Docker setup persists data across restarts:

| Volume | Path | Purpose |
|--------|------|---------|
| `opencode-data` | `~/.local/share/opencode` | OpenCode data |
| `opencode-state` | `~/.local/state/opencode` | OpenCode state |
| `opencode-cache` | `~/.cache/opencode` | OpenCode cache |
| `opencode-config` | `~/.config/opencode` | OpenCode config |
| `claude-auth` | `~/.claude` | Claude authentication |

Your workspace is mounted from `~/workspace` on the host.

## Troubleshooting

### "Claude not authenticated"

```bash
claude login
```

This opens a browser for OAuth. Your Claude Max subscription credentials are needed.

### "Proxy failed to start"

1. Check Claude auth: `claude auth status`
2. Try starting the proxy manually: `CLAUDE_PROXY_PASSTHROUGH=1 claude-max-proxy`
3. Check if the port is in use: `lsof -i :3456`

### "Proxy didn't become healthy within 10 seconds"

The proxy takes a moment to initialize. If this persists:
- Ensure `claude auth status` shows `loggedIn: true`
- Check your internet connection
- Try a specific port: `CLAUDE_PROXY_PORT=3456 oc`

### Port conflicts

The `oc` launcher uses a random available port by default. To use a fixed port:

```bash
CLAUDE_PROXY_PORT=3456 oc
```

### Updating components

```bash
npm update -g @anthropic-ai/claude-code opencode-ai opencode-claude-max-proxy
```

Or re-run the installer — it's idempotent and will update existing installations.

## Project Structure

```
opencode-with-claude/
├── install.sh           # curl | bash one-liner installer
├── README.md            # This file
├── Dockerfile           # All-in-one Docker image
├── docker-compose.yml   # Docker Compose config
├── config/
│   └── opencode.json    # OpenCode config template
└── bin/
    ├── oc               # Launcher: proxy + opencode in one command
    ├── setup.sh         # Git clone setup script
    ├── entrypoint.sh    # Docker entrypoint
    ├── claude           # Docker wrapper for claude CLI
    └── opencode         # Docker wrapper for opencode CLI
```

## Disclaimer

This project is **unofficial** and not affiliated with Anthropic or OpenCode. It uses Anthropic's public npm packages with your own authenticated account. Whether this complies with Anthropic's Terms of Service is your responsibility to determine.

## License

MIT
