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

## wip — Add tools/circuit_labeler.html (browser real-eval labeler) (2026-07-12)
- **What:** Self-contained HTML tool (no install/server): load a circuit image, **drag bounding boxes** (auto-assigned V1/R1/C1/L1/SW ids, auto-incrementing), fill all record fields (family, question_type, question, topology, perception-only vs numeric answer, optional gold_values, source/license/notes), and it exports a **`labeled.jsonl` line** in the exact real_eval schema. Boxes are emitted as **0–1000 normalized** coords; component inventory auto-derived from box ids.
- **Why:** the real-eval gold must stay **authoritative hand-labels** (human draws the boxes). This makes that fast, and matches `make_real_eval_record` so records drop straight into `data/real_eval/labeled.jsonl`.
- **Validated:** `node --check` on the embedded JS passes; brackets balanced. Schema fields verified against the 8 existing labeled records.


## wip — BrainLift: add Outcome section (did data->behavior hold?) (2026-07-12)
- **What:** Added an **Outcome** section to Part 2 of `behavior_spec_brainlift.md` — the brief's required "whether data->behavior held, with evidence." Maps each result to a Spiky POV: grounding (recall 0->~0.80, reactive-type ~0.5->1.0), honesty (fabrication ~0.3->0, honest-abstention ~0->~0.8-1.0), step-verification (Req 0->~0.5), the independent LLM-judge (spec 0->0.92, robustness 0->0.92, task 0.08->1.08, consistency 0->1.08), and an honest note that numeric arithmetic stays low (in-thesis: capability, not target behavior; tool-offload tried and didn't help). Plus a **data-validity note** (procedurally generated, not AI-generated -> exact labels by construction) and a **pending-headline** note (real-world eval expanding from 8 labeled).
- **Why:** required BrainLift deliverable; also captures the mentor's point that our data is deterministic ground-truth, not hallucination-prone synthetic.
- **Validated:** numbers reference results.md; deltas rounded/qualified where they vary across runs; real-world numbers explicitly flagged as pending.


## wip — Record LLM-as-judge results + error analysis in results.md (2026-07-12)
- **What:** Added the §9e LLM-as-judge results (vision Claude via TrueFoundry gateway, n=12) to `docs/results.md`: base->tuned means spec 0.00->0.92, robustness 0.00->0.92, task_quality 0.08->1.08, consistency 0.00->1.08, plus a required **error-analysis paragraph** synthesized from the judge's per-item notes (superficial verification; bridge/nodal self-inconsistency; OOD perception slips e.g. Zener-as-source; over-symbolic answers + the trailing PLAN line as a format break).
- **Why:** LLM-as-judge is a required graded deliverable; the holistic scores + error analysis complement the mechanical eval and tell the honest story (behavior/format instilled ~1.0; residual errors in verification, hard reasoning, and unfamiliar-component perception).
- **Validated:** numbers taken verbatim from the 12-item run output; means recomputed match (all 12 items returned scores).


## wip — §9e judge via TrueFoundry gateway (the chat credential) (2026-07-12)
- **What:** Reworked §9e to call the **TrueFoundry LLM gateway** (the credential that powers Claude Code) instead of a provider directly: Anthropic SDK with `base_url=TFY_BASE_URL` + **Bearer** `auth_token=TFY_TOKEN`. Set `JUDGE_MODEL` to a gateway-served model; preflight ping fails fast. Repo keeps a **placeholder** `TFY_BASE_URL` (internal gateway hostname not committed); token via Colab Secret, never printed.
- **Why:** the `AQ.`/`tfy` key is a TrueFoundry **gateway** token (Bearer), not a direct Anthropic/OpenAI/Gemini API key — which is why every direct-provider attempt 401'd or hit a 0-quota. The project is meant to use the gateway that powers the chat (`ANTHROPIC_BASE_URL` + `ANTHROPIC_AUTH_TOKEN`).
- **Validated:** both notebooks compile (21 code cells); anthropic image-block + JSON parse validated earlier. Auth confirmed live in Colab via the preflight.


