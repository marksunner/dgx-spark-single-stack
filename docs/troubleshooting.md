# Troubleshooting

## Common Issues

### OOM During CUDA Graph Capture

**Symptoms:** Container crashes within first 2–3 minutes of startup during the compilation phase.

**Cause:** `--gpu-memory-utilization` too high (typically 0.90) combined with MTP and default batch sizes.

**Fix:**
```bash
--gpu-memory-utilization 0.85
--max-num-batched-tokens 16384
--max-num-seqs 256
```

If it still OOMs, try `0.80` and `8192` batched tokens. You can increase gradually once stable.

### OOM Under Concurrent Load

**Symptoms:** Works fine for single requests but crashes when multiple users interact simultaneously.

**Cause:** No sequence limit + large batch token default = memory exhaustion under parallel requests.

**Fix:** The `--max-num-seqs 256` and `--enable-chunked-prefill` flags prevent this. Chunked prefill breaks large prompts into manageable chunks instead of allocating memory for the entire prompt at once.

### Thinking Tokens in Responses

**Symptoms:** Responses start with "Thinking Process:" or contain reasoning blocks.

**Fix:** Use the `unsloth.jinja` chat template:
```bash
--chat-template /models/qwen35-122b-hybrid-int4fp8/unsloth.jinja
```

See [thinking-modes.md](thinking-modes.md) for details.

### `content: null` in API Responses

**Symptoms:** API returns valid JSON but with `"content": null` in the message.

**Cause:** Using `--reasoning-parser qwen3`. This flag is incompatible with agent use.

**Fix:** Remove `--reasoning-parser qwen3` from the launch command. Do not use it.

### Model Download Instead of Local Load

**Symptoms:** Startup shows download progress bars (downloading 69+ GB from HuggingFace) instead of loading from local disk.

**Cause:** Model path not mounted into the Docker container, or mounted at wrong path.

**Fix:** Verify your `-v` mount matches the model path in the `serve` command:
```bash
-v /host/path/to/models:/models          # Mount
serve /models/qwen35-122b-hybrid-int4fp8  # Must match
```

### Hermes Log File Empty

**Symptoms:** `nohup` redirect produces an empty log file even though the process is running.

**Cause:** Python output buffering.

**Fix:** Use unbuffered Python:
```bash
PYTHONUNBUFFERED=1 nohup python -u -m hermes_cli.main gateway run --replace > hermes.log 2>&1 &
```

Check Hermes's own logs at `~/.hermes/logs/gateway.log` — these are always written regardless of stdout buffering.

### Discord Bot Connected But Not Responding

**Symptoms:** Logs show "Connected as bot#1234" but the bot doesn't respond to messages.

**Possible causes:**
1. **Message Content Intent not enabled** in Discord Developer Portal
2. **`require_mention: true`** — bot needs @mention to respond
3. **`DISCORD_ALLOWED_USERS`** doesn't include your user ID
4. **vLLM not ready** — check if `curl http://localhost:8000/v1/models` returns a response

### Power Cycle Recovery

If you need to hard-reset the Spark (e.g., after an unrecoverable OOM):

1. Power cycle the Spark
2. Wait 2–3 minutes for boot
3. SSH in and check Docker: `docker ps`
4. Containers with `--restart unless-stopped` will auto-recover
5. Hermes (if running as nohup) will need manual restart
6. Consider setting up a systemd service for automatic recovery (see [hermes-setup.md](hermes-setup.md))

## Performance Debugging

### Check Token Speed

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.5-122b","messages":[{"role":"user","content":"Write a 200 word essay about stars."}],"max_tokens":300}' \
  | python3 -c '
import json, sys, time
# Note: for proper tok/s measurement, time the request externally
r = json.load(sys.stdin)
print(f"Completion tokens: {r[\"usage\"][\"completion_tokens\"]}")
print(f"Prompt tokens: {r[\"usage\"][\"prompt_tokens\"]}")
'
```

### Check GPU Memory

```bash
nvidia-smi
```

### Check Running Containers

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```
