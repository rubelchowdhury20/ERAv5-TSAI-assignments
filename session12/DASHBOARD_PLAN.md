# GPU Communication Dashboard — Master Plan

> **Purpose of this file:** ground truth for the build. If context is lost or the
> conversation grows too long, re-read this file first. It defines the objective,
> the agreed design decisions, the curriculum, the math, and the phase plan with
> acceptance criteria. Do not drift from it without updating it.

---

## 1. Objective

Build a **single self-contained interactive website** that teaches — in
**great depth** — **how GPU communication happens during distributed LLM
training**, covering Data Parallelism (DP), ZeRO stages 1/2/3, Tensor
Parallelism (TP), and Pipeline Parallelism (PP).

The site is a **teaching tool first**, a simulator second. A learner who follows
it front-to-back should understand:

- What a training step actually does and where memory goes (weights, gradients,
  optimizer state, activations).
- Why models do not fit on one GPU (the "memory wall").
- What each parallelism strategy is, **why it exists**, the math behind it, and
  the memory/communication tradeoffs.
- What each collective primitive (All-Reduce, Reduce-Scatter, All-Gather,
  Broadcast, Send/Recv) does and when it fires.
- How to choose a strategy in practice (including hybrid 3D parallelism).

**Audience:** intermediate. Assume basic ML (gradients, SGD/Adam) but teach all
distributed-training concepts from the ground up.

**Non-goals:** real model execution, real benchmarks, beginner hand-holding,
quizzes, a papers/references section (deprioritized — may add inline citations
only if cheap).

---

## 2. Agreed Decisions (locked)

| Decision | Choice |
|---|---|
| Tech | Single self-contained `index.html`, **no build step** |
| Libraries | Tailwind CDN + vanilla JS + Chart.js CDN + Lucide icons + Google Fonts (Inter, JetBrains Mono) |
| Modes | 6: DP, ZeRO-1, ZeRO-2, ZeRO-3, Tensor Parallel, Pipeline Parallel (separate modes) |
| Topology | Multi-node: 1–4 nodes × 8 GPUs each (max 32). NVLink intra-node, InfiniBand inter-node |
| Math | Quantitative (real byte math), not qualitative |
| Learning flow | **Guided, paged course**: one fine-grained screen at a time (~18 steps), Next/Back + progress + resume. Replaces the single stacked page |
| Exploration | **Guided ⇄ Explore toggle** in one file; Explore = the full dashboard lab |
| Options | Prefs popover (math, deep-dives, glossary, advanced config, autoplay, reduced motion, theme) + reset progress, persisted |
| Experiments | Per-step prompt with a goal; completion is detected and saved (checkmarks) |
| Teaching layers | Intuition/analogies, math derivations, advanced deep-dives, glossary tooltips |
| Aesthetic | Clean, modern AI-hardware; smooth UX |
| Themes | **Dark (default) + light toggle**, same design language, CSS-variable tokens |
| First impression | Must NOT overwhelm: an **Objectives / orientation** block opens the page; advanced reference (charts, table) collapsed by default |
| Animation vision | GPUs on a **flow canvas**; packets traverse links between GPU cards during each training phase (detailed in §7) |

**Note:** the network is required for CDNs. Acceptable per user.

---

## 3. Deliverables / File Layout

- `session12/index.html` — the whole site (single file).
- `session12/DASHBOARD_PLAN.md` — this file.

Inside `index.html`, organize with clearly labelled section comments so it stays
maintainable, e.g.:
`/* ===== 1. THEME TOKENS ===== */`, `2. MATH ENGINE`, `3. STATE`, `4. TOPOLOGY`,
`5. VISUALS`, `6. ANIMATION ENGINE`, `7. LESSONS`, `8. CHARTS`, `9. GLOSSARY`,
`10. WIRING`.

---

## 4. Design System

### 4.1 Palette (semantic — locked)

| Token | Meaning | Color |
|---|---|---|
| weights | model weights (bf16) | **cyan** |
| grads | gradients (bf16) | **amber** |
| optimizer | optimizer state (fp32 master + m + v) | **purple** |
| activations | activations | **green** |
| nvlink | intra-node link | cyan |
| ib | inter-node link | amber |
| oom | out of memory | red |