## wip — §9e judge -> Google Gemini (free tier, vision) via OpenAI-compat (2026-07-11)
- **What:** Switched §9e to **Google Gemini** (`gemini-2.0-flash`, vision) using Gemini's **OpenAI-compatible endpoint** (base_url on the OpenAI SDK), so the judge code is unchanged bar the client init + model. Key = free `GEMINI_API_KEY` (aistudio.google.com), Colab Secrets/getpass, never printed; preflight ping fails fast. Both notebooks.
- **Why:** the user has neither a valid Anthropic API key (the "chat" credential is a subscription/OAuth, not an `sk-ant-api03` API key) nor a funded OpenAI key. Gemini's free tier unblocks the LLM-as-judge deliverable with no billing.
- **Validated:** both notebooks compile (21 code cells); OpenAI-compat content shape (base64 image_url data-URI) + JSON parse already validated. Live judging runs in Colab with a free key.


## wip — §9e judge back to Anthropic (Claude) + preflight key check (2026-07-11)
- **What:** Switched §9e back to the **Anthropic** SDK (`ANTHROPIC_API_KEY`, `claude-sonnet-5` vision) in both notebooks, and added a **preflight**: it prints the key format (length + sk-ant- prefix, never the key) and does a 5-token ping, raising a clear error if the key/model is bad — so a bad key fails instantly instead of after 12 silent 401s.
- **Why:** the user's OpenAI key didn't work; going back to Anthropic. Earlier the Anthropic run 401'd with no early warning — the preflight fixes that.
- **Validated:** both notebooks compile (21 code cells); offline checks pass (anthropic image block shape, JSON parse). Key handling unchanged (Secrets/getpass, never printed).


## wip — §9e judge switched to OpenAI (vision GPT-4o) + preflight (2026-07-11)
- **What:** Rewrote §9e to use the **OpenAI** SDK (`OPENAI_API_KEY`, `gpt-4o` vision via `chat.completions` with a base64 `image_url`) instead of the Anthropic SDK, in both notebooks. Added a **preflight ping** that fails fast on a bad key/model. Same rubric, same independent/blind scoring, same base-vs-tuned table + error analysis.
- **Why:** user has an OpenAI key, not an Anthropic one — and an OpenAI key can't authenticate against the Anthropic API regardless of the variable name (earlier run 401'd on every call).
- **Validated:** both notebooks compile; offline checks pass (JPEG data-URI shape, JSON parse). Key handling unchanged (Secrets/getpass, never printed).


## wip — Add §9e: LLM-as-judge (vision Claude, 4-dim rubric) (2026-07-11)
- **What:** New §9e cell in both notebooks. A **vision-capable Claude** scores base vs tuned outputs **0-2** on the brief's Appendix A dimensions — spec adherence, robustness, task quality, and **consistency (internal self-consistency)** — judging each output independently (blind) on the synthetic val set. Prints a mean-per-dimension base/tuned/delta table + an error-analysis list of tuned items scoring <2.
- **Why:** LLM-as-judge is a required graded deliverable; it measures the holistic axes the mechanical eval (§9) structurally cannot (clarity, spec-adherence in spirit, robustness, self-consistency).
- **Security:** `ANTHROPIC_API_KEY` is read from Colab Secrets or a hidden `getpass` prompt — never hardcoded or printed.
- **Validated:** both notebooks compile (21 code cells); offline tests pass — JSON extraction from a noisy judge reply, and image resize+JPEG-base64 encode (verifies JPEG magic). Live judging runs in Colab with a key.


## wip — _20260710 notebook: keep full pipeline, verify eval path (2026-07-11)
- **What:** Reverted `CircuitSight_training_20260710.ipynb` from the lean eval-only build back to the **full pipeline** (training + eval, 40 cells) — a clean, outputs-stripped copy of the canonical notebook — after asserting the eval path is correct (non-greedy `parse_final`, 896 cap everywhere / no `cap400`, Drive `DATASET_ZIP`/`MODEL_ZIP`, `EVAL_N=30`, `OFFLOAD_N=15`, load-model-from-Drive cell present).
- **Why:** user wants the whole pipeline retained but will only run the eval cells.
- **Validated:** all 20 code cells compile; eval-path assertions pass.


