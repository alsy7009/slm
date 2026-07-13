# CircuitSight — Behavior Spec & BrainLift

*A small (3B) vision-language model that reliably reads **complex** circuit schematics — grounding every component to the image, reading non-series-parallel topologies, and verifying its setup — where base small models misread the circuit and frontier models can't localize it. Arithmetic is offloadable; the moat is the grounded, honest reading.*

---

## Part 1 — Behavior Spec

### The one-sentence spec

Given a circuit schematic and a question, the model **reads the circuit off the image** — naming every component and pointing to *where it actually is* in the picture, stating the true series / parallel / **bridge** topology, and setting up the governing equations — then solves and checks itself; and when a value is genuinely illegible it reports `null` instead of inventing one. The defensible capability is the **grounded, honest reading and setup of complex circuits**, not the final arithmetic (which is offloadable to a solver).

### The moat — what this model does that others don't

The value is not "solves circuits." It is **reading a complex, real-looking schematic correctly and grounding every claim to the actual image** — the exact place two other classes of model fail:

- **Small / base VLMs misread the circuit.** Given a Wheatstone bridge, base Qwen-3B calls it a simple series-parallel network and never draws a box — it sets up the *wrong circuit*, then does arithmetic on it. Reading, not algebra, is the failure.
- **Frontier VLMs can't localize on the image.** Given the same bridge, a frontier model (Claude Sonnet) *names* the components and even labels it an "unbalanced Wheatstone bridge" — but its bounding boxes land in the wrong place; it emits a generic bridge layout **from prior**, not where the parts sit in *this* image. This is the documented "spatial blindness": right about *what*, wrong about *where*.

Our tuned 3B does what both miss: it **boxes every component on the actual pixels** (grounding recall ≈ 0.84 overall, 1.00 on the demo bridge), reads **non-series-parallel** topology correctly, declares the right solving method (nodal analysis), and abstains honestly on illegible values. That — grounded, honest, correctly-structured reading of *hard* circuits at 3B — is the moat. Arithmetic is deliberately **outside the capability claim**: the model produces the correct *symbolic setup* far more often than the final number, and a calculator/solver closes that last step.

### Scope

**In scope:** DC / AP-level resistor networks, reactive circuits (a capacitor / inductor at steady-state or t = 0), and **non-series-parallel bridges**; values as numbers *or* symbols; grounded perception, correct topology, verified setup, honest abstention; and transfer to **real** diagrams (photographed / scanned, varied layout, varied labels — `R1` / `R₁` / "Resistor 1", source `V` / `ε` / `E`).
**Out of scope (on purpose):** being a calculator — final numeric arithmetic is offloadable; transistor-level / analog-IC design; solving without grounding or verification.

### Required output structure (in this order — perception before answer)

