# Thinking / Reasoning Token Control

## The Problem

Qwen 3.5 122B includes a "thinking" mode where the model narrates its reasoning process before responding. This is useful for complex tasks but adds significant overhead for chat/agent use:

- **Extra tokens:** Thinking can add 100–500+ tokens per response
- **Slower TTFT:** The model generates thinking tokens before the actual response
- **Noisy output:** Discord/chat messages include "Thinking Process:" blocks

## Solutions

### Option 1: Chat Template (Recommended for Agent Use)

Use the `unsloth.jinja` chat template, which disables thinking by default:

```bash
--chat-template /models/qwen35-122b-hybrid-int4fp8/unsloth.jinja
```

This is the approach used in the main Docker run command. Clean responses, no thinking tokens.

### Option 2: Per-Request Control

If your vLLM is running with the default chat template (thinking enabled), you can disable it per-request:

```json
{
    "model": "qwen3.5-122b",
    "messages": [{"role": "user", "content": "Hello"}],
    "chat_template_kwargs": {"enable_thinking": false}
}
```

This is useful if you want thinking enabled by default but disabled for specific requests.

### Option 3: Generation Config Override

Add to the vLLM launch command:

```bash
--override-generation-config '{"enable_thinking": false}'
```

> **Note:** This approach may not work reliably with all Qwen 3.5 variants. The chat template approach (Option 1) is more reliable.

## When to Enable Thinking

Thinking mode improves quality on:
- Complex reasoning tasks
- Multi-step problem solving
- Code generation with constraints
- Mathematical proofs
- Investment/financial analysis

For these use cases, consider running a second vLLM instance (or using per-request control) with thinking enabled.

## Quality Impact

Informal observations (not rigorous benchmarks):
- Simple queries: No quality difference with thinking disabled
- Complex tasks: Noticeable improvement with thinking enabled
- Tool calling: Thinking can interfere with structured output — disable recommended

The `tool-eval-bench` scores (91/100 sequential) were achieved with thinking disabled.