## wip — Clean eval-only notebook (CircuitSight_training_20260710.ipynb) (2026-07-11)
- **What:** Rebuilt the downloaded `src/CircuitSight_training_20260710.ipynb` as a lean **eval-only** notebook (16 cells): install -> imports -> Config -> load dataset from Drive -> load model from Drive -> §9 harness -> §9 (base vs tuned) -> §9d (tool-offload). Dropped all training/save/download/duplicate cells.
- **Why:** the downloaded copy mixed canonical + hand-pasted cells: a stale §9 harness with the GREEDY `parse_final` (bug) plus a separate patch cell, `cap400` in places, and two §9d cells — correctness depended on paste/run order.
- **Validated:** every reused cell copied verbatim from the canonical notebook after asserting it is the correct version (non-greedy `parse_final`, 896 cap, Drive `DATASET_ZIP`/`MODEL_ZIP`, `EVAL_N=30`, `OFFLOAD_N=15`); all 8 code cells compile.


## wip — Raise eval generation cap 400 -> 896 (PLAN was being truncated) (2026-07-11)
- **What:** `max_new_tokens` 400 -> 896 in all eval/inference generators (§8 solve_image, §9c, §9d, the Drive-load cell, grader cells).
- **Why:** §9d showed tool (0.05) < raw (0.15) with mostly `None` outputs. Root cause: PLAN-retrained targets run median ~329 / p95 ~573 / max ~751 tokens, and the `PLAN` line is LAST, so a 400-token cap truncated it (and `FINAL` on big circuits) -> `tool=None`. `max_new_tokens` is a ceiling (outputs stop at EOS), so short circuits are unaffected; only long ones get room to finish.
- **Validated:** measured target lengths on the notebook `make_record` (INCLUDE_PLAN) to size the cap; both notebooks compile. Re-run §9d/§9 to get true numbers.


## wip — Add "load model from Google Drive" cell at the top of §9 (2026-07-11)
- **What:** New cell right under the §9 header that mounts Drive, loads the fine-tuned adapter from `MODEL_ZIP` (defaults to `MyDrive/slm_project/models/…`), integrity-checks the zip, and (re)defines `load_image`/`solve_image` — so §9 / §9d run in a fresh runtime without re-running §4/§8. Skippable if the model is already in memory.
- **Why:** sessions keep ending; the existing zip loaders defaulted to `/content` paths and lived in the utilities area, so there was no obvious "load the model to evaluate from Drive" step at the start of the eval section.
- **Validated:** both notebooks compile (20 code cells). Loader mirrors the validated §9c/loadzip loader.


