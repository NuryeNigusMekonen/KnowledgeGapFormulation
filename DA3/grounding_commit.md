# Grounding Commit — DA3

Artifact edited: `methodology_rationale.md`, the Week 11 evaluation write-up, and future preference-data design notes for TheConversionEngine Path B critic.

The main change after this explainer is conceptual but operationally important. Instead of describing the critic's weakness as a vague tendency to "latch onto shortcuts," I can now explain it as a property of pairwise preference training: the model is rewarded for separating chosen from rejected responses, so it will learn the cheapest predictive distinction first if the data allows it. That means future preference-data work should include more hard negatives that remain lexically compliant while differing in business usefulness, because those pairs force the critic to model the real commercial distinction rather than surface style cues. The related write-up change is that any claim about critic readiness should now distinguish style-and-compliance detection from deeper business-judgment calibration.
