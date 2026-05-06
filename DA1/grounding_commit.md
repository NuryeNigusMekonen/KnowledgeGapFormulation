# Grounding Commit — DA1

Artifact edited: `method.md`, `held_out_traces.jsonl`, and `agent/policies/service.py` in TheConversionEngine (Week 10 repo).

The explainer closed a real interpretability gap in the Week 10 and Week 11 artifacts. The change to `method.md` is conceptual: latency and cost should be described in terms of prefill versus decode, not only total tokens. The change to `held_out_traces.jsonl` is interpretive: rows with higher latency and prompt-token counts should be read as possible prompt-ingestion or repeated-prefix costs, not just "slower model behavior." The design implication for `agent/policies/service.py` is to keep stable policy and style instructions in a deterministic prefix and move volatile prospect-specific details later, so the system has a real chance of benefiting from prompt-prefix reuse when the provider supports it.

What changed and why: before this explainer I was treating repeated prompt content as if it were automatically reusable. After the explainer, I can distinguish stable prefix from volatile suffix, and I can make latency and cost claims that refer to the actual inference mechanism instead of only to total token counts.
