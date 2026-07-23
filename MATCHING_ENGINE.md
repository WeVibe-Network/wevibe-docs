# WeVibe Matching Engine — Architecture Reference

**Scope:** End-to-end retrieval architecture. How a query becomes a ranked, served, decay-affecting result.

**Last reviewed: Sprint 32 (2026-06)**

**Owning decisions:** D-4.1 (KeywordWeight primitive), D-4.2 (Earned Trust decay), D-4.3/D-4.5 (per-event TX), D-9.1 (Qdrant payload), D-9.2 (Qdrant hardening), D-9.3 (enriched ranking), D-9.4 (probabilistic exploration).

---

## The Problem

A memory retrieval system at scale faces two distinct failure modes. Both must be solved or the system cannot be load-bearing for user acquisition.

**Failure 1 — Bad memories persist.** When a memory is wrong and consumers deny it, the system needs to remove it from circulation. Raw-count decay (the previous D-4.2 formula) couldn't tell a good memory with occasional false-positive denials apart from a bad memory with deserved denials, so it punished both. Result: half the good memories died alongside the bad ones. New users joining a year-old org saw a thin store of stale knowledge.

**Failure 2 — Good memories die in ranking battles.** Even with a correct decay formula, deterministic top-N ranking creates a death spiral. Two memories match a query; the higher-weighted one wins forever; the loser starves of serves, idle-decays, and archives — even though it was a perfectly good memory that simply lost the first ranking battle. This is independent of the decay formula. It is a query-engine problem.

These two failures live at different layers of the system. Both fixes ship independently and compose cleanly.

---

## The Two-Layer Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│  CHAIN LAYER — authoritative state                                     │
│                                                                        │
│  Per memory, per keyword: { weight (bps), serve_count, denial_count }  │
│  Per memory totals:        { serve_count_total, denial_count_total }    │
│  Canonical decay formula:  D-4.2 Earned Trust (applyDecay)              │
│  Event source:             matched_keywords from serve/denial batches    │
│  Terminal:                 archive when all keyword weights ≤ 1500 bps   │
└────────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │ chain → hub sync
                                  ▼
