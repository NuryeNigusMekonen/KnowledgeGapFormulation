# DA2 Candidate Sources

## Core Sources

1. Anthropic documentation on tool use and prompt engineering for tool calling.
Why it matters: useful for separating native tool use from ordinary text generation with structured prompts.

2. OpenAI documentation on function calling / structured outputs.
Why it matters: helpful for understanding what guarantees provider-native schemas and tool calls add beyond plain JSON prompting.

3. MCP specification or official documentation.
Why it matters: useful if the explainer ends up covering how tool capability is exposed to a model and why tool descriptions matter.

## Local Evidence From My Project

1. `agent/orchestration/service.py`
Use it to show that tool order, side effects, retries, and delivery routing are mostly scaffolded in code.

2. `agent/generation/service.py`
Use it to show that the current model call rewrites a draft through prompted JSON output rather than native function calling.

3. `agent/policies/service.py`
Use it to show how hard-coded policy, risk flags, and scaffold assembly shape what the model is allowed to rewrite.

4. `method.md` and `README.md`
Use them to identify where the architecture is described in higher-level language that now needs a cleaner account of scaffolding versus model-led behavior.

## Likely Argument Map

1. A model can participate in an agent system without being the component that selects or executes tools.
2. Deterministic orchestration, native function calling, and structured JSON prompting are different control patterns with different guarantees.
3. The Tenacious system currently leans heavily on scaffolding and guardrails, so the explainer should clarify where the model actually contributes and where code enforces behavior.