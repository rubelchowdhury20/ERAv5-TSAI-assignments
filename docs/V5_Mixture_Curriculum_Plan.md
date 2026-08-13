# V5 Mixture & Curriculum Plan

> 100B-parameter model · 2T tokens · seven capability lanes · protected floors · curriculum to anneal.

## Mission

V5 is a **Codex-style assistant**: excellent at coding and agentic work — taking a long task, planning it, calling tools across multiple steps, reading results, recovering from failures, and holding a growing history in context. Its **differentiator** is native Indic generation: output quality, fluency, and generative richness in Indic languages that is **comparable to the English capability of the same model** and beyond what coding/agentic-leaning models offer today.

## 1. Capabilities & target benchmarks

**Method — compose backward from the benchmarks.** We do not start with round numbers for the mixture. We start with the capabilities we intend to prove, measure each with a benchmark, and translate each benchmark into the exact training-data format it rewards. A capability that is never measured is a capability nobody can verify was built; a share of the budget that cannot point to a benchmark it buys is excess weight the reviewer is entitled to question.

| # | Capability | Benchmarks it must win | What the benchmark actually tests → required data format |
|---|------------|------------------------|-----------------------------------------------------------|
| C1 | Code generation | HumanEval, LiveCodeBench, APPS | Function synthesis from a textual spec → natural-language ↔ code pairs with unit-testable ground truth |
| C2 | Science & math | MATH, GSM8K, GPQA | Multi-step solving with checkable answers → token-level chain-of-thought traces |
| C3 | General knowledge | MMLU, ARC-c | Single-hop recall of world facts → QA pairs over broad world text |
| C4 | Reasoning-effort control | ARC chosen subsets at low/high effort, GPQA subsets | The *same* problem solved at clearly different trace depths → a **distribution of trace lengths**, short to long, within one lane |
| C5 | Long-context | LongBench, multi-hop needle | Read long documents and answer from anywhere in them → long-form text and long-answer data, introduced late in the curriculum |
| C6 | Agentic tool use | BFCL, Tau-Bench, TerminalBench | Observe, choose, call, read, continue → conversation-style trajectories with tool calls interleaved |
| C7 | Native Indic generation | Indic eval suite (translational + generative quality) | Naturally-formed Indic text, not translated-flavoured → verified native Indic, judged against the English lane's bar |

### Binding data rule (agentic lane)

In every agentic training example the loss is placed on (a) the model's own tool **selection and arguments** and (b) the model's reasoning text around the call. The **tool-result span carries no loss** — it is masked ground truth the model may read and reason over but is never trained to reproduce. Training on tool outputs teaches the model to invent tool results instead of calling the tool; the trajectories we want actually result from observing real outputs.

### Indic headline bar

Indic is a headline lane, not a courtesy allocation: generative quality comparable to this model's own English output (fluency, coverage, fertility). Existing agentic/coding-strong models are weak here, which is exactly why the lane exists.

## 2. Budget & capability lane shares

### 2a. The total budget — Chinchilla-bound

Target model: **100B parameters**. Compute-optimal scaling (Chinchilla, Hoffmann et al., 2022) fits the efficient curve at roughly **20 tokens per parameter** — the same ratio as the seminal 70B / 1.4T model.

- **D = 20 × N = 20 × 100B = 2 Trillion tokens.**
- The total budget is **fixed**. Every share given to a lane is taken from another; every preset renormalizes to 100%.
- The annealing phase is not extra data: it is a **held-back slice of the 2T** (Section 6) with its own preset, which sums to 100% of that slice.

### 2b. The main-pretraining preset (average across stages)

Each lane exists because of Section 1: the **buys** column names the benchmarks; the **defense** is the one-line argument a reviewer can push on. These are average shares over the main run — the mixture is *temporal*, and per-stage presets are specified in Section 7.

| Lane | Share | Tokens (of 2T) | Buys (Sec. 1) | Defense in one line |
|---|---|---|---|---|
| General web | 40% | 800B | C3: MMLU, ARC-c | Knowledge breadth; the abundant, cheap foundation — V5 is a general assistant, not a math-only demon |
| Code | 22% | 440B | C1: HumanEval, LiveCodeBench | Core identity (Codex-style); large push over V4 (≈13% → 35% trajectory) |
| Science & math | 14% | 280B | C2: MATH, GSM8K, GPQA | Checkable rewards; the earliest chain-of-thought foundation |
| Reasoning (CoT) | 8% | 160B | C4: GPQA/ARC effort dial | Reserves a *distribution* of trace lengths; spent late (Sessions 17–18 raw material) |
| Long-context | 7% | 140B | C5: LongBench, NIAH | Introduced after reading is learned; real supply exists early |
| Agentic | 4% | 80B | C6: BFCL, Tau-Bench, TerminalBench | Sized at the edge of honest supply (see 2d) — the largest share real data supports |
| Indic | 4% | 80B | C7: native-gen Indic eval | The differentiator is *quality*, not bulk; verified-native supply is tiny (Section 4); concentrated again in the anneal (Section 6) |
| **Sum** | **100%** | **2T** | | |

### 2c. Temporal semantics (why 40%/22%/… are never a per-step constant)

The mixture moves at three levels:

1. **Stages (curriculum, Section 7):** the run proceeds seed → general → reasoning → long-context → anneal; each stage has its own rebalanced preset. V4's actual trajectory: general web **70% → 18%**, code 13% → 35%, science+math 7% → 39%. The table above is the *average* over the main run; Section 7 writes the stage schedule.
2. **Batches (selector, Section 5):** the loss-based selector scores candidate batches per iteration and retains the most useful fraction; the accepted distribution fluctuates per step. The **protected always-on floor** (Section 5) is the per-batch minimum that scarce lanes may never sit below.
3. **Transitions:** mixtures are never changed in one hard step — V4's experience is that abrupt distribution shifts spike the gradient norm and can diverge the run; every stage boundary is therefore **blended across warm-up bands of several billion tokens**.

### 2d. Agentic share sized against real supply

