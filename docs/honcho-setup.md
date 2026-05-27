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

### 2. Start Honcho

```bash
docker compose up -d
```

### 3. Configure Hermes to use Honcho

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