- Canvas: deep slate/black (approx `#0a0e14`), layered surfaces via subtle
  gradients, 1px `white/5` hairline borders, soft inner shadows. Glassy, not noisy.
- Legend taught once in L0, reused consistently everywhere.

### 4.2 Type & space

- UI font: **Inter**. Numbers/formulas/log: **JetBrains Mono** with
  `font-variant-numeric: tabular-nums` so live values do not jitter.
- 4/8px spacing scale, generous whitespace around hero visuals.
- Clear hierarchy: lesson title → concept → visual → detail.

### 4.3 Component vocabulary (reused everywhere)

`card`, `chip`/`badge`, `segmented control`, `stat pill`, `expander`
(disclosure), `slider` row, `tooltip`, `node box`, `gpu card`, `stacked memory
bar`, `primitive chip`, `narration log`, `transport bar`, `data table`.

### 4.4 Motion & UX

- Motion tokens (duration + easing) in one place; transitions 150–300ms,
  `cubic-bezier(.2,.8,.2,1)`.
- Animate transform/opacity only (GPU-composited); `will-change` where useful.
- Number tweening on slider change; bars morph vs snap; lesson change
  fade/slide with slight stagger.
- Packet animation: glowing dot + fading trail on SVG paths; pulsing node rings
  on send/receive; primitive-chip "ignition" flash.
- Micro-interactions: hover lift + glow, segmented-control sliding pill,
  tooltip fade with ~80ms delay, log auto-scroll that pauses when user scrolls up.
- Smoothness guardrails: respect `prefers-reduced-motion`; rAF loop; no layout
  thrash; debounced resize; Chart.js responsive.
- Navigation: keyboard shortcuts (Space=play/pause, ←/→=step, R=reset,
  1–6=lesson), visible focus rings, sticky stepper with progress fill,
  smooth-scroll, touch-friendly targets.

---

## 5. Curriculum (8 lessons)

Each lesson follows the same anatomy:
**Concept → Why it exists → Intuition/analogy → Math (expandable derivation) →
Live animation + narration → Tradeoffs/takeaways → Deep-dive (expandable).**

- **L0 — The training step & the memory wall.** What fills GPU memory; the
  16 B/weight breakdown; why models do not fit; why distribute at all.
- **L1 — Data Parallelism (baseline).** Replica model; gradient All-Reduce;
  ring reduce-scatter + all-gather; 2P.
- **L2 — ZeRO-1.** Shard optimizer states; why optimizer dominates; Reduce-Scatter
  grads + All-Gather params.
- **L3 — ZeRO-2.** Also shard gradients; grads stay sharded; 2P.
- **L4 — ZeRO-3.** Also shard weights; per-layer All-Gather on demand (fwd/bwd);
  3P; memory vs communication tradeoff.
- **L5 — Tensor Parallel.** Intra-layer slicing (column/row parallel); 2
  All-Reduces per layer; NVLink-bound; why intra-node only.
- **L6 — Pipeline Parallel.** Layer-range stages; microbatches; send/recv; the
  pipeline bubble and how to size it.
- **L7 — Choosing a strategy.** Tradeoff matrix; hybrid 3D parallelism; when
  memory vs comm vs complexity wins.

### 5.1 Analogies (draft)

- All-Reduce ≈ everyone ends up with the class average.
- Reduce-Scatter ≈ each person keeps only their assigned chapters.
- ZeRO-3 ≈ pass one chapter around the study group only when it's being read.

### 5.2 Deep-dives (draft)

- Ring all-reduce internals.
- Pipeline bubble fraction.
- TP row/column parallel + where comm sits.
- Activation memory & checkpointing.
- Comm/compute overlap.

---

## 6. Math Model (source of truth)

### 6.1 Per-weight state (replicated in DP)

```
weight bf16       2 B
gradient bf16     2 B
master copy fp32  4 B
Adam m fp32       4 B
Adam v fp32       4 B
----------------------
total            16 B / weight
```

`P` = params in bf16 = `2 * N` bytes (communication unit).
`N` = number of parameters.

### 6.2 Per-GPU state bytes (G = total GPUs)

