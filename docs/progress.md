# CircuitSight — Progress Log

A running changelog of what we update each iteration. Newest first. Keep entries short but specific: **what changed · why · how it was validated**. One entry per meaningful update (usually one commit).

> Template:
> ```
> ## <commit> — <short title> (<date>)
> - **What:** <the change, concretely>
> - **Why:** <the reason / problem it solves>
> - **Validated:** <how we know it works>
> ```

---

## wip — Real-world eval set: scraped + labeled (2026-07-08)
- **What:** Built the hand-labeled real transfer set in `data/real_eval/` (git-ignored). Scraped AP FRQ + scoring-guideline PDFs from College Board (Physics C E&M 2023–26; Physics 1 2023–26; Physics 2 2023–26) and auto-extracted circuit figures with PyMuPDF (stem-anchored page detection → crop embedded raster or vector-path bbox). Added 3 Wikimedia CC-BY-SA schematics (Wheatstone bridge, series/parallel with meters, resistors-in-parallel) via `Special:FilePath` + rasterize. **Auto-labeled 8 images** into `labeled.jsonl` (generalized `make_real_eval_record` schema): question, gold_components, gold_topology, gold_answer, and `gold_boxes` (0–1000). Boxes estimated from fractional coords and **verified against `overlays/`**.
- **Why:** The headline metric is base-vs-tuned on *real* images. These are different renderers/styling the model never saw — incl. the moat cases (R∥R perception in AP P2/C, a real Wheatstone bridge). 2 images have numeric gold (AP26 RL → 1.2 A; Wikimedia parallel → 18 mA); the other 6 are symbolic → score component/grounding/topology (the perception moat) only.
- **Findings:** AP **Physics 1** 2023–26 has **no** circuit FRQ (pivoted to **Physics 2**, which has clean DC networks). AP C E&M FRQs are calculus-heavy — labeled only the in-scope (steady-state / t=0 / R_eq / count / topology) parts. Dropped the AP25 single-loop figure (ambiguous supply symbol).
- **Validated:** Every gold box viewed on its overlay and corrected until it lands on-component. `.gitignore` extended (`data/real_eval/`, `data/scraped/`, `*.pdf`/`*.jpg`); `git check-ignore` confirms the whole set (PDFs, figures, overlays, `labeled.jsonl`) is untracked. Only `.gitignore` + this changelog are stageable.
- **Next:** run base-vs-tuned on `labeled.jsonl` with the identical prompt; fill numeric `gold_answer` for more images from the `*-sg-*` scoring guidelines if we want a bigger numeric slice.

## wip — Full intermediate verification: enriched targets + eval + filter (2026-07-08)
- **What:** (1) **Targets** now emit a concise `Branch currents: I(R1)=… A, …` line for every solved DC and bridge record (the current through *each* resistor, incl. how it splits in parallel blocks / flows in the bridge). (2) **Eval** parses those and verifies every substantive (≥1 mA) branch current against `gold_values`; new `intermediates_accuracy` axis, and `step_verified_accuracy` now requires component + R_eq + **all intermediates** + answer. (3) **Verify-as-filter** (`verify_record`) drops any record whose printed intermediates don't match the solver (guarantees dataset consistency; abstain/open-circuit records exempt).
- **Why:** This is the second half of the moat — "correct intermediate steps." Enriching the target *trains* the model to produce every branch current (where generic models fail: equal-splitting parallels, collapsing the bridge); the eval upgrade *measures* it and kills lucky guesses (right answer via wrong steps).
- **Validated:** scratch: perfect model 1.0 on `intermediates_accuracy`/`step_verified`; a corrupted-intermediates model craters to 0.0 intermediates / 0.14 step-verified while answer stays 1.0 (proves the axis bites). Notebook: both compile; ported real eval cell scores perfect model 1.0 on all axes; 300-record generation run drops 0 to the filter, keeps abstain, and every DC/bridge solved target carries branch currents.

## wip — Domain randomization for synthetic→real transfer (2026-07-08)
- **What:** New `domain_randomize()` applied to **every** image (chain, bridge, abstain) after render: small ±4° rotation (boxes carried through the rotation via a corner transform), faint graph-paper grid, off-white paper tint, brightness/contrast/sharpness jitter, Gaussian blur, sensor noise, and JPEG artifacts. Replaced the abstain-only `light_blur` (blur was leaking as an abstain *tell*). Prototyped in `scratch_pipeline.py`, ported verbatim to notebook cell 18 + wired into all three `make_record` branches.
- **Why:** All training images were clean matplotlib in one style; the headline metric is real-image transfer. Uniform augmentation prevents renderer-memorization and removes the blur→abstain leak.
- **Validated:** 387 scratch records + 150 notebook-`make_record` records → 0 invalid/out-of-range boxes, box-count matches gold, abstain still reports `null`. Rotation box-transform verified visually (overlays at exaggerated +12°: every box lands on its component, chain + bridge). Both notebooks compile; ported eval still scores a gold-fed perfect model 1.0 on all axes.

## wip — Real PRD with user story/flow (2026-07-08)
- **What:** Rewrote `docs/prd.md` (was a byte-for-byte copy of the Behavior Spec) into an actual PRD: problem, moat, goals/non-goals, primary/secondary users, a user story + 8-step user flow, functional requirements, success metrics, milestones, risks.
- **Why:** The PRD duplicated the Behavior Spec, giving reviewers no product-level "why/who".