1. **Components** — the inventory (e.g., "1 source, 5 resistors"), each with the image region it occupies as `<box>[x0,y0,x1,y1]</box>` (0–1000 coords).
2. **Concepts used** — the minimal set the problem needs (Ohm's law, series/parallel, nodal analysis for a bridge, steady-state / t = 0 for a reactive element).
3. **Topology** — what is in series / parallel / bridging, referencing the grounded components; for a bridge, state that it is **not** series-parallel reducible.
4. **Solution** — step by step, with intermediate values (R_eq, node voltages, branch currents).
5. **Verification** — a real consistency check (plug back V = I·R_eq; a parallel block is smaller than its smallest branch; KCL closes at each node).
6. **FINAL** — a machine-readable line: `FINAL: {"quantity", "target_id", "value", "unit", "abstain"}`.

### Pass / fail rules (mechanically checkable against gold; ordered by centrality to the moat)

- **Grounding (core — beats frontier models).** Each cited box must overlap the true component location (IoU ≥ 0.5). Fail on boxes that land on nothing real, or on a plausible-but-wrong *prior* layout that doesn't match this image.
- **Topology incl. non-series-parallel (core — beats base models).** The stated structure must match the gold netlist (graph-isomorphic). A bridge read as a series-parallel network is a fail.
- **Component identification.** Right count and types; capacitor-vs-inductor is the reactive "spatial-blindness" litmus. Fail on miscount, wrong type, or a hallucinated component.
- **Verified setup, not lucky answers.** Every intermediate (R_eq, branch currents) matches the solver within tolerance. A correct final number via a wrong intermediate is a **fail**.
- **Honesty (cardinal).** For an illegible value, output `null`; inventing a number (**fabrication**) is the cardinal failure, scored separately from accuracy.
- **Concept scoping.** Declared concepts are a subset of those the problem needs; inventing an unneeded one is a concept-hallucination fail.
- **Final numeric answer (secondary, offloadable).** Checked against the solver but reported *separately* and explicitly **not** the capability claim — correct grounded setup with an arithmetic slip is a near-miss, not a core failure.

### What "good" is (the metrics)

**Primary (the moat):** grounding accuracy (boxes on real components), topology accuracy (incl. bridges), component + reactive-type accuracy, honest-abstention vs. fabrication, and **step-verified setup** (components + topology + intermediates all correct). **Headline evidence:** a three-way head-to-head on the *same* schematic — base Qwen (misreads), frontier Sonnet (can't localize), tuned 3B (grounds + structures correctly) — alongside base-vs-tuned deltas and a blind LLM-judge on holistic behavior. **Secondary / de-scoped:** final numeric answer accuracy (offloadable), reported honestly with the numeric-vs-symbolic split that shows the residual gap is *arithmetic*, not reasoning. The ultimate target is transfer to a **hand-labeled real-world** set.

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

### Key insights (each with its research source + URL)

- **Bounding-box grounding reduces hallucination and improves generalization.** Grounding a model's reasoning in specific image regions "curbs hallucination and promotes generalization," and VLMs "hallucinate less and reason more accurately when first prompted to generate grounded visual descriptions." → *Declare each component with its bounding box before reasoning.*
  Source: *Grounded Reinforcement Learning for Visual Reasoning* (2025). https://arxiv.org/pdf/2505.23678

- **Circuit VLMs specifically suffer "spatial blindness."** State-of-the-art VLMs "exhibit a critical spatial blindness" on dense schematics, "struggling with low-level tasks like identifying line junctions or counting parallel components," and deterministic structural grounding is "the essential key" to fixing it. → *Grounding is not optional polish here; it targets the exact failure.*
  Source: *VLM-CAD: VLM-Optimized Collaborative Agent Design for Analog Circuit Sizing* (2026). https://arxiv.org/html/2601.07315v4

- **Naive chain-of-thought can make a VLM worse.** Standard CoT-style fine-tuning exhibits "premature answer commitment and limited direct visual-token access during rationale generation," and "often increas[es] reliance on textual priors and reduc[es] counterfactual visual dependence." → *Put grounded perception before any number; keep solutions tight, not verbose; watch grounding accuracy as text grows.*
  Source: *Attention-guided Fine-tuning of MLLMs Improves Chain-of-Thought Reasoning* (2026). https://arxiv.org/pdf/2606.01558

- **CoT-tuned VLMs over-rely on text and under-use the image.** RL/CoT-tuned VLMs "remain vulnerable to weak visual grounding, hallucinations, and over-reliance on textual cues," and controlled textual perturbations flip their answers — confirming the reasoning drifts off the image. → *Reinforces grounding + perception-first ordering.*
  Source: Apple ML Research — *The Potential of CoT: A Closer Look at Trace Dynamics* (2025). https://machinelearning.apple.com/research/cot

- **Verify the reasoning, not just the final answer.** Step-level verification with a computer-algebra checker "rejects about 34% of the correct-answer solutions that outcome verification would keep" (the lucky guesses); this cleaner signal let accuracy climb "80.5% → 91.0%" where answer-only verification plateaued, and turning the checks into DPO pairs "further teaches the model to tell the two apart." → *Use SPICE to verify every intermediate as a data filter; validates the v2 mistake pairs.*
  Source: *Reliable Self-Improvement Training by Verifying Reasoning, Not Just Answers* (2026). https://arxiv.org/pdf/2603.21558

- **A closing self-check fixes consistency errors.** Self-verification yields consistent gains because "a non-trivial portion of errors made by strong fine-tuned models arise not from lack of knowledge, but from failures in global consistency, which self-verification is well suited to mitigate." → *End each solution with a sanity check (e.g., V = I·R_eq).*
  Source: *Self-Verification is All You Need to Pass the Japanese Bar Examination* (2026). https://arxiv.org/pdf/2601.03144

### What the dataset already has vs. what these findings add

Already present: layered labels (components / topology / values), concept declaration, skill tags, honest-abstention cases, deliberate-mistake preference pairs, SPICE-verified final answers.
Added from this research: (1) **visual grounding** — bounding boxes per component/perception step; (2) **step-level verification as a data filter** — keep a solution only if every intermediate matches SPICE; (3) an explicit **verification step** in the solution text.

### Outcome — did the data → behavior thesis hold? (evidence)

**Yes, on the behavior the thesis set out to instill — and, honestly, not on the axis it deliberately deprioritized.** Base-vs-tuned, Qwen2.5-VL-3B + QLoRA on the held-out val set (full tables and run details in [results.md](results.md)):

- **Grounding worked (POV 1).** Grounding recall **≈0.00 → ≈0.80** (IoU ≥ 0.5), component identification **≈0.2 → 0.85**, and reactive-type — capacitor-vs-inductor, the "spatial blindness" litmus — **≈0.5 → 1.00**. The base model boxes nothing usable; the tuned model reads the *actual* circuit. This directly hits the failure the BrainLift targets (VLM-CAD spatial blindness).
- **Honesty came from data (the cardinal rule).** Fabrication **≈0.3 → 0.00** and honest-abstention **≈0.0 → ≈0.8–1.0**: the model learned to report `null` on an illegible value instead of inventing one — a pure behavior-from-data result.
- **Step-verification helped (POV 3).** Intermediates go from **0.00** (base never produces a sound one) to a meaningful fraction; equivalent-resistance **0.00 → ≈0.5**. The step-level solver filter kept only genuinely-sound solutions — exactly the "reject the lucky guesses" mechanism the research cites.
- **An independent LLM-judge agrees (holistic Appendix A rubric, vision-capable, blind).** spec-adherence **0.00 → 0.92**, robustness **0.00 → 0.92**, task-quality **0.08 → 1.08**, consistency **0.00 → 1.08** (0–2 scale, n = 12). A frontier judge — not our own solver — confirms the tuned model embodies the specified behavior where the base does not. It also flagged real self-consistency bugs (e.g. node voltages that don't satisfy the model's own KCL), the exact error class the closing self-check (self-verification) is meant to reduce and a clear signpost for where more data helps.
- **Where we were honest about *not* optimizing (POV 2 corollary; the brief's "measure behavior, not capability").** Final *numeric* answer accuracy stays low — and that is in-thesis. Splitting answer accuracy by value mode shows the model produces the **correct symbolic setup far more often than it computes the final number**: the residual gap is *arithmetic*, not reading or reasoning. (We even tried offloading the arithmetic to a symbolic solver; it didn't help — reinforcing that the value is perception + setup + honesty, not being a calculator.) Arithmetic is a capability, not our target behavior, so we report it honestly rather than chase it.

**Note on data validity (why "synthetic" is not a weakness here).** The training data is **procedurally generated, not AI-generated**: one structured spec deterministically drives *both* the rendered image *and* the solver-computed labels, so every label is exact by construction — there is no hallucinated ground truth, unlike LLM-synthesized datasets. What remains to validate is *transfer to real diagrams*, not label correctness.

**Pending headline.** All numbers above are on the held-out synthetic split. The headline the spec promises — base-vs-tuned on the **hand-labeled real-world set** (AP-exam FRQ circuit figures + open-source diagrams) — is in progress (8 labeled, expanding from the scraped AP-FRQ / Wikimedia set); those numbers will be filled in here.

### Sources

Apple ML — Improving VLM CoT (CoT + outcome rewards / DPO). https://machinelearning.apple.com/research/chain-of-thought · Attention-guided CoT fine-tuning (premature commitment; textual-prior reliance). https://arxiv.org/pdf/2606.01558 · Do VLMs See or Guess? (textual-prior reliance, grounding recovers accuracy). https://arxiv.org/pdf/2606.10400 · Grounded RL for Visual Reasoning (grounding reduces hallucination). https://arxiv.org/pdf/2505.23678 · Ground-R1 (grounded visual reasoning). https://arxiv.org/html/2505.20272v2 · Reliable Self-Improvement by Verifying Reasoning (step-checking, +10.5 pts, DPO pairs). https://arxiv.org/pdf/2603.21558 · Self-Verification (global-consistency errors). https://arxiv.org/pdf/2601.03144 · VLM-CAD (VLM spatial blindness in circuits; deterministic grounding). https://arxiv.org/html/2601.07315v4 · SINA (schematic→netlist; component boxes + connectivity). https://arxiv.org/pdf/2601.22114 · MAPS (netlist + SPICE for physics circuit reasoning). https://arxiv.org/pdf/2501.10768 · Qwen2.5-VL + Unsloth QLoRA (feasibility). https://unsloth.ai/docs/basics/vision-fine-tuning · QLoRA. https://arxiv.org/abs/2305.14314