```
DP      : 16 * N
ZeRO-1  : 4*N + 12*N/G        (2+2 full, 12/G sharded)
ZeRO-2  : 2*N + 14*N/G        (2 full, 14/G sharded)
ZeRO-3  : 16*N/G              (fully sharded)
```

Corresponding B/weight per GPU: `DP 16`, `Z1 4 + 12/G`, `Z2 2 + 14/G`,
`Z3 16/G`.

For TP (tp = tensor-parallel degree): per-GPU `16 / tp`.
For PP (pp = pipeline stages): per-GPU `16 / pp` (+ activations for its stage).

**Standalone degrees (implemented):** since modes are separate (no hybrid), in
Tensor mode `tp = min(G, gpusPerNode)` (intra-node); in Pipeline mode `pp = G`
(every GPU is a stage). All other modes use the full GPU count `G = nodes × 8`
for sharding.

### 6.3 Communication per step

```
DP      : 2P   (All-Reduce grads = reduce-scatter + all-gather)
ZeRO-1  : 2P   (Reduce-Scatter grads + All-Gather updated params)
ZeRO-2  : 2P   (Reduce-Scatter grads + All-Gather updated params)
ZeRO-3  : 3P   (All-Gather weights fwd + All-Gather weights bwd + Reduce-Scatter grads)
TP      : per layer, 2 All-Reduces of activation-sized tensors (intra-node)
PP      : (pp-1) * microbatches * activation-size, send/recv each direction
```

Rationale note: in ZeRO-1/2 weights stay replicated, so an All-Gather of the
updated parameters is required (hence 2P there too); ZeRO-3 adds a second
weight All-Gather for the backward pass (3P).

### 6.4 Activations (excluded from ZeRO state sharding)

Rough estimate, explicitly labelled: `batch × seq × hidden × A × layers_here × micro`
per GPU, where `A = 20` bytes per (token × hidden) per layer without
checkpointing and `A = 2` with checkpointing (only layer inputs stored), and
`micro = 8` microbatches in Pipeline mode. `hidden`/`layers` come from
interpolating a preset table (1B/7B/13B/30B/70B):
`1B→h2048,l22 · 7B→h4096,l32 · 13B→h5120,l40 · 30B→h7168,l60 · 70B→h8192,l80`.
Activations are **NOT** sharded by ZeRO state sharding and the attention `seq²`
term is omitted (stated in the UI).

### 6.4b Implemented formula details

- `shape(nB)`: piecewise-linear interpolation of the preset table (clamped at ends).
- `commBytes`: DP/Z1/Z2 = `2P`; Z3 = `3P`; TP = `layers × 4 × batch·seq·hidden·2`
  (two All-Reduces per layer per direction, rough); PP =
  `(pp−1) × 2 × microbatches × batch·seq·hidden·2`.
- `bubbleFraction` = `(pp−1)/(pp−1+micro)`, micro = 8.
- Config defaults: `strategy z2, nodes 1 (G=8), N 7B, batch 1, seq 4096,
  VRAM 80 GB, checkpointing off` — chosen so ZeRO-2 fits while DP OOMs at 7B/8 GPU.

### 6.5 OOM

`per-GPU bytes <= VRAM limit` → fits; otherwise red OOM badge. VRAM presets
40 / 80 / 141 GB.

### 6.6 Reference numbers (regression checks)

`N = 30B, G = 32`:

| Mode | Per-GPU state | GiB |
|---|---|---|
| DP | 480 GB | ~447.0 GiB |
| ZeRO-1 | 131.25 GB | ~122.2 GiB |
| ZeRO-2 | 73.125 GB | ~68.1 GiB |
| ZeRO-3 | 15 GB | ~14.0 GiB |

(GB here = 1e9; GiB = 1024^3. Verify against these in Phase 2.)

---

## 7. Visual Components

- **Topology view:** node boxes (NVLink domain, cyan-tinted) each holding 8 GPU
  cards; amber IB links drawn between node boxes.
- **GPU card:** stacked memory bar (weights cyan / grads amber / optimizer purple
  / activations green), `used / limit`, OOM badge, ownership chip (shard i/G · TP
  slice · PP stage).
