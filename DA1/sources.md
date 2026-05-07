# Sources

## Core Sources

1. Hugging Face Transformers documentation, "Caching."
Why it matters: this is the clearest primary technical reference for what a KV cache actually stores and why decoding can reuse past keys and values instead of recomputing them every step.

2. Kwon, Woosuk, et al. Efficient Memory Management for Large Language Model Serving with PagedAttention. SOSP 2023.
Why it matters: this is the systems-level source for why KV-cache memory management matters in real serving infrastructure, not just in a textbook attention diagram.

3. Anthropic documentation, Prompt Caching.
Why it matters: this is the clearest practical source for exact-prefix reuse, cache hierarchy, invalidation rules, and how cacheability depends on prompt structure.

4. OpenAI documentation, Prompt Caching.
Why it matters: this is the API-boundary source for automatic prompt caching, exact prefix matching, cached token reporting, and how repeated prompt prefixes affect latency and cost in hosted inference.

## Local Evidence From My Project

1. agent/policies/service.py
Use it to show how stable policy framing and volatile prospect-specific fields are currently mixed in the prompt context passed into the generation layer.

2. held_out_traces.jsonl
Use it to compare prompt tokens, completion tokens, latency, and number of model calls in a prompt-heavy workload.

3. method.md
Use it to show where the benchmark currently reports latency and cost without a prefill versus decode explanation.

## Argument Map

1. Hugging Face explains the within-request KV-cache mechanism that makes decode efficient.
2. PagedAttention explains why KV-cache management becomes a serving bottleneck at system scale.
3. Anthropic and OpenAI explain when repeated prompt prefixes can actually be reused across requests.
4. The local repo artifacts show why this is a real interpretability and prompt-design gap in TheConversionEngine.
