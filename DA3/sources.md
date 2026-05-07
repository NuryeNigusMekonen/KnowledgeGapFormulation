# Sources — DA3

## Canonical Sources

1. Rafailov et al., "Direct Preference Optimization: Your Language Model is Secretly a Reward Model."
Why it matters: this is the foundational paper for understanding pairwise preference optimization as relative scoring rather than absolute judgment.
Link: https://papers.neurips.cc/paper_files/paper/2023/hash/a85b405ed65c6477a4fe8302b5e06ce7-Abstract-Conference.html

2. Meng, Xia, and Chen, "SimPO: Simple Preference Optimization with a Reference-Free Reward."
Why it matters: this is the core reference for the average-log-probability reward and target-margin formulation that makes SimPO especially relevant to shortcut learning through easy separability.
Link: https://arxiv.org/abs/2405.14734

3. Hong, Lee, and Thorne, "ORPO: Monolithic Preference Optimization without Reference Model."
Why it matters: this shows how preference learning differs from ordinary supervised fine-tuning by explicitly penalizing the rejected style, which is central to the chosen/rejected formatting question.
Link: https://aclanthology.org/2024.emnlp-main.626/

4. Lambert et al., "RewardBench: Evaluating Reward Models for Language Modeling."
Why it matters: this is the strongest evaluation-side reference for why reward models and judges need challenging comparison sets and can look strong overall while still failing on subtle distinctions.
Link: https://arxiv.org/abs/2403.13787

## Local Grounding

1. `DA3/question.md`
Use it as the authoritative grounding artifact for the actual SignalForge failure mode: lexical compliance winning over commercial usefulness on medium-confidence email tasks.

2. Named artifacts from the peer question:
- `reports/executive_memo.md`
- `methodology_rationale.md`
- `training_data/path_b_preferences.jsonl`

Why they matter: these are the shipped artifacts the explainer is trying to clarify and improve, even though their contents were not directly available in this workspace.

## Argument Map

1. DPO explains why pairwise preference learning is fundamentally about relative separation.
2. SimPO explains why margin-based separation can reward whichever features most easily increase the chosen-over-rejected gap.
3. ORPO explains why rejected examples matter structurally and why this is not equivalent to SFT on chosen responses alone.
4. RewardBench explains why subtle evaluations are necessary when judging whether a preference-trained critic has learned the intended abstraction rather than a shortcut.
