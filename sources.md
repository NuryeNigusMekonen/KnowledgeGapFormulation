# Sources

## Core Sources

1. Kwon, Woosuk, et al. Efficient Memory Management for Large Language Model Serving with PagedAttention. SOSP 2023.
Why it matters: this is the clearest source for explaining KV-cache behavior at serving time and why memory management affects latency and throughput.

2. Anthropic Documentation, Prompt Caching.
Why it matters: this is the practical source for exact-prefix reuse, cache breakpoints, invalidation rules, TTL, and how to tell whether reuse happened in an API workflow.

## Optional Supporting Source

1. vLLM blog, vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention.
Why it matters: this gives a more accessible explanation of how PagedAttention manages KV-cache blocks and why that improves serving efficiency.

## Local Evidence From My Project

1. agent/policies/service.py
Use it to show which parts of the generation prompt are stable and which are prospect-specific.

2. held_out_traces.jsonl
Use it to compare prompt tokens, completion tokens, latency, and number of model calls.

3. method.md
Use it to show where the benchmark currently reports latency and cost without a prefill versus decode explanation.

## Argument Map

1. The PagedAttention source explains why prior-token state matters during inference.
2. The Anthropic docs explain when repeated prompt prefixes are actually reusable across requests.
3. The local repo artifacts show why this is a real gap in my own agent and benchmark work.