┌────────────────────────────────────────────────────────────────────────┐
│  RETRIEVAL LAYER — query engine (wevibe-hub + Qdrant)                  │
│                                                                        │
│  Candidate generation:     vector similarity + keyword overlap         │
│  Candidate ranking:        score = vector + γ·keyword_boost (D-9.3)    │
│  Position assignment:      top-1 deterministic, 2..N probabilistic     │
│  Contested detection:      score gap < threshold → twin-suppression    │
│  Lifecycle filter:         exclude ARCHIVED always                     │
└────────────────────────────────────────────────────────────────────────┘
```

The chain layer governs *what survives over time*. The retrieval layer governs *what surfaces in any single query*. They communicate only through keyword weights — the chain produces them, the retrieval layer consumes them.

---

## The Chain Layer: Earned Trust Decay (D-4.2)

The chain's only job in retrieval is to maintain per-keyword weights that reflect each memory's accumulated quality signal. Each serve and denial event updates keyword counters only for the keywords that actually matched that retrieval (`matched_keywords` from `ServeEntry` / `StoredServeReceipt`). Event handlers execute canonical `applyDecay` immediately; epoch-end processing runs an idle sweep only for memories with no events in that epoch. The central discriminator remains:

> **The discriminator is `denial_rate = denial_count / (serve_count + denial_count)`, not raw counts.**

A memory with 50 serves and 2 denials has a 4% denial rate. A memory with 2 serves and 2 denials has a 50% denial rate. Same raw counts, vastly different signal — and the formula treats them accordingly.

**Per-epoch decay formula:**

For each keyword, given `s` serves and `d` denials this epoch:

- **Serve boost:** `+ serveD × s × (serveFloor + (1 − serveFloor) × trust²)` where `trust = 1 − denial_rate`
- **Denial decay:** `− denialD × d × (denialFloor + (1 − denialFloor) × denial_rate)`
- **Idle decay** (when `s = 0 AND d = 0`):
  - If trust-earned (`serve_count ≥ trustMinServes AND denial_rate < trustMaxRate`): `− idleD × idleProtect`
  - Otherwise: `− idleD × idleUntrusted`

Memory archives (D-4.4) when every keyword weight ≤ `retrievalThreshold` (default 1500 bps).

**Why the trust gate is the load-bearing mechanism:**

The trust gate (`serve_count ≥ 1 AND denial_rate < 0.30`) decides whether a memory is protected from idle decay. This is what produces decoupling:

- **Untested memories drain at full rate.** A memory that has never been served has not demonstrated value. Idle decay drains it at `idleD × idleUntrusted` (full base rate) until it archives. There is no path for a bad memory to survive by simply hiding.
- **Tested-and-trusted memories barely decay.** A memory with ≥1 serve and low denial rate is protected at `idleD × idleProtect` (5% of base). Good memories stay alive across long quiet periods.
- **Tested-but-denied memories lose trust.** A memory that accumulates denials sees its denial rate climb past `trustMaxRate`, loses the trust gate, and reverts to full idle decay — while also taking explicit denial decay on each denial event.

The trust gate uses on-chain per-memory lifetime totals (`serve_count_total`, `denial_count_total`) and per-keyword counters. No fallback path exists for missing matched keywords: serve submissions with empty `matched_keywords` are invalid and rejected by chain validation.

**Hub-side enforcement (CO-033a):** the hub now mirrors the chain's non-empty `matched_keywords` constraint at ingress. `POST /v1/orgs/{orgID}/serves` requires a non-empty `matched_keywords []string` on every request; missing / empty / whitespace-only payloads return HTTP 400 (`internal/serves/serves.go normalizeMatchedKeywords`). The value is persisted on `serve_events.matched_keywords TEXT[] NOT NULL` (migration `wevibe-server/db/migrations/000005_add_serve_events_matched_keywords.up.sql`) and read back into `protocol.MemoryResult.MatchedKeywords` at retrieval time, where the hub computes the intersection of memory keywords and query keywords per result and filters candidates with zero overlap (consistent with the sim's `applyNewMemoryBoost` base==0 short-circuit at `wevibe-sim/ranking-fix.js:184`).

**Sprint status (CO-033b landed):**
- LANDED in CO-031 Rev 2: chain `matched_keywords` field on `ServeEntry` + `StoredServeReceipt`, per-keyword `matchedThisEpoch` gate in `applyDecay`, memory-level lifetime counters (`serve_count_total`, `denial_count_total`), archive predicate using `.every() ≤ retrievalThreshold`.
- LANDED in CO-032: hub tempered power-law sampler (positions 2..N) with new-memory boost, three retrieval env vars, `IdleDecayBPS` orphan removed.
- LANDED in CO-033a: hub `serve_events.matched_keywords TEXT[] NOT NULL` persistence (no default — pre-MVP wipe per D-13.9), strict 400-on-empty validator on `POST /v1/serves`, retrieval pass-through threading `MatchedKeywords` through `protocol.MemoryResult`.
- LANDED in CO-033b: `wevibe-protocol` JS bindings regen (`matchedKeywords` on `ServeEntry`); dashboard `buildServeBatchMsg` and live `handleSubmitBatch` broadcaster (replaces deprecated stub at `chain-submit/page.tsx:212`); MCP forwards `matched_keywords` on `POST /v1/serves` with non-empty validation; plugin threads `matched_keywords` from recall response through `toInject` loop to serve POST; dogfood-pipeline.test.ts step 1 fixture aligned with chain's 9-field canonical body; hub test infrastructure cleanup (`ListMembers` SELECT, reports/members test fixtures, `qdrantAvailable` 401 skip).
- LANDED in DMO-006 + DMO-007: `DECISIONS.md` D-4.2 + D-9.4 Implementation Clarifications subsections codify the per-keyword gate, lifetime counters, power-law-not-softmax mechanism, boost window arithmetic, and per-serve matched-keyword tracking.
- Validation workstream: chain fast-epoch primitive (`WEVIBE_EPOCH_DURATION_SECONDS` env var) + empirical replay harness measuring `chain.gap` against the sim Steady-State scenario. Sprint contract: `chain.gap ≥ 75pp AND |Δ vs sim| ≤ 5pp`.
- IMPLICATION as of CO-033b landing: every plugin → MCP → hub serve flow now sends `matched_keywords` end-to-end; the chain accepts these submissions. The remaining sprint deliverable is empirical measurement, not wiring.

**Locked parameters (chain-governance changeable):**

| Parameter | Default | Purpose |
|---|---|---|
| `serveD` | 220 bps | base serve boost |
| `denialD` | 900 bps | base denial decay |
| `idleD` | 600 bps | base idle decay |
| `grace` | 20 chain epochs | no decay during bootstrap |
| `trustMinServes` | 1 | minimum serves to earn trust |
| `trustMaxRate` | 0.30 | denial rate ceiling to keep trust |
| `idleProtect` | 0.05 | trusted idle decay multiplier |
| `idleUntrusted` | 1.0 | unverified idle decay multiplier |
| `serveFloor` | 0.4 | minimum fraction of serve boost |
| `denialFloor` | 0.3 | minimum fraction of denial decay |
| `retrievalThreshold` | 1500 bps | archive cutoff |

---

## The Retrieval Layer: Hybrid Ranking with Probabilistic Exploration (D-9.4)

The retrieval engine produces a ranked list of memories to inject into the agent's context. The key change from naive top-N ranking is a two-phase position assignment:

### Phase 1: Candidate scoring (D-9.3 unchanged)

Candidate score per memory:

```
keyword_boost = Σ(query_keyword_weight × memory_keyword_weight)
capped_boost  = min(γ × keyword_boost, δ × vector_score)
final_score   = vector_score + capped_boost
```

Defaults: `γ = 0.1`, `δ = 0.15`. ARCHIVED memories are filtered out. Per-keyword weights from the Qdrant payload (mirror of chain state per D-9.1) drive the boost.

### Phase 2: Position assignment (D-9.4 new)

```
Position 1: strict top-1 by final_score (deterministic system answer)
Positions 2..N: tempered power-law sample with temperature T
                w_i = (score_i / max_score)^(1 / T)
                sample without replacement
