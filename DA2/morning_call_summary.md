# Morning Call Summary — DA2

Both draft questions came in broader than they needed to be.

Samuel Lechisa's original draft asked why an agent pipeline ignores a reply message and books meetings anyway. The interrogation pushed it toward the concrete mechanism: not "why is this a bug" but "what would a function-calling tool that reads the message actually look like, and what is the model doing at the token level when it picks an intent label." That grounding in both the tool schema and the token-level mechanics made it resolvable in a single explainer and directly connected to the code in agent/orchestration/handoff.py.

My own draft asked how to describe an agent architecture accurately when the model is only one component. The call sharpened it to the specific boundary question: when is it correct to say the model is using tools versus the orchestrator is using tools while the model only rewrites content. That tied the question to concrete artifacts — the toolchain sequence in agent/orchestration/service.py, the generation rewrite in agent/generation/service.py, and the descriptions in method.md and README.md that currently overstate the model's role in tool selection.

The key thing the call resolved was that both questions are ultimately about the same underlying issue from opposite directions. Samuel's question is about what happens when the model is not involved in a decision that it should be. My question is about how to describe a system where the model's role is narrower than the language implies. Confirming that connection made both questions sharper.
