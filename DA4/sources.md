# Sources — DA4

## Canonical Sources

1. Wilson, E. B. (1927), "Probable Inference, the Law of Succession, and Statistical Inference."
Why it matters: this is the original source behind the Wilson interval, which is a much better way to report uncertainty for a 43/50 held-out accuracy than a naive Wald interval.
Link: https://doi.org/10.1080/01621459.1927.10502953

2. Dwork et al., "The reusable holdout: Preserving validity in adaptive data analysis."
Why it matters: this is the central source for why repeated use of a hold-out set during development weakens its reliability as a final out-of-sample estimate.
Link: https://research.ibm.com/publications/the-reusable-holdout-preserving-validity-in-adaptive-data-analysis

3. Dwork et al., "Preserving Statistical Validity in Adaptive Data Analysis."
Why it matters: this is the deeper technical companion to the reusable-holdout argument and helps explain the adaptive-evaluation problem more mechanistically.
Link: https://dwork.seas.harvard.edu/publications/preserving-statistical-validity-adaptive-data-analysis

4. Varma and Simon, "Bias in error estimation when using cross-validation for model selection."
Why it matters: this is the clearest model-selection bias reference for why an evaluation partition used repeatedly during development becomes optimistic as a final performance estimate.
Link: https://bmcbioinformatics.biomedcentral.com/articles/10.1186/1471-2105-7-91

5. Cawley and Talbot, "On Over-fitting in Model Selection and Subsequent Selection Bias in Performance Evaluation."
Why it matters: this generalizes the same lesson beyond one specific workflow and makes the variance/selection-bias tradeoff explicit.
Link: https://jmlr.org/papers/v11/cawley10a.html

## Local / Peer Grounding

1. Peer question details for Day 4
Use it to ground the explainer in the exact Week 11 SignalForge split:
- 109 training pairs
- 50 held-out pairs
- 94.3% training accuracy
- 86.0% held-out accuracy
- named artifacts: `train_simpo.py`, `eval_dev.py`, `MODEL_CARD.md`

2. My own Week 11 evaluation artifacts in TheConversionEngine
Use them as an adjacent grounding reference for why evaluation framing and statistical interpretation matter operationally:
- `ablation_results.json`
- `outputs/reports/current_ablation_report.json`
- `eval/ablation_harness.py`

## Argument Map

1. Wilson explains how to quantify small-sample uncertainty on a binomial held-out accuracy.
2. Dwork explains why repeated hold-out reuse weakens the validity of that estimate under adaptive development.
3. Varma and Simon plus Cawley and Talbot explain why repeated evaluation-driven model choice creates optimistic performance estimates.
4. The peer's Week 11 setup is exactly the kind of workflow where both uncertainty sources matter at once.