Default T = 0.7
```

Hub runtime knobs (wevibe-hub env vars):

- `RETRIEVAL_TEMPERATURE` (default `0.7`)
- `RETRIEVAL_NEW_MEM_BOOST_MULT` (default `0.5`)
- `RETRIEVAL_NEW_MEM_BOOST_WINDOW` (default `30`)

With chain `grace = 20`, the boost window used in ranking is `window = grace + boostWindow = 50` epochs.

**Why position 1 is deterministic:** the user always gets the system's best answer first. Predictability at position 1 is what makes the system feel reliable.

**Why positions 2..N are probabilistic:** this is the fix for the ranking-loss death spiral. A memory that scored well but not best in this query still has a meaningful probability of being served. Across enough queries, every reasonable memory gets opportunities to earn the trust gate. Bad memories get those opportunities too — and lose them via denials.

**New-memory boost (enabled by default).** For memories with age < `window` chain epochs, multiply query score by:

`query_score = raw_score × (1 + boostMult × max(0, 1 − age/window))`

where `window = grace + boostWindow`.

### Phase 3: Contested detection and twin-suppression (existing)

After position assignment, the hub computes the score gap between position 1 and position 2. If the gap is below a configured threshold (default 0.20), the memory pair is flagged `contested = true` in the response.

When the MCP client sees `contested = true`, the shipped mechanism is **deterministic twin-suppression**: it surfaces the clear winner (position 1) and suppresses the near-tied twin (position 2) rather than injecting both — no model call and no clarifying-questions loop. A model-based disambiguation (per-memory summaries / "best when" / tradeoff feeding user clarifying questions) has been considered as a future enhancement, but it is **not shipped** — it is deferred under the no-local-LLM-on-production-path rule (R-33).

The contested path remains the silent-failure-prevention mechanism: when the engine isn't confident enough to pick between near-tied answers, it surfaces a single clean winner rather than guessing between near-ties or silently shipping both.

---

## The Recall Governor — Consumer-Facing Recall (Walter-locked 2026-06-18)

**Owning decisions:** D-RECALL-INJECTION-VISIBLE, D-RECALL-CONSUMER-MATRIX-2x2, D-RECALL-FEEDBACK-FOUR-BUTTON, D-RECALL-GOVERNOR, D-RECALL-TRAJECTORY (DECISIONS.md). This section governs what happens **after retrieval, before injection** — the layer that turns a ranked candidate set into a tailored, fatigue-aware user experience.

**Implementation status (thin-client / fat-backend overhaul — SHIPPED 2026-06-19):** relevance governing lives in the **hub**, not the client. The hub accepts per-request `relevance_floor` + `surface_budget` knobs, filters below-floor candidates *before* the D-9.4 sampler, caps the governed top-N, and returns the governed set plus the full per-query scoring breakdown. The plugin **sends** its configured floor/budget (`~/.wevibe/plugin-config.json`, default floor 0.55 / budget 3) and trusts the hub-governed set — its old in-plugin floor/sort/budget governor and headless auto-accept were removed. The MCP forwards the knobs and decrypts only the governed set. Live-verified: identical on-topic query governed 6→3, off-topic 6→0 (cross-context bleed eliminated). The **D-9.3 δ-cap is now SHIPPED** too (hub + sim mirror in cross-language parity; populates gamma/delta/capped_boost; no recall regression — sim eval C3 unchanged at 0.967). **Contested-twin suppression is now SHIPPED** too — the plugin consumes the hub `contested` flag deterministically (surface the clear winner, suppress the near-tied twin; no LLM). Still **planned, not shipped:** trajectory recall (see below).

**Test mode + session-tied injection (D-RECALL-MODE-FLAG, 2026-06-21):** a single env flag **`WEVIBE_RECALL_MODE` ∈ {`prod`,`test`}** read independently by plugin + MCP + hub (default `prod`; `test` explicit + loudly logged) is the one switch for all governor bypassing used in test/dev mode. `prod` = the canonical defaults below (floor **0.55**, surface_budget **3**, recall limit **3**, hub trial-daily + recall rate-limit enforced). `test` = floor **0**, surface_budget **1000**, recall limit **1000**, hub throttles **bypassed** (trial-EXPIRY still enforced); it replaces hand-edited `plugin-config.json` hacks. Explicit per-knob config/request values still override the mode base. Two further refinements landed with it: **(a) session-tied injection** — the plugin now uses OpenCode's real `sessionID` (the old process-global random hex is removed; AMENDS D-SESSION-SERVE-DEDUP) and injects each memory **once per session** rather than re-pushing it every turn (a per-session injected-set gates the per-turn `experimental.chat.system.transform`; serve receipt fires once per real session, fixing the `session_served_memories(org,session_id,cid)` keying). **(b) inject observability** — the plugin logs every injection (`[inject] <ISO> sid=… injected N: <cid>(score,"preview")…`), the test/dev "see WHAT injects WHEN" surface. In `test` only, persisted Earned-Trust auto-accept (D-11.5/D-4.2) is disabled so each recall re-gates + re-counts every candidate through the one-at-a-time popup; `prod` keeps persisted approvals unchanged. Per-turn vs per-prompt is confirmed (OpenCode #17637): `system.transform` fires every model turn incl. no-human-prompt tool-loop turns (injection point); `chat.message` fires only on human input (recall point).

**The structural principle (carve this into the design):**

> Retrieval may be generous. Injection is earned by relevance. Interruption is earned by risk. Serve attribution is earned by surfacing past the bar. Quality judgment is earned by deliberate user action. Never fabricate a signal — in either direction — that the human did not give.

The root error the governor corrects: the current implementation treats *retrieval success* as both (1) evidence of relevance and (2) evidence that human review is warranted. Those are different questions. Relevance and risk are **orthogonal axes**:

```
                  LOW RISK                 HIGH RISK