The open inventory of agentic/trajectory data (surveyed 2026):

| Corpus | Trajectories | Tokens | Quality note |
|---|---|---|---|
| SWE-ZERO-12M | 12.3M | **111B** | Execution-free (no verifier); from 122K real PRs |
| AgentTrove | 1.7M | 19.4B | Second largest open corpus |
| NVIDIA SWE-Zero-hand | 318K | 8.9B | Synthesized SWE-style |
| Nemotron-Terminal-Corpus | 366K | 6.2B | Terminal use |
| mini-coder-trajs-400k | 397K | 5.3B | ~21% verified |
| Toucan-1.5M (MCP tool use) | 1.5M | ~few B | 567K real-executed; rest synthetic |
| Orca AgentInstruct (tool) | 1.05M | ~few B | Synthetic, instruction-style |
| Explorer (web) + rest of web corpora | ~1M | ~2B | Synthetic; dozens of small corpora |

- The **entire open inventory** is ≈15–20M trajectories ≈ **150–200B tokens**, of which a quality gate realistically lets through a fraction.
- The lane asks for 80B — **at the edge** of honest supply even before repetition/synthesis. Anything more would be wishful accounting; anything much less would underbuy the Codex-style identity the Mission names. Full per-lane accounting to Section 3; the best Tier-A slice of it sets in Section 6's anneal reserve.

### 2e. Indic share sized against real supply

The reference open corpus is **Sangraha** (AI4Bharat, 2024) — 251B tokens across the 22 scheduled Indic languages, internally tiered the same way this plan's Section 4 will be:

| Tier | Tokens (22 langs) | Composition |
|---|---|---|
| Verified | **64.3B** | Human-verified websites, OCR'd books/PDFs, transcribed audio-video |
| Unverified | 24.3B | CulturaX + MADLAD-400 after perplexity filtering vs. Verified |
| Synthetic | 162.7B | English Wikipedia translated to 14 languages + romanized variants |
| Total | 251.3B | |

Headline languages: Hindi 34.5B (12.6 verified), Bengali 30B, Gujarati 17.2B, Tamil 17.4B, Telugu 16.3B. Earlier verified corpora (IndicCorp 20.9B, IndicNLP) add to the verified pool. Translation feedstock exists in Samanantar (49.6M en↔xx pairs — the raw material for the translated tier).

**Two facts a reviewer must confront:**
1. The **entire verified-native corpus (64.3B, all 22 languages) is smaller than our 80B lane alone** — the lane's verified portion (Section 4) is freely fillable from Sangraha-Verified, but the lane as a whole *requires* unverified + translated + synthetic, by arithmetic, not by choice.
2. The lecture's 25%-Indic hypothetical would need 500B tokens — 2× the *entire* Indic universe (251B) — exactly the wishful accounting this plan refuses. Our 4% (80B) is the honest scale at which Indic is trained as a headline *quality* lane, not a volume lane.

### 2f. Filling the 80B Indic lane — tier arithmetic

Verified scarcity is real only for the verified tier. The lane as a whole is filled by tier, each ask sized beneath its supply:

