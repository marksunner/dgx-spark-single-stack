# Hermes Agent Setup

## Overview

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is an AI agent runtime that connects to LLM inference backends (like vLLM) and messaging platforms (Discord, Slack, Matrix, etc.). It provides tool use, conversation management, and gateway features.

Running Hermes on the same DGX Spark as vLLM gives the lowest possible latency — the agent talks to the model over localhost.

## Prerequisites

- vLLM running and serving on port 8000 (see [vllm-config.md](vllm-config.md))
- Python 3.11+ (DGX Spark ships with 3.12)
- Git
- A Discord bot token (or other messaging platform credentials)

## Installation

### 1. Install uv (Python package manager)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.local/bin/env
```

### 2. Clone and install Hermes

```bash
mkdir -p ~/.hermes
cd ~/.hermes
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv --python 3.11 venv
source venv/bin/activate
uv pip install -e '.[all]'
```

### 3. Create configuration

**`~/.hermes/.env`:**
```bash
DISCORD_BOT_TOKEN=your_discord_bot_token_here
DISCORD_ALLOWED_USERS=comma,separated,user,ids
CUSTOM_API_KEY=not-needed
CUSTOM_BASE_URL=http://localhost:8000/v1
DISCORD_HOME_CHANNEL=your_home_channel_id
DISCORD_ALLOW_BOTS=all
```

**`~/.hermes/config.yaml`:**
```yaml
model:
  default: qwen3.5-122b
  provider: custom
  base_url: http://localhost:8000/v1

toolsets:
  - hermes-cli
  - hermes-discord
  - discord

agent:
  max_turns: 90
  gateway_timeout: 1800

terminal:
  backend: local
  cwd: /home/your_user/agent-workspace
  timeout: 180
  persistent_shell: true

web:
  search_backend: brave

compression:
  enabled: true
  threshold: 0.5
  target_ratio: 0.2
  protect_last_n: 20

discord:
  require_mention: true
  auto_thread: true
  reactions: true

approvals:
  mode: manual
  timeout: 60
```

### 4. Create agent workspace

```bash
mkdir -p ~/agent-workspace
cat > ~/agent-workspace/AGENTS.md << 'EOF'
# Agent Workspace

Your agent's instructions go here.
EOF
```

### 5. Start Hermes

```bash
cd ~/.hermes/hermes-agent
PYTHONUNBUFFERED=1 nohup venv/bin/python -u -m hermes_cli.main gateway run --replace > ~/hermes.log 2>&1 &
```

Check the logs:
```bash
# Main gateway log
tail -f ~/.hermes/logs/gateway.log

# Agent interaction log
tail -f ~/.hermes/logs/agent.log
```

You should see:
```
[Discord] Connected as your-bot#1234
✓ discord connected
Gateway running with 1 platform(s)
```

## Resource Usage

| Metric | Value |
|--------|-------|
| RAM (idle) | ~140 MB |
| RAM (active conversation) | ~200–300 MB |
| CPU (idle) | <1% |
| CPU (active) | 2–5% (brief spikes) |
| GPU | 0 |
| Disk | ~500 MB (install + deps) |

Hermes is entirely a CPU/RAM workload. It does not compete with vLLM for GPU resources.

## Discord Bot Setup

1. Go to [Discord Developer Portal](https://discord.com/developers/applications)
2. Create a new application
3. Go to **Bot** → Create bot
4. Enable **Message Content Intent** (under Privileged Gateway Intents)
5. Copy the bot token
6. Generate an invite URL: **OAuth2** → **URL Generator** → Select `bot` scope
7. Permissions needed: Send Messages, Read Message History, Add Reactions, Use Slash Commands
8. Invite the bot to your server

## Running as a Service (Optional)

For production use, create a systemd user service:

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/hermes-agent.service << 'EOF'
[Unit]
Description=Hermes Agent Gateway
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/%u/.hermes/hermes-agent
ExecStart=/home/%u/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run --replace
Restart=on-failure
RestartSec=10
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable hermes-agent
systemctl --user start hermes-agent
loginctl enable-linger $USER
```
