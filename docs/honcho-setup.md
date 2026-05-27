# Honcho Setup (Conversation Memory)

## Overview

[Honcho](https://github.com/plastic-labs/honcho) provides persistent conversation memory for AI agents. It stores conversation history, user profiles, and metamemory (insights derived from conversations) in PostgreSQL with pgvector for semantic search.

## Why Honcho?

Without Honcho, each conversation starts fresh. With it:
- **Conversation persistence:** Resume conversations across sessions
- **User profiling:** Agent learns about users over time
- **Metamemory:** Agent develops insights from conversation patterns
- **Semantic recall:** Find relevant past conversations by meaning, not just keywords

## Installation via Docker Compose

Honcho runs as a set of Docker containers alongside vLLM.

### 1. Create docker-compose.yml

```yaml
# docker-compose.yml for Honcho
version: '3.8'

services:
  database:
    image: pgvector/pgvector:pg15
    restart: unless-stopped
    environment:
      POSTGRES_DB: honcho
      POSTGRES_USER: honcho
      POSTGRES_PASSWORD: your_secure_password
    volumes:
      - honcho_db:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"

  redis:
    image: redis:8.2
    restart: unless-stopped
    ports:
      - "127.0.0.1:6379:6379"

  api:
    image: honcho-api
    restart: unless-stopped
    depends_on:
      - database
      - redis
    environment:
      DATABASE_URL: postgresql://honcho:your_secure_password@database:5432/honcho
      REDIS_URL: redis://redis:6379
    ports:
      - "0.0.0.0:8001:8000"

  deriver:
    image: honcho-deriver
    restart: unless-stopped
    depends_on:
      - api
    environment:
      HONCHO_BASE_URL: http://api:8000

volumes:
  honcho_db:
```

### 2. Create .env for Honcho

Honcho needs configuration for embeddings and its "deriver" (which generates insights from conversations). Create a `.env` file:

```bash
LOG_LEVEL=INFO
AUTH_USE_AUTH=false
DB_CONNECTION_URI=postgresql+psycopg://postgres:postgres@database:5432/postgres

# Embeddings (OpenAI text-embedding-3-small — very cheap, ~$0.02/1M tokens)
EMBED_MESSAGES=true
EMBEDDING_MODEL_CONFIG__TRANSPORT=openai
EMBEDDING_MODEL_CONFIG__MODEL=text-embedding-3-small
LLM_OPENAI_API_KEY=sk-your-openai-key-here

# Deriver — uses the local Qwen model for generating insights
DERIVER_MODEL_CONFIG__TRANSPORT=openai
DERIVER_MODEL_CONFIG__MODEL=qwen3.5-122b
DERIVER_MODEL_CONFIG__OVERRIDES__BASE_URL=http://localhost:8000/v1
DERIVER_MODEL_CONFIG__OVERRIDES__API_KEY_ENV=DUMMY
DERIVER_WORKERS=1

VECTOR_STORE_TYPE=pgvector
```

> **Note:** The deriver points at the local vLLM instance — so Honcho's memory processing uses the same Qwen model at zero additional cost. Only embeddings use the OpenAI API (text-embedding-3-small is extremely cheap).

### 3. Build and Start Honcho

```bash
docker compose up -d --build
```

First build takes a few minutes (building the API and deriver images). Subsequent starts are instant.

### 4. Configure Hermes to use Honcho

Add to `~/.hermes/config.yaml`:

```yaml
honcho:
  enabled: true
  baseUrl: http://localhost:8001
  peerName: default
  recallMode: hybrid
  writeFrequency: async
  saveMessages: true
```

## Resource Usage

| Component | RAM | CPU | GPU |
|-----------|-----|-----|-----|
| PostgreSQL (pgvector) | ~200 MB | <1% idle | 0 |
| Redis | ~50 MB | <1% idle | 0 |
| Honcho API | ~150 MB | <1% idle | 0 |
| Honcho Deriver | ~100 MB | <1% idle | 0 |
| **Total** | **~500 MB** | **<1%** | **0** |

Like Hermes, Honcho is entirely a CPU/RAM workload with zero GPU usage.

## Verification

```bash
# Check all containers are healthy
docker compose ps

# Test the API
curl http://localhost:8001/health
```

## Data Persistence

Honcho stores all data in a Docker volume (`honcho_db`). To back up:

```bash
docker exec honcho-database-1 pg_dump -U honcho honcho > honcho_backup.sql
```
