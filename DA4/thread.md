# Thread

1. This thread answers Kidane Gebremedhin Gidey's Day 4 gap: how much should we trust an 86.0% held-out accuracy when the held-out set has only 50 examples and has been reused throughout development?

2. The first issue is small-sample uncertainty. An accuracy of 43/50 is still a noisy binomial estimate, so it should be reported with an interval, not just a headline point estimate.

3. The second issue is adaptive reuse. If the same held-out split guided repeated development decisions, it is no longer a clean final test set. It becomes a validation estimate with some optimism baked in.

4. Those are different problems. A confidence interval captures sampling noise. It does not fix bias from repeatedly consulting the same held-out partition.

5. That means the 8.3-point train–held-out gap should be reported as an observed train–validation gap, not as definitive proof of true generalization.

6. The honest reporting move is: give the numerator and denominator, give a Wilson-style interval, and explicitly say the split was reused during development.

7. The clean long-term fix is a truly untouched test set after the pipeline is frozen, ideally once the V0.2 dataset expansion gives enough examples to afford it.