- **Weight-distribution view:**
  - DP/ZeRO → parameter-space bar `0→N` split into G labeled shards (DP shows
    "replicated ×G").
  - TP → weight-matrix grid with column/row stripes across the TP group.
  - PP → layer-range band `layer 0→L` split across stages.
- **Animation engine:** rAF timeline of phase objects
  `{ label, primitive, band, explain, traffic[], memDelta, highlight[], duration }`.
  SVG bezier paths between card centers; glowing packets; intra-node cyan,
  inter-node amber.
- **Transport bar:** Play / Pause / Step Forward / Step Back / Reset + speed
  (0.5×/1×/2×) + phase progress.
- **Primitive chips:** All-Reduce, Reduce-Scatter, All-Gather, Broadcast,
  Send/Recv light up as triggered.
- **Narration log** (sticky): timestamped plain-English explanation, e.g.
  *"Reduce-Scatter gradients: each of 32 GPUs keeps 1/32 of its gradients
  (23 MiB) for the optimizer shard it owns."*
- **PP Gantt:** stage × time chart highlighting bubbles.

### 7.1 Per-mode timelines (draft)

- **DP:** broadcast weights (init) → forward → backward → **All-Reduce grads** →
  optimizer.
- **ZeRO-1:** forward → backward → **Reduce-Scatter grads** → optimizer on owned
  shard → **All-Gather updated params**.
- **ZeRO-2:** forward → backward → **Reduce-Scatter grads** (stay sharded) →
  optimizer → **All-Gather updated params**.
- **ZeRO-3:** per-layer **All-Gather weights** (fwd) → compute → per-layer
  **All-Gather weights** (bwd) → **Reduce-Scatter grads** → local optimizer.
- **TP:** per layer: forward All-Reduce → backward All-Reduce (intra-node pulse,
  two per layer per direction).
- **PP:** microbatch pipeline — Gantt chart with forward/backward sends and
  bubble regions highlighted.

---

## 8. Charts & Comparison Table (Chart.js)

- Stacked bar: per-GPU memory breakdown for the active mode + VRAM capacity line.
- Bar: comm GiB/step across all 6 modes.
- Table: mode · B/weight/GPU · GiB/GPU · fits? · comm/step ×P · topology
  (NVLink vs IB) · when to use.

---

## 9. Layout (top → bottom)

1. Sticky header: title + lesson stepper with progress fill.
2. Lesson panel: concept / why / math expander / deep-dive expander.
3. Config bar: nodes, GPUs/node (=8), model size, batch, seq, VRAM limit,
   checkpointing toggle (bound live in Phase 2+).
4. Main area: topology view + weight-distribution view alongside sticky narration
   log.
5. Transport bar.
6. Charts + comparison table.
7. Footer: legend + formula reference.

Responsive: fluid 1→4 node grids; panels stack < 1024px; charts resize; touch
friendly.

---

## 10. Phases & Acceptance Criteria

Each phase ends with an **openable, working `index.html`**. Review checkpoints
after every phase; do not proceed until the user is happy. Phases 1–2 are the
highest-leverage reviews.

### Phase 1 — Scaffold + design system
- Layout shell, Tailwind theme tokens, dark aesthetic, component vocabulary.
- Static mock of the topology with dummy GPUs and dummy numbers.
- **Accept:** page opens, looks clean/modern, all component styles visible,
  responsive at 1440px and ~800px. User approves the look.

### Phase 2 — Math + state engine
- 16 B/weight, per-GPU memory for all 6 modes, activation estimate, comm volume,
  OOM; live config sliders bound to numbers.
- **Accept:** numbers match §6.6 regression values; sliders update live and
  smoothly (tweened).

### Phase 3 — Static visualizations
- GPU cards with stacked memory bars, weight-distribution views for all modes,
  memory + comm charts, comparison table. No animation yet.
- **Accept:** every mode renders correct, readable visuals and table row.

### Phase 4 — Animation engine + lesson content wiring
- rAF timeline, SVG packets, primitive chips, narration log, transport controls
  (play/pause/step fwd/back/reset/speed).
- **Accept:** all 6 timelines animate correctly and narration matches each phase.

