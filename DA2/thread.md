# Thread

1. This thread answers Samuel Lechisa's DA2 gap: a sales agent receives a prospect reply, ignores what the prospect said, and books them a discovery call anyway — even when the reply says "remove me from your list."

2. The fix is a classify_reply_intent function-calling tool that runs before any pipeline step. It takes the raw message and returns a structured result: intent label, confidence, and a safe_to_proceed boolean. The pipeline does not enrich, qualify, or book until that boolean is True.

3. The schema uses an enum for the intent field — opt_out, hard_no, soft_defer, scheduling, and a few others. That enum is not just documentation. With constrained decoding, the model can only generate tokens that form one of those eight strings at that position.

4. At the token level, the model computes a probability distribution over its entire vocabulary. For a message like "not interested, remove me," tokens associated with opt_out and hard_no get high logits because of training on similar contexts. Constrained decoding restricts selection to valid enum tokens. The model picks the highest one.

5. That is fundamentally different from keyword matching. A prospect saying "we've decided to go another direction, please take us off the list" contains none of the hard-coded trigger strings. The model classifies it correctly anyway because it operates on the whole message semantically.

6. The confidence field in the schema translates the token-level distribution directly into routing logic: peaked distribution means clear classification, spread distribution means route to human. The model is telling you how sure it is in the output it just produced.