| Tier | Lane ask | Supply | Verdict |
|---|---|---|---|
| Verified | 8B (10%) | 64.3B Sangraha-Verified + 20.9B IndicCorp + IndicNLP | Covered by selection, not synthesis — no scarcity |
| Unverified | 20B (25%) | 24.3B Sangraha-Unverified; extensible by re-running the CCNet perplexity filter on CulturaX/MADLAD-400 | Covered |
| Translated | 32B (40%) | English feedstock is abundant (800B in this plan's own web lane); Samanantar 49.6M pairs seed the pipeline | Effectively unlimited |
| Synthetic | 20B (25%) | Teacher distillation, paraphrase/expansion of verified seeds, templated generation | Effectively unlimited |
| **Total** | **80B** | 251B Indic universe | Fillable with zero repetition, zero cross-lane cannibalization |

The lane stays deliberately under-dosed on verified (10%) even though supply allows more: verified corpora are news-register-dominated, and a lane that is 80% one register fails the native-generative-quality differentiator. Diversity across tiers is the design, not the constraint.

## 3. Lane inventory & supply accounting — what is real, what is repeated, what is synthesized

Method (same as 2d/2e, applied to every lane): each lane is accounted for by named datasets with token counts and a status per token — **R** real human text, **T** machine-translated, **S** generated (distillation/templates), **P** repetition of a slice already in the budget. No lane silently depends on generated or repeated data; where a share requires S or P, this plan says so in the lane's own row. All counts are pre-cross-lane-dedup and renormalized after (3f).

### 3a. General web — 800B

| Dataset | Tokens | Status |
|---|---|---|
| FineWeb-Edu (score ≥ 3) | 1.3T | R — quality-gated CC, best open web data on MMLU/ARC |
| FineWeb (base, 96 snapshots) | 15T | R — fill for non-Edu remainder |
| FineWeb-Edu-score-2, DCLM, RefinedWeb 600B, SlimPajama 627B | several T | R — diversity backup |

**Verdict: fully real, zero S, zero P.** The only cost is filtering compute (FW-Edu's classifier ≈ 6,000 H100-hours at 1.3T scale — affordable and one-time). The lane is a *selection* problem, not a supply problem.

### 3b. Code — 440B

| Dataset | Tokens | Status |
|---|---|---|
| The Stack v2 (train-full, 600+ langs, repo-grouped) | ~900B | R |
| The Stack v2 (train-smol, 17 top langs) | ~525B | R — quality subset |
| GitHub PRs / Issues / Jupyter (StarCoder2 extras) | ~45B | R |
| CodeContests, APPS, MBPP, HumanEval-style seeds | ~1B | R — high-value curated |

**Verdict: fully real, zero S, zero P.** StarCoder2 already trained 913B unique code tokens from this family; 440B is a quality-gated subset (top-17 languages + repository grouping for repository context). Overlap with the agentic lane's SWE traces is dedup'd at 3f.

### 3c. Science & math — 280B

| Dataset | Tokens | Status |
|---|---|---|
| arXiv (2.3M papers, text + LaTeX) | ~36B | R |
| ProofPile-2 (arXiv + math + OpenWebMath + books) | 55B | R |
| S2ORC (8M papers), PubMed Central | ~60B | R |
| OpenWebMath 14.7B, MathPile 9.5B, StackExchange math/science | ~35B | R |
| **Real subtotal** | **~155B** | |
| Worked-example & proof generation from verified seeds (established recipe, e.g. AMPS-style synthesis at 5–10× per seed) | ~95B | S |
| Top-slice repetition (MATH/AMPS/olympiad-class sets) | ~30B | P |

**Verdict: real majority (~55%), shortfall ~45% — explicitly S + P.** This is the first lane where the plan admits generation, by arithmetic: open science/math *pre-training* text is ~150–170B; closing 280B without S/P is impossible.

### 3d. Reasoning CoT — 160B

| Dataset | Tokens | Status |
|---|---|---|
| OpenThoughts2-1M (1M traces), OpenThoughts-114k | ~5–8B | R (traces of real problems) |
| OpenR1-Math 600K verified, NuminaMath 1M, SkyT1, s1, rStar-Math, Qwen DeepThink seeds | ~15–25B | R |
| **Real subtotal** | **~25B** | |
| Teacher distillation: seed problems × multi-sample traces (OpenThoughts measured **16× per question**) with verifier/unit-test gating | ~110B | S |
| Top-tier trace repetition (Tier-A slice also feeds the Section 6 anneal) | ~25B | P |

**Verdict: synthesis-dominated — the biggest S lane in the plan, admitted openly.** Every open reasoning corpus is SFT-scale (a few B tokens); pretraining-scale CoT (160B) *cannot exist* without generation. The defense is the known recipe: a few million curated seeds → 16× multi-sample distillation → execution/verifier filtering (which is also where the agentic-lane execution machinery is reused). Cross-lane note: long traces double as long-context data — counted once, attributed to the lane where the loss is computed (3f).

### 3e. Long-context — 140B

| Dataset | Tokens | Status |
|---|---|---|
| Books (Gutenberg, BookCorpus2, Pile Book3), arXiv full text, patents, legal (FreeLaw), Wikipedia long articles | ~80–100B raw | R |
| After length ≥ 4k / quality gate | ~60–70B | R |
| Long-doc QA + multi-hop synthesis from the real book/paper corpus (chunk → question → answer pipelines, e.g. LongBench-family recipes) | ~50B | S |
| Length-upsampled repetition of highest-value long docs | ~20B | P |

**Verdict: real ~50%, then explicitly S + P.** Honest marker: beyond ~32k tokens the length distribution is essentially all synthetic or repeated — that is arithmetic, and the plan says so now rather than letting a reviewer find it out.

### 3f. Cross-lane dedup & attribution rules

1. GitHub-hosted web text is dedup'd against the code lane; arXiv/paper HTML against the science lane; SWE/agentic traces against the code lane (SWE-ZERO's 122K PRs are code, but their *loss* is agentic — attributed to agentic, counted once).
2. Reasoning traces ≥ 4k tokens that also qualify as long-context are **counted once**, attributed to the lane whose loss design owns them (Section 5 floors resolve ambiguity: floor-covered lanes keep their tokens).
3. All lane shares in Section 2 renormalize **after** cross-lane dedup — the plan's percentages are real exposure, not raw-corpus overlap.
4. Repetition is not free: repeated slices are budgeted tokens and capped by the 3–5 epoch guidance (Muennighoff et al. saturation) for the highest-value slices only.
5. **Why repetition exists at all — and only on high-quality data.** Repetition is never a supply mechanism; it is an *emphasis* mechanism. It exists for one purpose: to concentrate compute on the highest-value slices (the top decile of quality-gated corpora, Tier-A verified traces, olympiad-class math) once unique supply is exhausted. Evidence is Muennighoff et al. (2023): repeating a fixed dataset up to ~4 epochs is nearly equivalent to unique data — repeats are a legitimate compute multiplier — and beyond ~4 epochs gains saturate and quality degrades. Hence: P buys emphasis, never knowledge; it sharpens what the model already holds, and its canonical use in this plan is the Tier-A anneal (Section 6). Where a lane is short, that gap is closed by synthesis (S), never by repetition — the reasoning lane's 25B P row is only the verifier-gated top-trace slice, and because multi-sample distillation (3d) produces *distinct* traces per seed, the plan's true exact-duplicate repetition is well under that row.

### 3g. Summary — the lane ledger

| Lane | Tokens | R | T | S | P | Verdict |
|---|---|---|---|---|---|---|
| General web | 800B | 800 | — | — | — | Fully real |
| Code | 440B | 440 | — | — | — | Fully real |
| Science & math | 280B | 155 | — | 95 | 30 | Real majority; S+P declared |
| Reasoning CoT | 160B | 25 | — | 110 | 25 | Synthesis-dominated, admitted |
| Long-context | 140B | 70 | — | 50 | 20 | Real ~half; then S+P |
| Agentic | 80B | 40 | — | 25 | 15 | Edge of open supply (2d) |
| Indic | 80B | 28 | 32 | 20 | — | Tiered, zero repetition (2e/2f) |
| **Total** | **1,980B** | **1,558** | **32** | **300** | **90** | **~79% real, 16% synthetic, 5% repeated** |

Two rows stand out and are intentional: the **reasoning lane is 69% generated** (there is no honest alternative at 160B), and **agentic sits at the edge of real supply** (2d) — the two lanes where a reviewer's "where does this come from?" gets an arithmetic answer, not a claim.

## 4. The Indic lane — tier split per language

### 4a. The four tiers and what each one teaches

The 80B lane is split by tier; each tier has a job, not just a token count:

| Tier | Tokens | Share | Teaches | Role in differentiator |
|---|---|---|---|---|
| Verified (V) | 8B | 10% | Native *register* — human-vetted news, books, speech transcripts | The quality anchor: what "native" sounds like |
| Unverified (U) | 20B | 25% | Broad *coverage* — web-scale volume, current idioms | The volume layer; perplexity-gated vs V |
| Translated (T) | 32B | 40% | *Coverage of technical/formal domains* English owns: science, law, code docs | Bridges the domain gap at native quality cost |
| Synthetic (S) | 20B | 25% | Code-mixed & short-form reality (Hinglish), creative/instructional registers | The behavioral layer: how people actually type |

The verified share stays capped at 10% by design, not by scarcity: verified corpora are news-register-dominated (2f), and a lane that is 80% one register cannot deliver the native-generative-quality headline. Each tier is a *different register of the same language*.

### 4b. Per-language budget (B tokens)

Language share follows a mix of population-weighted importance and per-tier supply headroom (the five headline languages plus the remaining 17 of the 22):

| Language | V | U | T | S | Lane total | V supply available |
|---|---|---|---|---|---|---|
| Hindi | 2.6 | 6.2 | 8.6 | 5.0 | 22.4 | 12.6 (20% used) |
| Bengali | 1.4 | 3.0 | 4.4 | 2.4 | 11.2 | ~5 (28% used) |
| Tamil | 1.0 | 1.8 | 2.4 | 1.2 | 6.4 | ~3 (33% used) |
| Telugu | 0.9 | 1.7 | 2.3 | 1.1 | 6.0 | ~2.5 (36% used) |
| Gujarati | 0.7 | 1.5 | 2.0 | 1.0 | 5.2 | ~2 (35% used) |
| Other 17 | 1.4 | 5.8 | 12.3 | 9.3 | 28.8 | remainder (22% used) |
| **Total** | **8.0** | **20.0** | **32.0** | **20.0** | **80.0** | |

Every verified ask consumes ≤ 36% of that language's verified supply — the verified tier is a *selection* problem, never a scarcity problem. The U column (20B) sits at 82% of Sangraha-Unverified (24.3B), with the extensibility path (re-running the perplexity filter on CulturaX/MADLAD-400) as the buffer. The T column (32B) draws on an English source slice of ≈35B raw tokens **excluded from the web lane's 800B spend** — the translated tier never double-counts English exposure. The S column includes romanized/code-mixed generation (Hinglish dominates real-world Indic keyboard traffic); Hindi's S (5.0B) carries ≈1.5B of it.

**Distribution basis — demand-led, supply-checked.** The language split is driven first by native-speaker weight (Hindi ≈ 28%, Bengali ≈ 14%, Tamil/Telugu/Gujarati ≈ 6.5–8% each, the remaining 17 languages ≈ 36% — Marathi, Punjabi, Kannada, Malayalam, Odia, Urdu and the others inside that bucket), and only *checked* against availability on the tier where it binds (V, verified). It is deliberately **not** availability-led: a pure supply split of the 251B universe would flatten Hindi to ~13.5% and lean on the synthetic-heavy tail. Availability constrains only the verified tier; the low-resource languages in the "other 17" are carried by the U/T/S tiers (translation is where their real coverage comes from), which is also why their verified asks are the smallest.

### 4c. Quality gates per tier

- **V:** provenance is the gate — Sangraha-Verified, IndicCorp, IndicNLP only; license-cleaned; dedup'd against the Section 1 Indic eval suite (contamination hold-out). **Quality inside the tier is judged in layers** (this is the least mature part of the open stack for Indic — no public FW-Edu-equivalent for all 22 languages): (1) mechanical signals — language-ID confidence, native-typicality perplexity, and OCR/ASR confidence (Sangraha-Verified includes OCR'd books and transcribed audio-video; machinery errors are the one class human "verification" misses); (2) a per-language quality scorer trained on a native-speaker-rated seed set (a few thousand docs per language rated on fluency, coherence, informativeness, native-idiom) distilled into a classifier via the FineWeb-Edu recipe, then scaled to the full corpus; (3) stratum-aware selection — the "best 20%" is a rank *within* strata (news / books / transcripts / wiki), so the chosen slice never collapses to a single register; (4) audit samples — 1–2% of each batch re-scored by native raters with inter-annotator agreement tracked, to catch scorer drift; (5) validation of the gate itself in the Step 8 proxy experiments — a gate earns its place only if its top slice beats the unselected pool on the C7 Indic metrics.
- **U:** language-ID confidence ≥ 0.9 + perplexity filter trained against V (CCNet-style), MinHash dedup; the gate is what separates U from noise.
- **T:** Samanantar 49.6M pairs seed the engine; each translated segment passes a back-translation score gate (COMET/round-trip) before admission; identity-shift and hallucinated segments are dropped.
- **S:** teacher-distilled from a strong English base with Indic-native prompts; 1–2% human review sample per generation run; register controls to prevent the known failure mode — English syntax leaking into "Indic" text; flagged low-fluency batches are rejected, not absorbed.