### Phase 5 — Guided app shell
- Two views (guide/explore) in one file, panel relocation, options + prefs,
  progress/resume, step + lesson dots, keyboard, experiment goals.
- **Accept:** the page lands on a single small step; Next/Back and dots navigate;
  Explore shows the full dashboard; prefs persist; goals tick; no console errors.

### Phase 6 — Guided content
- Full narrative, analogies, math derivations and deep-dives for all ~18 steps;
  refine experiment prompts/goals.
- **Accept:** a reader can go start→recap and understand the full picture; all
  expanders/tooltips/goals work and match the visuals.

### Phase 7 — UX / polish pass
- Motion tokens, transitions between steps, hover/focus states, reduced-motion,
  responsive/touch pass, recap + explore layout, performance check.
- **Accept:** interactions feel smooth; no jank; keyboard + reduced-motion work;
  layout solid at 1440px and 800px.

---

## 11. Verification

- Serve: `python3 -m http.server` in `session12/`, open `index.html`.
- Math: confirm §6.6 reference values in the on-page table.
- Walk all 8 lessons end-to-end; exercise play/pause/step/reset on every mode.
- Check layout at 1440px and 800px.
- Confirm glossary tooltips, math/deep-dive expanders, keyboard shortcuts, and
  reduced-motion behavior.

---

## 12. Session Grounding Notes

- The original `session12_zerosharding_sim.ipynb` is **not** the source of truth;
  the dashboard is built from scratch. The notebook's numbers informed §6 but the
  site must stand alone.
- Keep this file updated whenever a locked decision changes.

---

## 13. Build Progress

- [x] **Phase 1** — scaffold + design system + static mock. Revised after user
  feedback: added Objectives/orientation block, dark **and light** theme toggle,
  calmed landing density (default 1 node, collapsed Reference section).
- [x] **Phase 2** — math + state engine, live-bound to sliders. All 6 modes,
  activations, comm, OOM, per-mode distribution view, per-mode timeline log,
  real memory/comm charts and comparison table. Verified against §6.6 reference
  values (30B/G=32 → 447.0 / 122.2 / 68.1 / 14.0 GiB).
- [x] **Phase 3** — Chart.js integrated. Memory chart = stacked per-GPU breakdown
  across all 6 modes with a dashed VRAM-limit line and red OOM axis labels;
  comm chart = GiB/step per mode. Theme-aware (rebuilds on light/dark toggle),
  lazy-initialized when the Reference section is opened. Verified headless: no
  console errors, correct labels/data, canvas sized.
- [x] **Phase 4** — animation engine landed. SVG flow canvas overlays the GPU
  cards (`#flowSvg` inside `.topo-stage`); packets travel bezier arcs along
  ring/chain/star edges (intra-node cyan NVLink, inter-node amber IB). Each
  timeline phase carries animation metadata (`flow`/`prim`); compute phases
  stagger a GPU pulse, comm phases emit packets (ring 8×2, PP chain 7×3).
  Transport bar fully wired: Play/Pause, Step fwd/back, Reset, 0.5×/1×/2× speed,
  progress bar, phase dots and label; narration log auto-scrolls to the active
  phase. Rebuilds on config change, resize (debounced) and theme toggle. Verified
  headless: no console errors across all 6 strategies; All-Reduce/Reduce-Scatter/
  All-Gather/Send-Recv packets present and positioned, compute pulses fire.
- [x] **Phase 5** — **guided app shell** (see §14). One `index.html`, two views
  (`#/guide` default, `#/explore`) switched by a topbar toggle. Guide shows a
  single centered step at a time with Next/Back; explore is the original full
  dashboard. Panels are **singletons relocated** into the active step via
  captured home parents (`captureHomes`/`clearPanels`/`mountForStep`), so the
  math/animation/chart engine is reused untouched. Options popover with persisted
  prefs (math, deep-dives, glossary, advanced config, autoplay, reduced motion,
  theme) + reset progress; progress/resume in `localStorage`; step dots + lesson
  dots; keyboard (←/→, Space, Esc); goal-detected experiment checkmarks + toasts.
  Verified headless: 18 steps mount the declared panels, no console errors,
  explore restores all 7 panels, charts render inside a step, goals/options/
  persistence/theme/keyboard all work. **Cinema stage** (§14.2) keeps each
  animation's flow + narration + transport in one viewport with guided scroll.
