# Thread

1. My Week 12 gap came from a real mistake in my agent work: I could report latency, cost, and token counts, but I could not explain what part of that time was prompt ingestion versus generation.

2. The clean split is prefill versus decode. Prefill is the model reading the prompt. Decode is the model generating the answer token by token. If an agent keeps sending a large repeated prompt, most of the pain usually shows up in prefill.

3. I had also been blurring KV cache and prefix caching. They are related, but not the same. KV cache helps within a generation. Prefix caching helps across requests when the serving system can reuse the same prompt prefix.

4. The important catch is that reuse is stricter than it sounds. "Mostly the same prompt" is not enough. Reordered fields, timestamps, dynamic metadata, or unstable JSON key order can all break the shared prefix.

5. In my TheConversionEngine repo, that means the stable policy and style instructions should stay in one deterministic prefix, and the prospect-specific signal details should come later as the volatile suffix.

6. That changed how I read my own benchmark traces. Higher latency is not just "more tokens." It can mean I paid the prompt-ingestion cost again because I failed to preserve a reusable prefix.

7. The broader lesson for agent systems is straightforward: if you cannot say which tokens are stable, which are volatile, and where prefill dominates, then your latency and cost story is still incomplete.