### 4d. Why this split serves the headline

The differentiator (Section 1) is native generation comparable to the model's own English — fluency, coverage, fertility. Register variety is the mechanism: V teaches the sound, U the breadth, T the domains English owns, S the way people actually type. The eval in C7 therefore reports *per-tier* provenance on generative quality (a strong plan can be audited: where did this fluency come from?), and the per-stage rollout of the tiers is a curriculum decision deferred to Section 7 (stage 1: U + T volume; stage 2: S; stage 3/anneal: V and the top slices).

## 5. Protected always-on floor & selector behavior

### 5a. Why a floor exists

The mixture selector scores candidate data by what the current model finds learnable (loss on held-out samples) and picks each batch's mix. The proxy is necessarily dominated by the model's own distribution — and the model is trained overwhelmingly on English web/code. The consequence is a stable attractor: data the proxy *distrusts* — scarce, noisy, or long-format data — gets rejected batch after batch, and a lane's realized contribution **silently decays toward zero** even when its budget says otherwise. V4 learned this with Indic and fixed it by pinning Indic at 8% of every batch, outside the selector's control. This plan extends the same protection to the three lanes where the proxy bias is most destructive: **Indic, agentic, and reasoning.**

### 5b. The protection set — and who is deliberately left out

| Lane | Protected? | Why |
|---|---|---|
| Indic | **Yes** | V4 lesson; scarce supply; the proxy's English bias is strongest against it. Unprotected → contribution → 0. |
| Agentic | **Yes** | At the edge of supply (2d); trajectories are long, multi-format, tool-result-heavy — exactly the data the loss-proxy penalizes early. |
| Reasoning | **Yes** | 69% synthetic (3d); synthetic traces read as "typical" and learnable *weakly*; the proxy cannot be trusted to protect the effort-dial distribution the plan promises (C4). |
| Long-context | No | Books/papers are English and proxy-friendly; it needs *curriculum* (length ramp), not protection. |
| Science & math | No | Abundant and proxy-friendly (educational text scores well). |