HIGH RELEVANCE    auto-inject (serve)      HUMAN GATE (interrupt)
LOW  RELEVANCE    suppress                 suppress (never surfaced)
```

Only the high-relevance + high-risk quadrant earns an interruption.

### Injection is earned by relevance (D-RECALL-GOVERNOR)

- **Absolute relevance floor (SHIPPED, hub-side).** Previously there was no floor — Qdrant was queried with no `score_threshold` and `probabilisticRank` returned exactly `limit` items regardless of absolute score. Now the hub accepts a per-request `relevance_floor` and drops candidates whose combined score is below it *before* the sampler. The floor is calibrated and configurable (default 0.55; client-sent). **Zero-injection is a valid, healthy outcome.** Top-K is not relevance; top-K is "the least-bad candidates."
  - **Floor scale — pre-freshness (clarified 2026-07-16, D-RECALL-GOVERNOR amendment).** The absolute floor gates the **pre-freshness semantic-relevance score** (the D-9.3 combined `vector + capped keyword boost`); the D-9.4 new-memory freshness boost affects **ordering only among admitted candidates**, never admission. An absolute floor must be corpus-age-independent. The hub currently compares the floor against `row.Final`, which already includes the freshness boost (×1.5 at age 0) — a conformance bug that makes admission corpus-age-dependent and effectively disables the floor across a freshly-seeded corpus. This is a stage fix only (which score the floor reads); it does not touch the freshness multiplier, decay, or serve arithmetic (R-DECAY-FROZEN). Until it is corrected the production floor stays 0.55.
- **Use the hub score, not substring re-rank (SHIPPED).** The plugin previously discarded the hub's authoritative score and re-sorted by naive `text.includes(promptWord)` substring hits (with a "no hits → inject everything" fallback). That client-side heuristic was removed; the plugin now trusts the hub-governed, hub-ranked set.
- **δ-cap the keyword boost (SHIPPED).** The D-9.3 cap `capped_boost = min(γ·keyword_boost, δ·vector_score)` (γ=0.1, δ=0.15) is implemented in `ranking_core.go` and mirrored in `wevibe-sim/recall-sim/pipeline/rank.mjs`, kept in cross-language parity via the shared `recall-ranking-parity.json` fixture (Go + JS parity tests green). It bounds keyword domination when vector similarity is low; a no-op when keyword boost is already within δ·vector. The breakdown now surfaces `gamma`/`delta`/`capped_boost`.
- **Surface budget (attention cap — SHIPPED, hub-side).** The hub accepts a per-request `surface_budget` that caps the governed top-N (client-sent, default 3) — distinct from the arbitrary semantic caps removed earlier. The floor decides *relevance*; the budget protects *cognitive bandwidth*. It survives model/embedding/ranking swaps.
- **Contested-twin suppression (SHIPPED).** The hub's `contested` flag is now consumed client-side via deterministic twin-suppression (no longer ignored by the plugin): surface one of a near-tied pair cleanly rather than injecting both. Full LLM disambiguation stays deferred (needs a capable local LLM; would block the non-blocking plugin path).

### Interruption is earned by risk; visibility is mandatory (D-RECALL-INJECTION-VISIBLE, D-RECALL-CONSUMER-MATRIX-2x2)

- **No memory is injected the user cannot see.** The **approval popup** (TUI dialog) is the canonical surfacing surface and shows the full memory + trust/risk detail. OpenCode cannot render plugin-injected text into the visible transcript (issue #885), so the popup — plus a toast + session review log for the direct-inject modes — is how the invariant holds. Hiding injection in the invisible system-prompt path with no visible trail is disallowed.
- **Popup border is color-coded by risk:** green (safe/trusted/guard-clean) / amber (low-trust or low-confidence) / red (guard-flagged).
- **Four consumer options (2×2, retires "risk appetite"):** content `[All memories | Negative signals only]` × gate `[Direct inject | Approval gate]`. Default = All + Approval gate.
- **Non-defeatable safety override:** in ALL four modes a wevibe-guard-flagged candidate forces a one-off popup. "No gate" governs the routine path only; the scanner is never skipped.
- **Deferred-but-planned:** per-candidate quality labelling, the batched quality tray, and the "gated on risk" smart-gate third mode. For now the four binary options stand, and the floor + surface budget contain fatigue by keeping injected volume small.

### Feedback: Accept / Deny / Block / Report (D-RECALL-FEEDBACK-FOUR-BUTTON)

| Action | Meaning | Local | Corpus (D-4.2) |
|---|---|---|---|
| Accept | use it | injected | **serve** (positive) |
| Deny | "not useful now" | suppress this session/context | **neutral** (context ≠ quality) |
| Block | "not useful ever" | permanent personal blacklist | **global denial** (aggregate of distinct blockers) |
| Report | harmful / wrong | block + escalate | on-chain accountability |

The global corpus down-vote moves **Deny → Block** (Deny was previously conflated as both local-forever + global down-vote). The decay FORMULA is unchanged (R-DECAY-FROZEN); only the upstream action that emits a denial event moves. **Auto-injected memories emit a *serve*, never an *approval*** — the relevance floor is what makes that auto-serve an honest D-4.2 signal. The Block/Deny negative path is the load-bearing self-correction once the safe majority auto-serves; keep it cheap, and never fabricate a synthetic negative (an ignored injection is not a denial).

### Anticipatory / "story" recall (D-RECALL-TRAJECTORY — planned, tuning-gated)

Recall evolves from isolated per-prompt queries to **trajectory-aware** retrieval: the session need-card accumulates the direction of the work, position-1 stays the "right-now" answer, and later memories surface progressively as the work moves — recall as a story, anticipating the next need. Recall already re-fires per prompt (so mid-course change already happens); the enhancement is trajectory-carry + progressive non-repeating surfacing, client-side in the need-card. Additive — does not touch the frozen decrypt flow or the frozen decay. Compute is cheap (ranking math); embedding is a cloud API call, Qdrant runs on the hub VPS, and a modest no-GPU box covers alpha unless a local per-turn LLM rerank is later required.

### Observability: Recall Health dashboard (D-RECALL-OBSERVABILITY — ALPHA-ONLY, privacy-bounded)

The hub persists, per recall query, the ranking inputs it receives (keyword_weights, relevance_floor, surface_budget, model, vector dim) plus the **full per-candidate scoring breakdown tagged with a disposition** (`returned` / `below_floor` / `over_budget_unsampled`) — tables `query_log` + `query_candidate_scores`. The leader-only dashboard "Recall Health" page reads aggregate health from these via `GET /v1/orgs/{org}/recall-health` and renders an at-a-glance gauge strip (floor fidelity / restraint / zero-injection / contested / serve:denial) + a disposition stacked bar, alongside the **recall-alignment sim scorecard** (the C0→C3 `matrix.json` scorecard) as a validated reference.

The honest framing is load-bearing: the **sim has ground truth** (real Recall@k/MRR/nDCG); the **live system does not** — its metrics are behavioral proxies. The page shows whether the deployed system operates in the regime the sim validated, **not** a head-to-head Recall@k. Live "floor fidelity" (returned-vs-below-floor score gap) is the closest cousin of the sim's `mean_separation` but is computed differently and on a different scale — never read them as the same number.

**Privacy boundary (CANONICALUX §0 / D-MISSION-INVARIANT):** `query_log` is durable, hub-only, unanchored, non-re-derivable state — exactly what §0 forbids — permitted ONLY as time-bounded sacrificial fine-tuning telemetry while alpha is single-user (leader = only member = hub operator), and raw query text is **not** logged. Before any multi-tenant / external-operator deployment this persisted view is **replaced by a realtime, non-persisted** observability surface so no future hub operator can read members' recall activity. This replacement is a hard precondition of multi-tenant.

---

## Recall Flow (end-to-end)

```
Agent calls wevibe_recall(query)
        │
        ▼
