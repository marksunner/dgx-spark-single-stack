# Complete Walkthrough: Building Steve from Scratch

A step-by-step guide to building a complete single-Spark AI agent from a fresh DGX Spark. Follow these steps in order.

**Total time:** ~2–3 hours (mostly waiting for downloads and builds)

**Prerequisites:**
- A DGX Spark with network access
- SSH access to the Spark
- A HuggingFace account (for model downloads)
- A Discord account (for bot creation)
- An OpenAI API key (for Honcho embeddings — text-embedding-3-small, very cheap)

---

## Phase 1: Model & Inference (~90 minutes)

### 1.1 Build the Hybrid Checkpoint

```bash
cd ~
git clone https://github.com/albond/DGX_Spark_Qwen3.5-122B-A10B-AR-INT4.git
cd DGX_Spark_Qwen3.5-122B-A10B-AR-INT4
./install.sh
```

This downloads models (~75 GB), builds the hybrid checkpoint (~30 min), and creates the Docker image. See [hybrid-checkpoint.md](hybrid-checkpoint.md) for details and troubleshooting.

**Output:** Docker image `vllm-qwen35-v2` and checkpoint at `~/models/qwen35-122b-hybrid-int4fp8/`

### 1.2 Get the Chat Template

Download or copy the `unsloth.jinja` template into your model directory. This suppresses thinking tokens for agent use.

```bash
# If you have it from another source:
cp unsloth.jinja ~/models/qwen35-122b-hybrid-int4fp8/

# Or extract from the Qwen model's tokenizer_config.json and modify
# See thinking-modes.md for details
```

### 1.3 Launch vLLM

```bash
docker run -d --name vllm-qwen35 \
    --gpus all --net=host --ipc=host \
    -v ~/models:/models \
    -v ~/.cache/vllm:/root/.cache/vllm \
    -v ~/.cache/flashinfer:/root/.cache/flashinfer \
    vllm-qwen35-v2 \
    serve /models/qwen35-122b-hybrid-int4fp8 \
    --served-model-name qwen3.5-122b \
    --max-model-len 131072 \
    --gpu-memory-utilization 0.85 \
    --port 8000 --host 0.0.0.0 \
    --load-format fastsafetensors \
    --attention-backend FLASHINFER \
    --enable-chunked-prefill \
    --max-num-batched-tokens 16384 \
    --max-num-seqs 256 \
    --enable-prefix-caching \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --generation-config auto \
    --chat-template /models/qwen35-122b-hybrid-int4fp8/unsloth.jinja \
    --override-generation-config '{"temperature": 0.3, "top_p": 0.95, "top_k": 20, "presence_penalty": 0.0, "repetition_penalty": 1.0}' \
    --speculative-config '{"method":"mtp","num_speculative_tokens":2}'
```

**Wait ~3.5 minutes** for model loading, compilation, and warmup.

### 1.4 Verify Inference

```bash
curl http://localhost:8000/v1/models
# Should return: {"data":[{"id":"qwen3.5-122b",...}]}

curl -s http://localhost:8000/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{"model":"qwen3.5-122b","messages":[{"role":"user","content":"Say hello."}],"max_tokens":50}' \
    | python3 -m json.tool
# Should return a clean response without "Thinking Process:" blocks
```

---

## Phase 2: Conversation Memory (~15 minutes)

### 2.1 Clone and Configure Honcho

```bash
cd ~
git clone https://github.com/plastic-labs/honcho.git
cd honcho
cp docker-compose.yml.example docker-compose.yml
```

Edit `docker-compose.yml` — make the API port accessible:
```yaml
# Change the api ports line to:
ports:
  - "0.0.0.0:8001:8000"
```

### 2.2 Create Honcho .env

```bash
cat > .env << 'EOF'
LOG_LEVEL=INFO
AUTH_USE_AUTH=false
DB_CONNECTION_URI=postgresql+psycopg://postgres:postgres@database:5432/postgres

# Embeddings (OpenAI text-embedding-3-small — very cheap)
EMBED_MESSAGES=true
EMBEDDING_MODEL_CONFIG__TRANSPORT=openai
EMBEDDING_MODEL_CONFIG__MODEL=text-embedding-3-small
LLM_OPENAI_API_KEY=sk-your-openai-key-here

# Deriver — uses the local Qwen model
DERIVER_MODEL_CONFIG__TRANSPORT=openai
DERIVER_MODEL_CONFIG__MODEL=qwen3.5-122b
DERIVER_MODEL_CONFIG__OVERRIDES__BASE_URL=http://host.docker.internal:8000/v1
DERIVER_MODEL_CONFIG__OVERRIDES__API_KEY_ENV=DUMMY
DERIVER_WORKERS=1

VECTOR_STORE_TYPE=pgvector
EOF
```

