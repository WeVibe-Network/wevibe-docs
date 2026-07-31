# RECALL-PIVOT-SPEC — the conclusive event-schema + boundary spec

**Date:** 2026-07-28 · **Status:** LIVE pivot spec, amended by Walter's 2026-07-30 rulings · **Canon basis:** `RECALL-PIVOT.md` (Walter's canonized decisions)
**Evidence basis:** `28-07-26-0758-SIM-PROOF-events-edge-useleg` (the numbers) · `28-07-26-0554-SIM-AUDIT-RESPONSE` (the audit) · `28-07-26-0321-OVERHAUL-MAP-per-keyword-to-per-memory` (137 surfaces) · `28-07-26-0249` (SMOKE-5 serve-400 triage)
**Relationship to the numbers findings:** SUPPLEMENTARY — this document carries no new measurements beyond the 2026-07-30 target-level corrections; it fixes the schema, the boundary, the invariants, the targets, the production mirror, and the inventory that the proved numbers made actionable.

---

## 1. HEADER + HOW TO READ (the whole review at a glance)

**What was decided (the inversion).** The chain stores **EVENTS** — serve / block / outcome (use-leg) / contest / sponsorship / validity-predicate outcome / cost-to-discover attestation / convergence — immutable, append-only, content-free, consumer-signed, directly broadcastable (hub relay = retry-fallback only). **Standing** — "should this memory surface, and where" — is **COMPUTED at the edge** as a deterministic pure function of (the event stream, a published policy version whose hash is anchored on-chain). Nothing on-chain is ever a verdict: no weights, no standing, no scores, no content. The per-keyword decay formula, per-keyword weights, and the `matched_keywords` serve-gate leave consensus entirely. The load-bearing signal is **outcome** ("did it work", episode-level), not attention; serve-semantics (license vs renewable) is demoted to a revisable edge-policy draft.

**Why, in four links (each proven):** (1) the frozen `matched_keywords` gate rejected **24/24 legitimate serves, zero for cause** (SMOKE-5); (2) **~100% of retirement was idle-driven** because Deny is canon-neutral and Block is rare — the formula starves; (3) **attention ≠ outcome** — a right memory injected still saw the worker collapse (SMOKE-2); (4) the **frozen constants are unverifiable** — the grid winner silently changed regime (66.7% vs ~12% break-even) inside horizon parity, the top-50 feasible points sit inside seed noise ±0.0196, and badPersist ≤0.20 fails at realistic Block rates for BOTH models (0.768–0.860).

**What proves the fix:** **Q1** — adding the outcome signal at ≥80% observability restores bad-retirement at every realistic Block rate: badPersist **0.115–0.133, all CI95s exclude 0.20** in the original immediate-credit proof; the ratified held-credit Aw arm accepts a ~4-point good-survival cost while preserving the mechanism; gap restored 0.08–0.18 → **0.75–0.85**, with **zero new consensus constants** — *same guillotine, new evidence*. **Q2** — license vs renewable is a **statistical tie at every rate** (|Δ| ≤ 0.0021 inside noise ±0.02); the audit's inversions were retrieval-feedback artifacts → the semantics choice is demoted to edge policy. **Q3** — **one event stream drove 8 policies** with no re-simulation, and standing replayed from the saved log is **bit-exact** (parity max|Δ| = 0.000e+0) — D-HUB-REBUILDABLE demonstrated at sim scale.

**How to read this document.** §1 is the complete at-a-glance review. Sections 2–8 are the probe-depth: §2 the full reasoning chain, §3 the event schema + the boundary rule, §4 every invariant pressed against the design, §5 the measurable contract (benchmark targets from the sim), §6 the sim→production mirror + how the bench analyzes production, §7 the three core work components, §8 the FULL INVENTORY — KEPT / REMOVED / UPDATED.

---

## 2. THE REASONING (the conclusive case)

### 2.1 The four-link evidence chain

**Link 1 — the frozen gate rejects only the legitimate.** SMOKE-5 (kimi-k3 worker, bench `f2b7ad9`) recorded 24 serve-receipt HTTP 400s. Triage case-split (report `28-07-26-0249`, corroborated by the overhaul map): **24/24 were legitimate serves** rejected purely for empty keyword intersection (recall was tool-failure-dominated → no query keywords → zero overlap); **0** were plugin bugs; **0** unknown. The 400 (`"matched_keywords is required, non-empty (D-4.2 Implementation Clarifications)"`) was emitted by MCP `http-server.ts:1246`. The requirement catches nothing and rejects reality — the first empirical proof that the per-keyword gate is wrong, not merely unfashionable.