The floor is about **starve-to-zero prevention**, not about favoring weak lanes: long-context and science get their budget through the selector's normal operation and the curriculum.

### 5c. Floor levels per stage — and the compliance rule

Stage lengths are the plan's curriculum backbone (details in Section 7): Stage 1 foundations 700B (35%), Stage 2 skill-build 900B (45%), Stage 3 reasoning + long-context 300B (15%), anneal 100B (5%) — total 2T.

| Protected lane | S1 floor | S2 floor | S3 floor | Anneal floor | Floor-weighted avg | Lane target |
|---|---|---|---|---|---|---|
| Indic | 25B (3.6%) | 32B (3.6%) | 10B (3.3%) | 6B (6%) | 3.7% | 4% |
| Agentic | 7B (1%) | 36B (4%) | 12B (4%) | 8B (8%) | 3.2% | 4% |
| Reasoning | 14B (2%) | 54B (6%) | 30B (10%) | 15B (15%) | 5.7% | 8% |
| **Sum** | **46B (6.6%)** | **122B (13.6%)** | **52B (17.3%)** | **29B (29%)** | **12.5%** | |

Note: the Indic floors are the maximum the 80B lane budget admits — a naive 4% floor in every stage would demand 28 + 36 + 12 + 6 = **82B**, more than the lane holds. The floor is therefore set to the lane's feasible maximum (25 + 32 + 10 + 6 = **73B** minimum realized), and the selector's above-floor top-up carries the lane the rest of the way to its 4% target.

Reading the table: the floor guarantees a *minimum*, never the target — the selector's above-floor choices must supply the remainder (reasoning: 2.3 pp above floor; agentic: 0.8 pp; Indic: 0.3 pp). Even at its peak (anneal, 29%), the floor leaves the selector ≥ 71% of every batch free. In the anneal, floors rise because that is where the Tier-A slices (Section 6) are spent by design.

**Compliance rule (operational, not aspirational):** every batch logs realized shares per lane; over any rolling 1B-token window, each protected lane's realized share must be ≥ its stage floor. A violation is a fail-stop — training pauses and the sampler is inspected. This makes "protected" a measured invariant, not a stated intention.

> **Worked example (plain language).** Take one 1M-token batch in Stage 2. The floor part is 13.6% ≈ 136K tokens, filled automatically: ≈36K Indic + 40K agentic + 60K reasoning — the loss-proxy does not vote on these. The remaining ≈864K tokens are the free pool: web, code, science, long-context, plus *extra* reasoning/agentic if the proxy approves them. Early training the proxy approves none, so reasoning runs at its floor; late in training the model is strong enough that reasoning traces look "learnable," the proxy starts approving them, and the lane's average rises toward its 8% target. The floor prevents starvation; the selector provides the rest; the compliance rule checks both.

### 5d. Selector operating rules above the floor

1. **Floor-fill, then free pool.** Each batch is constructed in two parts: (a) the floor quota per protected lane, filled by that lane's *in-lane sampler* (not the loss-proxy), and (b) the free pool, allocated by the selector among all lanes.
2. **In-lane samplers carry the curriculum.** Within the floor quota, the sampler chooses *which* data by stage: reasoning = short traces only in S1 (the "trickle" — the model learns the form of reasoning without wasting long-trace exposure), medium traces in S2, long traces in S3/anneal — this is how the C4 effort-dial distribution is enforced even inside the protected quota. Indic = U+T volume first, V+S later (§4d). Agentic = execution-verified trajectories first, synthesized later.
3. **Loss-window selection above the floor.** For the free pool, the selector admits candidates whose current-model loss falls in the learnable window (not trivial, not hopeless) — the standard recipe — with the unprotected lanes (web/code/science/long-context) regulated by a moving-average controller so each lane's run-exposure tracks its preset share.
4. **Per-batch caps.** No single lane may exceed a per-batch cap (e.g., web ≤ 55% of any batch) — protection against the selector over-concentrating when the proxy spikes.
5. **Composition within the floor is not frozen.** The floor guarantees *quantity*; the in-lane sampler is free to rotate registers/tiers to keep the batch diverse — floors and diversity are separate controls.

**Implementation maturity (honest):** the per-batch selector is *not* exotic research — it is a few hundred lines inside the dataloader (a loss-window sampler over shard samples plus the floor-fill logic). No off-the-shelf library ships the whole thing, because the behavior is pipeline-specific; the standard practice is to build on building blocks: **datatrove** (Hugging Face — offline dedup/filter/tokenize, the FineWeb stack), the **Megatron-LM / NVIDIA NeMo** dataloader sampler APIs, and **DoReMi** (Google, public reference implementation) for the proxy-reweighting alternative. Qwen's technical reports document the same loss-window recipe operationally. The heavy lifting in this plan is offline curation (Section 4's quality stack), not the selector itself.

### 5e. What the floor is not

- **Not a fixed per-batch share for all lanes** — that would delete the selector's value; only three lanes are protected, and only as a lower bound.
- **Not a curriculum override** — the floor sets *how much*, the curriculum sets *which* (Section 7); the two never conflict because the in-lane sampler honors the stage's composition rules.
- **Not cheap slack** — floor tokens are real budget and count against the lane's share; protected lanes are still accountable to the ledger (3g). The floor is how the plan *guarantees* the ledger's numbers are realized, not an excuse to inflate them.