> **Note:** On DGX Spark with `--net=host`, you can use `http://localhost:8000/v1` or `http://127.0.0.1:8000/v1` for the deriver base URL instead of `host.docker.internal`.

### 2.3 Build and Start Honcho

```bash
docker compose up -d --build
```

Wait for all containers to be healthy:
```bash
docker compose ps
# Should show: database (healthy), redis (healthy), api (healthy), deriver (running)
```

### 2.4 Verify Honcho

```bash
curl http://localhost:8001/health
# Should return: {"status":"ok"}
```

---

## Phase 3: Agent Runtime (~20 minutes)

### 3.1 Create a Discord Bot

1. Go to [Discord Developer Portal](https://discord.com/developers/applications)
2. Create new application → name it
3. **Bot** → Create bot → copy the token
4. Enable **Message Content Intent** (Privileged Gateway Intents)
5. **OAuth2** → URL Generator → select `bot` scope
6. Permissions: Send Messages, Read Message History, Add Reactions, Use Slash Commands
7. Use the generated URL to invite the bot to your server

### 3.2 Install Hermes

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.local/bin/env

# Clone and install
mkdir -p ~/.hermes
cd ~/.hermes
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv --python 3.11 venv
source venv/bin/activate
uv pip install -e '.[all]'
```

### 3.3 Configure Hermes

**`~/.hermes/.env`:**
```bash
DISCORD_BOT_TOKEN=your_bot_token_here
DISCORD_ALLOWED_USERS=your_discord_user_id
CUSTOM_API_KEY=not-needed
CUSTOM_BASE_URL=http://localhost:8000/v1
DISCORD_HOME_CHANNEL=your_channel_id
DISCORD_ALLOW_BOTS=all
# Optional: web search
BRAVE_SEARCH_API_KEY=your_brave_key
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
  cwd: ~/agent-workspace
  timeout: 180
  persistent_shell: true

web:
  search_backend: brave

compression:
  enabled: true
  threshold: 0.5
  target_ratio: 0.2
  protect_last_n: 20

honcho:
  enabled: true
  baseUrl: http://localhost:8001
  peerName: default
  recallMode: hybrid
  writeFrequency: async
  saveMessages: true

discord:
  require_mention: true
  auto_thread: true
  reactions: true

approvals:
  mode: manual
  timeout: 60
```

### 3.4 Create Agent Workspace

```bash
mkdir -p ~/agent-workspace
cat > ~/agent-workspace/AGENTS.md << 'EOF'
# My Agent

Instructions for your agent go here.
Customise this to define your agent's personality and capabilities.
EOF
```

### 3.5 Launch Hermes

```bash
cd ~/.hermes/hermes-agent
PYTHONUNBUFFERED=1 nohup venv/bin/python -u -m hermes_cli.main gateway run --replace > ~/hermes.log 2>&1 &
```

### 3.6 Verify Agent

Check the logs:
```bash
tail -f ~/.hermes/logs/gateway.log
# Should show: "[Discord] Connected as your-bot#1234" and "Gateway running with 1 platform(s)"
```

Then @mention your bot in Discord. It should respond using the local Qwen model.

---

## Phase 4: Monitoring (Optional, ~5 minutes)

### 4.1 Uptime Kuma

```bash
docker run -d --name uptime-kuma \
    --restart unless-stopped \
    -p 3001:3001 \
    -v uptime-kuma-data:/app/data \
    louislam/uptime-kuma:1
```

Access at `http://your-spark-ip:3001`. Add monitors for:
- vLLM: `http://localhost:8000/health`
- Honcho: `http://localhost:8001/health`

---

## Verification Checklist

After completing all phases, verify the full stack:

```bash
# vLLM serving
curl -s http://localhost:8000/v1/models | python3 -m json.tool

# Honcho healthy
curl -s http://localhost:8001/health

# Hermes running
ps aux | grep hermes

# All Docker containers
docker ps --format 'table {{.Names}}\t{{.Status}}'

# Expected containers:
# vllm-qwen35        Up X hours
# honcho-api-1       Up X hours (healthy)
# honcho-database-1  Up X hours (healthy)
# honcho-redis-1     Up X hours (healthy)
# honcho-deriver-1   Up X hours
# uptime-kuma        Up X hours (healthy)
```

## What You Now Have

A single DGX Spark running:
- **122B parameter LLM** at 41–47 tok/s with tool calling
- **AI agent** with Discord integration and 90-turn conversations
- **Persistent memory** across sessions with semantic search
- **Health monitoring** dashboard
- **Total cost:** One DGX Spark + electricity + an OpenAI key for cheap embeddings

🎉
