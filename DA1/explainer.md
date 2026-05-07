# Prefill, Decode, and Prompt Reuse in a Multi-Turn Sales Agent

## Peer Question

Your question, restated plainly, is this: in TheConversionEngine, what part of inference time is spent reading the prompt, what part is spent generating the answer, and when does repeated prompt content actually get reused instead of recomputed? More specifically, how do prefill, decode, KV cache, and provider-level prefix caching interact in a multi-turn sales agent, and what prompt-assembly choices would reduce latency and cost without changing the behavior of the agent?

## Why This Question Matters

This matters because your Week 10 and Week 11 artifacts already make latency and cost claims, but right now those claims are mostly stated in token-count terms rather than mechanism terms. In `agent/policies/service.py`, the policy layer passes a large structured context payload into `generation_service.draft_email_from_scaffold()`. That payload includes segment labels, signal summaries, confidence-by-signal entries, top-quartile practices, safe-gap framing, do-not-claim rules, and risk flags. In the Week 11 evaluation artifacts, `held_out_traces.jsonl` and `method.md` report latency, cost, prompt tokens, completion tokens, and model-call counts. The missing link is causal explanation: when latency goes up, is the system paying for a longer decode, or is it repeatedly paying to ingest a long prompt prefix that could have been structured better?

That distinction affects both interpretation and design. If most of the avoidable cost sits in prompt ingestion, then the system should be redesigned around stable prefixes and volatile suffixes. If the main cost sits in long outputs, then the optimization target is different. Without this split, it is hard to defend either the benchmark write-up or the prompt architecture.

## Short Answer

Prefill is the model reading the input prompt and constructing the internal attention state it needs to start generation. Decode is the model generating output tokens one by one after that state exists. KV cache is the within-request memory that makes decode efficient by storing past keys and values so they are not recomputed every step. Prefix or prompt caching is a separate serving-layer optimization across requests: it can reuse the precomputed prefix state from an earlier request, but only when the repeated prefix matches closely enough and the provider retains that cache entry.

For TheConversionEngine, the main avoidable waste is likely repeated prefill on large, mostly stable prompt prefixes. Output length still matters, but for a short outreach draft the repeated prompt prefix is the more suspicious latency and cost driver. The practical fix is to keep policy and scaffold text deterministic and early, and push prospect-specific volatile fields later.

## Literature Review

Four primary sources are especially useful here.

The Hugging Face Transformers caching documentation explains the basic KV-cache mechanism used during autoregressive inference. Its key point is simple: without a cache, every next-token step would recompute the old keys and values again; with a cache, only the current token's new key and value must be added, which makes decoding much cheaper than re-reading the whole prompt every step. That is the clearest reference for the within-request distinction between prefill and decode. Source: [Hugging Face, "Caching"](https://huggingface.co/docs/transformers/cache_explanation).

