# RECALL-PIVOT.md — 2026-07-28 direction decisions (canonized by Walter's order; pivot LIVE as of 2026-07-30)

## GOAL:

The inversion: the chain stores EVENTS; standing is COMPUTED.

Serve receipts, blocks, sponsorships, contests, validity-predicate outcomes, cost-to-discover attestations, and convergences all live on-chain as immutable, append-only, content-free, consumer-signed events, directly broadcastable by the consumer. Nothing on-chain is ever a verdict: no weights, no standing, no scores. Standing — the answer to "should this memory surface, and where" — is computed at the edge from that evidence, by policy that lives OUTSIDE consensus and is revisable weekly.

Standing must be a deterministic pure function of (the event stream, a published policy version), with the policy version hash anchored on-chain, so that any replacement hub reconstructs standing exactly. This preserves D-HUB-REBUILDABLE and strengthens the Four Guarantees: nothing deleted (KILL), nothing recomputed-into-record (REWRITE), no party's verdict authoritative (WITHHOLD), evidence content-free (READ).

And the load-bearing signal is outcome, not attention. Every "used" must eventually answer "did it work." That answer is carried with attestation and provenance as the THIRD provenance leg — the consumer-session use-leg, joining the existing producer and extraction legs (D-PROVENANCE-ADMISSIBILITY).

## PROBLEM:

The 2026-07-28 evidence chain:

1. **The frozen per-keyword formula rejected 24/24 legitimate serves in SMOKE-5, zero for cause** (report 28-07-26-0249; the `matched_keywords` gate, D-4.2).
2. **Decay executes on the idle timer** — ~100% of archives are idle-driven — because Deny is canon-neutral and Block is rare (28-07-26-0554).
3. **Attention signals are not outcome signals.** Served / injected / accepted say nothing about whether the memory worked; a memory can be right and the attempt still fail for unrelated reasons (SMOKE-2: the right memory was injected and the worker collapsed anyway).
4. **The frozen constants are unverifiable.** The grid winner silently changed regime (serve +15 vs trusted idle −30 ⇒ a 66.7% break-even service rate vs the legacy ~12%) while holding ±0.002 parity inside a 200–300-epoch horizon; the top-50 feasible points sit inside seed noise ±0.0196; and badPersist ≤0.20 fails at realistic Block rates for BOTH models (0.768–0.860).

A bad formula frozen in consensus becomes an unamendable fact about the corpus; changing it is a governance event.

## TASK STATUS:

(a) **Canonized** — this document.

(b) **NUMBERS PROVED; PIVOT LIVE.** The sim was re-pointed from constant-fitting to policy-evaluation: evidence-stream → candidate standing policies → the existing 9-scenario × 7-seed suite, validating the events+edge approach. Serve-semantics (renewable-resource vs one-time license) and the denial-model re-spec (Deny-neutral + rare Block) are demoted by the inversion from chain-constants decisions to edge-policy drafts.

(c) Development steps are solidified in `RECALL-PIVOT-SPEC.md`; live implementation status is recorded below.

## WHAT HAS BEEN DONE:

(2026-07-28; evidence pointers)

- **Full memory funnel proven end-to-end on-chain** — SMOKE-4 (report 27-07-26-2153).
- **kimi-k3 reasoning-monster root-caused (prompt-behavior) and fixed** — bench `f2b7ad9`; SMOKE-5 ON worker: 85 tools / 32 writes / 1001 TS lines.
- **Serve-400 triage: 24/24 legitimate, zero for cause** — report 28-07-26-0249.
- **Per-memory decay sim** — module + 6-stage grid + audit-response with a variance floor of ±0.0196 (28-07-26-0430, 28-07-26-0554).
- **Overhaul map** — 137 surfaces, 69R / 19X / 44K / 4A, plus pivot-cruft and ordering (28-07-26-0321-OVERHAUL-MAP).
- **Bench attempt-pacing knobs** (`68480ac`) + **harness logging restored** (`24d2c14`).
- **Directives stashed** — R-42 (never-abort), R-43 (docs-full-consumption), R-44 (no-waste-efficiency), R-45 (proxy-hands-off).
- **Joint amendment applied; pivot live** — chain stores EVENTS only; standing = `f(events, anchored policy_version)` computed at the EDGE.
- **`edge-policy-v1` anchored and verified** — hash `2d2faa14461aa51bb72735b05debf30defff039750e5f90c1922ae813c87899e` anchored at height 45; hub reports `status=anchor_verified`.
- **Memory-approval admission gate live** at both MCP edge and hub intake.
- **Aw income fix applied inside the anchored policy with no drift** — `worked_serve_quanta: 2`, worked-channel only; a serve+worked pair credits TWO quanta.
- **Pre-pivot keyword gate dead** — a serve with empty or absent `matched_keywords` returns 201 in live smoke.
- **D1+D3 serve-income-hold live** — serve income is HELD pending an outcome or a `serve_pending_window_epochs: 1440` window; unpaired past-window serves are VOID, contributing NOTHING (not positive, not negative). The 1440 window is a PROVISIONAL DRAFT VALUE to tune in production from real outcome-lag data, not a bench deliverable and not a launch gate.

## LIVE PROGRAM NOW IN FORCE:

The program, in force:

1. **Event schema + consensus/edge boundary spec.** What events exist, what each carries, and what may NEVER appear in an event — no verdicts, no weights, no standing, no content. The genesis wipe needs no permission (0 DAU, pre-MVP, Walter's machine); the one constraint is never wipe while a benchmark run is in flight, and before wiping confirm no run in flight plus build+test green. The joint canon amendment is applied — D-4.1 + D-4.2 + I-7 + R-DECAY-FROZEN amended as ONE amendment; the inversion IS the amendment. Serve-recording dropped the non-empty `matched_keywords` requirement (vindicated 24/24; live smoke returns 201 for empty/absent keywords). Consumer-signed receipts remain directly broadcastable; the hub relay becomes a retry-fallback only.

2. **The use-leg (third provenance leg).** Episode-level attribution: a memory fired for THIS need → did THIS need resolve — never session-level pass/fail. Harvested, not reported: the plugin observes tool signals (errors dying, tests red→green); a user "didn't work" report is the fallback/dispute path. Expensive to fake: evidence-referenced, on the GSTV-sealed pattern, welded to memory CID + session identity. Convergence (T4) stands as the positive twin of contest.

3. **Validity predicates + knowledge categories.** Invariant / version-bound / environment-bound / negative-DND / ephemeral — machine-checkable predicates checked against the consumer's harvested stack. Staleness killed by fact, not by timer.

4. **Contextual suppression fields.** A Block stores the need-card embedding as a negative anchor; suppression = query↔anchor similarity; topic-scoped trust is recovered in the dense space; evidence is monotone; the kernel is tunable edge policy. **NOTE: the READ disclosure question re-arises here — anchors are on-chain embeddings. Walter's ruling is required.**

5. **Cost-to-discover as a first-class attribute.** Bench telemetry: cycles / tool-calls / attempts-to-green. Expensive to fake.

6. **Demand-steered extraction.** The zero-injection / below-floor map becomes extraction directives; the extraction prompt is versioned, with an accept/inject-rate scoreboard.

7. **Negative-corpus-as-moat evaluation.** A product-lead candidate — Walter's call.

**PARKED / SPECULATIVE:**

- **Sponsorship tiers** (hot / warm / cold; presence, never position) — needs a ROADMAP economics decision.
- **Contest-as-primitive** — needs supersession built first; supersession becomes load-bearing, no longer deferred.