## 7bdb916 — Cap training to `MAX_STEPS`, smaller images (2026-07-08)
- **What:** Added `CFG["MAX_STEPS"]=150` (training cell uses `max_steps` when set, else epochs); `MAX_IMAGE_PX` 768→512; warmup 10→5, logging 25→10, `save_strategy="no"`. Prints images actually seen.
- **Why:** A "1 epoch" run was ~2,600 optimizer *steps* over ~10k images (~6.5 h on a T4). This narrow behavior converges in a few hundred steps → target ~15–25 min on a T4.
- **Validated:** All training cells compile. Validation stays on the held-out synthetic split (`val_synthetic.jsonl`).

## 5ef9d31 — Layout diversity: flips & mirroring (2026-07-08)
- **What:** Both renderers (chain + bridge) randomly apply horizontal (~40%) / vertical (~25%) flips on training images (source on the right, branches swapped, 180° rotation). Implemented via axis inversion so gold boxes follow the flip. Previews stay upright.
- **Why:** Prevent overfitting to "source-top-left horizontal chain"; improve transfer to real diagrams.
- **Validated:** Forced all four variants — every box stays on-component and in-range; full-notebook run 0 FINAL/gold mismatches; eval grounding still 1.0 on a gold-fed perfect model.

## 4feb496 — Wheatstone bridge family (Step 3) (2026-07-08)
- **What:** New non-series-parallel bridge family: `gen_bridge`/`solve_bridge` (5 resistors via MNA), vertical resistor symbols + bridge renderer with boxes, nodal-analysis worked solution (both KCL equations, midpoint voltages, `R_eq`, balanced/unbalanced check). Questions: R_eq, bridge current/voltage, total current. `make_record` mixes dc/reactive/bridge via `BRIDGE_FRAC=0.15`.
- **Why:** Sharpest litmus case — base VLMs collapse a bridge as `(R1+R3)||(R2+R4)` and are wrong when unbalanced.
- **Validated:** Bridge fuzz 0/5000 KCL failures; full-notebook run mixes all 3 families, 0 mismatches; eval scores bridges (incl. `R_eq` intermediate) 1.0 on a gold-fed perfect model.

## 25aa2e7 — Eval harness for generalized schema (2026-07-08)
- **What:** Rewrote base-vs-tuned harness for `FINAL: {quantity,target_id,value,unit,abstain}`. New axes: component types (R/C/L), `reactive_type_accuracy` (cap-vs-inductor), grounding IoU≥0.5 + hallucination rate, `step_verified_accuracy` (answer AND `R_eq`), per-qtype answer accuracy; kept fabrication/abstention + concept-hallucination. Updated training `INSTRUCTION` (boxes + new FINAL), the printout, and the real-eval template + `make_real_eval_record` helper.
- **Why:** Match the new data schema; measure the axes the BrainLift actually targets.
- **Validated:** Both notebooks compile; ported eval scores a gold-fed perfect model 1.0 on every axis and correctly fails corrupted outputs (wrong answer, type confusion, shifted boxes, fabrication).

## 806fd36 — Capacitor/inductor family + diversified questions (2026-07-08)
- **What:** Reactive family (one C or L per circuit) solved at DC steady-state (t→∞) or t=0 via open/short rules, reduced to a resistor network on the same MNA solver. New capacitor/inductor render symbols + boxes. Diversified question types (current, voltage, power, R_eq, total current, charge Q=CV, energy ½CV²/½LI²). Generalized `FINAL` schema + `gold_answer`, `family`/`regime`/`question_type` tags, new skill tags. `make_record` draws from both families; loop/dedup/abstention/v2-mistakes preserved.
- **Why:** Widen toward AP Physics C while staying in the perception moat (reading the circuit, not arithmetic).
- **Validated:** Physics sanity + 20k fuzz (0 failures); full-notebook run 0 errors, 0 FINAL/gold mismatches, all boxes valid; symbols/boxes eyeballed.

## 22898f3 — Scale to 11k, larger family, local export, bbox viz (2026-07-08)
- **What:** `N_TRAIN=11000`, `MAX_LINKS=5`, `MAX_PARALLEL=4`; export dataset zip to `/content/downloads` for manual download; cell overlays gold boxes on samples for visual verification. (Reflects Colab dependency-conflict fixes in the training notebook.)
- **Why:** Scale up the dataset and make it easy to pull off Colab and sanity-check boxes.

## f73d910 — Initial commit (2026-07-08)
- **What:** CircuitSight project: dataset-generation notebook (netlist → schematic → numpy MNA solve, grounded boxes, step-verification filter, abstention slice, flag-gated v2 mistake pairs) + training/eval notebook (Qwen2.5-VL-3B QLoRA via Unsloth, OOM smoke test, base-vs-tuned harness) + docs (Behavior Spec/BrainLift, PRD). Large dataset zips excluded via `.gitignore` (persisted to Google Drive).

---

### Open items / next
- **Regenerate the dataset** with the updated generation notebook (new schema + families) and point the training data-load cell at the fresh zip — the old `circuitsight_dataset_11k_4_3.zip` predates these changes and is incompatible with the new eval.
- First real **base-vs-tuned** run on the new data.
- Deferred (per plan): v2 mistake pairs → DPO; hand-labeled real-world transfer set.
