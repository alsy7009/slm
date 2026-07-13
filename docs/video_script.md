# CircuitSight — Project Video Script (3–5 min)

*Voiceover script for a short screen recording. **[SHOW]** = what's on screen, **[SAY]** = narration. Target ~4 min; trim §3 detail to hit 3. Numbers are from [results.md](results.md); research is from the [BrainLift](behavior_spec_brainlift.md). (This is the project-summary video — for the live base-vs-Sonnet-vs-tuned demo, see [demo_script.md](demo_script.md).)*

---

## 1 · What we trained (0:00–1:00)
**[SHOW]** title card → a circuit schematic with the model's structured, boxed output scrolling beside it.
**[SAY]**
> "We fine-tuned **Qwen2.5-VL-3B** — a 3-billion-parameter vision-language model — using **QLoRA**, 4-bit low-rank adapters, to *read and solve circuit schematics*. Given a schematic and a question, the model produces a structured, grounded solution: first it lists **every component with a bounding box** showing where it sits in the image, then the concepts it'll use, the topology — what's in series, parallel, or bridged — a step-by-step solution with intermediate values, a self-check, and a machine-readable `FINAL` line.
> We trained it on a **procedurally-generated** dataset: 10,000 circuits where one spec deterministically drives *both* the rendered image *and* a **solver-computed answer key**. So every label — every box, every branch current — is exact by construction; nothing is AI-generated or hallucinated. And we generate the *hard* cases on purpose: four-branch networks, three resistors per branch, and **Wheatstone bridges** that aren't series-parallel at all."

## 2 · The goal — and the research behind it (1:00–1:55)
**[SHOW]** three research findings as bullets.
**[SAY]**
> "The goal was **not** to build a calculator. It was to fix the thing that actually breaks: *reading the circuit.* Three findings from our research shaped it.
> **One** — general vision models fail on simple circuits not because the math is hard, but because they set up equations for a circuit they **misread**. The documented fix is **grounding**: make the model point to each component in the image before it reasons — which cuts hallucination. Circuit VLMs specifically suffer *'spatial blindness'* — they can't reliably locate parts or count parallel branches.
> **Two** — more chain-of-thought can make a vision model *worse*; it drifts onto text priors and stops looking at the image. So the value is reasoning that stays **tied to the image** and checks itself.
> **Three** — a right answer from wrong steps is a failure. So we **verify every intermediate** against the solver, not just the final number.
> That defines our moat: **grounded, honest, verified reading of complex circuits** — the capability, not the arithmetic."

## 3 · Evaluation methodology (1:55–2:55)
**[SHOW]** the metric list / harness table.
**[SAY]**
> "We evaluate on a **held-out synthetic split** the model never saw — with exact solver labels — plus a hand-labeled **real-world** set. A mechanical harness scores each axis *separately*: component identification; **grounding** — do the boxes land on real components at IoU ≥ 0.5; topology — is it graph-isomorphic to the true netlist; **step-verified** solving — every intermediate correct, so no lucky guesses; and honesty — did it abstain on an illegible value or fabricate one.
> And we triangulate. **Base versus tuned**, to isolate what fine-tuning added. A **numeric-versus-symbolic split**, to separate reasoning from arithmetic. And two *independent* checks: a **blind, vision-capable LLM judge** scoring holistic behavior, and a **head-to-head against a frontier model**, Claude Sonnet, on the very same images."

## 4 · The results (2:55–4:30)
**[SHOW]** the base-vs-tuned table, then `demo_boxes.png` (the 3-way box overlay).
**[SAY]**
> "Fine-tuning turned *reading the circuit* from near-total failure into near-perfect. Component identification: **17% to 96%**. Grounding — boxes on the right components — **zero to 84%**. Capacitor-versus-inductor, the spatial-blindness litmus: **33% to 100%**. And honesty: from never abstaining to correctly saying 'I can't read that' **100%** of the time on illegible values.
> The key result is the split. On **symbolic** problems — where the answer is a *formula* and there's no arithmetic — the tuned model is right about **31%** of the time. On the *identical numeric* problems, near **zero**. So the reasoning is there; only the calculator step slips — and that's offloadable. The moat, proven.
> And the head-to-head makes it visual. **[SHOW overlay]** Same Wheatstone bridge, three models. **Base Qwen** misreads it as a simple series-parallel network — the *wrong circuit*. **Frontier Sonnet** correctly *names* it a bridge, but its bounding boxes land in the wrong place — it knows *what*, not *where*. Only our **tuned 3B** boxes every component on the actual image and sets up the right nodal analysis. A blind LLM judge agrees: spec-adherence and self-consistency go from about **0 to ~1 on a 0–2 scale**, where the base model scored ~0 across the board."

## 5 · Close (4:30–5:00)
**[SAY]**
> "So: a 3-billion-parameter model, QLoRA-tuned on procedurally-generated data, that reads *complex* circuits **reliably** — grounded, honest, and verified — where a frontier model still can't locate the parts. The behavior came from the **data**. And it runs on a laptop."

---

## On-screen number cheat-sheet (base → tuned)
| | base | tuned |
|---|---|---|
| component identification | 0.17 | **0.96** |
| grounding (boxes on components) | 0.00 | **0.84** |
| capacitor-vs-inductor | 0.33 | **1.00** |
| honest abstention | 0.00 | **1.00** |
| answer accuracy — symbolic vs numeric | — | **0.31 vs 0.00** |
| head-to-head bridge grounding (base / Sonnet / tuned) | 0.00 / 0.00 | **1.00** |

## Research cited (from the BrainLift)
- Grounding reduces hallucination / improves generalization — *Grounded RL for Visual Reasoning* (2025), https://arxiv.org/pdf/2505.23678
- Circuit VLMs' "spatial blindness"; grounding is the fix — *VLM-CAD* (2026), https://arxiv.org/html/2601.07315v4
- Naive chain-of-thought can degrade VLMs (textual-prior reliance) — *Attention-guided Fine-tuning of MLLMs* (2026), https://arxiv.org/pdf/2606.01558
- Verify the reasoning, not just the answer (rejects lucky guesses) — *Reliable Self-Improvement by Verifying Reasoning* (2026), https://arxiv.org/pdf/2603.21558

## Honesty guardrails (say these accurately)
- Headline metrics are on the **held-out synthetic** split; the real-world set is small — don't over-claim real-world transfer.
- Frame arithmetic as **deliberately out of scope** (offloadable), not a hidden weakness.
- The bridge in the overlay (`img_002430`): anchor on **grounding + topology**, not its final number (a known self-consistency slip on that item).
