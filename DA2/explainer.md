# What the Model Is Actually Doing When It Chooses an Intent

This explainer answers Samuel Lechisa's question about a real failure in a sales agent pipeline. The system receives a prospect's reply email in a parameter called `message_preview`. Then it runs seven steps: enrich the company, qualify them, book a discovery call, send a confirmation. The problem is that `message_preview` is never read by any of those steps. The pipeline does not look at what the prospect said before deciding to book them. So if a prospect replies "Not interested, please remove me from your list," the agent qualifies their company, books a discovery call anyway, and sends a confirmation email. The agent is completely unaware that the prospect asked to be removed.

The question was: what would a `classify_reply_intent` function-calling tool look like that reads the message before the pipeline runs, and what is the model actually doing at the token level when it picks an intent?

## The Tool Schema

A function-calling tool works by giving the model a typed schema with an explicit set of valid outputs. For intent classification, the schema looks like this:

```python
classify_reply_intent_tool = {
    "name": "classify_reply_intent",
    "description": (
        "Classify the intent of a prospect's inbound reply before any pipeline "
        "step runs. The pipeline is gated on this result — it will not enrich, "
        "qualify, book, or send anything until this tool has been called."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "message_preview": {
                "type": "string",
                "description": "The full text of the prospect's reply, unmodified."
            },
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
                ]
            },
            "confidence": {
                "type": "number",
                "description": "Model confidence in this classification, 0.0 to 1.0."
            },
            "safe_to_proceed": {
                "type": "boolean",
                "description": (
                    "False for opt_out, hard_no, and legal_handoff. "
                    "True for everything else. The pipeline halts when this is False."
                )
            }
        },
        "required": ["message_preview", "intent", "confidence", "safe_to_proceed"]
    }
}
```

The pipeline checks `safe_to_proceed` before any enrichment, qualification, or booking step runs. If it is `False`, the pipeline stops and routes to a human. The model has read the message. The pipeline has not assumed anything.

This is the structural fix. The important question is what the model is actually doing when it fills the `intent` field.

## What Happens at the Token Level

Language models generate one token at a time. At each position, the model computes a probability distribution over its entire vocabulary — typically 32,000 to 100,000+ tokens. The next token is selected from that distribution, usually by taking the highest-probability token (greedy decoding) or by sampling.

When the model sees the `intent` field in the schema, it knows from the enum that the only legal values are `"opt_out"`, `"hard_no"`, `"soft_defer"`, `"pricing_query"`, `"scheduling"`, `"curious"`, `"legal_handoff"`, and `"general"`. With constrained decoding, the model's probability distribution is masked at each step to allow only tokens that could form one of those valid outputs. This is why function-calling reliably returns structured output instead of free text: the generation process itself is constrained to legal continuations.

The classification decision happens at the token where the enum value begins. Given a prospect message like "Not interested, please remove me from your list," the model computes logits for all vocabulary tokens. The tokens associated with `"opt_out"` and `"hard_no"` will have high logit values because the model has seen many training examples where those phrases co-occur with suppression, unsubscribe, and opt-out contexts. The constrained decoding mask then restricts the selection to only valid enum tokens, and the model picks the highest one among the allowed options.

This is fundamentally different from keyword matching. The keyword-matching approach in the existing `route_inbound_message()` checks whether specific strings are present: `"stop" in body`, `"not interested" in body`. A message like "we've decided to go a different direction, please take us off your list" contains none of the exact tokens in `_HARD_NO_TOKENS`, so it falls through to the scheduling branch and gets a booking confirmation. The model-based approach does not check for exact token presence. It operates on a dense representation of the whole message, so paraphrases, indirect phrasing, and polite variations all activate the right distribution.

## Why the Enum Matters

The enum is not just documentation. It directly shapes the token-level output. When the model reaches the position where the intent value should be, the constrained decoder only allows tokens that start one of the eight enum strings. The model's task is reduced to picking the best match from a closed set. This is what makes structured output from function-calling reliable: the output space is bounded, not just suggested.

This also explains why the enum labels should be chosen carefully. Labels like `"opt_out"` and `"hard_no"` are worth separating because they carry different severity and different downstream actions: `opt_out` typically means the prospect asked to unsubscribe via a normal channel, while `hard_no` means they said explicitly not to contact them again. Giving the model two distinct labels lets it put higher probability on the right one given the message's phrasing, rather than conflating them into a single catch-all bucket.

## Adjacent Concept: Structured Output vs JSON Prompting

It is worth naming the difference between provider-native function calling and asking the model to output JSON in a system prompt. JSON prompting tells the model what format to use, but generation is unconstrained. The model can produce malformed JSON, hallucinate field names, or write a valid JSON structure that includes an intent value not in the enum. Provider-native function calling with constrained decoding enforces the schema at the generation level, so the output is guaranteed to be valid. For a gate that controls whether a pipeline books a meeting, that guarantee matters.

## The Adjacent Concept Worth Watching

The token-level mechanism described here also explains why the `confidence` field in the schema is useful. A high `confidence` value means the model's logit distribution was peaked, with one enum label clearly dominant. A low `confidence` value means the distribution was spread, and the model's selection was close. Routing low-confidence classifications to a human reviewer is a direct translation of what the token-level distribution is telling you: the model is not sure, and the pipeline should not act on an uncertain classification the same way it acts on a clear one.

## Sources

- Anthropic tool use documentation: describes how function-calling schemas constrain model output and how to structure enum inputs for classification tasks.
- Brown et al., "Language Models are Few-Shot Learners" (GPT-3 paper): section on constrained decoding and structured generation provides the theoretical grounding for why token-level masking over valid continuations is the right mechanism.
- Local artifact: agent/orchestration/handoff.py lines 841–883, which shows the current keyword-matching approach that the `classify_reply_intent` tool would replace at the top of `route_inbound_message()`.