- [ ] **Phase 6** — guided **content**: full narrative, analogies, Math
  derivations and deep-dives for every step (Phase 5 ships concise placeholders).
- [ ] **Phase 7** — UX polish: transitions, reduced-motion, responsive/touch,
  recap + explore layout, final QA.

**Notes:** Phase 4's transport/animation remain intact; the old hardcoded lesson
panel and objectives block were replaced by the guide steps. Number tweening is
still deferred.

---

## 14. Guided App Shell (Phase 5)

**Why:** the single stacked page overwhelmed a newcomer moving toward DP. The fix
is a paged, guided course that reveals one idea per screen, with an escape hatch
to free exploration.

**Views.** `body.view-guide` / `body.view-explore`. Guide is default; the topbar
**Explore / Guide** toggle switches. Guide chrome (progress, lesson dots) is
`.guide-only`, hidden in explore.

**Two views, one document.** All lab panels are singletons with stable ids in
`PANELS`. `captureHomes()` records each panel's original parent + next sibling at
init; `clearPanels()` restores them; `mountForStep(step)` relocates the panels a
step declares into `#stepVisual` as rows (`step.layout`, e.g.
`[['topology','log'],['transport']]`). Explore is simply "restore all homes".
Because ids never change, `renderAll` and every renderer work unmodified.

**Steps.** `STEPS[]` (~18 screens) drives everything: `kicker`, `title`, `lede`,
optional `analogy`, `math`, `deep`, `takeaway`, `panels`/`layout`, and optional
`experiment { prompt, cfg, goal }`. `renderStep()` applies `cfg` to `CONFIG`,
syncs controls, fills text + expanders, mounts panels, then `renderAll()`.

**Experiments.** `checkGoal()` runs at the end of every `renderAll()`; when
`step.experiment.goal(CONFIG, evaluate(CONFIG))` is true the step is marked in
`PROGRESS.completed`, the badge flips to **Done**, a toast fires and progress is
saved.

**Preferences.** `PREFS` (persisted `dt-prefs`): `showMath`, `showDeepDive`,
`glossary`, `advancedConfig`, `autoplay`, `reducedMotion`, `theme`. `applyPrefs()`
sets body classes (`pref-no-glossary`, `pref-advanced`, `pref-reduced`) and opens
`.math-box` / `.deep-box` details. Advanced config controls carry `.adv-only`.

**Persistence.** `dt-progress = {view, step, completed[]}`, `dt-prefs`, `dt-theme`
(unchanged). Reset progress restores defaults and returns to step 0.

**Keyboard.** `←/→` step, `Space` play/pause (animation steps), `Esc` closes the
options popover. Lesson dots jump to a lesson's first step.

### 14.1 Step manifest (implemented)

| # | id | lesson | kind | panels |
|---|---|---|---|---|
| 1 | welcome | – | intro | stats |
| 2 | l0-state | 0 | concept | stats |
| 3 | l0-wall | 0 | experiment | stats, distribution, config |
| 4 | l0-why | 0 | takeaway | stats |
| 5 | l1-concept | 1 | concept | stats, distribution |
| 6 | l1-exp | 1 | experiment | topology, log, transport |
| 7 | l1-take | 1 | takeaway | stats |
| 8 | l2-exp | 2 | experiment | stats, distribution, config |
| 9 | l2-anim | 2 | experiment | topology, log, transport |
| 10 | l3 | 3 | experiment | topology, distribution, log, transport |
| 11 | l4 | 4 | experiment | topology, distribution, log, transport |
| 12 | l5-concept | 5 | concept | stats, distribution |
| 13 | l5-anim | 5 | experiment | topology, log, transport |
| 14 | l6-concept | 6 | concept | stats, distribution |
| 15 | l6-anim | 6 | experiment | topology, log, transport |
| 16 | l7-trade | 7 | experiment | stats, config, transport, reference |
| 17 | l7-choose | 7 | concept | topology, distribution |
| 18 | recap | – | recap | stats |