**Link 2 — the formula starves under canon semantics.** Canon: Deny is **neutral** (local-only); only **Block** feeds decay, and Block is rare (plausibly single-digit %). The audit (`28-07-26-0554`, slice A) swept the true-positive denial rate at canon-realistic {0.02, 0.05, 0.10, 0.20}: **badPersist breaches at every rate for BOTH calibrated models — 0.768–0.860** (constraint ≤0.20). Cause instrumentation: **100% of bad archives are idle-caused** at those rates (99.8% even at the sim's native 55–70% denial regime). The denial quantum's direct execution role is ~nil; the ONLY lever that retires anything is the trust gate (serves≥1 && denialRate<0.30) arming the idle guillotine (−30 → −600 bps/epoch). And at low denial rates **any served memory — bad or good — earns that trust**, so one early serve shelters a bad for the entire 200–300-epoch horizon.

**Link 3 — attention is not outcome.** Serves, injections, accepts say nothing about whether the memory *worked*. SMOKE-2 supplied the cleanest possible counter-example: the right memory was injected for the need, and the worker collapsed anyway for unrelated reasons. A system that prices memory health on attention optimizes for being looked at, not for being right.

**Link 4 — the frozen constants are unverifiable, so freezing them froze an unknown.** The audit showed: (a) the grid winner (serveD 15) silently **changed regime** — a clean serve (+15) vs trusted idle (−30) implies a **66.7% break-even service rate** vs the legacy ~12% — while holding ±0.002 parity inside the 200–300-epoch horizon, so the regime change is *invisible* to the harness that produced it; measured serve-rate medians are 5.7–19.7%, so under the winner essentially nothing outside pinned knowledge breaks even. (b) The **top-50 feasible grid points span 0.0101 — inside one seed-noise half-width ±0.0196**; ranks inside the band are seed luck. (c) badPersist ≤0.20 — the headline feasibility claim — fails at realistic Block for **both** models. A bad formula frozen in consensus is an unamendable fact about the corpus; changing it is a governance event. That is the trap the inversion exits.

### 2.2 The inversion

The chain stores **events**; standing is **computed**. Serve receipts, blocks, outcomes, contests, sponsorships, validity-predicate outcomes, cost-to-discover attestations, and convergences live on-chain as immutable, append-only, content-free, consumer-signed events, directly broadcastable by the consumer. Nothing on-chain is ever a verdict. Standing is computed at the edge from that evidence by policy that lives **outside consensus and is revisable weekly** — a deterministic pure function of (events, published policy version), with the policy-version hash anchored on-chain so any replacement hub reconstructs standing exactly. And the load-bearing signal becomes **outcome**: every "used" must eventually answer "did it work", carried with attestation and provenance as the **third provenance leg** — the consumer-session use-leg, joining the producer and extraction legs (D-PROVENANCE-ADMISSIBILITY).

### 2.3 The proof (what the sim demonstrated)

**Q1 — the money question: does the outcome signal restore bad-retirement? YES, at ≥80% observability, with margin.** Over one fixed set of 252 evidence streams (tpDeny {0.02,0.05,0.10,0.20} × 9 scenarios × 7 seeds), six policy arms were replayed. badPersist (constraint ≤0.20, mean over 63 cells, per-seed CI95):

| arm | tp=0.02 | tp=0.05 | tp=0.10 | tp=0.20 |
|---|---|---|---|---|
| noOut (audit baseline) | 0.8585 ±0.0057 BREACH | 0.8489 ±0.0105 BREACH | 0.8305 ±0.0082 BREACH | 0.7575 ±0.0115 BREACH |
| out@ρ0.5 | 0.3554 BREACH | 0.3151 BREACH | 0.2635 BREACH | 0.1967 marginal |
| out@ρ0.65 | 0.1985 marginal | 0.1884 marginal | 0.1638 **PASS** | 0.1405 **PASS** |
| **out@ρ0.8** | **0.1332 ±0.0172 PASS** | **0.1283 ±0.0140 PASS** | **0.1214 ±0.0077 PASS** | **0.1152 ±0.0136 PASS** |
| out@ρ1.0 | 0.1049 PASS | 0.1049 PASS | 0.1017 PASS | 0.1031 PASS |
| out@ρ0.8, degraded θ(0.8/0.2) | 0.1971 marginal | 0.1817 PASS | 0.1637 PASS | 0.1426 PASS |

All ρ0.8 CI95s **exclude 0.20** in the immediate-credit proof (worst upper bound 0.1504). Paired per-seed diffs vs noOut are **−0.50…−0.75** with CIs ±0.007–0.026 — the effect is **25–100× the noise floor**. In the original proof, goods were not harmed: goodSurv 0.9400 → 0.9532–0.9551, gfa 0.060 → 0.045–0.049, across immediate-credit outcome arms and rates. Under the ratified held-credit Aw arm, Walter accepted the honest cost at ρ0.8 (goodSurv 0.9535 → 0.9145; T5 gives the corresponding gfa levels). The decoupling **gap restores from 0.08–0.18 to 0.75–0.85** (ρ≥0.65) — at/above the native-regime 0.758. The noOut column reproduces the audit's independently-produced slice-A baselines **exactly** (0.8585/0.8489/0.8305/0.7575) — cross-validation of the new architecture.

**Mechanism — same guillotine, new evidence.** No new constants were invented: an observed *didn't-work* is treated as a denial-equivalent and an observed *worked* as a serve-equivalent, in both the trust accounting and the weight quanta. At ρ=0.8 a bad accumulates ~0.72 didn't-work per serve (0.8 × 0.9) → denialRate ≈ 0.4+ → the **trust gate trips** → idleProtect lost → **−600/epoch idle guillotine** → archive in ~14 idle epochs. Goods at ρ=0.8 see ~0.08 didn't-work per serve → trust retained. The trust gate was already the binding lever (audit finding 2); the outcome signal is precisely the evidence that trips it for bads at realistic Block rates. Diminishing returns above ρ=0.8 (ρ1.0: 0.101–0.105), and robustness to a degraded outcome channel (θ 0.8/0.2 passes 3/4 rates). **Engineering target: harvest outcome for ≥~80% of uses; below ~0.65 the leg fails at low Block rates.**

**Q2 — license vs renewable: a coin flip at every rate → demoted.** Both serve-semantics were replayed as pluggable standing policies over the SAME 315 streams. Every paired |Δ| ≤ 0.0021 — **10× under the ±0.02 noise floor** (coin flip at 0.02/0.05/0.10/0.20 AND at native); goodSurv identical (0.939–0.941). Without the outcome signal, BOTH breach badPersist catastrophically (0.754–0.859) — the trust-gate shelter dominates both shapes. And the audit's on-policy results do **not** reproduce over shared evidence: the winner's native edge (+0.0246, CI [+0.0072,+0.0455]) **vanishes** (−0.0006 ±0.0013) and the 0.05/0.20 inversions never appear — proving they were properties of the **retrieval-feedback loop**, not of standing arithmetic. Under the pivot's architecture (one on-chain stream, N edge policies) the semantics are indistinguishable at every denial regime. The choice is therefore demoted to a second-order **edge-policy draft**, decidable later on design grounds (renewable: 12% duty cycle sustains; license: trust-flag dominated) — what retires bads is the use-leg, not the serve shape.

**Q3 — the architecture itself: SOUND.** Events were generated **once**; **8 policies** were replayed per stream at every realistic tier (504 replay rows / 63 streams) with zero regeneration. Replay-from-disk in a fresh process reproduces standing hashes **exactly (4/4 MATCH)**; regenerating a stream from its seed yields **byte-identical** event JSON (sha256 equal); and the deepest form — replaying the serving policy over its own event log — reconstructs the on-policy engine's standing **bit-exactly** (max|Δ| = 0.000e+0 on all 7 metric fields, all 63 native cells + 4 spot cells). This is **policy velocity** (N policies over one immutable evidence stream, no re-simulation) and **rebuildability** (standing as a pure function of events + policy code) holding in miniature — **D-HUB-REBUILDABLE demonstrated at sim scale.**

**Disclosed limits (stated, not hidden):** production θ (good/bad resolution rates) is unknown — proven at θ(0.9/0.1), robust at θ(0.8/0.2); noisier production resolution raises required observability (measurement task for the use-leg build). The replay frame does not model the retrieval-feedback loop (a policy altering future serves) — an on-policy sim variant is the natural follow-up. fpDeny stayed scenario-native (no joint tp×fp sweep). The sufficiency boundary inside (0.5, 0.65) at Block ≤0.05 is unresolved — ρ≥0.8 is the safe target either way. The sim can no longer serve as independent evidence for the policy it selected; the bench now validates the simulator as much as the policy. The robust θ(0.8/0.2) box is a **class-conditional-noise** result, while the harvester's real error rate is instance-dependent, so thresholds do not transfer cleanly — report θ **per rung**, never pooled. The outcome window creates selection pressure toward fast-resolving knowledge: architectural insight that takes three days to validate may receive no credit, and no credit plus idle decay equals retirement. Walter's not-yet-built hedge is for window expiry to emit an explicit **unresolved-within-window** observation, so a future policy can distinguish "never seen" from "seen but slow."

---

## 3. EVENT SCHEMA + BOUNDARY

All events share one envelope: **`(event_type, org_id, memory_cid, epoch/window, signer_pubkey, nonce, [event fields], signature)`** — ed25519 over a canonical body by the consumer's per-org serve key (D-SERVE-CONSUMER-SIGNED lineage). Every event is **content-free** (references and fingerprints only — never plaintext, keys, or memory content; D-MISSION-INVARIANT / R-37), **consumer-signed**, and **directly broadcastable** by the consumer; the **hub relay is a retry-fallback only**, never the required path. The org serving key signs only the transaction envelope (gas carrier via org feegrant, D-S32-ORG-FEEGRANT-ALL); it cannot mint event content.

### 3.1 The eight events

| # | Event | Payload fields (inside the signed body, beyond the envelope) | Notes |
|---|---|---|---|
| E1 | **serve** | *(none beyond envelope)* | A memory was delivered to a consumer in a window. `matched_keywords` is **OUT of the signed preimage** (sig scheme `wevibe-serve-v1`→**v2**); it survives only as optional, never-gated descriptive metadata off-chain. The 400-on-empty class is **deleted** (vindicated 24/24). |
| E2 | **block** | `serve_ref` (fingerprint of the originating serve) · **`neg_anchor` = need-card embedding** *(gated — see READ flag, §4)* | The rare true-negative ("never for this need"). The contextual-suppression anchor makes topic-scoped trust recoverable in dense space (pivot item 4); the embedding field ships **only with Walter's READ-disclosure ruling** — schema carries the slot, policy decides activation. |
| E3 | **outcome (use-leg)** | `episode_ref` (session identity + need reference) · `worked` bit · `evidence_ref` (GSTV-sealed pattern) | **The load-bearing event.** Episode-level: a memory fired for THIS need → did THIS need resolve — never session-level pass/fail. Welded to memory CID + session identity; expensive to fake (evidence-referenced). **Harvested, not reported** (§6). |
| E4 | **contest** | `counter_evidence_ref` | A memory is challenged as wrong. **PARKED semantics** — contest-as-primitive needs supersession built first (pivot PARKED list); schema slot defined, activation deferred. |
| E5 | **sponsorship** | `tier ∈ {hot, warm, cold}` — **presence, never position** | A principal stakes presence on a memory. **PARKED semantics** — needs the ROADMAP economics decision; schema slot defined, activation deferred. |
| E6 | **validity-predicate outcome** | `predicate_id` (hash) · `result ∈ {pass, fail, absent}` · `evidence_ref` | Machine-checkable predicates (invariant / version-bound / environment-bound / negative-DND / ephemeral) checked against the consumer's harvested stack. **Staleness killed by fact, not by timer.** Fail-closed: a timed-out/failed predicate records ABSENCE — never an inferred result. |
| E7 | **cost-to-discover attestation** | `cycles` · `tool_calls` · `attempts_to_green` · `evidence_ref` | Bench telemetry of what it cost to reach the knowledge. Expensive to fake; feeds standing as a first-class attribute (pivot item 5). |
| E8 | **convergence (T4)** | `convergence_ref` (independent rediscovery evidence) | The **positive twin of contest**: independent sessions converge on the same knowledge. |

### 3.2 THE BOUNDARY RULE

**What may NEVER appear in an event:** no verdicts, no weights, no standing, no scores, no trust values, no archive flags, no derived judgments of any kind — and no content (no plaintext, no DEK, no memory-content embeddings; fingerprints and content-free evidence references only). An event is an **observation with provenance**, never a conclusion. Even E3's `worked` bit is a raw observed fact bound to evidence; what it *means* for standing is policy's business, computed off-chain.

**Standing = `f(events, policy_version)`** — a deterministic pure function. The policy is **published and versioned; the policy-version hash is anchored on-chain**. Consequences: (a) any party — replacement hub, client, auditor — recomputes standing **identically** from the public event stream (Q3 proved this bit-exact at sim scale); (b) policy revision is a new anchored version, never an edit of history; (c) the chain's only durable authority is the **event log**; every standing projection (hub Postgres, Qdrant payload) is derived, disposable, and exactly rebuildable (D-HUB-REBUILDABLE).

---

## 4. INVARIANT PROBE (certainty, when pressed)

Each invariant is pressed against the design as a question an adversary would ask. The guarantee language: **no single party may unilaterally READ plaintext, WITHHOLD the network's function, REWRITE the record, or KILL an org's knowledge** (D-MISSION-INVARIANT / CANONICALUX §0; RECALL-SYSTEM §8).

| Invariant | The press | The answer | Verdict |
|---|---|---|---|
| **READ** | Do events leak content? | Events are content-free by construction (§3.2): fingerprints + evidence references only; hub never receives `epoch_sk`/DEK/plaintext — unchanged. | **HOLDS** — with ONE OPEN FLAG below |
| **READ (suppression anchors)** | E2's `neg_anchor` puts a **need-card embedding on-chain** — a semantic shadow visible to all observers. Does that widen the disclosed envelope beyond the keyword shadow already ratified (D-EMBEDDING-HONEST-CLAIM)? | **Genuinely open.** The pivot canon itself flags it: *"the READ disclosure question re-arises here — anchors are on-chain embeddings. Walter's ruling is required."* The schema carries the slot **inert** until ruled; suppression policy does not activate without it. | **⚠ OPEN — Walter ruling pending (carried, not papered)** |
| **WITHHOLD** | Can a hub (or WeVibe) stop a consumer's evidence from entering the record? | Events are consumer-signed and **directly broadcastable**; the hub relay is retry-fallback only. A hub that refuses to relay is routed around, and one that refuses to *serve* is replaceable — standing recomputes from the same public stream. Gas rides the org feegrant (inherited canon, unchanged — the consumer needs no wallet). | **HOLDS — strengthened** (submission path no longer hub-mediated) |
| **REWRITE** | Can anyone alter history — events, or standing-as-record? | Events are immutable append-only. Standing is **never written into the chain record** — it is recomputed, so there is nothing to rewrite into. Policy change = a **new anchored version**, never an edit; old standing remains exactly reproducible under its own version. The genesis wipe (pre-MVP, D-13.9) is the honest vehicle precisely because in-place migration would be a rewrite. | **HOLDS — strengthened** |
| **KILL** | Can withdrawing infrastructure destroy knowledge or standing? | Nothing is ever deleted: archival is an **edge-computed visibility outcome**, not a chain deletion. The event log + commitments persist; any key-holder replays events + anchored policy and reconstructs standing **exactly** (Q3: bit-exact at sim scale). | **HOLDS — strengthened** |
| **D-HUB-REBUILDABLE** | Can a replacement hub reconstruct without incumbent cooperation? | Rebuild = replay the event stream under the anchored policy version → identical standing (a pure function). This is *stricter* than the old rebuildable (which rebuilt per-keyword weight state): now the hub carries **no authoritative state at all**. Acceptance stays top-k parity on a fixed query set. | **HOLDS — strengthened; demonstrated at sim scale (Q3)** |
| **Human approval gate** | Does outcome-harvesting bypass the consumer's gate? | Untouched. The four-button gate / D-RECALL-INJECTION-VISIBLE governs injection; the harvester **observes post-use tool signals** (errors dying, tests red→green) and never gates, never injects, never auto-judges. A user's "didn't work" report is the **fallback/dispute path**, not the data source. | **HOLDS — untouched** |
| **Deny-neutral** | Does the outcome leg re-arm Deny as a corpus signal? | No. Deny stays canon-neutral (local-only). The chain's negative events are **Block** (rare, deliberate) and **outcome didn't-work** (harvested evidence) — Deny emits nothing. The sim's denial model used Block rates (0.02–0.20), exactly the canon regime. | **HOLDS — preserved by construction** |
| **Decay never hub-side-authoritative** | The hub now *computes* standing for retrieval — has authority quietly moved hub-side? | The hub computes, but **mints nothing**: standing is a deterministic pure function of public inputs (events + anchored policy). Any party can recompute and **catch a lying hub**; the chain event log is the only durable authority; every hub projection is derived + rebuildable. **Pressed honestly:** the guarantee's teeth are determinism + the *capability* for independent recomputation — a client-side/verifier recompute path is a **build dependency** of the pivot program (today's client trusts the hub's standing), not something to assume away. | **HOLDS by design — with a named build dependency (not a failure; flagged loudly)** |
| **R-DECAY-FROZEN** | Was the frozen formula amended by stealth? | No. The freeze held throughout: the proof ran **sim-only**, zero chain/constant edits (md5 pins on the decay module verified unchanged before and after). The formula is **replaced only by Walter's explicit joint amendment — D-4.1 + D-4.2 + I-7 + R-DECAY-FROZEN amended as ONE** (the inversion IS the amendment; RECALL-PIVOT.md). One signature, no partial edits, no silent drift. | **HOLDS — satisfied procedurally, amendment is explicit and atomic** |

**Invariant probe summary:** no invariant **FAILS**. One is **OPEN** (READ vs suppression anchors — Walter ruling pending, carried in §3 E2). One carries a **named build dependency** (independent standing recomputation gives the hub-never-authoritative guarantee its teeth). The **disclosed limits** ride forward from the proof (§2.3): production θ unknown (measurement task for the use-leg build, reported per rung), the retrieval-feedback loop unmodeled in the replay frame (on-policy follow-up sim), the sim no longer independent evidence for its chosen policy, CCN-vs-IDN transfer limits, the window's selection pressure toward fast-resolving knowledge, and E4/E5 semantics PARKED pending supersession and the ROADMAP economics decision respectively.

---

## 5. BENCHMARK TARGETS (the measurable contract, from the sim)

These are the numbers the production system must reproduce **through the bench against the real path** (§7.3) — not sim aspirations. Statistics discipline is part of the contract: every comparison is published with **per-seed CI95**, and **no claim is made inside the noise floor** (±0.02 @ n=7; the audit's published variance).

| # | Target | Threshold | Sim evidence (basis) |
|---|---|---|---|
| T1 | **bad survival paired policy contrast** | Replaying the SAME bench event log under `edge-policy-v1` as shipped vs the same policy with E3 outcomes ignored, compare survival of the SAME planted bads by exact one-sided McNemar on discordant pairs, direction pre-registered. Decision requires the four-way conjunction: direction `b > c`, risk difference ≥ 0.50, reverse-discordant cap `c ≤ 1`, and `p < 0.05`. At K=14 archetypes, rd≥0.50 requires `b-c ≥ 7` (not 6; the old "6 of 8" was scoped to K=8). Absolute badPersist is secondary: report a Clopper-Pearson 90% interval, with 0.20 as a descriptive reference line, not a gate. | 0.1152–0.1332 @ ρ0.8, all four Block rates; worst CI upper bound 0.1504; paired diffs −0.50…−0.75 (25–100× noise floor) |
| T2 | **RETIRED — redundant with T5** | No gate. Structural finding: `goodSurv + gfa = 1` exactly across all 244 arm × tp sim rows (verified in `wevibe-sim/runs/policy-sim-arms/analyze.txt`), so T5 (`gfa ≤ level`) is `goodSurv ≥ (1-level)` and is strictly stronger than T2's old ≥0.90. T2 carries no information T5 lacks. | Retired in place; T3–T6 labels intentionally unchanged. |
| T3 | **gap** (goodSurv − badPersist) | ≥ 0.75 | 0.75–0.85 @ ρ≥0.65; native-regime reference 0.758 |
| T4 | **outcome-harvest observability ρ** | ≥ 0.80 **PROVISIONAL** pending measurement of the ρ ceiling; ρ is capped by the fraction of episodes resolving inside the window. | ρ0.5 fails at Block ≤0.10 (0.2635–0.3554); ρ0.65 marginal at 0.02–0.05; **below ~0.65 fails at low Block**; ρ1.0 adds little over 0.8. Live probe returned N=0 because `serve_events` carries no `episode_ref` and no `session_id`, so no direct join path exists; only a weak proxy exists via `outcome_events.session_id` → `session_served_memories` → `serve_events` memory hash. The threshold may be structurally unreachable until measured. This does **not** make T4 a bench launch gate; Walter ruled the window itself is tuned in production. |
| T5 | **gfa** (good false-archive) | Level set from the ratified Aw arm, not the pre-D1 artifact: tp=0.10 values are `0.0855 @ ρ0.8` and `0.0632 @ ρ1.0`; worst case across tp at ρ0.8 is `0.0869` (tp values 0.0869 / 0.0865 / 0.0855 / 0.0868). The old ≤~0.05 was reachable only pre-D1 (`out@rho0.8` gfa=0.0465 at tp=0.10) and is unreachable under any held-credit policy. | Aw@ρ0.8 gfa = 0.0855 at tp=0.10 and 0.0869 / 0.0865 / 0.0868 at tp 0.02 / 0.05 / 0.20. Aw@ρ1 gfa = 0.0632 at tp=0.10 and 0.0641 / 0.0642 / 0.0634 at the other tp. Walter accepted the resulting ~4-point good-survival cost (goodSurv 0.9535 → 0.9145 at ρ0.8). |
| T6 | **ttaBad reported SPLIT** | fraction-of-bads-archived **and** median-time-among-archived, separately; arm labels must not be mixed. | the audit's "58" was a horizon-placeholder artifact — (8 scenarios × 34 + founding's no-bads placeholder 250)/9. Under the ratified Aw arm, `ttaBadMedianArchived = 33.84 @ ρ0.8` and `33.13 @ ρ1`; the immediate-credit `out@rho0.8` value is 34.57. The widely quoted "33.71 vs 34.36" pair is the D1 arm, not Aw. T6 got slightly **better** post-D1, contrary to the expected direction, because everything crosses the retirement threshold sooner when everything earns less. Caveat: the sim emits the outcome in the SAME epoch as the serve, so outcome lag is zero by construction and the sim structurally cannot see the latency cost. **No placeholder-averaged figure is admissible.** |
| T7 | **serve events: zero keyword-gated rejections** | the 400 class is **deleted** | 24/24 legitimate serves rejected in SMOKE-5 → 0 by construction (no keyword condition exists in E1) |
| T8 | **funnel seams non-null** | every funnel-seam metric must be **non-null** before a run counts as evidence | the no-blind-run observability discipline (bench §4A): a blind seam voids the run, loudly |

T4 is the pivot's engineering burden made measurable: the use-leg must **harvest** outcome for ≥80% of uses — which is why the design is "harvested, not reported" (plugin-observed tool signals, not user reports; §6). Its ρ≥0.8 threshold is provisional until the production-resolving-window ceiling is measured; Walter ruled that the window value itself is tuned in production from real outcome-lag data, not delivered by the bench and not used as a launch gate.

**Mandatory disclosure — off-policy replay contrast.** T1's paired replay contrast is off-policy for one arm: under the no-outcome policy, different memories would actually have been served. The bias direction is conservative for the stated contrast. Without outcomes, planted bads survive longer, get served more, and generate more evidence, so the true gap would, if anything, be wider than the fixed-log replay reports.

---

## 6. PRODUCTION MIRROR (sim → production, clean separation)

The sim's architecture transfers one-for-one. The separation rule: **the chain carries evidence; the edge carries judgment; the bench carries measurement.**

| Sim component (proved) | Production mirror | Boundary guarantee |
|---|---|---|
| **Event stream** — append-only log generated once (`generateStream`), consumed by every arm | **Chain event log** — the 8 events of §3 as immutable, content-free, consumer-signed chain records (x/serve refactored from receipt-validation to event-log; new event types added) | Nothing else on-chain judges: no weights, no standing, no formula in consensus |
| **Standing policy** — `replayPolicy`, a swappable pure function over (events, policy) | **Hub standing engine** — an edge service: versioned policy code, published; the **policy-version hash anchored on-chain**; every projection it emits is derived + rebuildable | Policy lives **outside consensus**, revisable weekly; anchored version ⇒ anyone recomputes identically |
| **Trust gate + idle guillotine** — `serves≥1 && denialRate<0.30` arming −30→−600/epoch | **Edge policy code** — the same arithmetic as the first policy draft (the frozen constants migrate here as *draft values*, no longer chain-frozen) | **Never consensus.** A bad policy draft is a bad deploy, not a governance event |
| **Outcome harvest** — worked/didn't-work at ρ≥0.8, episode-level (`out{u,r,wk}` events) | **Plugin use-leg harvester** — observes **tool signals**: errors dying, tests red→green, the episode closing; emits E3 welded to CID + session identity, GSTV-sealed evidence pattern | **Harvested, not reported.** The user's "didn't work" report is the fallback/**dispute path**; expensive to fake, never session-level pass/fail |
| **Event replay** — replay-from-disk ≡ on-policy standing, bit-exact | **Hub rebuild procedure** — replay chain events under the anchored policy version → identical standing; acceptance = top-k parity (D-HUB-REBUILDABLE) | The hub holds **zero authoritative state** |
| **Ground-truth sidecar** — `isGood`, which policies never see | **Bench-only** — planted ground truth lives in the harness, never in production | Measurement never contaminates the measured system |

### 6.1 The bench-analysis plan — how each §5 target is proved against production

| Target | Measured through | Status in the harness |
|---|---|---|
| T1 paired bad-survival contrast / T3 gap / T6 ttaBad-split | **Planted-bad corpus**: the bench seeds known-bad memories (ground truth in the sidecar) and tracks their retirement through the funnel, then replays the same event log under `edge-policy-v1` as shipped vs E3-ignored for the pre-registered McNemar contrast; ttaBad is split as fraction-archived vs median-time-among-archived | **⚠ BENCH WORK ITEM — planted-bad retirement tracking (new)** |
| T5 gfa (T2 retired as redundant) | Known-good corpus false-archive tracking via the funnel scanners; goodSurv is derived because `goodSurv + gfa = 1` structurally | Extends existing `RecallFunnelScan` family (mostly built) |
| T4 outcome-harvest ρ | `uses_with_harvested_outcome / total_uses` per run, from the plugin harvest log; provisional threshold pending production measurement of the window-limited ρ ceiling | **⚠ BENCH WORK ITEM — outcome-harvest-rate metric (new)** |
| T7 zero keyword-gated rejections | `serve_receipt_failures` by status — the keyword-gated class must not exist | Existing `ServeInjectParse` scanner (built; the class self-empties by deletion) |
| T8 funnel seams non-null | Seam scanners gate the run: a null seam voids the run loudly | Built (no-blind-run observability; pacing knobs `68480ac`; harness logging `24d2c14`) |
| CI discipline | per-seed/per-rep CI95 published with every comparison; no claims inside the noise floor | Audit convention carried into bench scorecards |

The two flagged work items are the **only** harness additions the pivot requires; everything else is already-built observability re-pointed at the new events.

---

## 7. THE THREE CORE WORK COMPONENTS (Walter's framing, made concrete)

### 7.1 Production code carries the burden

What must be true in each repo (full granularity in §8):

- **wevibe-chain** — the decay formula, per-keyword weights, and the `matched_keywords` requirement have left consensus; x/serve is the **event log** for the event stream; the **policy-version hash anchor** is live; all of it rides the **genesis wipe** discipline (0 DAU, pre-MVP, Walter's machine; never wipe during a benchmark). The joint amendment is applied and the pivot is live.
- **wevibe-server (hub)** — the **standing engine** (versioned edge policy, pure function of events), event replay/rebuild, retrieval + Qdrant payloads consuming standing instead of keyword weights, the watcher re-pointed from mutation to event-mirroring. One atomic hub change-set, verified on the redeployed artifact (R-DELIVERY-INTEGRITY).
- **wevibe-mcp** — serve-signing **v1→v2** (lockstep with chain + hub verify + parity tests), the 400 gate deleted, outcome-event (E3) emission client-side, retrieval response shapes re-based on standing.
- **wevibe-opencode-plugin** — serve-forward already intact (no change needed there); the **use-leg harvester** is the new component: observes tool signals (errors dying, tests red→green), emits episode-level outcome events; user report = dispute path.

### 7.2 The benchmark carries the measurement + the automation-simplified workflow

**Mostly built:** no-blind-run observability (funnel seam scanners, T8), the deterministic golden-oracle gate suite, attempt-pacing knobs (`68480ac`), spend meters, pervasive harness logging (`24d2c14`), resumable per-unit checkpointing conventions. **The adds (the only two):** the **outcome-harvest-rate metric** and **planted-bad retirement tracking** (§6.1). The bench's job is unchanged in kind and narrowed in scope: it already measures a real system through the real transport; it now also measures *retirement* and *harvest*.

### 7.3 Target numbers met inside the benchmark

**Acceptance = the §5 table reproduced by the bench against the real path, not the sim.** The sim proved the *mechanism* (the outcome signal trips the existing trust gate for bads at realistic Block rates — same guillotine, new evidence). The bench must prove the *product*: real plugin harvest toward the provisional ρ≥0.8 target while production tunes the window (T4), real planted bads passing the paired McNemar policy contrast with absolute badPersist reported secondarily (T1), goods protected by the accepted T5 gfa level (T2 retired), the gap holding (T3), ttaBad honestly split (T6), zero keyword-gated rejections (T7), no blind seams (T8) — all through the real transport (R-EVAL-INTEGRITY) on the rebuilt artifact (R-DELIVERY-INTEGRITY), every comparison published with per-seed/per-rep CIs and no claims inside the noise floor. A number that only holds in the sim is a hypothesis; the contract is bench-measured.

---

## 8. FULL INVENTORY — KEPT / REMOVED / UPDATED

Derived from the overhaul map's **137 surfaces** (69 REPLACE / 19 REMOVE / 44 KEEP / 4 ADD), re-verdicted against the pivot. The re-verdict logic: the map's KEEPs survive unchanged; its REMOVEs stay removed; its REPLACEs now **split** — surfaces whose *function* dies outright (the formula, per-keyword weights, the matched_keywords requirement) become **REMOVED**, while surfaces that persist with a new shape become **UPDATED**; of its 4 ADDs, the chain `trust_weight_bps` field is **dead on arrival** (the chain holds no standing at all), the hub trust store re-scopes to a *derived standing projection*, the dashboard display stays optional, and extraction demand-steering defers to pivot item 6. The pivot's own new surfaces (chain event log + policy anchor, mcp E3 emission, plugin harvester, two bench metrics) are marked **NEW**.

### 8.1 KEPT (24 rows)

| Repo | Surface | Why it survives |
|---|---|---|
| wevibe-chain | Lifecycle supporting infra (epoch store, contributors, active-count, save/load, counts, eligibility, decodeCID; `clampFloat64`) | Event windowing still needs the epoch machinery |
| wevibe-chain | `params.proto` reserved fields 6,7,8,11-14 | Proto hygiene; record of the expunged pivot |
| wevibe-protocol | No `.proto` sources here; bindings regen-only | Chain proto is the SoT (map validation #5) |
| hub | `retrieval.go` `contestedThreshold=0.20` | Keyword-independent (verified) |
| hub | `retrieval.go` `Gate:false` rank option | Intersection-drop already disabled in production; moot |
| hub | `serves.go RecordServe` + `serve_events` table | Becomes the hub's mirror of chain serve events |
| hub | Denial path `'{}'` matched-set convention | Denials reference their serve by fingerprint |
| hub | Leader curation handlers (Submit/Verify/Update/Remove/List) + nearDup consts | Vocabulary management is orthogonal to standing |
| hub | `schema.sql` `org_keywords` + `keyword_candidates` | Org vocabulary stays |
| dashboard | `keywords-section.tsx` + `keyword-seeding-banner.tsx` | Vocab CRUD UI, orthogonal |
| mcp | `retrieve-cli.ts` RecallGovernor (floor 0.55 / budget 3 / limit 3) | No keyword governor exists (map correction #11) |
| mcp | `retrieval-card.ts` (need-card + prompt digest) | Pure-deterministic card; rendering unchanged |
| mcp | `canonical.ts submitMemoryMessage` | Already keyword-free — no signature bump needed |
| mcp | `mc1/keywords.ts boostKeywordsByVocab` + INV-7 vocab boost | Boost-not-gate ranking, orthogonal to standing |
| mcp | `session.ts dissect_to_keywords` · `guard.ts` metadata · keyword-vote route | None carry decay semantics |
| plugin | Serve-submit forward + `?? []` coalesce + fire-and-forget **with R-37 failure logging** | Already aligned; needs NO change for the 400-class deletion |
| plugin | Keyword-free served store `{cid,text,session_ids,last_used_at}` | Already the pivot shape |
| bench | `RecallFunnelScan` · `ServeInjectParse` · preflight keyword seeding · `cumulative/types.py` telemetry sentinels · backend adapter parse · inject header | The observability the pivot re-points (§6.1) |
| wevibe-sim | `per-memory-decay.js` engine (md5-pinned, unchanged) + `runs/policy-sim.js` | **KEPT as THE policy-evaluation harness** — every future edge-policy draft is proved here first |
| wevibe-sim | `recall-sim/` retrieval-eval harness (no applyDecay anywhere) | Ranking mirror; separate system |
| mcp | Extraction core (`extraction.ts` emit path; EXTRACTION-FLOW) | Extraction produces keywords regardless of decay role; **demand-steered extraction lands later** (pivot item 6) |
| docs | D-RECALL-ALIGNMENT (boost-not-gate formula) | Untouched bar a one-phrase trim ("decay/accounting") |
| docs | D-RECEIPT-LABEL-NEVER-GATE | Orthogonal (verified) |
| docs | D-KEYWORD-AT-EXTRACTION · D-MC1-CHAIN-LEG | Vocabulary + rebuild anchors unaffected |

### 8.2 REMOVED (17 rows)

| Repo | Surface | Why it dies |
|---|---|---|
| wevibe-chain | **The D-4.2 decay formula in consensus** — `applyDecay` / `ApplyEpochDecay` (as decay driver) / `ApplyServeBoost` / `ApplyDenialDecay` (`lifecycle.go`) | **Events replace the formula.** The chain stores evidence; standing is computed at the edge (joint amendment, §8.4) |
| wevibe-chain | `resolveOrgIdleDecayConfig` + `getMatchedKeywords` + per-keyword `big.Rat` weight plumbing | Per-keyword machinery dies with the formula |
| wevibe-chain | `params.go` 13 BPS decay constants | Migrate **OUT of consensus** — they become revisable values in the first edge-policy draft, not frozen law |
| wevibe-chain | **Per-keyword weights** — `KeywordWeight` proto msg + weights embedded in `StoredMemoryCommitment` | Per-keyword weights die entirely; keywords → flat label metadata (commitment itself: UPDATED) |
| wevibe-chain | `MatchedKeywordsPrefix` index + Store/Get `MatchedKeywordsForEpoch` + denial re-index + `expected_keepers` iface | The whole matched-keyword index |
| wevibe-chain | **`matched_keywords` non-empty requirement** — `msgs.go ValidateBasic` gate + `msg_server` denial re-check (`rejectedNoKeywords`) | The requirement rejected 24/24 legitimate serves and caught nothing; deleted (vindicated) |
| hub | `api/handlers/serves.go serveEntryFromRecord` empty-check + `chain/submit.go buildServeEntries` empty-check (dead path) | 400-class removal, lockstep with chain + mcp |
| hub | `keyword_extraction.go` sum-to-1.0 curation gate | No weights → no sum gate |
| wevibe-chain | Cruft: `minKeywordWeight` · `graceEpochsRemaining` · `calculateDenialRateAndTrust` | Confirmed zero-caller/discarded (map §D) |
| wevibe-chain | The old map's ADD — `trust_weight_bps` on the commitment | **Dead on arrival**: the chain holds NO standing under the pivot |
| hub | **The flat shadow decay** — `ServeBoostBPS=100`/`DenialDecayBPS=500` + `ApplyServeBoostLocal`/`ApplyDenialDecayLocal`/`GetKeywordWeights` | Conditional-cruft #14's load-bearing question is **resolved by the pivot**: neither chain nor hub computes decay; events + edge standing do |
| hub | `SubmissionRecord.MatchedKeywords` (hardcoded `[]` everywhere) | Vestigial (cruft #6) |
| dashboard | `lib/keyword-weights.ts` `renormalizeFromBase` + `displayWeight` | Die with curation weights (cruft #16) |
| mcp | `approveSubmissionMessage` v1 + `keywordsHash` (zero prod callers) + the wevibe-meta test-lib duplicate | Dead v1 approval canonical (cruft #7/#8) |
| mcp | `extraction.ts` `normalizeKeywordBucket` / `assignRankDecayWeights` / `normalizeOrRankDecay` | Weight normalization dies with weights (cruft #15) |
| wevibe-sim | Five standalone `applyDecay` duplicates: `sim-runner.js`, `iterate.js`, `final-analysis.js`, `suppress-idle-analysis.js`, `unique-floor.js` | Subsumed by the per-memory + policy-sim harness — **extract IP before deletion** (permanent floor ×2 dedupe, unique-consumer/top1 gates, tenure/rep theorems) |
| wevibe-sim | `app/` Next.js viz shell | Walter keep/kill call (conditional cruft #17) |

**The class deletion, stated plainly:** the **400-on-empty-`matched_keywords` rejection class is deleted** across mcp (`http-server.ts:1246`) + hub + chain. This work is live: the pre-pivot keyword gate is dead, and a serve with empty or absent `matched_keywords` returns 201 in live smoke — 24/24 legitimate rejections → **zero**, by construction (no keyword condition exists in E1).

### 8.3 UPDATED (34 rows)

| Repo | Surface | What changes |
|---|---|---|
| wevibe-chain | `MsgSubmitServeBatch` / `ServeEntry` / `StoredServeReceipt` + `CanonicalServeBody` | → event schema E1/E2; `matched_keywords` **out of the signed preimage** (sig `wevibe-serve-v1`→**v2**), optional descriptive metadata only |
| wevibe-chain | `StoredMemoryCommitment` proto | Drops keyword weights; keeps ciphertext + provenance anchors; keywords → flat repeated labels |
| wevibe-chain | Genesis import/export | The genesis wipe needs no permission (0 DAU, pre-MVP, Walter's machine); before wiping, confirm no benchmark run is in flight and build+test green, then drop the matched-keyword re-index call |
| wevibe-chain | **NEW** — event-log storage for E1–E8 + **policy-version hash anchor** | The pivot's chain addition (replaces the old map's dead standing-field ADD) |
| wevibe-protocol | `js/` TS bindings | Regen after chain `.proto` edits (R-19: Docker `make proto-gen`; never hand-edit) |
| hub | `ranking_core.go` `RankCandidate`/`keywordScore`/`ScoreAndRank` + retrieval γ/δ consts | Per-keyword weight vector → **standing scalar** from the standing engine |
| hub | `UpdateKeywordWeights` (Qdrant) + payload sync | `keyword_weights` map → **standing projection** scalar (derived, rebuildable) |
| hub | `watcher_serve.go` / `watcher_memory.go` | From event-time weight **mutation** → chain-event **mirroring** into the standing store |
| hub | `serves.go normalizeMatchedKeywords` | Gate deleted: empty → `[]` accepted, logged; the hub leg of the 400-class removal |
| hub | **NEW** — the **standing engine** (versioned edge policy; pure function of events + anchored policy version) + **event replay/rebuild procedure** | The pivot's hub addition; acceptance = top-k parity (D-HUB-REBUILDABLE) |
| hub | `schema.sql` (wipe-on-change): `serve_events.matched_keywords DEFAULT '{}'` · `memory_keywords` drop weight → topical join · `query_log`/`query_candidate_scores` → standing columns · **NEW standing store** | SoT schema mirrors the mirror; standing store is a derived projection |
| hub | `verify/canonical.go` LIVE v1 approval verify | v1 removal + confirm v2 (map deviation #22: the live v1 is hub-side) |
| hub | Keyword curation weight types → weightless include/exclude | Curation = vocabulary, not weights |
| hub | `serves_test.go` contract tests | Flip to empty-allowed |
| dashboard | `moderator-review-panel.tsx` / `leader-pipeline-panel.tsx` | Weight display + renormalization → pure pills, include/exclude |
| dashboard | `hub-client.ts` / `verify-queue.ts` / `chain-client.ts` weight-bearing types | Weightless wire types; `matched_keywords` stays optional metadata |
| dashboard | Optional standing display | Only if Walter wants the observability |
| mcp | `http-server.ts handleServes` + `ServeRequestBody` | **The 400 origin**: drop the empty-disjunct; accept + log vector-only serves |
| mcp | `serve-signing.ts` `CANONICAL_SERVE_VERSION` v1→v2 | `keywordsJoined` out of the preimage; **lockstep** chain `canonical.go` + hub verify + parity tests in ONE train |
| mcp | `retrieve-cli.ts` no-keywords hard gate | Soften to warn + vector-only recall proceeds (`no_keywords` stops being a wall) |
| mcp | Recall forward / `MemoryOutput` / contribution metadata / `deserialize.ts` / `org-client.ts` / mc1 schema+types | Keyword-weight shapes → standing/label shapes |
| mcp | **NEW** — outcome-event (E3) emission | The use-leg client leg: harvest signals → sign → broadcast (hub relay = retry-fallback) |
| plugin | Serve-forward payload tolerance | Field stays optional; coalesce already handles empty (code change minimal) |
| plugin | **NEW** — the **use-leg outcome harvester** | Observes tool signals (errors dying, tests red→green), emits episode-level E3; user report = dispute path |
| bench | `keyword_match_rate.py` + Cell/scorecard fields | → per-memory standing/attribution report (lockstep with the payload change) |
| bench | `backends/base.py RecalledMemory.matched_keywords` | → descriptive id-only labels |
| bench | **NEW** — outcome-harvest-rate metric + planted-bad retirement tracking | The two flagged work items (§6.1) — the only harness additions |
| wevibe-sim | `ranking-fix.js` | Superseded as decay SoT — DECISIONS `:409`/`:1286` SoT repoint to the per-memory/policy-sim frame, **inside the joint amendment**; keeps its legacy comparison role |
| wevibe-sim | `recall-sim/pipeline/rank.mjs` (hub γ/δ mirror) | Re-based on the standing scalar; MUST stay synced with `retrieval.go` (D-RECALL-ALIGNMENT) |
| wevibe-sim | `DECAY-SIM.md` / READMEs | Re-framed per the SoT repoint |
| docs | **DECISIONS.md D-4.1 + D-4.2 + I-7 + R-DECAY-FROZEN** | **THE ONE JOINT AMENDMENT — the inversion IS the amendment**: formula → events; the gate deleted; sim-wins repoint; frozen-surfaces clause reconciled. ONE signature; a partial amendment self-contradicts |
| docs | DECISIONS D-4.4 / D-4.6 / D-5.4a / D-9.1 / D-9.3 / D-9.4 / D-SERVE-CONSUMER-SIGNED | Consequent amendments in the same stream (archive predicate → standing-threshold visibility; sig-body role → descriptive; Qdrant payload → standing) |
| docs | RECALL-SYSTEM.md (glossary archive line · the :478 400 note · chain-decay description · §11.5 SoT pointer · B2.10 mirror gap) | Narrative conformance to the inversion (B2.10 **simplifies**: the mirror target becomes one scalar — a positive interaction) |
| docs | WHITEPAPER.md (TRACKED, **deleted-in-working-tree** — map validation #17) | The per-keyword→events rewrite **folds into the restore decision**; do NOT silently restore — flagged to Walter independently |

### 8.4 Ordering constraints (preserved from the map, re-expressed under the pivot)

1. **Joint amendment applied; pivot live** — D-4.1 + D-4.2 + I-7 + R-DECAY-FROZEN have been amended as ONE; the inversion IS the amendment. The SoT repoint (:409/:1286 → per-memory/policy-sim frame) is in force. Remaining Walter calls are separate scope: sim viz shell keep/kill, optional dashboard standing display, and the READ-disclosure ruling on E2 suppression anchors (§4).
2. **Genesis wipe needs no permission; never wipe during a benchmark** — 0 DAU, pre-MVP, Walter's machine. The one constraint is economic/operational: before wiping, confirm no benchmark run is in flight and build+test green.
3. **R-19** — chain `.proto` edits → Docker `make proto-gen` → protocol regen; never hand-edit generated code.
4. **Signature-bump lockstep** — mcp serve-signing v2 ↔ chain `canonical.go` ↔ hub verify + parity tests in ONE train.
5. **400-gate removal is live** — mcp + hub `serves.go` + `api/handlers` + chain msgs/keeper dropped together; the plugin needed NO change. Live smoke proves empty or absent `matched_keywords` returns 201.
6. **Hub change-set atomic** — schema (wipe-on-change) + standing engine + retrieval + watcher land as ONE hub rebuild, verified on the redeployed artifact (R-DELIVERY-INTEGRITY).
7. **Wire-shape changes atomic** — hub↔mcp↔plugin↔dashboard flip together; `matched_keywords`-becomes-optional is the backward-tolerant exception.
8. **Canon serializes** (R-CANON-SERIAL) — DECISIONS + RECALL-SYSTEM + WHITEPAPER in one stream.
9. **Bench LAST** — no standing math lives there; the two metric adds land after the payload change.
10. **Sim-first is necessary but not sufficient for timing-dependent mechanisms** — the sim is the retained policy-evaluation harness (Q3's policy velocity), but any mechanism whose effect depends on timing still requires real-path measurement before it is treated as proven at the edge.

### 8.5 Inventory counts

| Verdict | Rows | Of which NEW (pivot additions) |
|---|---|---|
| **KEPT** | 24 | — |
| **REMOVED** | 17 | — |
| **UPDATED** | 34 | 6 (chain event log + policy anchor · hub standing engine · mcp E3 emission · plugin harvester · 2 bench metrics) |

### 8.6 Live status truth (2026-07-30)

The pivot is **LIVE**, not conditional. Chain stores events only; standing = `f(events, anchored policy_version)` is computed at the edge. `edge-policy-v1` hash `2d2faa14461aa51bb72735b05debf30defff039750e5f90c1922ae813c87899e` is anchored at height 45, and the hub reports `status=anchor_verified`. The memory-approval admission gate is live at both the MCP edge and hub intake. The Aw income fix is applied inside the anchored policy with no drift: `worked_serve_quanta: 2`, worked-channel only, so a serve+worked pair credits **two** quanta. The pre-pivot keyword gate is dead: empty or absent `matched_keywords` returns 201 in live smoke. D1+D3 serve-income-hold is live: serve income is held pending an outcome or a `serve_pending_window_epochs: 1440` window; unpaired past-window serves are **VOID**, contributing nothing — not positive, not negative. The `1440` window is an explicit **provisional draft value** to be tuned in production from real outcome-lag data; it is not a bench deliverable and not a launch gate.

Reconciliation to the map's 137 surfaces (69 REPLACE / 19 REMOVE / 44 KEEP / 4 ADD, +1 info row): the 19 REMOVEs remain REMOVED (folded into §8.2 with their cruft-list kin); the 44 KEEPs remain KEPT (consolidated to §8.1's 24 rows at file granularity); the 69 REPLACEs **split** — those whose function dies (the formula, per-keyword weights/state, the requirement, the hub shadow decay, curation-weight machinery) moved to REMOVED, the rest are §8.3's UPDATED rows; of the 4 ADDs, one is dead-on-arrival (chain `trust_weight_bps`), one re-scoped (hub standing store = derived projection), one deferred (demand-steered extraction → pivot item 6), one optional (dashboard display). Row granularity here is file-level (the map's was symbol-level), so counts are table rows, not a 1:1 symbol recount; the map remains the symbol-level reference.

---

**End of spec.** The numbers proved the mechanism; the joint amendment is applied; the pivot is live. This spec fixes the schema, the boundary, the invariants, the amended targets, the mirror, the inventory, and the current live-status truth.
