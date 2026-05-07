# Sign-off — DA3

Gap status: closed.

Before this explainer I could say that a preference-tuned critic was "learning shortcuts," but I could not explain why that happened in training-mechanics terms. The explainer made the important shift from description to mechanism: pairwise objectives like SimPO and ORPO do not directly teach commercial usefulness, they teach relative separation between chosen and rejected responses, so a small LoRA judge will often internalize the cheapest reliable features first. What I understand now that I did not before is why hard negatives matter so much: if lexical compliance remains different across the pair, the critic can win through shallow cues; if lexical compliance is held constant, the model is forced to learn the deeper business distinction. That changes how I would explain critic limitations and how I would design the next round of preference data.
