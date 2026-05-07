# Structured Reply Classification Before the Pipeline Runs

## Peer Question

Samuel Lechisa's question, restated plainly, is this: in a sales pipeline that currently ignores the meaning of an inbound reply and keeps moving toward booking, what would a real function-calling reply-classification step look like, and what is the model actually doing at the token level when it selects an intent such as `opt_out`, `hard_no`, or `scheduling`?

## Why This Question Matters

This matters because the failure is not cosmetic. In TheConversionEngine, the inbound reply path currently routes based on deterministic keyword checks in `agent/orchestration/handoff.py`. That means the pipeline can detect very obvious strings like `stop` or `unsubscribe`, but still miss semantically equivalent replies that use different wording. If the system fails there, the error propagates into real downstream actions: enrichment, qualification, booking previews, CRM updates, and follow-up drafts may all proceed even though the prospect effectively said no.

For Week 10 and Week 11 portfolio work, this gap affects both system design and evaluation. It changes how the architecture should be described, how safety boundaries should be implemented, and how reply-handling failures should be benchmarked. If the pipeline is going to take consequential actions after reading a prospect reply, then reply interpretation cannot stay a brittle side condition.

## Short Answer

A proper reply-classification step should be a structured decision point before the rest of the pipeline runs. The model should receive the inbound message plus a schema that defines a small set of valid reply intents, such as `opt_out`, `hard_no`, `soft_defer`, `pricing_query`, `scheduling`, `curious`, `legal_handoff`, and `general`. At generation time, structured output or tool calling constrains the model to produce one of those valid values instead of arbitrary text.

At the token level, the model still computes a probability distribution over its vocabulary, but constrained decoding masks that distribution so only continuations consistent with the schema remain legal. So the model is not "thinking in JSON" in the informal prompting sense. It is choosing among a bounded set of valid continuations. That is why provider-native function calling is materially safer than asking the model to "please return JSON."

## Literature Review

Three primary sources are load-bearing here.

Anthropic's tool-use documentation explains how tool calling works in the Messages API. Tools are supplied as structured definitions with a name, description, and `input_schema`, and the model can emit `tool_use` blocks that the host executes. Anthropic also documents an important implementation detail: when tools are provided, the API constructs a special system prompt around those tool definitions. That matters because it clarifies that tool use is not just "the model happened to output JSON," but a first-class API behavior with explicit schema conditioning. Source: [Anthropic, "How to implement tool use"](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use) and [Anthropic, "Tool use with Claude"](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview).

OpenAI's Structured Outputs documentation is the clearest primary reference for schema-constrained generation from the hosted-model side. It states that Structured Outputs ensures responses adhere to the supplied JSON Schema and explicitly highlights enum support. That is directly relevant to reply classification because the intent label is naturally a closed-set enum decision. Source: [OpenAI, "Structured model outputs"](https://platform.openai.com/docs/guides/structured-outputs).

OpenAI's function-calling guide gives the broader application-level framing for tool calls as a multi-step interaction between the application and the model. That source matters because reply classification is not useful unless the application actually branches on the result. Source: [OpenAI, "Function calling"](https://platform.openai.com/docs/guides/function-calling/lifecycle).

Together, these sources support the central mechanism. Anthropic explains tool schemas and emitted tool-use blocks. OpenAI explains schema-constrained outputs and enum-safe generation. Both show that structured outputs are not merely a formatting request but a change in the allowed output space.

## Core Mechanism

Step 1 is input encoding. The inbound reply is tokenized and passed to the model along with either a tool schema or a structured-output schema. That means the model is conditioned on both the message content and the allowed output format.

Step 2 is semantic processing. Internally, the transformer computes contextual representations over the entire reply. This is why a model can detect that "we've decided to go a different direction, please take us off your list" is semantically close to an opt-out or hard refusal even when no exact trigger phrase matches.

Step 3 is constrained generation. At the point where the model must output the `intent` value, the decoder still begins from logits over the vocabulary. But if the schema says the valid values are only:

- `opt_out`
- `hard_no`
- `soft_defer`
- `pricing_query`
- `scheduling`
- `curious`
- `legal_handoff`
- `general`

then the decoding process masks away continuations that would produce invalid outputs. Only token paths that can form one of those enum values remain legal. This is the crucial mechanism.

Step 4 is application branching. The orchestrator reads the structured result and decides whether to continue, stop, or escalate. The model is responsible for classification; the application is responsible for acting on it.

So the system-level division of labor becomes:

- model: interpret reply meaning and return a bounded classification
- orchestrator: enforce the safe branch associated with that classification

That is much safer than a plain-text reply draft being interpreted indirectly by downstream code.

## Case Study From Our Work

The current reply path in `agent/orchestration/handoff.py` uses deterministic checks like:

```python
if any(token in body for token in ("stop", "unsubscribe", "unsub")):
    next_action = "handoff_human"
elif any(token in body for token in _HARD_NO_TOKENS):
    next_action = "handoff_human"
elif self._is_legal_handoff_request(body):
    next_action = "handoff_human"
```

This works for exact-string cases, but it is structurally weak for paraphrases. A message like "we're going another direction, please remove us from future outreach" may carry the same practical meaning as an opt-out without matching the hand-built trigger list exactly.

That is why Samuel's question is so well grounded. The failure is not hypothetical. The current system can route correctly only when the prospect's wording lands inside the curated lexical pattern set.

## Concrete Demonstration

A safer classification boundary would look conceptually like this:

```python
classify_reply_intent_tool = {
    "name": "classify_reply_intent",
    "description": "Classify a prospect's inbound reply before any pipeline step runs.",
    "input_schema": {
        "type": "object",
        "properties": {
            "message_preview": {"type": "string"},
            "intent": {
                "type": "string",
                "enum": [
                    "opt_out",
                    "hard_no",
                    "soft_defer",
                    "pricing_query",
                    "scheduling",
                    "curious",
                    "legal_handoff",
                    "general",
                ],
            },
            "confidence": {"type": "number"},
            "safe_to_proceed": {"type": "boolean"},
        },
        "required": ["message_preview", "intent", "confidence", "safe_to_proceed"],
    },
}
```

Then the application would gate the rest of the pipeline:

```python
if not result["safe_to_proceed"]:
    route_to_human()
else:
    continue_pipeline()
```

This is the important contrast with the current generation layer in `agent/generation/service.py`. That file sends:

```python
payload = {
    "model": settings.openrouter_model,
    "messages": messages,
    "temperature": 0.2,
}
```

There is no `tools` parameter there. So the current system is not doing provider-native tool calling at that point. It is standard chat generation with a JSON-style prompt discipline. That distinction matters because the guarantees are different.

The worked example is straightforward:

- Input reply: `"Not interested, please remove me from your list."`
- Keyword matcher: catches it only if exact tokens overlap the rule list
- Structured classifier: maps it into `opt_out` or `hard_no` because the model can use the full semantic representation and then choose among valid enum outputs

That makes the mechanism visible: the classifier is not safer because it is "more intelligent" in the abstract, but because semantics plus constrained output gives the application a cleaner control point.

## Adjacent Concepts

The first adjacent concept is structured output versus JSON prompting. If you only prompt for JSON, the model may still produce malformed JSON or an invalid label. With provider-native structured outputs, the schema shapes the allowed output space directly.

The second adjacent concept is orchestration boundaries. A model classifying intent is not the same as a model planning the whole pipeline. The classifier can be model-led while the rest of the pipeline remains orchestrator-led. This distinction matters for accurate architecture descriptions.

The third adjacent concept is confidence-aware routing. Once the model returns a bounded label, a confidence field becomes meaningful operationally. Low-confidence classifications are ideal candidates for human review instead of automatic continuation.

## Limitations and Tradeoffs

There are four important tradeoffs.

First, label design matters. If the enum is too coarse, downstream actions become blunt. If it is too fine-grained, the model may struggle to separate near-overlapping classes consistently.

Second, constrained outputs improve reliability of structure, not correctness of meaning. A structured classifier can still choose the wrong legal label if the prompt, schema, or examples are poor.

Third, provider-native structured outputs are safer than free-form JSON prompting, but they still require application-side safety logic. A correct label is only useful if the orchestrator branches correctly on it.

Fourth, classification adds cost and latency. Introducing a new model decision before every reply path creates an extra inference step, so the system should use it where the downstream safety benefit is worth the additional overhead.

## What I Would Change in the Portfolio After Learning This

I would make three changes.

First, I would document the current reply route in `agent/orchestration/handoff.py` as deterministic keyword handling rather than model classification, so the architecture description stops overclaiming what the system is doing.

Second, I would mark reply classification as the correct insertion point for a structured-output or tool-calling upgrade, with a bounded intent schema and a human-handoff branch for unsafe or uncertain cases.

Third, I would evaluate reply handling separately from outbound generation quality. A system that writes good drafts can still fail badly if its reply classification boundary is brittle.

## Sources

- Anthropic, "How to implement tool use": https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use
- Anthropic, "Tool use with Claude": https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
- OpenAI, "Structured model outputs": https://platform.openai.com/docs/guides/structured-outputs
- OpenAI, "Function calling": https://platform.openai.com/docs/guides/function-calling/lifecycle
- Local project evidence: `agent/orchestration/handoff.py` and `agent/generation/service.py` in TheConversionEngine

Direct answer to the question: the right fix is a structured reply-classification step before the rest of the pipeline runs, where the model reads the inbound message and generates an intent from a bounded schema. At the token level, the model still scores vocabulary tokens normally, but constrained decoding masks generation so only valid enum continuations remain legal. That is why provider-native structured outputs or tool use are safer than plain JSON prompting for reply gating in a sales pipeline.