## 6. Anneal reserve & cooldown budget

### 6a. The reserve — declared, held back, and write-protected

The anneal is a **declared reserve of 100B tokens (5% of the 2T)**, not extra data and not a fresh-mixture experiment. It is composed of each lane's **Tier-A slice** — the top of the top by the Section 4 quality stack — *held back from the pool that stages 1–3 sample from*:

| Lane | Tier-A slice definition | Anneal share |
|---|---|---|
| Web | FineWeb-Edu top decile by scorer (stratum-aware) | 30B (30%) |
| Code | The Stack v2 top-language, repo-grouped slices incl. test files | 20B (20%) |
| Science & math | Olympiad-class + ProofPile-2 top slice + solvability-verified synthetic | 12B (12%) |
| Reasoning | Verifier-gated top traces (the C4 effort-dial top band) | 15B (15%) |
| Long-context | Longest, highest-quality docs + verified long synthetic QA | 8B (8%) |
| Agentic | Execution-verified trajectories (Toucan real-executed, SWE verified, Nemotron verified) | 8B (8%) |
| Indic | Verified top + best synthetic by per-language scorer | 6B (6%) |
| Headroom | Scorer-buffer slack | 1B (1%) |
| **Total** | | **100B** |

**Write-protection rule (the discipline that makes "reserved" mean something):** the sampler is blocked from consuming any Tier-A slice during stages 1–3 — this is a guardrail in the same family as the floors (5c), with the same fail-stop compliance. The anneal is the *only* consumer of Tier-A. This is what the assignment means by a declared anneal reserve: a reviewer can point at it, size it, and verify nothing leaked early.

### 6b. Why this preset

The anneal preset is the Section 2 main preset with the weights **tilted from bulk-knowledge lanes toward capability lanes**: web drops 40% → 30%, code 22% → 20%, science/math 14% → 12%, while reasoning rises 8% → 15%, long-context 7% → 8%, agentic 4% → 8%, Indic 4% → 6%. The reasoning is structural, not arbitrary: by the anneal, the model has consumed the knowledge lanes; what remains is *sharpening*, and sharpening happens on the lanes the Mission (Section 1) names — reasoning depth, agentic competence, Indic fluency. Note the anneal floors of 5c (Indic 6 / agentic 8 / reasoning 15) *equal* this preset — during the cooldown the floors absorb the full lane share, which is exactly the intended hand-off.

### 6c. The mechanism — why the last 5% works

1. **LR decay concentrates learning.** Over the 100B, the LR cosine-decays to ~2% of peak. At low LR the model is in the refinement regime: it updates weights around the current basin instead of exploring new ones — so it can absorb *high-quality re-exposure* without the forgetting that same data would cause mid-training.
2. **Repetition is legitimate here** — and this is where the ledger's P rows (3g: 90B total) land: roughly 60B of the anneal is Tier-A re-exposure, within the 3–5 epoch cap (3f). The anneal is the canonical place every frontier recipe repeats its best data (Llama-3, Gemma, MiniCPM-class recipes all do).
3. **The evaluated checkpoint is the anneal's output.** The final checkpoint is what the benchmark board (Section 1) measures, so the reserve directly shapes the headline numbers — which is why the best data, not the most data, closes the run.
4. **Tier-A is not novelty.** Nothing in the anneal is new content — every token was seen at least once in stages 1–3. The cooldown trades *coverage* for *mastery*; new domains or new data types are excluded by rule (the write-protect rule cuts both ways: no late additions).

### 6d. Cooldown mechanics & what is not in the anneal

