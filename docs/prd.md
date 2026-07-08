# CircuitSight — Behavior Spec & BrainLift

*A small vision-language model that reads an AP-Physics-C circuit schematic and solves it — grounding every claim in the image and verifying its own steps.*

---

## Part 1 — Behavior Spec

### The one-sentence spec

Given a schematic image and a question, the model identifies the components (pointing to where each one is in the image), states the topology, writes the governing equations, solves for the requested quantity, and checks its own answer — or, if a value is genuinely illegible, says so and reports `null` rather than inventing it.

### Required output structure (in this order — perception before answer)

1. **Components** — the inventory (e.g., "1 voltage source, 3 resistors"), each with the image region it occupies.
2. **Concepts used** — the minimal set of solving concepts the problem actually requires (Ohm's law, parallel/series combination).
3. **Topology** — what is in series / parallel, referencing the grounded components.
4. **Equations & solution** — step by step, with intermediate values (node voltages, branch currents).
5. **Verification** — a consistency check (e.g., plug the answer back: V = I·R_eq; confirm a parallel combination is smaller than its smallest branch).
6. **FINAL** — a machine-readable line: `FINAL: {answer_id, answer_current, R_eq, n_resistors, abstain}`.

### Pass / fail rules (all mechanically checkable against gold)

- **Component identification** — right count and types. Fail on miscount, wrong type, or a hallucinated component.
- **Grounding** — cited regions overlap the true component locations. Fail on regions that don't correspond to a real component (a hallucinated coordinate).
- **Topology** — the stated series/parallel structure matches the gold netlist (graph-isomorphic).
- **Every intermediate value** — node voltages and branch currents match the solver within tolerance. A correct final answer reached through a wrong intermediate step is a **fail** (no lucky guesses).
- **Final answer** — matches SPICE within tolerance.
- **Concept scoping** — declared concepts are a subset of those the problem needs. Inventing an unneeded concept is a **concept-hallucination** fail.
- **Honesty** — for an illegible value, output `null`. Inventing a number for an unreadable value (**fabrication**) is the cardinal failure, scored separately.

### What "good" is (the metrics, base vs. tuned)

Primary: component accuracy, grounding accuracy, topology accuracy, **step-verified solve accuracy** (final answer *and* all intermediates correct), and **fabrication rate** vs. **honest-abstention rate**. Secondary: concept-hallucination rate; a judged clarity score. The headline result is base-vs-tuned on a **hand-labeled real-world** set, not synthetic.

---

## Part 2 — BrainLift (research grounding)

### Purpose

Prove that a small (~3B) VLM can be fine-tuned to solve textbook circuits *reliably* — where general models fail on even simple ones — by making the training data do two things the literature says actually move the needle: **ground reasoning in the image** and **verify reasoning step-by-step, not just the final answer**.

**In scope:** DC / AP-level circuits; image → grounded, verified worked solution; honest `null` on illegible values; synthetic data with exact labels; a real-world transfer eval.
**Out of scope:** transistor-level analog IC design; solving in the model without verification; baking in changing facts.

### Spiky points of view

1. **The failure isn't arithmetic, it's reading the wrong circuit — so ground the reasoning in the image.** General VLMs miss simple circuits because they set up equations for a circuit they misread or half-invented, not because Ohm's law is hard. The fix isn't more reasoning text; it's forcing every perception step to point at the image region it came from, which is documented to cut hallucination and improve generalization.

2. **More chain-of-thought can make a VLM worse; grounding and verification are what help.** Naive CoT fine-tuning increases reliance on textual priors and causes premature answer commitment, sometimes degrading visual reasoning. So the value is not in longer solutions but in solutions that stay tied to the image and end by checking themselves.

3. **A right answer from wrong reasoning is a failure, not a success.** Keeping solutions on final-answer-correctness alone teaches "lucky guesses." Because we have a circuit solver, we can verify every intermediate value and keep only genuinely-sound solutions — a cleaner signal that raises the achievable ceiling.

### Key insights (with evidence)

- **Grounding reduces hallucination.** VLMs "hallucinate less and reason more accurately when first prompted to generate grounded visual descriptions"; grounding in visual regions "curbs hallucination and promotes generalization." Circuit VLMs specifically show "spatial blindness," struggling to count parallel components or identify junctions. → *Add bounding-box grounding per perception step.*
- **CoT can backfire for VLMs.** Standard CoT-SFT shows "premature answer commitment and limited direct visual-token access," "increasing reliance on textual priors." → *Perception (grounded) must come before any number; keep solutions tight, not verbose.*
- **Verify reasoning, not just answers.** Step-level verification with a computer-algebra checker "rejects ~34% of correct-answer solutions that outcome verification would keep," and this cleaner signal lifted accuracy 80.5%→91.0% where answer-only verification plateaued. → *Use SPICE to verify every intermediate value as a data filter, and include a verification step in the target.*
- **Preference pairs sharpen sound-vs-flawed reasoning.** Turning step-checks into DPO pairs "further teaches the model to tell the two apart." → *Validates our v2 mistake pairs (correct = sound reasoning, wrong = flawed).*
- **Self-verification fixes consistency errors.** A "non-trivial portion of errors arise not from lack of knowledge but from failures in global consistency," which a self-check mitigates. → *End each solution with a sanity check.*

### What the dataset already has vs. what these findings add

Already present: layered labels (components / topology / values), concept declaration, skill tags, honest-abstention cases, deliberate-mistake preference pairs, SPICE-verified final answers.
Added from this research: (1) **visual grounding** — bounding boxes per component/perception step; (2) **step-level verification as a data filter** — keep a solution only if every intermediate matches SPICE; (3) an explicit **verification step** in the solution text.

### Sources

Apple ML — Improving VLM CoT (CoT + outcome rewards / DPO). https://machinelearning.apple.com/research/chain-of-thought · Attention-guided CoT fine-tuning (premature commitment; textual-prior reliance). https://arxiv.org/pdf/2606.01558 · Do VLMs See or Guess? (textual-prior reliance, grounding recovers accuracy). https://arxiv.org/pdf/2606.10400 · Grounded RL for Visual Reasoning (grounding reduces hallucination). https://arxiv.org/pdf/2505.23678 · Ground-R1 (grounded visual reasoning). https://arxiv.org/html/2505.20272v2 · Reliable Self-Improvement by Verifying Reasoning (step-checking, +10.5 pts, DPO pairs). https://arxiv.org/pdf/2603.21558 · Self-Verification (global-consistency errors). https://arxiv.org/pdf/2601.03144 · VLM-CAD (VLM spatial blindness in circuits; deterministic grounding). https://arxiv.org/html/2601.07315v4 · SINA (schematic→netlist; component boxes + connectivity). https://arxiv.org/pdf/2601.22114 · MAPS (netlist + SPICE for physics circuit reasoning). https://arxiv.org/pdf/2501.10768 · Qwen2.5-VL + Unsloth QLoRA (feasibility). https://unsloth.ai/docs/basics/vision-fine-tuning · QLoRA. https://arxiv.org/abs/2305.14314
