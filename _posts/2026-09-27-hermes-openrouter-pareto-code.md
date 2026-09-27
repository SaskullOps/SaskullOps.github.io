---
title: "Routing Hermes coding requests with OpenRouter Pareto Code"
date: 2026-09-27 11:00:00 +0200
categories: [Desarrollo, Automatización]
tags: [hermes-agent, openrouter, ai, automation]
image: /assets/img/posts/hermes-openrouter-pareto-code-cover.png
---

I use [Hermes Agent](/posts/hermes-agent-personal-ai-assistant/) as a terminal assistant. For cloud requests, I recently switched its default model to `openrouter/pareto-code`. The name looks like a model ID, but it is a router: OpenRouter chooses a coding model behind it. That matters when you are trying to understand which model answered a request, or why the bill changed.

## The configuration

This is the relevant part of my Hermes configuration, without credentials or machine-specific details:

```yaml
model:
  default: openrouter/pareto-code

openrouter:
  min_coding_score: 0.65
  response_cache: true
  response_cache_ttl: 300

provider_routing:
  sort: price
  require_parameters: true
  data_collection: deny
```

Hermes sends `min_coding_score` to OpenRouter in the `pareto-router` plugin when the model is `openrouter/pareto-code`. The `provider_routing` settings ask OpenRouter to prefer cheaper provider endpoints, require support for the requested parameters and deny data collection. They do not pin a specific model. Response caching is a separate Hermes/OpenRouter setting; its 300-second TTL is not the router's coding score.

The score is easy to misread. `0.65` does **not** mean a model needs to pass 65% of a coding benchmark. OpenRouter maps values from `0.33` up to (but not including) `0.66` to its **medium** coding tier. `0.66` starts the high tier. Within a tier, the router normally picks the cheapest available model on its curated shortlist. Leaving the score out defaults to high.

I chose medium as a cost/quality starting point. It is not a spending cap: the shortlist, model prices and availability can change.

## Checking the model that actually answered

The API response has a `model` field containing the concrete model ID. This small standard-library script makes one test request using the same score and provider preferences. It reads `OPENROUTER_API_KEY` from the environment; the key is not passed as a command-line argument or printed.

```python
import json
import os
import urllib.request

key = os.environ["OPENROUTER_API_KEY"]
payload = {
    "model": "openrouter/pareto-code",
    "plugins": [{"id": "pareto-router", "min_coding_score": 0.65}],
    "provider": {
        "sort": "price",
        "require_parameters": True,
        "data_collection": "deny",
    },
    "messages": [{"role": "user", "content": "Reply OK."}],
    "max_tokens": 16,
}
request = urllib.request.Request(
    "https://openrouter.ai/api/v1/chat/completions",
    data=json.dumps(payload).encode(),
    headers={
        "Authorization": f"Bearer {key}",
        "Content-Type": "application/json",
    },
    method="POST",
)
with urllib.request.urlopen(request, timeout=30) as response:
    result = json.load(response)
print(result["model"])
```

In one test on 27 September 2026, the response reported `google/gemini-3.8-flash`. That is what handled **that request**, not a permanent mapping for my Hermes sessions. A different request may resolve differently after a shortlist update, a provider error or a change in tier. OpenRouter also tries to keep a conversation on the same model and provider for a short period to improve consistency and caching; that stickiness is best-effort.

The response for my 17-token test cost $0.00005175 according to its `usage.cost` field. This number is not a useful estimate for an agent session with long context and tool calls. The router itself adds no fee; the underlying model determines the charge.

## What I would check before changing tiers

For a coding-heavy task, I would compare the resolved `model` and the per-request usage rather than assuming `pareto-code` is always the same model. If responses are not good enough, raise `min_coding_score` to at least `0.66` and repeat the comparison. If latency matters more than price, OpenRouter also offers the `:nitro` variant, which favours throughput within the tier.

Pareto Code is tuned for coding. I would not treat it as an automatic choice for every kind of assistant task. For a predictable model or a fixed price, select a concrete model ID instead.

Sources: [OpenRouter Pareto Router documentation](https://openrouter.ai/docs/guides/routing/routers/pareto-router) · [Hermes Agent](https://hermes-agent.nousresearch.com/docs)
