# DA2 Sources

## Canonical Sources

1. Anthropic documentation on tool use.
Why it matters: this is the clearest primary source for how provider-native tool calling works, including typed schemas, constrained generation, and the `tool_use` / `tool_result` pattern.

2. OpenAI documentation on function calling and structured outputs.
Why it matters: useful for contrasting provider-native structured output with ordinary JSON prompting and for explaining why schemas constrain what the model is allowed to emit.

3. Brown et al., "Language Models are Few-Shot Learners" (GPT-3).
Why it matters: helpful for the token-by-token generation framing behind the explainer's discussion of logits, vocabulary distributions, and constrained continuation.

## Local Evidence From TheConversionEngine

1. `agent/orchestration/handoff.py`
Use it to show the current keyword-matching reply handling path that the proposed `classify_reply_intent` tool would replace.

2. `agent/generation/service.py`
Use it to contrast ordinary prompted generation with provider-native function calling and to show where the current generation layer is only rewriting content.

3. `agent/orchestration/service.py`
Use it as the broader pipeline context for where message handling and downstream actions currently sit in the orchestrated flow.

## Tool / Pattern Used

1. Function-calling schema design with an enum-constrained intent field.
Why it matters: this is the concrete pattern the explainer used to make the reply-classification mechanism visible instead of describing it only in abstract terms.