**Phase 6 will deepen** each step's `lede`/`analogy`/`math`/`deep` and refine the
experiment prompts and goals. The shell is complete and content-agnostic.

### 14.2 Cinema stage (keep the flow, its narration and its controls together)

A step flagged `stage:true` (the six animation steps: `l1-exp`, `l2-anim`, `l3`,
`l4`, `l5-anim`, `l6-anim`) mounts its panels into a **two-column stage** instead
of stacked rows: `.stage-left` holds the flow (`#panelTopology`), any distribution
view and the transport bar; `.stage-right` holds the narration log. Because the
log is a live, phase-by-phase description of what the animation is doing, it sits
**beside** the flow so the two never drift apart.

Fit + focus rules:
- `.stage-right #panelLog` is capped to `calc(100vh - 104px)`, so the stage is
  inherently at most one viewport tall — the animation, narrative and controls
  land in **a single window** on laptop heights (verified 1024×768 → 1440×900).
- The transport bar is `position:sticky; bottom:10px`, so **Play/Pause/Step and
  the phase label stay reachable** even when the topology grows (e.g. 4×8).
- **Guided scroll** (`scrollToStage`, wired into `renderStep`, Play and Step)
  nudges the page only as much as needed: if the stage fits it leaves the step
  title in view; if the stage is taller than the screen it pins the stage top.
  Below 980px the stage collapses to one column and the log un-pins.
- Animation steps also get compact type/padding (`.step--anim`) so the head +
  experiment prompt + stage all fit.

---

## 15. Redesign — GPU ring, training loop, reordered course

User-directed rework of three things: the GPU visual, the animation semantics,
and the course sequence. Agreed decisions:

- **Canvas:** GPU view sits on a fixed **light "paper"** surface in *both* themes.
- **Layout:** each node is a **circle of GPU icons**; up to **2 node circles side
  by side**, bridged by a labeled InfiniBand link. More nodes → a **ghost ring**
  ("+N more nodes").
- **Capping/readability:** at most **8 GPU icons per node**; extras are blurred +
  a **"+N more" tail**. Icons scale with the node via container-query units so
  they never overlap (verified spacing ratio ≈ 2.2 across 1/2/4 nodes, perNode 16).
- **Animation:** a real **training loop** — forward → backward → **pause & memory
  reveal** (what is replicated on every GPU vs what is sharded) → communication →
  optimizer sharing — run for **exactly 2 iterations**, ending in a clear
  "complete" state (no anonymous dots).