MCP extracts keywords + embedding from query (LLM-based)
+ forwards client relevance_floor + surface_budget
        │
        ▼
Hub → Qdrant: vector + keyword candidate retrieval (D-9.3 inputs)
        │
        ▼
Hub: verify producer-model attestation + resolve producer tier
     from hub-materialized producer provenance + registry snapshot
     (D-PRODUCER-MODEL-PROVENANCE, D-CAPABILITY-REGISTRY)
        │
        ▼
Hub: capability-eligibility admission filter (pre-scoring admission; filter-only)
     (admit only producer tier ≤ consumer tier; unknown relation = fail-closed;
     exact-self reuse only when identity is proven)
     (D-CAPABILITY-ELIGIBILITY — DESIGN CANONIZED 2026-07-23; implementation UNBUILT)
        │
        ▼
Hub: drop candidates below relevance_floor (D-RECALL-GOVERNOR)
        │
        ▼
Hub: candidate set scored + ranked by final_score (D-9.3)
     (eligibility already applied as pre-scoring admission;
      not a relevance-scoring input)
        │
        ▼
Hub: position 1 = strict top-1
     positions 2..N = tempered power-law sample by score (D-9.4)
        │
        ▼
Hub: cap governed set to surface_budget (top-N)
        │
        ▼
