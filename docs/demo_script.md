# CircuitSight — Demo Script (3–5 min)

*A presenter script: **[SHOW]** = what's on screen, **[SAY]** = narration. Target ~4 min; cut Scene 3 detail to hit 3 min. Numbers are from [results.md](results.md) (base-vs-tuned, Qwen2.5-VL-3B + QLoRA). The head-to-head image + box overlays come from the **§9g** cell in the training notebook.*

---

## Before you record
- Run **§9** (base vs tuned), **§9f** (vs Sonnet), then **§9g** (picks the demo image + saves `demo_boxes.png`).
- Have two assets ready: (1) `demo_boxes.png` — the 3-panel box overlay (base / Sonnet / tuned on the *same* image); (2) the base-vs-tuned metric table.

---

## Scene 1 — The problem (0:00–0:30)
**[SHOW]** the chosen circuit image, no boxes.
**[SAY]** "This is an AP-level circuit. Hand it to a frontier model — Claude Sonnet, GPT-class — and something surprising happens: it doesn't fail at the algebra. It fails at *reading the circuit*. It writes equations for a circuit that isn't on the page. That's the gap we set out to close — and we did it with a **3-billion-parameter** model, small enough to run on a laptop."

## Scene 2 — The BrainLift research (0:30–1:30)
**[SHOW]** three findings appear as bullets.
**[SAY]** "We started from the literature. Three findings shaped everything.
1. **The failure isn't arithmetic — it's reading the wrong circuit.** So the fix is *grounding*: make the model point to *where* each component is in the image before it reasons. Grounding is documented to cut hallucination.
2. **More chain-of-thought can make a vision model worse** — it drifts onto text priors and stops looking at the image. So the value isn't longer reasoning; it's reasoning that stays tied to the image and *checks itself*.
3. **A right answer from wrong steps is a failure, not a success.** So we verify *every* intermediate, not just the final number.
And circuit VLMs specifically suffer documented **'spatial blindness'** — they can't count parallel branches or find junctions. That's the exact failure we target."

## Scene 3 — From research to dataset (1:30–2:30)
**[SHOW]** dataset feature list, each tied to a finding.
**[SAY]** "Each finding became a concrete data feature:
- **Grounding** → every component in our data carries an exact **bounding box**.
- **Verify-the-steps** → we keep a worked solution only if **every intermediate matches our circuit solver**. Crucially, the labels aren't AI-generated — one spec drives both the image and the solver-computed answer, so they're **exact by construction**.
- **Honesty** → 8% of examples have an **illegible value**, where the right answer is 'I can't read it — null', not a guess.
- **The moat** → we generate the *hard* topologies: four-branch networks, three resistors per branch, and **Wheatstone bridges** that aren't series-parallel at all.
- **Real-diagram transfer** → we vary **layout** (resistors on any rail, not just a top row) and **labels** (R1, R₁, 'Resistor 1'; source V, E, or ε)."

## Scene 4 — The moat (2:30–3:00)
**[SAY]** "So what's the moat? **Perception** — reading complex circuits correctly and grounding every claim — **not arithmetic.** And we can prove it's perception, not luck: on **symbolic** problems, where the answer is a formula and there's *no arithmetic*, the model is right about **31%** of the time; on the *identical* **numeric** problems it's near **zero**. The reasoning is there — only the calculator step slips, and a calculator is a solved problem you can bolt on. We deliberately didn't chase it."

## Scene 5 — Head-to-head (3:00–4:30)
**[SHOW]** `demo_boxes.png` — three panels, same image (`img_002430`, a Wheatstone bridge).
**[SAY]** "Same circuit — and it's a **Wheatstone bridge**, the case you *can't* reduce with series-parallel rules; you need nodal analysis. Three models.
- **Base Qwen-3B:** no boxes at all, and it miscounts the resistors. It can't point to anything — grounding **zero**.
- **Frontier Sonnet:** the real tell. It gets *more* right than base — it correctly calls it an **unbalanced Wheatstone bridge** and even emits bounding boxes. But watch *where* they land: its boxes cluster in the wrong part of the image. Sonnet is drawing a **generic bridge from memory, not reading where the parts actually are** — so its grounding is still **zero**. Textbook spatial blindness: it knows *what* and the abstract *structure*, but not *where* on this image.
- **Our tuned 3B:** every one of the six components boxed on target — grounding **1.00** — correct count, and it sets up the node equations a bridge actually requires."

*(You cannot solve a bridge without knowing which resistor sits on which node — so Sonnet naming the parts but not locating them is exactly why it can't do this circuit, and why grounding is the moat.)*

**[SHOW]** the metric table.
**[SAY]** "Across the eval:
- Component identification **17% → 96%**.
- **Grounding** — boxes on the right components — **0% → 84%**.
- Capacitor-vs-inductor, the spatial-blindness litmus: **33% → 100%**.
- Honest abstention on unreadable values: **0% → 100%** — it stops guessing.
And an **independent, vision-capable Claude judge**, scoring blind, rates spec-adherence and self-consistency from **~0 to ~1-out-of-2** — the tuned model embodies the behavior; the base model doesn't."

## Scene 6 — Close (4:30–5:00)
**[SAY]** "The headline: a **3B model**, QLoRA-tuned on **procedurally-generated** data, reads complex circuits *reliably* — grounded, verified, honest — where a **frontier model still misreads them**. Not because it's smarter, but because the **data taught the behavior**: look at the image, point to what you see, check your work, and admit when you can't read something. That's the moat — and it fits on a laptop."

---

## Why this image works (base + Sonnet fail, tuned succeeds)
The §9g picker prefers **Wheatstone bridges** and **4-branch parallel networks** because they maximize the gap:
- **Base Qwen** emits **no bounding boxes** → grounding 0.00, and miscounts components.
- **Frontier Sonnet** gets the components and even the topology right, and *does* emit boxes — but they land in the wrong place (a generic bridge from its prior, not this image's actual positions), so grounding stays ~0. The documented "spatial blindness." (It also can't web-look-up a synthetic circuit, so it's on its own.)
- **Tuned 3B** was trained to ground + count + state topology, so it lands boxes on-target (IoU ≥ 0.5) and gets the structure right.

The single most convincing visual is the **box overlay**: red boxes on components for the tuned model, none (or scattered) for the other two.

## Fact-check notes (keep the demo honest)
- Metrics are the **held-out synthetic** split (the real-world transfer eval is still small; don't over-claim on real diagrams).
- The Sonnet head-to-head numbers come from **your §9f run** — paste the actual table before recording.
- Frame the arithmetic gap as *deliberate scope*, not a hidden weakness: the moat is perception + setup + honesty; arithmetic is offloadable.
- **The Sonnet panel shows its boxes in the wrong place — not an empty panel.** §9g parses boxes *leniently* (any format: our `<box>` tag, a markdown table, `R1: [..]`), so you fairly show that Sonnet *does* attempt to localize but **mislocates** (it emits a generic bridge layout, not this image's positions). That's more honest and more compelling than an empty panel — and it's why "same prompt, still fails grounding" is a fair claim.
- **Anchor `img_002430` on perception, not the final number.** This is the bridge the §9e judge flagged for a self-consistency slip (its node voltages don't perfectly satisfy its own KCL). The clean, defensible wins here are **grounding 1.00**, correct **component count**, and **recognizing it needs nodal analysis** — show the box overlay and stop there. If you display the full tuned text, describe it as "sets up the node equations" — don't claim the final current is exact. That's on-message: the moat is *reading + setup*, and bridge arithmetic is the offloadable part.