The vLLM / PagedAttention paper adds the serving-systems view. Kwon et al. show that large-scale inference is often bottlenecked not just by model math in the abstract, but by how the KV cache is stored, shared, and managed in memory. Their contribution is especially important because it separates two issues people often blur together: the existence of a KV cache and the engineering problem of serving many requests efficiently with that cache. Source: [Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention"](https://arxiv.org/abs/2309.06180).

The Anthropic prompt-caching documentation is the clearest practical source for cross-request reuse rules. It explains that cacheable prefixes are structured hierarchically as tools, then system, then messages, and that changing an earlier level invalidates later cached content. It also makes explicit that static content should go first and that cache reuse is sensitive to prompt structure rather than semantic similarity. Source: [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching).

OpenAI's prompt-caching guide makes the API-boundary story concrete for current hosted models: prompt caching works only for exact prefix matches, it starts at a minimum cacheable length, and cached tokens are observable in usage fields. It also clarifies that caching reduces input-side cost and latency, not output generation logic. Source: [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching).

Taken together, these sources give a full stack view. Hugging Face explains why KV caching is necessary for decode. vLLM explains why KV-cache memory management matters for serving throughput. Anthropic and OpenAI explain when repeated prompt prefixes can be reused across separate API calls. That combination is exactly what your question needs.

## Core Mechanism

Step 1 is prompt assembly and tokenization. Your application builds a final prompt from system instructions, scaffold text, structured context, and user-specific details. That string or message array is tokenized into model tokens.

Step 2 is prefill. During prefill, the model processes all input tokens and computes the key and value tensors for every layer and attention head. This is the expensive "read the prompt" phase. Time-to-first-token is heavily affected by this step, because the model cannot begin generating output until prefill has built the initial state.

Step 3 is decode. After prefill, the model predicts one output token at a time. At decode step `t`, the model does not need to recompute every old key and value from the prompt again. It reuses the cached past keys and values and only computes the new token's contribution. This is what the KV cache is for. So decode is sequential and can still be expensive for very long outputs, but it is not equivalent to repeatedly rereading the full prompt.

Step 4 is provider-level prefix reuse across requests. This is where prompt caching enters. If request 2 begins with the same long prefix as request 1, the serving layer may be able to reuse the earlier precomputed prefix state instead of running full prefill again. But that only works if the reusable prefix is actually stable enough and if the provider's routing, retention, and cache policy allow a hit. This is not the same thing as the model's within-request KV cache, even though both involve reusing key/value tensors.

So the key split is:

- KV cache: reuse within one generation
- Prefix or prompt caching: reuse across separate requests

That split matters because a system can benefit from KV cache during every single decode step and still get zero reuse across turns if its prompt prefix changes too much.

## Case Study From Our Work

In `agent/policies/service.py`, the `draft_initial_decision()` path calls `generation_service.draft_email_from_scaffold()` with a large `context` dictionary. That dictionary currently mixes several kinds of information:

- relatively stable framing: `primary_segment`, `recommended_pitch_angle`, `safe_gap_framing`, `do_not_claim`
- volatile prospect-specific details: `signals`, `confidence_by_signal`, `top_quartile_practices`, `risk_flags`, `recommendation_memory`

This is the architectural reason your question matters. If those stable and volatile fields are serialized into one early prompt block, then a small change to a confidence score, a risk flag, or the order of structured fields can force the provider to treat the next request as a new prefix and rerun prefill. That means the system may be paying the long-prompt cost repeatedly even though the high-level policy instructions did not really change.

Your Week 11 traces already suggest this workload is prompt-heavy. In `held_out_traces.jsonl`, the baseline rows show prompt tokens such as 318 and 295 against completion tokens such as 142 and 158. That does not prove cache reuse or non-reuse by itself, but it does show a regime where input-side work is large enough to matter materially.

## Concrete Demonstration

The current prompt-assembly risk looks roughly like this:

```python
context = {
    "primary_segment": prospect.primary_segment_label,
    "segment_confidence": round(prospect.segment_confidence, 2),
    "recommended_pitch_angle": hiring_signal_brief.recommended_pitch_angle,
    "signals": [signal.summary for signal in hiring_signal_brief.signals[:4]],
    "confidence_by_signal": [
        {"signal_name": item.signal_name, "score": item.score}
        for item in hiring_signal_brief.confidence_by_signal
    ],
    "top_quartile_practices": competitor_gap_brief.top_quartile_practices[:3],
    "safe_gap_framing": competitor_gap_brief.safe_gap_framing,
    "do_not_claim": hiring_signal_brief.do_not_claim,
    "risk_flags": risk_flags,
}
```

The problem is not that these fields are wrong. The problem is that fields that change often are mixed together with reusable policy framing.

A more cache-friendly assembly would conceptually separate them like this:

```python
stable_prefix = {
    "policy_rules": stable_policy_rules,
    "style_constraints": stable_style_constraints,
    "safe_gap_framing": stable_safe_gap_framing,
    "do_not_claim_rules": stable_do_not_claim_rules,
    "email_schema": stable_output_schema,
}

volatile_suffix = {
    "company_name": prospect.company_name,
    "signals": current_signal_summaries,
    "confidence_by_signal": current_confidence_scores,
    "risk_flags": current_risk_flags,
    "peer_note": current_peer_note,
}
```

If the stable prefix is serialized deterministically and kept unchanged across many calls, then provider-level prompt caching has a real chance to help. If the whole prompt is rebuilt with unstable ordering, dynamic metadata, or frequently changing early fields, then the system will likely pay full prefill again.

That is the concrete design lesson: prompt reuse is not just about repeated meaning. It is about repeated structure at the token level.

## Adjacent Concepts

The first adjacent concept is time-to-first-token versus total latency. Prefill mostly affects time-to-first-token because the model must finish reading the prompt before it can emit the first output token. Long outputs increase total latency, but not in the same way.

The second adjacent concept is billing. Providers usually expose billing as input tokens and output tokens, not as separate line items for "prefill" and "decode." But mechanistically, input-token cost maps more closely to prefill work, while output-token cost maps more closely to decode work. Prompt caching changes the economics of the input side, not the fact that outputs are still generated anew.

The third adjacent concept is memory pressure. KV caches accelerate decode, but they also consume memory proportional to sequence length and model size. This is why systems papers like vLLM matter: long-context inference is not only a math problem, but also a memory-management problem.

## Limitations and Tradeoffs

There are four important limitations.

First, provider behavior is not identical. Anthropic exposes explicit cache breakpoints and invalidation rules; OpenAI exposes automatic prompt caching with usage reporting. A design that is cache-friendly in principle is not guaranteed to produce identical hit rates across providers.

Second, caching is best-effort. Even a perfectly stable prefix can miss because of retention limits, routing, cache eviction, or insufficient prefix length. That means prompt redesign should be paired with instrumentation rather than treated as a guaranteed win.

Third, aggressive prompt minimization can hurt quality. Some repeated context exists for a reason. If the system removes too much policy or grounding information to save tokens, it may reduce latency at the cost of worse outputs.

Fourth, prompt caching does not change the semantics of generation. It speeds up reuse of the prompt-side state, but decode still happens normally and the output is still computed anew.

## What I Would Change in the Portfolio After Learning This

I would make three changes.

First, I would revise `method.md` so latency and cost are described in terms of prefill versus decode, not just total tokens. A slower row should no longer be explained only as "more tokens used."

Second, I would treat the prompt in `agent/policies/service.py` as two layers: a deterministic stable prefix and a volatile suffix. The stable portion should contain policy, style, and reusable framing rules. The suffix should contain prospect-specific signals, confidence values, and temporary state.

Third, I would add explicit instrumentation for cache-related evidence whenever the provider makes that possible. On OpenAI, that means logging cached prompt tokens from usage fields. On any provider, it means logging enough prompt-assembly metadata to tell whether the repeated prefix was truly stable.

## Sources

- Hugging Face Transformers documentation, "Caching": https://huggingface.co/docs/transformers/cache_explanation
- Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention": https://arxiv.org/abs/2309.06180
- Anthropic documentation, "Prompt caching": https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- OpenAI documentation, "Prompt caching": https://platform.openai.com/docs/guides/prompt-caching

Direct answer to the question: in your sales agent, prefill is the expensive prompt-reading phase, decode is the token-by-token generation phase, KV cache speeds decode within a request, and prefix caching can only reduce repeated prefill across requests when the shared prompt prefix remains stable enough to match at the token level. For TheConversionEngine, the biggest practical improvement is to separate stable policy text from volatile prospect-specific context so the system can stop paying full prompt-ingestion cost every turn.
