# Week 12 Knowledge Gap Formulation

This repository collects a set of paired research artifacts focused on closing mechanism-level gaps from my Week 10 and Week 11 portfolio work. The package combines daily questions, technical explainers, public writing, call summaries, source trails, and grounded portfolio updates.

The package is organized by day. Each daily folder captures the full paired workflow required by the Week 12 challenge:

- my final sharpened question
- the morning call summary
- the explainer I wrote for my peer's question
- the public thread draft
- the evening call summary
- sign-off and grounding materials for the explainer I received
- the source list used for the explainer

## Week Scope

The work completed here covers four paired research days:

- **Day 1 — Inference-Time Mechanics:** prefill versus decode, KV cache versus prefix caching, and prompt reuse in TheConversionEngine.
- **Day 2 — Agent Tool-Use Internals:** orchestrator-driven pipelines versus model-driven tool use, plus structured reply-intent classification.
- **Day 3 — Training and Post-Training Mechanics:** preference tuning with SimPO and ORPO, shortcut learning in critics, and hard-negative design.
- **Day 4 — Evaluation and Statistics:** reliability of small held-out sets, adaptive reuse of validation splits, and statistical interpretation of benchmark claims.

## Public Artifact URLs

- Day 1 blog post: https://open.substack.com/pub/nuryenigus/p/what-actually-gets-faster-in-a-sales?r=8bo5sh&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true
- Day 2 blog post: https://open.substack.com/pub/nuryenigus/p/what-the-model-is-actually-doing?r=8bo5sh&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true
- Day 3 blog post: https://open.substack.com/pub/nuryenigus/p/why-preference-tuned-critics-learn?r=8bo5sh&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true
- Day 4 blog post: https://open.substack.com/pub/nuryenigus/p/how-much-should-you-trust-a-small?r=8bo5sh&utm_campaign=post&utm_medium=web&showWelcomeOnShare=true

## Repository Structure

- `DA1/`: Day 1 package, including question, explainer, thread, call summaries, sign-off, grounding commit, sources, and day README
- `DA2/`: Day 2 package, including question, explainer, thread, call summaries, sign-off, grounding commit, sources, and day README
- `DA3/`: Day 3 package, including question, explainer, thread, call summaries, sign-off, grounding commit, sources, and day README
- `DA4/`: Day 4 package, including question, explainer, thread, call summaries, sign-off, grounding commit, sources, and day README
- `docs/`: challenge briefs and supporting reference materials for Weeks 10, 11, and 12

## Day-by-Day Focus

### Day 1

Grounded in TheConversionEngine prompt assembly and Week 11 latency artifacts. The main gap was understanding what actually happens during prefill and decode, how KV cache differs from provider-level prompt caching, and what prompt changes break reuse across turns.

### Day 2

Grounded in TheConversionEngine reply-routing and orchestration code. The main work separated deterministic orchestration from model-led tool use and also explored how structured reply classification and constrained outputs can create a safer boundary before downstream actions run.

### Day 3

Grounded in Week 11 Path B critic work and peer research on preference-tuned judges. The main focus was why pairwise objectives like SimPO and ORPO can reward lexical shortcuts before deeper business judgment, and how hard negatives can force a critic to learn the intended distinction.

### Day 4

Grounded in Week 11 evaluation artifacts and paired research on validation reliability. The main focus was how to interpret held-out accuracy when the held-out split is small and repeatedly reused during development, and how to separate sampling uncertainty from adaptive evaluation bias.

## Purpose

The purpose of this repository is not only to collect blog posts. Each day is meant to close a real gap that affected how I described, evaluated, or designed systems I had already built in Weeks 10 and 11. The package therefore combines:

- public-facing technical writing
- paired research and feedback records
- source-backed explanations
- grounded portfolio implications