- **Schedule:** last 100B tokens, LR cosine-decays to ~2% of peak; full context length maintained; the mixture is the 6b preset at floor-locked shares (the selector's free pool shrinks to ~71% — web/code/science still selectable, so the cooldown retains some adaptability).
- **Excluded:** new domains, untested synthetic generators, un-gated data — anything that has not already passed the Section 4 quality stack. The anneal is not a testing ground; the proxy experiments (Section 8) are.
- **Exit criterion:** the run ends with the last anneal checkpoint; if a mid-anneal evaluation shows capability regression (e.g., reasoning length-band drop), the rewind is *within* the reserve (re-tilt the preset), never by extending the budget — the 100B is hard.

## 7. Curriculum — stage presets & the difficulty × reasoning-length bands

### 7a. Stage structure and presets

Four stages (lengths fixed in 5c): **S1 Foundations 700B (35%), S2 Skill-build 900B (45%), S3 Reasoning + long-context 300B (15%), Anneal 100B (5%).** The per-lane token matrix below is solved so that every lane's row sums exactly to its Section 2 budget and every stage's column sums exactly to its stage budget (floors from 5c all satisfied):

| Lane (B tokens) | S1 (700B) | S2 (900B) | S3 (300B) | Anneal (100B) | Lane total |
|---|---|---|---|---|---|
| General web | 375 | 306 | 89 | 30 | **800** |
| Code | 164 | 216 | 40 | 20 | **440** |
| Science & math | 70 | 168 | 30 | 12 | **280** |
| Reasoning CoT | 35 | 55 | 55 | 15 | **160** |
| Long-context | 20 | 52 | 60 | 8 | **140** |
| Agentic | 7 | 53 | 12 | 8 | **80** |
| Indic | 25 | 39 | 10 | 6 | **80** |
| Headroom | 4 | 11 | 4 | 1 | **20** |
| **Stage total** | **700** | **900** | **300** | **100** | **2,000** |

Stage narrative (the V4 trajectory, adapted — web falls 53.6% → 34% → 29.7%, capability lanes rise):
- **S1 Foundations:** web 53.6% + code 23.4% teach vocabulary, syntax, world knowledge; reasoning appears as a short-trace trickle (5%); Indic at its floor; agentic at its floor. The model learns *what each data form is*.
- **S2 Skill-build:** code and science peak (24% / 18.7%), reasoning 6.1%, agentic 5.9%, long-context 5.8% — the skill lanes' mass phase. S2 is the "middle of the S-curve": everything is learnable, so the loss-proxy is most useful here.
- **S3 Reasoning + long-context:** reasoning 18.3% and long-context 20% dominate a third of the stage's mass while web drops to 29.7% — the longest traces and the deepest difficulty bands arrive (7c). S3 is where the C4 effort-dial is exercised hardest.
- **Anneal:** the Section 6 preset (30/20/12/15/8/8/6/1) — capability-heavy, floor-locked, LR decaying.

**Transitions:** stage boundaries are never abrupt. Presets are blended over a **warm-up band of ~10B tokens** (first ~5B of a stage ramps the new preset in, last ~5B of the old stage ramps it out), which prevents the gradient spikes and mixture-shock the lecture flags — the same mechanism as 2c's semantics.

### 7b. The difficulty × reasoning-length bands (3 × 3)

Difficulty levels: **L1 basic application** (single skill, formulaic), **L2 multi-step** (chain of 3–10 reasoning steps, standard contest/competition), **L3 research-level** (open-ended, graduate/olympiad-frontier, requires planning and self-correction). Trace lengths: **short < 512 CoT tokens, medium 512–4k, long > 4k.**

| | Short (<512) | Medium (512–4k) | Long (>4k) |
|---|---|---|---|
| **L1 basic** | **e.g.** "A train departs at 9:15; the journey takes 2h 45m. What time does it arrive?" — one-step trace. Stage 1. | **e.g.** GSM8K-class: "Priya buys 3 kg of rice at ₹85/kg and 2 L of oil at ₹160/L; she pays with ₹1000. Change?" — 3-step trace. Stages 1–2. | **e.g.** Single-claim verification over a 30k-token document (multi-hop retrieval + one check). Stage 3. |
| **L2 multi-step** | **e.g.** Debug a single function with an off-by-one error — locate, explain, fix, in a compressed trace. Stages 1–2. | **e.g.** AMC/APPS-class: "Prove the quadratic is never negative and find its minimum" — 5–8 steps with checkpoints. Stage 2 (mass). | **e.g.** An olympiad problem with an extended proof (construction, lemma, main argument), or long-context multi-hop QA with synthesis. Stages 3 + anneal. |
| **L3 research-level** | **e.g.** GPQA-class MCQ with deep inference compressed into few dense steps. Stages 2–3. | **e.g.** IMO-shortlist / LiveCodeBench-hard: a research-scale coding problem with design phase + iteration. Stage 3. | **e.g.** Frontier-style: multi-part theorem or system design over a long artifact with plan–execute–verify–repair cycles. Stage 3 + anneal (Tier-A). |

### 7c. The effort-dial distribution per stage

The C4 target is a *distribution over these bands*, set per stage (percent of the reasoning lane's tokens in that stage):

| Band | S1 | S2 | S3 | Anneal |
|---|---|---|---|---|
| L1 × short | 60% | 20% | — | — |
| L1 × medium | 30% | 30% | 5% | — |
| L1 × long | — | 10% | 10% | — |
| L2 × short | 10% | 30% | 10% | 5% |
| L2 × medium | — | 10% | 30% | 25% |
| L2 × long | — | — | 25% | 25% |
| L3 × short | — | — | 5% | 5% |
| L3 × medium | — | — | 10% | 20% |
| L3 × long | — | — | 5% | 20% |

The dial is turned, not thrown: S1 spends its trickle on L1×short so the model learns the *form*; S2 moves mass into L2; S3 and the anneal spend the bulk on L2×medium/long and L3 — and the Tier-A reserve (Section 6) is exactly the top of this distribution. The evaluation (C4) samples the same bands, so the plan's reasoning claims are checkable cell-by-cell: a model whose long-band scores lag while short-band scores lead has mis-received the dial — a Section 8 proxy can catch it before the 100B run.

## 8. Proxy experiments — 1B/3B validation spec

### 8a. What the proxy validates (and what it cannot)

The 100B run is one shot; the proxy is the cheap gatekeeper. It validates **mechanisms, not final scores**: mixture-and-floor behavior (P1), curriculum ordering (P2), the quality gates (P3), and the anneal (P4). 1B/3B models do not exhibit emergence, so the proxy's verdicts transfer *qualitatively* to 100B (the mechanism — what data the model receives — is scale-invariant); absolute scores do not transfer.

**This plan specifies the proxy; it does not run it** — the runs are budgeted and gated as the first checkpoint before the 100B run is approved.

### 8b. The experiment matrix

Fixed across all runs: the custom Indic+English tokenizer, the C1–C7 eval suite (same contamination hold-outs as the full plan), the same Section 4 pipeline outputs. Scale: **1B/20B tokens and 3B/60B tokens** (Chinchilla D = 20N at both scales — same regime as the main run). Estimated cost at 40% MFU: ≈4 H100-days per 1B run, ≈30 per 3B run; the full matrix below ≈ **150 H100-days** — two orders of magnitude below the 100B run's budget, which is the point.

| Exp | Question | Design | Primary metric | Confirm if | Refute if |
|---|---|---|---|---|---|
| **P1** | Are the protected floors load-bearing? | Full 7-lane preset with floors (as designed) vs. the same preset with floors disabled (selector free) | C6 + C7 composite (mean of C6 benchmarks + Indic fluency/fertility) | No-floor run ≥ 3 points worse on the composite | No-floor within 1 point (floors are overhead; relax them) |
| **P2** | Is the S1 reasoning trickle right? | Trickle (S1 = 2% reasoning, short traces only) vs. heavy-early (S1 = 8% reasoning incl. long traces) | C4 long-band (traces ≥ 4k) score | Heavy-early run ≥ 5 points *worse* on C4-long (wasted-exposure confirmed) | Heavy-early within 2 points on C4-long (move reasoning earlier in the curriculum) |
| **P3** | Do the quality gates buy anything? | Gated top-slice (Section 4 stack output) vs. unselected pool, for the Indic and reasoning lanes separately | C7 fluency/fertility (Indic) and C4 short/medium (reasoning) | Gated run ≥ 2 points better on its lane's metric | Gated within 1 point of the pool (rewind to the Layer-2 scorer rubric — a gate that changes nothing is a gate that is wrong) |
| **P4** | Does the anneal reserve pay? | With 5% anneal on Tier-A vs. no anneal (same tokens, no cooldown) | Mean of C1–C7 headline metrics at the final checkpoint | No-anneal run ≥ 3 points worse | No-anneal within 1 point (reconsider the reserve's size) |

### 8c. Decision semantics

- **All four confirm** → proceed to the 100B run exactly as specified; the README's numbers stand as written.
- **Any refute** → the plan rewinds to the owning section, not the whole plan: P2-refute touches §7c's dial, P1-refute touches §5's floors, P3-refute touches §4c's gate (the quality-judgment stack's Layer-2 rewind — already anticipated in 4c), P4-refute touches §6's reserve size. The proxy is a *gate*, and the gate's verdict is binding — a refute is a design change, not a footnote.
- **Reporting:** each run logs the primary metric with 3 seeds (variance is real at 1B scale; a 2-point lead with seed variance ±2 is noise, and the thresholds above are set wide enough to respect that).

### 8d. Why "specify, don't run" is a defensible position for this deliverable

The assignment asks for a plan justified by proxy-scale reasoning, not for the runs themselves. The spec is written so a reader can audit the *design* of the validation without burning the compute: named metrics, named thresholds, named rewinds. If this plan were approved, P1–P4 would be the first line item on the run budget — the proxy is the plan's own first checkpoint, and it is cheaper than every later mistake it prevents.

## 9. Risks, commitments & verification checklist

### 9a. Risk register (each with its mitigation, and where it lives)

| # | Risk | Severity | Mitigation (section) |
|---|---|---|---|
| R1 | Synthetic reasoning collapses to a few trace templates → reasoning diversity dies | High | Verifier/unit-test gating (3d), multi-teacher distillation, band quotas in the effort-dial (7c), audit samples |
| R2 | English syntax leaks into Indic synthetic text ("Indic-sounding English") | High | S-tier register gates + 1–2% human review (4c); C7 fluency measures it directly |
| R3 | Floor/compliance machinery fails in production (silent starvation) | High | Fail-stop over 1B-token windows (5c); the selector is a few hundred lines, not a framework (5d) |
| R4 | Indic quality scorers drift as new data arrives | Medium | Layer-4 audit samples with inter-annotator tracking (quality_judgement_stack.md) |
| R5 | Repetition overfits top slices / benchmarks contaminated | Medium | 3–5 epoch cap (3f); contamination hold-outs in C1–C7; eval domains excluded from every tier (4c) |
| R6 | Translated tier's MT quality fails the native-idiom bar | Medium | Back-translation score gate before admission (4c) |
| R7 | Budget shrinks mid-plan | Medium | Firm cut order: floors first (they *are* the identity), then anneal, then S3 reasoning mass — never the ledger (3g) |
| R8 | Sangraha-Unverified (24.3B) is only 21% above the 20B ask | Low | Named extension path: re-run the perplexity filter on CulturaX/MADLAD-400 (4b) |
| R9 | The effort-dial evaluation (C4 bands) is immature as a methodology | Low | Commit: release the band-labeled eval set with the run (7c) |

### 9b. Commitments (the plan's invariants)

1. **The 2T budget is hard.** No lane is extended to fix a shortfall; shortfalls are renegotiated across lanes within 2T or refused (3g).
2. **Floors are binding.** Rolling 1B-token compliance, fail-stop on violation (5c). A floor is not a preference.
3. **Tier-A is write-protected** until the anneal; the anneal is the only consumer (6a).
4. **The proxy gate is binding.** All-confirm → run as written; any-refute → rewind to the owning section (8c). The 100B run does not start on a refute.
5. **Repetition is budgeted and capped** (3–5 epochs), counted against the 2T, and never used as supply (3f).
6. **No late data.** Anything not passed through the Section 4 stack by the start of S3 is excluded by rule (6d).
7. **Eval honesty.** Contamination hold-outs are fixed before data collection begins; the C1–C7 board is frozen at run start (1).

### 9c. Verification checklist — the 8 assignment components

| # | Assignment requirement | Where it lives in this plan | Status |
|---|---|---|---|
| 1 | Share of the budget for every capability slot | §2b main preset (40/22/14/8/7/4/4) + §7a per-stage matrix | ✔ |
| 2 | Indic slot split: verified / unverified / translated / synthetic | §4a tiers (8/20/32/20B) + §4b per-language matrix | ✔ |
| 3 | Agentic, reasoning & long-context slots named with the inventory datasets filling them | §2d agentic, §3b code, §3d reasoning, §3e long-context (SWE-ZERO, AgentTrove, OpenThoughts, Stack v2, ProofPile-2, …) | ✔ |
| 4 | Protected always-on floor the selector cannot cross | §5c floors (25/32/10/6, 7/36/12/8, 14/54/30/15 B) + compliance rule | ✔ |
| 5 | Declared anneal reserve | §6a 100B Tier-A reserve, write-protected, §6b preset | ✔ |
| 6 | Difficulty × reasoning-length bands with a concrete example in each | §7b 3×3 table (9 examples) + §7c dial | ✔ |
| 7 | Justified via 1B/3B proxy with a named confirm/refute metric | §8b P1–P4 (metrics + thresholds + seeds) | ✔ |
| 8 | Honest supply math — shares reachable only via repetition/generation said plainly | §3g ledger (79% R / 16% S / 5% P), §2d/2e/2f scarcity facts, §3f rules | ✔ |

All eight components are present, each with a number a reviewer can check and a section they can audit. The plan closes where it opened: **compose backward from the benchmarks, account for every token, and say plainly where a number is a promise rather than a fact.**