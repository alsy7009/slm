# CircuitSight — Product Requirements Document

*A small vision-language model (SLM/VLM) that reads an AP-Physics-C circuit schematic and solves it — grounding every claim in the image and verifying its own steps.*

> Companion docs: [behavior_spec_brainlift.md](behavior_spec_brainlift.md) (exact I/O contract + research grounding), [progress.md](progress.md) (changelog). This PRD is the "why / who / what / how we win" layer; the Behavior Spec is the precise "what the output must look like."

---

## 1. Problem

General-purpose LLMs and VLMs are unreliable on circuits — and they fail on *simple* ones. Benchmarking work and our own testing show the failure is not arithmetic (Ohm's law is easy) but **perception**: the model sets up equations for a circuit it misread. Documented failure modes:

- **Spatial blindness** — miscounting parallel branches, missing line junctions, confusing a capacitor for an inductor.
- **Text-prior over-reliance** — answering from what circuits "usually" look like instead of *this* image; controlled text perturbations flip answers.
- **Lucky guesses** — a right final number reached through wrong intermediate steps, which naive final-answer training rewards.

## 2. What we're building

A ~3B VLM (Qwen2.5-VL-3B) QLoRA-fine-tuned on synthetic circuits with **exact, solver-derived labels**, so that on an image + question it produces a *grounded, step-verified* worked solution: it names each component and points to where it is in the image, states the topology, solves with correct intermediate values, checks itself, and abstains (`null`) rather than inventing an illegible value.

### Our moat (the two things generic models can't do reliably)

1. **Grounded perception** — correctly identify component **types**, **counts**, and **locations** (bounding boxes), including nodes/junctions.
2. **Correct intermediate steps** — every node voltage / branch current / R_eq is right, not just the final answer.

Both are backed by research and both are **mechanically verifiable** against our solver — see the Behavior Spec's pass/fail rules and BrainLift sources.

## 3. Goals / Non-goals

**Goals**
- Beat the base (un-tuned) model on a **hand-labeled real-world** eval, on: component accuracy, grounding accuracy, topology accuracy, step-verified solve accuracy, and honest-abstention vs. fabrication.
- Cover **AP Physics C (E&M)** DC/steady-state content: resistor networks (series/parallel), Wheatstone bridge, RC/RL at t=0 and t→∞, and basic transient reasoning (time constants).
- Fit the compute budget: dataset generated on CPU; training ≈15–25 min on a single T4 (one short QLoRA run, few hundred steps).

**Non-goals**
- Transistor-level / analog IC design; AC phasor/frequency-domain analysis; full transient waveforms `v(t)` over time.
- Solving "in the model's head" without a visible verification step.
- Baking in changing facts; this is a reasoning-on-the-image task.

## 4. Users & user story

**Primary user:** an AP Physics C student (or a tutoring/grading tool acting on their behalf) who has a schematic — from a textbook, a worksheet, or a photo of the board — and a question about it.

**Secondary user:** the project team, who need every output to be auto-gradable against gold so we can chart base-vs-tuned.

### User story

> *As a student, I photograph a circuit from my homework and ask "what's the current through R2?" I want the model to first show me it read the circuit correctly — which components are where, what's in series/parallel — then walk the solution with real intermediate numbers and a sanity check, so I can trust the answer and learn the method. If a value in my photo is smudged, I want it to tell me, not make one up.*

### User flow

1. **Input** — the user supplies a schematic image + a natural-language question (e.g., "Find the voltage across the capacitor at steady state.").
2. **Perception (grounded)** — the model lists components with image regions: `R1 <box>[x0,y0,x1,y1]</box> (100Ω); C1 <box>…</box> (10µF); V1 <box>…</box> (12V) …`, plus nodes/junctions.
3. **Concepts** — the minimal concept set the problem needs (Ohm's law, parallel combination, "capacitor is open at steady state").
4. **Topology** — series/parallel structure referencing the grounded ids.
5. **Solve** — step by step with intermediate values (reductions, R_eq, node voltages, branch currents).
6. **Verify** — a consistency check (e.g., V = I·R_eq; a parallel combo is smaller than its smallest branch; a balanced bridge carries zero bridge current).
7. **FINAL** — one machine-readable line: `FINAL: {"quantity","target_id","value","unit","abstain"}`.
8. **Abstain path** — if a required value is illegible, the relevant step reports `?`/`null` and `FINAL` has `abstain:true, value:null` — never a fabricated number.

*(Steps 2–7 are the required output order — perception strictly before any number. See the Behavior Spec for the exact contract and tolerances.)*

## 5. Functional requirements

**Data generation** (`src/CircuitSight_dataset_generation.ipynb`)
- Synthesize circuits with a netlist → schematic render → numeric MNA solve, so every label is exact.
- Families: **DC resistor** (series/parallel chains), **reactive** (one C or L, solved at t=0 or t→∞), **Wheatstone bridge** (non-series-parallel, nodal). Mixed per configurable fractions.
- Emit per record: image, question, `gold_components` (type inventory), `gold_topology`, `gold_answer`, `gold_values` (R_eq, I_total, node voltages, branch currents), `gold_boxes` (0–1000 normalized), `skills`, `concepts`, and the `target_output` worked solution.
- **Grounding:** every component (and node/junction) has a bounding box in the target and in `gold_boxes`.
- **Honesty:** an abstention slice occludes one value; its target reports `null`.
- **Diversity:** render-time domain randomization (styles, line widths, flips, rotation, backgrounds, noise, label jitter) so the model transfers to real images rather than memorizing one renderer.
- **Preference data (deferred):** flag-gated v2 "deliberate mistake" pairs for later DPO.

**Training** (`src/CircuitSight_training.ipynb`)
- Qwen2.5-VL-3B, QLoRA via Unsloth; capped by `MAX_STEPS` for a ~15–25 min T4 run; validation on a held-out synthetic split.

**Eval** (same notebook)
- Base-vs-tuned harness scoring the moat axes (see §6), all mechanically checkable against gold; plus a hand-labeled real-world transfer set.

## 6. Success metrics

**Primary (headline = on the real-world set, base vs. tuned):**
- Component accuracy (count + type), reactive type accuracy (C-vs-L).
- Grounding accuracy (IoU ≥ 0.5 vs. gold) and grounding-hallucination rate.
- Topology accuracy (graph-isomorphic to gold).
- **Step-verified solve accuracy** — final answer *and every checked intermediate* correct.
- **Fabrication rate** vs. **honest-abstention rate**.

**Secondary:** concept-hallucination rate; per-question-type answer accuracy; a judged clarity score.

**Bar:** tuned model beats base on grounding, step-verified accuracy, and fabrication rate by a clear margin on the real set.

## 7. Milestones / status

- ✅ Generation pipeline (3 families), grounded boxes, generalized `FINAL` schema, abstention slice, eval harness (see progress.md).
- 🔄 **Now:** domain randomization for transfer; full intermediate verification (score + enrich every intermediate, not just R_eq).
- ⏭️ First real base-vs-tuned run on regenerated data (tomorrow's GPU window).
- ⏭️ Hand-labeled real-world transfer set; later: v2 mistake pairs → DPO.

## 8. Risks & mitigations

- **Overfitting to the matplotlib style → fails on real images** (kills the headline metric). *Mitigation:* aggressive render-time domain randomization; the real-world eval is the source of truth, not synthetic val.
- **Tiny training budget (few hundred steps)** underfits a broad task. *Mitigation:* keep the behavior narrow and consistent; balance the dataset by family/qtype/difficulty so every capability appears in the steps seen.
- **CoT verbosity hurting VLMs** (documented). *Mitigation:* perception-first ordering, tight solutions, watch grounding accuracy as text grows.
- **Reward-hacking via lucky guesses.** *Mitigation:* step-verified accuracy as the primary solve metric; intermediates checked against the solver.