Hub: contested check (score gap pos1 vs pos2 < 0.20?)
        │
        ▼
Hub returns: { results[] (governed set), contested: bool, scoring_breakdown }
        │
        ▼
MCP decrypts the hub-governed set (PRE re-encryption flow)
        │
        ├── NOT contested ──▶ Plugin renders approval UI with top result
        │                     User accepts/denies/reports
        │
        └── CONTESTED ──────▶ Deterministic twin-suppression (no LLM):
                              surface the clear winner, suppress the near-tied twin
                              Plugin renders approval UI with the surfaced winner
        │
        ▼
On Accept: plugin queues serve receipt (per-org pseudonymous key)
                    including matched_keywords intersection (required)
On Deny: local blacklist + denial event queued
        │
        ▼
Leader settles batches on chain: MsgSubmitServeBatch / MsgSubmitDenialBatch
Chain applies Earned Trust decay (D-4.2) on serve/denial events,
plus idle sweep for no-event memories at epoch end
Hub mirrors chain state to Qdrant payload
```

**Pipeline invariant boundary (eligibility addition):** the D-CAPABILITY-ELIGIBILITY gate above is an admission filter only. It does **not** change D-9.3 `final_score` arithmetic, D-9.4 sampling/position assignment, the frozen D-4.2 decay model, or the pinned D-RECALL-ALIGNMENT `nomic-embed-text:v1.5` 768-d embedder/query-construction path.

**Provenance-admissibility SERVING gate (2026-07-23, D-PROVENANCE-ADMISSIBILITY-2026-07-23 §22; UNBUILT).** Orthogonal to the eligibility filter and to moderation/receipts: a memory is served only with admissible provenance for BOTH its producer-session and extraction-session legs (pathways P1 `ATTESTED_EXECUTION` / P2 `PROVIDER_WITNESSED`; absence states never served; commit ≠ serveable). Enforced two-tier — hub pre-scan (this stage, before ranking/gas-bearing fetch/re-encryption) + receiving-client final check before injection; the client wins; unknown/missing/invalid fails closed. Also admission-only: same invariant boundary — no change to D-9.3/D-9.4/D-4.2/embedder.

### Deterministic need-card query construction (RATIFIED 2026-07-22; feed implementation pending)

Recall query construction runs through the deterministic need-card (D-RECALL-ALIGNMENT): harvested session signals (`intent`, `task`, `stack`, `deps`, `errorStrings`, `files`) are assembled into one canonical query input shared by the product plugin and the benchmark path. Per D-FIXLOOP-RECALL (Walter-ratified 2026-07-22), the harvest seam `buildFailing` / `testFailing` is now mandated to be fed from live build/test failure signals so repair-round recall queries include the actual failing checks and error strings via this same card. No per-recall LLM call is added on the recall path.

---

## Approval Flow (memory submission, unchanged from prior ADR)

```
Moderator decrypts pending submission
        │
        ▼