## wip — Fix greedy FINAL parser (PLAN line broke it) + results.md v2 model (2026-07-11)
- **What:** (1) `parse_final` was greedy + DOTALL (`FINAL:\s*(\{.*\})`), so on the PLAN-retrained model's output (`FINAL: {…}\nPLAN: {…}`) it swallowed the PLAN line, the JSON parse failed, and every dc-resistor answer scored 0. Fixed to non-greedy, no-DOTALL (`\{.*?\}`) in the §9 harness + `scratch_eval`. (2) Added the **Model v2** (`…20260710_1`, PLAN-retrained, n=60) results to `docs/results.md`: reliable base→tuned gains (component 0.23→0.85, reactive-type 0.55→1.00, grounding 0.00→0.80, **Req 0.00→0.54**, intermediates 0.00→0.25, concept-declared 0.02→0.92, honest-abstention 0.20→0.80) and a caveated v1-vs-v2 comparison.
- **Why:** the v2 run showed `answer_accuracy 0.087→0.018` and `symbolic 0.308→0.000`, which looked like a regression but was a **measurement artifact** of the new PLAN line breaking the FINAL parser — not a model change. v2 actually improved the key reasoning intermediate (Req 0.18→0.54).
- **Validated:** reproduced the bug and the fix on a `FINAL … PLAN …` string (old parser → JSON fail/None; new parser → correct value 0.0218, and still returns a symbolic expression when there's no PLAN). Ported-eval self-check still 1.0; both notebooks compile. True v2 answer numbers pending a re-run of §9/§9d with the fixed parser.


## wip — Tool-offload as a RETRAIN feature: bake PLAN into targets + train for it (2026-07-10)
- **What:** Made tool-offload a trained capability instead of a zero-shot gamble. **Data-gen:** new `CONFIG["INCLUDE_PLAN"]=True`; `plan_of()` derives the answer as a FORMULA in the component labels (via a fully-symbolic solve of the same ladder) + the numeric readings, and `make_record` appends a `PLAN: {expression, values, ...}` line to every legible dc-resistor target (numeric/symbolic/mixed; not reactive/bridge/abstain). **Training:** `INSTRUCTION` now asks for the `PLAN` line after `FINAL` for resistor networks. **§9d** rewritten to a single generation → `FINAL` (model's arithmetic) vs `PLAN`→sympy (tool), head-to-head on numeric resistor problems; warns if the model wasn't PLAN-trained.
- **Why:** the value-mode split proved the gap is arithmetic, not reasoning. Training the model to emit the setup as a formula (its strength) and letting sympy do the arithmetic (a solved problem) should lift numeric toward the symbolic level — the moat framing, made real. Retraining removes the format-compliance gamble of the earlier zero-shot §9d.
- **Note:** `INCLUDE_PLAN` and the `INSTRUCTION` PLAN sentence are coupled — keep both on (default) or both off, else train/prompt mismatch.
- **Validated:** oracle in `scratch_offload.py` — `plan_of`→sympy reproduces the numpy gold answer **400/400 for numeric, symbolic, AND mixed**; tool 6/6 compute + safe-abstain on injection/missing. End-to-end on the **notebook's** `make_record` (INCLUDE_PLAN=True): PLAN present in every dc target (symbolic 108/108, numeric 68/68, mixed 52/52), **0 mismatches vs gold**, none leaking into reactive/bridge. Both notebooks compile (19 code cells).


## wip — Add §9d: tool-offloaded solving (zero-shot, pre-retrain) (2026-07-10)
- **What:** New §9d cells (later reworked into the retrain feature above). The model reads components + values and emits the answer as a **formula** (`PLAN`); a deterministic **sympy** `tool_compute` substitutes the readings and evaluates the number.
- **Why:** the value-mode split proved the weakness is arithmetic, not reasoning — offloading the arithmetic to a calculator should lift numeric accuracy toward the symbolic level.
- **Validated:** tool logic in `scratch_offload.py` — compute cases correct; unsafe/missing cases (incl. an `__import__` injection attempt) correctly abstain.


## wip — Commit Colab run notebook (with outputs) as evidence (2026-07-10)
- **What:** Added `src/colab_CircuitSight_training.ipynb` — the Colab working copy *with execution outputs* from the 350-step run: training loss, the base-vs-tuned metric dicts + DELTA table, and the value-mode split. It is the source of every number in `docs/results.md`.
- **Why:** preserve the actual run outputs in-repo as reproducible evidence for the reported results. The clean canonical notebook stays `src/CircuitSight_training.ipynb`.
- **Validated:** scanned for secrets/tokens/emails before committing (none); 436 KB, text-only outputs (no embedded images).


## wip — Add docs/results.md (evaluation results write-up) (2026-07-10)
- **What:** New `docs/results.md` consolidating the measured results for the fine-tuned adapter `circuitsight_qlora_final_20260709_1.zip`. Run A (base→tuned, n=24): component 0.167→0.958, reactive-type 0.333→1.000, grounding 0.000→0.842, honest-abstention 0.000→1.000, concept-declared 0.000→0.833; solving Req 0.000→0.182, answer 0.000→0.087, step-verified 0.000→0.087. Run B (value-mode split, n=36): numeric 0.000/14, symbolic 0.308/13, mixed 0.333/6. Includes a plain-language metric glossary, full description of the held-out synthetic val set (500 records; families, value modes incl. training mix 5271/2750/1479, question types, abstention), per-run sample sizes/composition, decoding settings, training details (loss ≈0.16, A100), the key finding (arithmetic gap, not a reasoning gap), caveats, and how to reproduce.
- **Why:** the user needs a self-contained results doc a reader unfamiliar with the test set can understand, naming the exact model artifact.
- **Validated:** all numbers taken verbatim from the saved Colab run outputs (`src/colab_CircuitSight_training.ipynb`), including the full base/tuned metric dicts and the DELTA table — nothing inferred.


## wip — §9c: drop monkeypatch (re-run-safe) + zip integrity check (2026-07-10)
- **What:** §9c no longer monkeypatches `score_record`/`aggregate`. Re-running the cell wrapped a function around itself (`_orig = score_record` captured the previous wrapper) → `RecursionError`. It now just calls the §9 `evaluate` and reads the harness's built-in `answer_accuracy_by_value_mode`. Also added a zip **integrity check** (`PK` magic + `testzip()` + `adapter_config.json` present) that asserts with a clear "truncated upload / fix the path" message instead of a raw `BadZipFile`.
- **Why:** user hit `BadZipFile` (truncated Colab upload) then `RecursionError` (re-ran the patch cell). Both are now handled: the eval is idempotent and bad zips fail loudly and clearly.
- **Validated:** both notebooks compile (`scratch_check.py`, 18/18 code cells). Split path unchanged — `aggregate` already reports `answer_accuracy_by_value_mode` (added earlier), so no patching is needed inside the notebook.


## wip — Add §9c: load a saved model .zip → score val set by value mode (2026-07-10)
- **What:** New cell (after the load-a-zip utility) that reloads the fine-tuned adapter **from an uploaded zip** and runs the numeric/symbolic/mixed answer split on the held-out synthetic val set. Self-contained: unzips + loads the model, redefines `load_image`/`solve_image` (so it works after a disconnect without re-running §4/§8), monkeypatches `score_record`/`aggregate` for the split, and calls the §9 `evaluate`. `MODEL_ZIP` + `EVAL_N_SPLIT` knobs at top.
- **Why:** the user's Colab runtime disconnected after training; a fresh runtime loses the in-memory model, so we need a no-retrain path that reloads the downloaded model zip and re-runs the split. Needs a **GPU runtime** (4-bit load + `generate`).
- **Validated:** both notebooks compile (`scratch_check.py`, 18/18 code cells). Reuses the already-validated load-from-zip loader and the value-mode split (matches the §9 print).


## wip — Eval: answer-accuracy split by value mode (numeric/symbolic/mixed) (2026-07-10)
- **What:** `score_record` now records each item's `value_mode`; `aggregate()` reports **`answer_accuracy_by_value_mode`** (numeric / symbolic / mixed); the §9 base->tuned run cell prints the split. Diagnostic only — no schema, dataset, or training change.
- **Why:** base-vs-tuned showed perception axes jumped but "solving" (answer/Req/intermediates) stayed low. Symbolic answers (`R_eq = R1+R2`) require correct reading + reasoning but **no arithmetic**; numeric answers require both. Splitting answer accuracy by mode separates a *reasoning* gap from an *arithmetic* gap — if symbolic >> numeric, the model gets the physics right and only slips on in-head math (which is offloadable to a calculator), so the moat (perception + setup) is stronger than the aggregate number suggests.
- **Validated:** perfect-model self-check reports `answer_accuracy_by_value_mode={'numeric':1.0,...}` (symbolic/mixed None on the numeric-only self-test set); both notebooks compile; ported-eval cell validated. Re-runnable on the already-trained model (no retrain) via a monkeypatch cell.


## wip — Mid-run checkpoints (keep 2 most recent) for disconnect safety (2026-07-09)
- **What:** Full-training cell was `save_strategy="no"` (nothing on disk until §11). Now `save_strategy="steps", save_steps=100, save_total_limit=2` into `CKPT_DIR`: if `SAVE_TO_DRIVE` -> `DRIVE_DIR/checkpoints` (auto-persists to Drive during training); else `/content/circuitsight_qlora/checkpoint-<step>/`. Keeps the 2 most recent; resumable via `resume_from_checkpoint`.
- **Why:** the user wants to grab partial models mid-run in case the runtime disconnects; previously there was nothing to grab until training finished.
- **Validated:** both notebooks compile. (Checkpoint writes exercised in Colab during the run.)


## wip — Eval: add grounding P/R/F1 + abstention precision/recall (2026-07-09)
- **What:** `aggregate()` (base-vs-tuned §9) now also reports **grounding_recall / grounding_precision / grounding_f1** (recall = gold components localized at IoU>=0.5; precision = predicted boxes that hit) and **abstention_precision / abstention_recall** (positive class = "abstain", over all records). These auto-appear in the §9 base->tuned delta table. Existing metrics unchanged (component/reactive-type/answer/Req/intermediates/step_verified accuracy, fabrication vs honest-abstention, concept-hallucination, per-qtype).
- **Why:** the user wanted precision/recall-style stats for the comparison; grounding and abstention are the two axes where P/R is genuinely meaningful.
- **Validated:** scratch + notebook: perfect model -> grounding P/R/F1 = 1.0; a constructed abstention mix gives precision=recall=2/3 as expected; both notebooks compile; perfect-model self-check still 1.0 on all accuracy axes.


## wip — Effective batch 4 -> 8 (BATCH=2) + smoke test uses real batch (2026-07-09)
- **What:** Training BATCH 1->2 (GRAD_ACCUM=4 => effective batch 8; ~2,800 images seen at 350 steps). The section-4 smoke test now uses per_device_train_batch_size=CFG["BATCH"] so its peak-VRAM readout reflects the real run. OOM fallback in CFG: BATCH=1, GRAD_ACCUM=8 (same effective 8, no extra VRAM).
- **Why:** effective batch 4 was small; the smoke test showed ~5.7/15.6 GB headroom on a T4, so BATCH=2 uses parallelism for smoother gradients + more coverage. A100 can go BATCH 4-8.
- **Validated:** both notebooks compile; effective batch = 8. Real VRAM confirmed by running section-4 in Colab (the go/no-go gate).


## wip — Grader "load & run" cell (turn-in) (2026-07-09)
- **What:** Added a self-contained §12 cell to the training notebook: reloads the fine-tuned model (reuses the in-session model, else loads base 4-bit + the LoRA adapter from `ADAPTER_DIR`) and runs it on one circuit image (`IMAGE_PATH` + `QUESTION`). No HuggingFace account needed — a grader unzips the adapter snapshot, sets an image path, and runs.
- **Why:** Makes the assignment submission turnkey (idea borrowed from the example notebook's merge/inference path; skipped its bnb/quant/pipeline bits since Unsloth already handles those better). Hub-push deferred (no HF account).
- **Validated:** training notebook compiles (15 code cells); grader cell AST-parses; reuse-in-session vs fresh-load branch + graceful no-image message.

## wip — JPEG images + SAVE_TO_DRIVE flag (resilience + turn-in) (2026-07-09)
- **What:** (1) Dataset images now saved as **JPEG** (make_record/v2 use `.jpg`; `domain_randomize` saves quality 85). ~20 KB/image vs ~300 KB PNG -> the 10k zip is ~200 MB (was ~3 GB), so download/upload between Colabs is quick. (2) New `SAVE_TO_DRIVE` flag (default **False**) in both notebooks + `DRIVE_DIR`. When True: the generation notebook copies `circuitsight_dataset.zip` to Drive; the training notebook snapshots the trained adapter to `DRIVE_DIR/models/circuitsight_qlora_<timestamp>.zip` and **keeps only the 2 most recent** (auto-prune). Survives a Colab disconnect and gives a runnable turn-in artifact.
- **Why:** Smaller JPEGs make the zip handoff painless; Drive save protects against runtime loss and provides the deliverable. Note: recency != best — pick the turn-in model by the section-9 eval, not by which is newest or by training loss.
- **Validated:** both notebooks compile; JPEGs valid + legible (schematic/labels crisp at q85); a 40-image run averaged 20 KB; generate->zip->unzip round-trip loads with images resolving; flags default False; prune-to-2 unit-tested.

## wip — Optional connectivity feature (INCLUDE_CONNECTIVITY, off) (2026-07-09)
- **What:** New `CONFIG["INCLUDE_CONNECTIVITY"]` (default **False**). When True, each record gains `gold_connectivity` (node -> list of component ids meeting there; node 0 = − rail, derived from the netlist) and a concise `Connections (nodes): ...` line appended to `target_output` before FINAL. Also fixed symbolic records to carry node-labeled `gold_netlist` (`_symbolic_netlist`) so connectivity works on them too.
- **Why:** Per-junction adjacency ("which components meet here") is the junction-reading skill VLM-CAD flags as the core circuit-VLM failure — more granular than the series/parallel topology string. Off by default: it lengthens targets (dilution risk on a short run), so treat it as an A/B to run *after* v1, not a default. Grounded node-dot boxes would be a stronger follow-up.
- **Validated:** both notebooks compile; flag stays False; with flag True, 160-record run has 0 bad — every family (incl. symbolic) gets node-labeled netlist + matching gold_connectivity + a Connections line, box counts intact, and the eval still parses the answer after the inserted line.

## wip — Rebalance mix toward symbolic + MAX_STEPS 275->350 (2026-07-09)
- **What:** CONFIG rebalanced — BRIDGE 0.15, REACTIVE 0.35->0.25 (dc-resistor -> ~0.60), and within DC SYMBOLIC 0.35->0.45 / MIXED 0.20->0.25. Net ~37% of the set is now symbolic/mixed (was ~27%), numeric still the majority. Training MAX_STEPS 275->350 (~1,400 images seen, ~30-40 min on a T4).
- **Why:** The real transfer eval is mostly symbolic; the seen-image mix (not dataset size) is what a ~1,400-step run learns, so the mix is the lever. Note: symbolic applies to the DC-resistor family only — reactive/bridge stay numeric (reactive-symbolic is a possible follow-up).
- **Validated:** 500-record draw hits the target mix (dc 61 / reactive 25 / bridge 14; symbolic 28 / mixed 13 / numeric 59 of non-abstain); both notebooks compile.

## wip — Modernized v2 mistake/correction generator (all families + question types) (2026-07-09)
- **What:** Rebuilt the flag-gated v2 pair generator on the current pipeline (was legacy pure-resistor, numeric-DC, "total current" only). It now spans **all 3 families** (dc/reactive/bridge) and **all question types**, emitting `correct_solution` + `wrong_solution` in the same v1 worked-solution format, plus `error_type / first_error_step / error_explanation / correction / critique_target / correct_final / wrong_final`. Perception-relevant errors: value_misread, omit_branch, parallel_as_series, **regime_confusion** (cap/inductor open-short swap), **bridge_collapse** ((R1+R3)∥(R2+R4), ignore R5). Each mistake is verified to change the asked answer by > the eval's 3% tolerance. Supports DPO (chosen/rejected) or critique SFT.
- **Why:** So v2 (a post-v1 DPO/critique stage) is ready to flip on later without a rewrite. `INCLUDE_MISTAKES` stays **False** — does not run in tomorrow's v1 generation.
- **Validated:** scratch (250 pairs) + notebook's own cell (60 pairs, `INCLUDE_MISTAKES=True`): 0 schema errors, every chosen scores answer-correct and every rejected scores answer-wrong through the eval harness; coverage across all families/qtypes/error types. Both notebooks compile; MAX_STEPS bumped 150→275; full generation→JSONL→eval pipeline re-verified end-to-end (perfect model 1.0 on all axes). Symbolic mistakes deferred (numeric only for now).

## wip — Symbolic/variable circuits + unified schema + eval robustness (2026-07-08)
- **What:** (1) **Symbolic + mixed** value modes for the DC-resistor family: a component's value is a number, its own id-symbol (`R3`, source `V`), or a per-component mix. Solved analytically with **sympy** (exact algebraic `R_eq`/branch-currents/answers); `gold_answer.value` + `gold_values` become expression strings with a `symbolic:true` flag. CONFIG `SYMBOLIC_FRAC=0.35`, `MIXED_FRAC=0.20` (bridge/reactive stay numeric; set to 0 to fall back to numeric). Every symbolic record is validated against the trusted **numpy solver as an oracle** (random substitution) and dropped if it disagrees. (2) **Eval hardening**: symbolic-answer equivalence via `sym_equal` (sympify + random substitution — accepts any equivalent form the model emits); guards so perception-only rows (`gold_answer=null`, missing `gold_values`) no longer crash; `question_type` `topology`/`components`; parser ids extended (`Rx`,`Rn`,`SW`,`VM`); **id-agnostic IoU grounding** for real images (`ground_mode="iou"`). (3) **Unified schema**: real-eval `labeled.jsonl` + synthetic records share `gold_answer{…,symbolic}`.
- **Why:** The real transfer eval is 6/8 symbolic, but training was 100% numeric — a train/test mismatch on the headline metric. Symbolic tests the moat more purely (a symbolic `R_eq=2R` is right only if the model truly read a series pair). The eval also **crashed** on the real rows until hardened.
- **Validated:** scratch: perfect model 1.0 on symbolic/mixed (answer/Req/intermediates/step-verified via `sym_equal`); corrupted symbolic answer → 0.0; oracle drops 0 legit records; ~0.13 s/record. Notebook (its own `make_record`): 124 numeric / 36 symbolic / 15 mixed over 180, **0 bad**, string FINALs + `symbolic` flag, verify wrapper passes; both notebooks compile (gen 18 cells); ported real eval scores 0 crashes on `labeled.jsonl`, perfect synthetic model still 1.0. Ported via NotebookEdit (new symbolic cell + in-place render/make_record/CONFIG/pip edits); numeric path untouched.
- **Next:** first base-vs-tuned run on regenerated data (numeric + symbolic + mixed) for tomorrow's GPU window.

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