- **Course order:** Terms → Assumptions → Concepts (each animates a **tuned
  per-strategy** preset) → Playground (strategy, nodes, GPUs/node, model, VRAM +
  **live feasibility** grid, then **Lock**) → Walkthrough (the **locked**
  strategy's loop).

### 15.1 Progress

- [x] **Phase A — white ring canvas (static).** Replaced the card grid with
  `renderTopology` → node circles of GPU-chip icons on the white paper stage.
  Capping (`MAX_RING_GPUS=8`, `MAX_RING_NODES=2`), the `+N` tail, ghost ring and
  caption note; icons scale via `cqw`; `layoutFlow` now anchors on each `.chip`.
  Verified headless: correct icon/node counts across all 18 steps and 6
  strategies, spacing ratio ≈ 2.2 (no overlap), canvas stays `#f7f8f5` in dark
  **and** light theme, no console errors. **Fixed a latent bug**: `renderStep`
  applied `step.cfg` but presets live at `step.experiment.cfg`, so no step preset
  was ever applied — now fixed.
- [x] **Phase A.1 — live commentary subtitle.** A `.stage-caption` bar pinned to
  the bottom of the ring canvas narrates the run as a subtitle: the phase chip
  (`Pass`/`All-Reduce`/…) plus a **3-beat** message per phase (`NARR`, keyed by
  prim; every TP/PP/z3 phase carries its own accurate `beats`). `drawFrame`
  advances the beat from `anim.t`, so commentary tracks the motion and updates
  during real playback. Verified: playback walks all phases/beats in order, the
  caption never overlaps the GPU ring (30–36px clearance at 820–1440px wide), no
  console errors.
- [x] **Phase A.2 — "GPU X-ray".** Inside the topology card, the ring is now the
  left column of a responsive **split view** (`#xrayHost`); the right column is a
  **single magnified GPU** that answers *"what is shared and what is replicated,
  and how do they update?"*. Six segments — `master` / `w` / `g` / `m` / `v`
  (the optimizer is split into **master + Adam m + Adam v**) / `acts` — each with a
  short badge (`Replica ×8` / `Shard 3/8` / `Local`) plus a one-line legend.
  Bars **morph live** with the timeline (`xrayBase` → `xraySeq` → `xrayAt(k,t)`
  lerp inside `drawFrame`). **Click any ring GPU to focus it** (`.sel` glow,
  `wireRingClick`); standalone (no-ring) steps default to **G0**. Replaces the old
  `renderDistribution`/`panelDistribution` entirely. Verified headless: badges &
  values correct for all 6 strategies, morph sequences correct (e.g. Z1 grads
  `13.0→13.0→1.6→0→0`; Z3 weights bump to `2.0` on the gather), mounts exactly
  once (hosted in the split only for animation steps, standalone otherwise), click
  moves `.sel` + header, no overlap (2-col ≥1024px, stacks <980px), no console
  errors.
- [x] **Phase B — training-loop animation.** The timeline is now an explicit
  **2-iteration training loop** instead of a single flat pass. Each iteration is
  the strategy's phases with a new **`Reveal — what is shared?`** pause inserted
  after the gradients are produced (per-strategy index in `REVEAL_AFTER`);
  `buildPhases` runs the iteration exactly `LOOP_ITERS=2` times, then appends a
  terminal **`Training step complete`** phase. The reveal dims the ring, glows the
  X-ray (all six segments highlighted) and explains replicated-vs-sharded for the
  active strategy; completion stamps the stage and caption. The X-ray snapshot
  stream (`xraySeq`) is generated from the same composition, so `XRAY.length ===
  PHASES.length` for every strategy and the bars morph continuously across both
  iterations. Log rows carry an `iter 1/2` tag; reveal/complete rows are tinted.
  Per-phase `dur` multiplier added (reveal 1.9×, complete 1.3×). Iteration 2 is an
  identical repeat per the agreed decision. Verified headless: P==X for all six
  strategies (dp 11, z1/z2/z3 13, tp/pp 11 phases), playback reaches the
  `loop-complete` state, reveal toggles on/off, full 18-step sweep clean, no
  console errors.
- [x] **Phase C — course restructure.** Full content rewrite into a
  **Terms → Assumptions → Concepts → Playground+Lock → Walkthrough** arc. 10
  lessons / **23 steps**: welcome; Terms (`t-memory`, `t-share`, `t-wall`,
  `t-why`); Assumptions (`a-rules`, `a-letters`); then **concept + lab** pairs for
  every strategy (`dp`, `z1`, `z2`, `z3`, `tp`, `pp`) where the concept step carries
  a tuned `preset` and plays the loop inline (`stage:true`), and the lab carries the
  interactive `experiment`/goal; Playground (`playground` — all knobs + feasibility
  grid + Lock, plus `decision`); Walkthrough (`walkthrough`, `stage:true`, applies
  the locked setup). New **Lock** system (`LOCKED`, persisted `dt-lock`,
  `lockSummary`/`lockCurrent`/`unlockCurrent`) feeds the Walkthrough, which shows a
  lock badge in the step head and overrides `CONFIG` with the locked setup. New
  **feasibility grid** (`renderFeas`) ranks all six strategies at the current
  model/topology/VRAM with fits/OOM and comm/step, clickable to switch strategy.
  Config panel split into **Nodes** + **GPUs/node** segments. Verified headless:
  23 steps mount correct panels, `PHASES.length === XRAY.length` for every staged
  step, reveal toggles at both iteration points and completion at the end, lock →
  walkthrough applies the setup + badge (and clears on unlock), feasibility rows and
  segment/theme clicks update state, layout has no new overflow (only the known
  pre-existing 6px), zero console errors.
- [ ] **Phase D — polish** (transitions, reduced-motion, responsive/touch) + docs.