Hub runs similarity query against existing index
using new memory's keywords + embedding
        │
        ├── No similar memories ──▶ Normal approval flow
        │
        └── Similar memories found ──▶ Surface them to moderator
                                       Moderator reviews side-by-side
                                       Decides: distinct / supersede / deny
                                              │
                                              ▼
                                       On approve: keywords (generated at extraction,
                                       D-KEYWORD-AT-EXTRACTION) carry forward to leader
                                        curation; embedding (computed at moderator approval) is indexed in Qdrant
                                        at chain commit (D-6.2). Memory enters chain via leader batch
```

The moderator can ask the local LLM to compare the new memory against similar existing ones. The LLM produces a structured diff: what's the same, what's different, which is more specific, which is more general. This is a convenience for moderators reviewing unfamiliar domains. It is not required.

---

## Component Authority

| Component | Decides | Does not decide |
|-----------|---------|-----------------|
| Chain (`x/memory`) | Per-keyword weight evolution (D-4.2), archive transitions (D-4.4) | Which position a memory was served in, vector similarity |
| Hub (`internal/retrieval`) | Candidate scoring (D-9.3), position assignment (D-9.4), contested detection, **pre-scoring capability-eligibility admission filtering + relevance-floor + surface-budget governing of the surfaced set (D-CAPABILITY-ELIGIBILITY, D-RECALL-GOVERNOR)**, full per-query scoring breakdown | Memory content, decay formula, which memory is "better" |
| MCP client | Keyword extraction, embedding, **forwarding the client floor/budget knobs to the hub**, deterministic twin-suppression of near-tied results, encryption/decryption of the governed set | Scoring, ranking, the floor/budget values, what to surface |
| Local LLM | Keyword generation, memory summaries, comparison analysis | Approval/denial, ranking |
| Moderator | Content quality, duplicate resolution, supersession | Keyword weights, scoring parameters |
| Agent | Which memory to apply based on user context, what clarifying questions to ask | What memories exist, how they scored |
| User | Clarifying questions; Accept/Deny/Report decisions | Everything else |

---

## What This Architecture Achieves

**Simulation output (`wevibe-sim/ranking-fix.js`) across 9 org scenarios × 7 seeds × 300 epochs (not production telemetry):**

| Configuration | Avg Gap (good_surv − bad_persist) | Good Survival | Bad Persistence | False-Archive of Good |
|---|---|---|---|---|
| Previous D-4.2 (flat-count decay) + deterministic top-N | 15pp | 68% | 54% | 30% |
| New D-4.2 (Earned Trust) + deterministic top-N | 78pp | 94% | 16% | 5.5% |
| **New D-4.2 + D-9.4 (probabilistic top-N)** | **79.5pp** | **94.5%** | **15.1%** | **5.5% (16.4% in cold storage)** |

5× improvement in decoupling, 5.5× reduction in good-memory false-archive. Robustness (worst-case scenario gap) improves from 0.4pp to 68.2pp.

**Specific failure modes resolved:**

1. **Silent wrong-memory delivery** — contested detection plus deterministic twin-suppression
2. **Bad memory accumulation** — Earned Trust decay archives unproductive memories
3. **Good memory ranking-loss death** — probabilistic position assignment gives every reasonable memory chances
4. **Stale memories outliving relevance** — idle decay still applies, but only to unverified memories; trusted ones survive long quiet periods

---

## What This Architecture Does NOT Do

1. **Does not replace human moderation.** The chain governs survival; moderators govern admission. Decay cannot replace the quality gate at submission time.

2. **Does not handle topic drift.** A memory that was correct under React 17 and is wrong under React 19 still requires human supersession through moderation similarity review in the approval flow. Decay catches contradiction at the same point in time; it does not catch deprecation across time.

3. **Does not work without consumer feedback.** If consumers never deny or accept, the chain has no signal to apply. The plugin's four-button UX (Accept, Deny, Block, Report — D-RECALL-FEEDBACK-FOUR-BUTTON) is the data source. Orgs with `serve_receipt_required = false` and few denials will see slower convergence.

4. **Does not eliminate contested ambiguity.** Probabilistic exploration breaks the ranking-loss death spiral but does not make every query unambiguous. The contested path remains the safety net for near-tied scores.

5. **Does not *require* a local LLM for contested cases.** The shipped contested handling is **deterministic twin-suppression** — surface the clear winner, suppress the near-tied twin — with no model on the recall path. A model-based rerank to sharpen contested ordering is a **deferred, optional** future enhancement, never a hard dependency: if it is ever enabled it would run non-blocking with deterministic twin-suppression as the always-present fallback and the original order preserved on error or timeout. Contested recall always completes safely without any rerank.

   **R-33 clarification (for the deferred rerank).** R-33 (no local LLM on a production path) targets a **hard dependency** on a slow/serial/org-hosted LLM sitting inline on the recall path — NOT an opportunistic, consumer-side rerank. IF that deferred rerank is later shipped, it is carved out because it would be: *opportunistic* (only for contested top-K) · reached via **the consumer's own separately-configured model** (through the `LlmProvider` abstraction — its own OpenRouter/local HTTP endpoint, NOT the host agent's LLM and NOT an org/leader LLM) · **non-blocking** (D-PLUGIN-NONBLOCKING) · **bounded-timeout** · backed by a **deterministic fallback** · and **off or pinned in test**. Under those constraints the rerank would never become a production hard-dependency, so R-33 is satisfied.

---

## What To Watch For

### Short term

- **Trust-gate empirical validation.** The `trustMinServes = 1, trustMaxRate = 0.30` defaults were tuned against synthetic scenarios. Production data may shift these. Both are chain-governance parameters, changeable via vote without a code release.

- **Probabilistic exploration temperature.** `T = 0.7` was the sweet spot in simulation. Lower T (toward 0.3) approaches deterministic; higher T (toward 1.0) approaches uniform. Monitor production query distribution; tune via hub config without consensus changes.

- **Contested threshold tuning.** Score gap = 0.20 default. If contested rate climbs above 20% in production, the scoring engine (D-9.3) needs work — too many ties signals keywords aren't discriminating enough.

### Medium term

- **Disambiguation LLM latency (deferred rerank only).** The shipped contested path is deterministic (twin-suppression) and adds no model latency to a recall. IF the deferred model-based rerank is later enabled, it would add ~3-5 seconds to a contested recall; at that point, monitor what percentage of recalls trigger it, and raise the contested threshold or use a faster endpoint if the rate is high.

- **Memory supersession chains.** When memory B supersedes A and later C supersedes B, the lineage must be tracked so deprecated memories don't appear in recall or similarity comparisons.

- **Contested patterns as org signal.** Repeated contested results on the same query across users is a signal the org should consolidate memories. Surface this to leaders as a curation hint.

### Long term

- **Contested rate vs memory count.** As an org grows from 100 to 10,000 memories, near-misses multiply. If contested rate grows linearly with memory count, the contested path (deterministic twin-suppression) becomes the primary experience rather than the exception. At that point: better pre-filtering (vocabulary tightening), hierarchical memory organization (topics), or selective contested gating (only on high-stakes queries).

- **Cross-org memory sharing.** If orgs federate, contested detection becomes the interface between "our way" and "their way." Currently out of scope.

- **Topic-aware consumer modeling.** The Unique-Consumer signal (count of distinct serve-pubkeys per memory) was tested as a candidate trust signal and rejected in our current simulation because consumers were modeled as topic-agnostic. In production, consumers do have topic preferences, and distinct-consumer count may genuinely add discrimination. Worth re-testing with production data before committing chain primitives to it.
