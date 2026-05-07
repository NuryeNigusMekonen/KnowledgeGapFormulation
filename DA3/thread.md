# Thread

1. This thread answers Ephrata Nebiyu's Day 3 gap: why a preference-tuned critic can learn lexical shortcuts faster than deeper commercial usefulness.

2. Pairwise objectives like DPO, SimPO, and ORPO do not directly teach "business value" in the abstract. They teach the model to score the chosen response above the rejected one.

3. That means the model usually learns the easiest reliable separator first. If chosen responses are often short, clean, question-led, and banned-phrase-free, those surface cues can become a cheap shortcut.

4. A small LoRA judge is especially vulnerable to this because it has limited adaptation capacity. It is easier to encode lexical compliance than a deeper notion of commercial usefulness.

5. Hard negatives matter because they remove the easy lexical route. If both responses are equally clean on the surface, but only one has a commercially useful ask, the model has to learn the deeper distinction to reduce loss.

6. That is why harder negatives are not just "more data." They change the reward geometry the critic sees during post-training.

7. The practical lesson for Path B systems is simple: if your rejected examples are too easy, your critic may become a strong style-and-compliance detector while still being weak at the actual business judgment you care about.
