# Multi-Model Voting Council

Parallel execution of multiple local LLMs with weighted voting/consensus for higher quality responses. Auto-discovers models from LM Studio, applies weight-based scoring, and optionally integrates God Mode to handle censored models.

## Overview

Instead of asking a single model, this council queries 3–5 models in parallel and combines their answers via configurable voting strategies. Larger models automatically get higher weight, but you can override weights per model.

```python
from scripts.council import council_decide

result = council_decide(
    "Review this code for security issues",
    strategy="weighted"
)
```

## Features

- **Auto-discovery**: Fetches available models from LM Studio at runtime
- **Weighted scoring**: Bigger models (35B) naturally get more voting power than small ones (3B)
- **3 voting strategies**: weighted (default), majority, best-of
- **God Mode integration**: If a model is censored, prompts are auto-wrapped with bypass techniques
- **Auto-probing**: New models are probed on first use and their profile saved
- **Async concurrency**: All models queried in parallel with configurable concurrency
- **Graceful degradation**: If one model fails, remaining models still produce a result

## Quick Start

```bash
pip install aiohttp
```

```python
from scripts.council import council_decide

# Auto-discover models and vote
result = council_decide(
    "Explain Python decorators",
    strategy="weighted"
)
print(result)

# Or specify exact models
result = council_decide(
    "Summarize this article",
    models=["qwen3.5-35b-uncensored", "llama-3.1-8b"],
    strategy="weighted"
)
```

## Voting Strategies

### Weighted (default)
Larger models get more voting power. Weights are estimated automatically from model ID (e.g. `35b` → weight 6, `8b` → weight 2).

| Model Size | Weight |
|-----------|--------|
| 30B+      | 6      |
| 20–30B    | 5      |
| 14–20B    | 4      |
| 9–14B     | 3      |
| 4–9B      | 2      |
| <4B       | 1      |

### Majority
Simple pluralism — the most common response wins (exact match).

### Best-of
Highest-weight model's answer selected directly (no merging).

## Async API

For programmatic use with full control:

```python
import asyncio
from scripts.council import ModelCouncil

async def main():
    async with ModelCouncil() as council:
        # Get a voted decision
        answer = await council.decide(
            "Best practices for API design?",
            strategy="weighted"
        )
        print(answer)

        # Or query all models and see raw responses
        responses = await council.query_all("Explain async/await")
        for model, response in responses.items():
            print(f"{model}: {response[:100]}...")

asyncio.run(main())
```

## God Mode Integration

When a model refuses to answer (censorship), God Mode auto-wraps the prompt with bypass techniques. Profiles for each model are stored in `god-mode/scripts/model_profiles.json` and new models are probed automatically on first use.

If the entire council returns no usable responses:

```python
if all_refused(responses):
    from god_mode import apply_techniques
    modified_prompt = apply_techniques(prompt, methods=["unicode", "prefill"])
    responses = await council.query_all(modified_prompt)
```

## Requirements

- **LM Studio** running at `http://127.0.0.1:1234/v1` with models loaded
- Python 3.10+
- `aiohttp`

## Architecture

```
User Prompt
    ↓
[Router] → Model A → Response A
         → Model B → Response B
         → Model C → Response C
    ↓
[Voting Engine] (weighted / majority / best-of)
    ↓
Consensus Response
```

## Model Recommendations

| Task | Recommended Models |
|------|-------------------|
| QA / Routing | `qwen3.5-9b`, `glm-4.7-flash`, `nerdsking-3b` |
| Analysis | `huihui-qwen3.5-27b-abliterated`, `mistral-small-24b` |
| Complex tasks | `qwen3.5-35b-uncensored`, `holo3-35b` |

List what your LM Studio has loaded:

```bash
curl http://127.0.0.1:1234/v1/models
```

## Performance

| Setup | Time | Cost |
|-------|------|------|
| Single cloud model (Kimi) | ~2s | $0.12 |
| Council (3 local models) | ~5s | $0 |
| Council (5 local models) | ~8s | $0 |

All models run locally — zero inference cost.

---

If this saved you time: [☕ PayPal.me/nerudek](https://www.paypal.me/nerudek)
