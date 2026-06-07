Writing `DECISIONS.md` now. Organizing by topic, preserving the WHY for every WHAT, using updated versions where duplicates exist.

---

# DECISIONS.md

# WeVibe Network — Architecture & Design Decisions

This document records every locked decision in the WeVibe Network architecture along with the rationale (the WHY behind the WHAT). It complements `MASTER.md` (which describes the current UX) by explaining why the system is shaped the way it is.

Decisions are organized by topic. Within each topic, foundational decisions come first, with derived/operational decisions following.

---

## Table of Contents

1. [Identity & Wallet Architecture](#1-identity--wallet-architecture)
2. [Cryptography & PRE (Proxy Re-Encryption)](#2-cryptography--pre-proxy-re-encryption)
3. [Hub Role & Trust Model](#3-hub-role--trust-model)
4. [Memory Lifecycle & Keyword Weights](#4-memory-lifecycle--keyword-weights)
5. [Memory Types & Extraction](#5-memory-types--extraction)
6. [Moderation & Approval](#6-moderation--approval)
7. [Reports & Accountability](#7-reports--accountability)
8. [Emissions & Payouts](#8-emissions--payouts)
9. [Qdrant & Vector Search](#9-qdrant--vector-search)
10. [Operational Procedures & Recovery](#10-operational-procedures--recovery)
11. [Architecture Tradeoffs (Documented Acceptance)](#11-architecture-tradeoffs-documented-acceptance)
12. [Sprint 25 Architecture: Multi-Org, Discovery, Profiles, Activity](#12-sprint-25-architecture-multi-org-discovery-profiles-activity)
13. [Sprint 25 Architecture: Chain Hardening — Accountability + Social Graph Foundation](#13-sprint-25-architecture-chain-hardening--accountability-social-graph-foundation)
14. [Sprint 29 Chain Foundation Decisions](#15-sprint-29-chain-foundation-decisions)
15. [Pattern B Tier 2 Verification Anchor (Current Design)](#16-pattern-b-tier-2-verification-anchor-current-design)

**New decisions in this update:** D-13.1 (Moderator Pubkey Persistence on Memory), D-13.2 (Upheld Report Plaintext + Ciphertext + Capsule Triplet), D-13.3 (Hub-Side Manipulation Alarm via BlockResults), D-13.4 (Social Graph Data On-Chain, Display Layer Separate), D-13.5 (Reputation Active at Genesis, Additive-Only), D-13.6 (Memory State Cleanup — 7-State Lifecycle Locked at Code Level), D-13.7 (Cross-Module Event Wiring), D-13.8 (Reputation as Tiering Signal — total_approved_memories), D-13.9 (Chain Wipe Acceptable Pre-MVP), D-13.12 (Chain Broadcast via Comet RPC), D-13.13 (Chain Pruning + IAVL Dev Settings); updates to D-2.2 (Umbral container) and D-13.10 (only host exception is Ollama).

---

## 1. Identity & Wallet Architecture

### D-1.1: Cosmos Wallet Is Primary Identity

> ⚠ **SUPERSEDED for onboarding (Sprint 32, `D-IDENTITY-PROGRESSIVE-CUSTODY`).** The wallet is **no longer the primary identity nor a prerequisite to participate.** The primary identity is now a **client-held, passkey-protected account key** created at first run with no wallet. A Cosmos wallet (Keplr/Leap) is an **optional later upgrade**, *linked* as the canonical chain authority + funding anchor via a staged handover — it is not the root every other key derives from. D-1.1 is retained below as historical context for the original alpha design. The economy-binding rationale still holds for users who upgrade; it simply no longer gates entry.

---

### D-1.2: Delegated Hot Key via `x/authz MsgGrant`

**Decision:** Users delegate signing authority to a locally-generated hot key via a one-time `x/authz MsgGrant`. The hot key signs all routine operations without further wallet popups.

**Why:** Requiring a wallet popup for every operation (memory submission, serve attestation, retrieval) would make the UX intolerable. The hot wallet pattern enables smooth daily operation while preserving security:
- The local key handles all routine signing (session retrieval, memory submission, serve attestations)
- No further Keplr popups for normal operation
- The main wallet's funds are never at risk (grant is scoped to WeVibe messages only)
- The grant can be time-limited (e.g., 30 days) and renewed
- User can revoke from the wallet at any time

---

### D-1.3: Security-Critical Operations Require Wallet Signature

**Decision:** Certain operations require the wallet itself to sign, not the delegated key:
- Org configuration changes for `required_approvals` and `report_vote_threshold`
- Report chain commitment (`MsgReportMemory`)
- Leadership transfer
- Org closure

**Why:** These operations have permanent or quorum-affecting consequences. Ed25519 delegate keys are designed for high-volume routine signing and have a weaker security posture. Wallet `signArbitrary` for consequential actions ensures the user explicitly approved them through their wallet's UI.

---

### D-1.4: PRE Identity Derived from Wallet via BIP-32 (Planned)

> ⚠ **AMENDED (Sprint 32, `D-IDENTITY-PROGRESSIVE-CUSTODY`).** The PRE encryption identity is now **generated client-side at first run** (passkey-protected), NOT derived from the Cosmos wallet — because a guest contributes/recalls before any wallet exists. Wallet-BIP-32 derivation is no longer the source. When a wallet is later linked, the **client-generated PRE key is RETAINED** (no re-key event), so linking a wallet does not invalidate any kfrags. The isolation rationale below still holds (PRE key ≠ signing key); only the derivation source changes.

---

### D-1.5: Hub Owns Member ↔ Epoch Relationship Tracking

**Decision:** The chain `x/org` module stores membership but not epoch access history. The hub's kfrag store (keyed by `org_id, epoch_id, member_pk`) is the authoritative record of which members have re-encryption keys for which epochs.

**Why:** Chain-side epoch membership tracking would balloon state and require chain transactions for every epoch access change. The hub already needs this data to apply re-encryption keys, so making it authoritative on the hub side avoids duplication. `DeleteKFrags` operates on hub-side data only — chain emits `MemberRemoved` events with org_id and member_pk, and the hub handles the rest.

---

## 2. Cryptography & PRE (Proxy Re-Encryption)

### D-2.1: PRE Solves the Real Vulnerability

**Decision:** Proxy Re-Encryption (Umbral) is the production retrieval mechanism. The hub never sees plaintext during retrieval.

**Why:** The actual critical vulnerability in the prior system was **removed member exfiltration**: a member who left the org still possessed cached epoch keys and could decrypt unlimited memories from their previous epoch with zero detection. This requires zero attacker sophistication — the member already has the keys on their laptop.

PRE eliminates this attack: without re-encryption keys (kfrags) generated by the hub for that specific member, the member cannot decrypt anything new. Deletion of their kfrags is instant and cryptographically complete for un-retrieved content.

| Attack | Difficulty | Pre-PRE System | PRE System |
|--------|-----------|----------------|------------|
| Removed member exfiltration | Trivial — has cached keys | CRITICAL — unlimited decryption | Eliminated — no kfrags = no decryption |
| Active member machine compromise | Moderate — endpoint pwn | CRITICAL — all epoch memories exposed | Bounded — only that member's PRE key; rate-limited |
| Epoch SK compromise | Hard — governance attack | CRITICAL | Hard — requires 2-of-3 Shamir collusion |

---

### D-2.2: GPL Boundary Isolation via Umbral Sidecar

**Decision:** Umbral PRE (umbral-pre 0.11.0, GPL-3.0) runs as a separate binary in a dedicated Docker container (wevibe-umbral, image built from wevibe-server/Dockerfile.umbral-sidecar). The main codebase (Apache-2.0) communicates with the sidecar via gRPC at the container's internal address (umbral-sidecar:4460 by default, overridable via WEVIBE_UMBRAL_SIDECAR_ADDR).

**Why:** Direct dependency on a GPL library would force the entire WeVibe codebase to GPL-3.0. The sidecar pattern isolates the GPL boundary to a single process, preserving Apache-2.0 licensing for everything else. The sidecar exposes:
- gRPC server (:4460) for hub use
- CLI subcommands (`encrypt`, `decrypt-reencrypted`) for wevibe-mcp use

Sprint 25 (CO-258) evolved the sidecar from a hub-spawned subprocess into a first-class Docker service. The architectural property (GPL boundary isolation via process separation) is preserved — the container is just a more disciplined process boundary than a subprocess. Healthchecks, restart policy, and resource limits all become declarative in docker-compose.yml rather than implicit in hub spawn logic.

**Note:** Umbral-sidecar is a Docker service, NOT a host exception. The only documented host exception is Ollama (see D-13.10).

---

### D-2.3: secp256k1 for PRE Identity, Aligning with Cosmos

**Decision:** PRE identity uses secp256k1 (via the `k256` crate), matching the Cosmos wallet curve. The previous Ed25519/X25519 stack is retained for non-PRE operations (signing, envelope encryption of mod-key submissions).

**Why:** Available PRE libraries dictate the curve:
- `umbral-pre` uses secp256k1 — same curve family as Cosmos wallets
- `recrypt-rs` uses a custom pairing curve — incompatible with everything else

Aligning PRE identity with the wallet curve enables BIP-32 key derivation (D-1.4) and avoids running three incompatible curve stacks. Ed25519 remains for fast signing where PRE is not involved.

---

### D-2.4: Approval Encrypts DEK Under Epoch Public Key

**Decision:** At memory approval time, the moderator's wevibe-mcp client:
1. Unseals the contributor's wrapped DEK with the moderator's private key
2. Decrypts the ciphertext to plaintext (for keyword extraction)
3. Re-encrypts the DEK under the **epoch public key** via Umbral, producing a capsule + ciphertext
4. Submits the new capsule + ciphertext to the hub for storage

**Why:** This is the moment plaintext briefly exists for moderation review. After approval, the memory's DEK is encrypted under the epoch key — the hub can re-encrypt the capsule for any active member (via their kfrags) without ever seeing plaintext. This decoupling is what enables the "hub never sees plaintext during retrieval" property.

---

### D-2.5: Chain Is Authority for Memory Confidence; Hub Mirrors

**Decision:** Per-memory keyword weights are stored on-chain. The hub maintains a local mirror for retrieval ranking but resyncs from chain state on restart.

**Why:** The chain provides immutable, auditable history of memory health signals. Without a single source of truth, hub-side state could drift from chain reality, leading to consumers retrieving stale-confidence memories. On every confirmed serve or denial TX, the hub applies the same decay/boost formula locally (R-ATOMIC) and on crash recovery calls `SyncKeywordWeightsFromChain` to reconcile.

---

## 3. Hub Role & Trust Model

### D-3.1: Hub Is an Active Re-Encryption Proxy, Not a Pure Observer

**Decision:** Under the PRE architecture, the hub plays three active roles:
- **Re-encryption proxy** — applies kfrags to produce cfrags without seeing plaintext
- **Authorization authority** — gates all retrieval based on membership and policy
- **Policy enforcement engine** — rate limits, ~~credit deduction~~ **[per-query credit deduction REMOVED — billing is now a hub-internal subscription gate; see D-S32-CO047-SUBSCRIPTION-CREDITS]**, audit logging

**Why:** Earlier whitepaper language described the hub as a "pure observer" that never signs transactions. PRE fundamentally changes this: re-encryption is an active cryptographic operation that the hub must perform on every retrieval. This is not a regression — it's the mechanism that achieves "hub never sees plaintext" while still enabling efficient access control. The hub's trust assumptions must reflect this expanded role.

---

### D-3.2: Phase 1 Metadata Exposure Is Accepted Risk

**Decision:** During Phase 1 (the period before encrypted vector search and metadata protection ship), the hub operator can observe retrieval patterns, access graphs, and temporal correlations. This is acceptable.

**Why:**
- Phase 1 hub operator is the WeVibe Network team (or trusted managed operator) — not zero-trust
- Chain observers see only ciphertext + transaction metadata, not retrieval patterns
- The alternative (delaying Phase 1) means shipping nothing while catastrophic vulnerabilities (removed member exfiltration) persist
- Metadata exposure cannot be retroactively anonymized, so Phase 2 PRE adoption with metadata protection is the only path forward

Documented as acceptable risk gated behind operational/contractual controls during Phase 1.

---

## 4. Memory Lifecycle & Keyword Weights

### D-4.1: Per-Memory Confidence Field Killed; Keyword Weights Are the Health Metric

**Decision:** The monolithic `confidence_bps` field on each memory has been removed. Each memory carries a list of `KeywordWeight { keyword, weight, serve_count, denial_count }` entries on-chain. Memory health is computed from its keyword weights.

**Why:** A single confidence value collapses too much signal. A memory might be highly relevant to one topic but degrading on another — the monolithic field can't express that. Keyword weights replaced confidence for three structural reasons:

1. **Granularity argument.** A single confidence value cannot represent topic-specific health. A memory about "Postgres connection pooling" might be highly validated on that topic but completely wrong on "MongoDB alternatives." The monolithic field flattened this into one number, losing all topical specificity.

2. **Redundancy argument.** Confidence was derived from the same signals — serves and denials — that already drive keyword weights. Storing a computed confidence field alongside the raw signals that produce it is duplicate signal storage with weaker semantics. Keyword weights are the primitive; confidence was the derivative.

3. **Decay-locality argument.** When a memory degrades, it degrades on specific topics. If a memory about "React hooks" gets 10 denials, its health on "React hooks" drops while its health on "state management" might remain intact. Per-keyword decay captures this naturally. Monolithic confidence flattened everything into one number — a memory that was wrong on one topic looked equally wrong on all topics.

---

### D-4.2: Earned Trust Decay Model

**Decision:** Per-memory keyword weights decay under an **Earned Trust** formula. Each memory carries `serve_count` and `denial_count` aggregates (D-4.1 primitives, unchanged). The decay rules at the end of each chain epoch:

**Effective signal: `denial_rate = denial_count / (serve_count + denial_count)`**

This is the discriminator. Raw counts do not drive decay — the ratio does.

**Per-keyword weight updates per chain epoch (chain authority):**

For each keyword the memory carries, given `s` serves and `d` denials touching that keyword this epoch:

- **Serve boost:** `+ serveD × s × (serveFloor + (1 - serveFloor) × trust²)`
  where `trust = 1 - denial_rate`. Good memories accelerate; suspect memories get a damped boost.
- **Denial decay:** `- denialD × d × (denialFloor + (1 - denialFloor) × denial_rate)`
  Bad memories die faster as their denial rate climbs.
- **Idle decay (when `s = 0 AND d = 0`):**
  - If **trust-earned** (`serve_count >= trustMinServes AND denial_rate < trustMaxRate`): `- idleD × idleProtect`
  - Otherwise: `- idleD × idleUntrusted`

Memory archives when **every** keyword weight ≤ `retrievalThreshold` (default 1500 bps = 15% of MAX_W).

**Locked default parameters:**

| Parameter | Value | Meaning |
|---|---|---|
| `serveD` | 220 bps | base serve boost per serve event |
| `denialD` | 900 bps | base denial decay per denial event |
| `idleD` | 600 bps | base idle decay per epoch |
| `grace` | 20 chain epochs | no decay during bootstrap |
| `trustMinServes` | 1 | minimum served events to gate trust |
| `trustMaxRate` | 0.30 | denial rate ceiling to keep trust |
| `idleProtect` | 0.05 | trusted memories idle-decay at 5% of base |
| `idleUntrusted` | 1.0 | unverified memories idle-decay at full base |
| `serveFloor` | 0.4 | minimum fraction of boost regardless of trust |
| `denialFloor` | 0.3 | minimum fraction of decay regardless of rate |
| `retrievalThreshold` | 1500 bps | weight below which a keyword is non-retrievable |

All parameters are chain-governance changeable.

**Why:**

The previous formula (flat 500 bps/denial keyword decay, 100 bps/serve boost, 50 bps/epoch idle decay) used raw counts. Empirically validated against 9 org scenarios × 7 seeds × 300 epochs, raw-count decay could not separate good memories from bad. A good memory with 50 serves and 2 false-positive denials decayed almost the same as a bad memory with 2 serves and 2 true-positive denials. Result: ~68% good-memory survival and ~54% bad-memory persistence — barely better than noise.

The Earned Trust formula decouples them by three structural moves:

1. **Signal change: use the ratio, not the counts.** `denial_rate` discriminates 50/2 from 2/2 immediately. The chain already tracks both `serve_count` and `denial_count` per memory (D-4.1) — no new state.

2. **Trust gate on idle decay.** Memories that have demonstrated value (≥1 served event with denial rate < 30%) are protected from idle decay. Memories that have never been served — and therefore never demonstrated value — drain at full base rate and archive. This is the load-bearing insight: **untested memories must demonstrate value or archive**. Bad memories that get queried accumulate denials → lose the trust gate → drain fast. Bad memories that never get queried drain anyway because they never earned the gate. No path for a bad memory to survive.

3. **Boost amplified by trust, decay amplified by denial rate.** Both the positive and negative signals scale with the memory's current quality. Good memories accelerate; bad memories collapse.

**Empirical results (9 scenarios × 7 seeds × 300 epochs):**

| Metric | Old (flat-count) formula | Earned Trust formula |
|---|---|---|
| Average decoupling gap (good-surv minus bad-persist) | 15pp | **78pp** |
| Good memory survival | 68% | **94%** |
| Bad memory persistence | 54% | **16%** |
| Bad memory time-to-archive (median) | 191 epochs | **58 epochs** |
| Good memory false-archive rate | 30% | **5.5%** |

5× improvement in decoupling, 5.5× drop in false-archive of good memories. Same chain primitives, same transactions, same state. Only the math expression in the decay step changes.

**Note on grace duration:** The 20-epoch grace period is measured in **chain epochs** (D-4.7), not wall time. With a 12-hour chain epoch, 20 epochs ≈ 10 days. With a 24-hour chain epoch, 20 epochs ≈ 20 days. Verify against actual chain epoch configuration via `wevibed query epochs epoch-info wevibe_epoch`.

#### D-4.2 — Implementation Clarifications (2026-05-28 GO-005 synthesis)

The Earned Trust Decay Model is specified by the canonical sim source at
`wevibe-sim/ranking-fix.js`, function `applyDecay` (lines 113–141), helpers
`makeMemory` (lines 40–48), and the per-epoch loop in `simulate` (lines
154–199). The following clarifications resolve ambiguities in the original
D-4.2 lock and are derived from GO-005's verbatim sim extraction.

**Source of truth.** `wevibe-sim/ranking-fix.js` is the authoritative
specification. Where this prose and the sim diverge, the sim wins.
Implementation must reproduce sim behavior.

**Chain-side state extension.** The per-keyword `matchedThisEpoch` gate in
this section requires per-serve matched-keyword tracking added to `x/serve`
in CO-031 Rev 2 (2026-05-28). See "Per-serve matched-keyword tracking" below.

**Trust computation.** Trust is computed inline at the start of each
applyDecay invocation. Not stored, not cached, not a persisted field.
Inputs are memory-level lifetime counters:

    denial_rate = m.denials / (m.serves + m.denials)   if (m.serves + m.denials) > 0
    denial_rate = 0                                     otherwise
    trust       = max(0, 1 - denial_rate)
    trustSq     = trust * trust
    trustEarned = (m.serves >= trustMinServes) AND (denial_rate < trustMaxRate)

Edge case: a memory with zero serves AND zero denials has denial_rate = 0
and trust = 1, but trustEarned is FALSE because m.serves < trustMinServes=1.
Such a memory receives full-multiplier serve boost (irrelevant — no serves
to boost on) but does NOT pass the idle-protection gate.

**New chain state required.** `MemoryCommitment` gains two `uint64` fields:
`serve_count_total` and `denial_count_total`. Both increment by 1 per
respective TX (serve, denial) in `x/memory/keeper/msg_server.go`. These
counters are memory-level lifetime, never reset, never decremented.

The per-keyword `KeywordWeight.ServeCount` and `KeywordWeight.DenialCount`
fields (D-4.1) remain unchanged and continue to increment on matched
keywords only. The canonical Earned Trust formula does NOT read the
per-keyword counters; they remain available for future per-keyword analytics.

Pre-MVP wipe acceptable per D-13.9; no migration required.

**Per-keyword `matchedThisEpoch` gate.** All three operations — serve boost,
denial decay, AND idle decay — are gated per-keyword on whether that specific
keyword matched the query that drove the event. Concretely, given per-epoch
event counters `servesThisEpoch`, `denialsThisEpoch` (the chain's existing
`getMemoryServeCount` / `getMemoryDenialCount` indexed queries) and the set
of keyword IDs matched that epoch `kwIdsMatched`:

    for each keyword k in memory.Keywords:
      matched = (k.id ∈ kwIdsMatched)
      if servesThisEpoch  > 0 AND matched:
        k.w += serveD * servesThisEpoch  * (serveFloor  + (1 - serveFloor)  * trustSq)
      if denialsThisEpoch > 0 AND matched:
        k.w -= denialD * denialsThisEpoch * (denialFloor + (1 - denialFloor) * denial_rate)
      if (NOT matched) OR (servesThisEpoch == 0 AND denialsThisEpoch == 0):
        idleMult = trustEarned ? idleProtect : idleUntrusted
        k.w -= idleD * idleMult
      k.w = clamp(k.w, 0, MAX_W)

Consequence: an active memory's unmatched keywords still receive idle decay
each epoch. Matched keywords stay strong; unmatched keywords trend toward
archive. The empirical decoupling-gap result depends on this per-keyword
discrimination.

**Grace period gates all three operations uniformly.** If
`current_epoch - memory.created_epoch < grace`, applyDecay returns early —
no boost, no decay, no idle decay. The early-return precedes any per-keyword
loop. Grace blanket-protects new memories from all three operations during
their first `grace` epochs.

**Archive trigger semantics (clarifies D-4.4).** Replace the prior `allZero`
predicate with:

    archive iff memory.Keywords.every(k => k.weight <= retrievalThreshold)

This means **ALL** keywords (not ANY) must be at or below the threshold for
archive to fire. Sim source: `ranking-fix.js:198`, identical predicate in
`sim-runner.js:117`, `iterate.js:233`, `final-analysis.js:158`, and
`app/page.js:182`. The archive transition fires at end-of-epoch after
applyDecay completes for the memory.

Archive is **terminal** — no resurrection path. `MemoryCommitment.archived_epoch`
(`uint64`, default 0 / "not archived") records the epoch at which transition
occurred for audit purposes.

**Per-epoch event counter source.** The `servesThisEpoch` and
`denialsThisEpoch` inputs to applyDecay correspond to the chain's existing
indexed queries `getMemoryServeCount(ctx, orgID, cidHex, currentEpoch)` and
`getMemoryDenialCount(ctx, orgID, cidHex, currentEpoch)`. The matched
keyword set `kwIdsMatched` is derived from the same per-epoch index — every
serve event records the matched-keyword IDs at submission time; idle decay
reads the union for the epoch.

**Per-serve matched-keyword
tracking (added 2026-05-28 via CO-031 R-ABORT resolution, Option A).**
The per-keyword `matchedThisEpoch` gate requires the chain to know which
keywords were matched by the query that drove each serve (and the
corresponding denial, if any). This information is computed hub-side at
query time but is not currently persisted on-chain. Option A extends
`x/serve` to carry this state.

**Proto extensions to `x/serve`:**

    // proto/wevibe/serve/v1/tx.proto, message ServeEntry:
    //   add field:
    repeated string matched_keywords = <next>;

    // proto/wevibe/serve/v1/state.proto, message StoredServeAttestation:
    //   add field:
    repeated string matched_keywords = <next>;

The keyword strings are the same `Keyword` identifier used in
`KeywordWeight.Keyword` (D-4.1). Order is not significant; the set semantics
apply (duplicates are equivalent to a single match for that keyword).

**Hub submission.** When the hub submits a serve event, it includes the
exact matched-keyword set used to compute that serve's `rawScore`. Per the
sim source `wevibe-sim/ranking-fix.js:184`, this is the intersection of the
memory's keywords and the query's keyword set:

    matched_keywords = [k.Keyword for k in memory.Keywords if k.Keyword ∈ query_keywords]

Hub submission of an empty `matched_keywords` array is invalid — every
serve has at least one matched keyword by construction (`rawScore > 0`
filter at sim line 167). The chain validates `len(matched_keywords) > 0`
on serve TX submission and rejects empty sets as malformed.

**Denial parity.** A denial event is conditional on a preceding serve event.
The denial inherits the serve's `matched_keywords` — the same set used to
serve the memory is the set negated by the denial. The hub does not
re-compute matched keywords on denial submission; it references the
preceding serve. The chain enforces this by requiring the denial TX to
reference (by `nullifier` or `serve_key`) the originating serve.

**Keeper method.** A new method on the serve keeper exposes the union of
matched keywords for a memory across an epoch:

    GetMatchedKeywordsForEpoch(ctx, orgID, cidHex, epoch) (map[string]bool, error)

The map keys are keyword strings; the value is always `true`. (The map
type is chosen for O(1) per-keyword lookup inside `applyDecay`'s
per-keyword loop.) The method aggregates over all serve attestations for
this memory in this epoch and returns the union.

If a memory has had no serves this epoch, the method returns an empty
non-nil map. `applyDecay` treats an empty map as the "no matched keywords"
case — every keyword takes the `!matched` branch, which is the pure
idle-decay path.

**Memory keeper integration.** `x/memory/keeper/lifecycle.go` `applyDecay`
consumes `GetMatchedKeywordsForEpoch` via the serve keeper interface
declared in `x/memory/types/expected_keepers.go`:

    type ServeKeeperExpected interface {
        GetMemoryServeCountForEpoch(ctx, orgID, cidHex, epoch) (uint64, error)
        GetMemoryDenialCountForEpoch(ctx, orgID, cidHex, epoch) (uint64, error)
        GetMatchedKeywordsForEpoch(ctx, orgID, cidHex, epoch) (map[string]bool, error)
    }

The `kwIdsMatched` parameter to `applyDecay` (per the formula in this
section above) is the return value of `GetMatchedKeywordsForEpoch`.

**Cross-references.**
- D-4.1: `KeywordWeight.Keyword` is the canonical keyword identifier
  consumed by matched-keyword tracking.
- D-13.9: Pre-MVP wipe acceptable for the new `matched_keywords` fields
  and the resulting epoch-indexed state.

**Empirical contract.** This expansion preserves the 79.5pp decoupling-gap
result. The per-keyword gate is the mechanism that produces it; without
per-serve matched-keyword tracking, the gate cannot be executed faithfully.

**Implementation source.** CO-031 Rev 2 (2026-05-28) implements this in
`proto/wevibe/serve/v1/tx.proto`, `proto/wevibe/serve/v1/state.proto`,
`x/serve/keeper/msg_server.go`, `x/serve/keeper/keeper.go`,
`x/memory/types/expected_keepers.go`, and `x/memory/keeper/lifecycle.go`.
Pre-MVP wipe per D-13.9; no migration.

**Locked parameter values (binding, all governance-changeable):**

| Parameter           | Value | Source (ranking-fix.js) | Notes |
|---------------------|-------|-------------------------|-------|
| serveD              | 220   | ET_BASE:269             | Serve boost coefficient (bps) |
| denialD             | 900   | ET_BASE:269             | Denial decay coefficient (bps) |
| idleD               | 600   | ET_BASE:269             | Idle decay coefficient (bps) |
| serveFloor          | 0.4   | ET_BASE:270             | Minimum serve-boost multiplier |
| denialFloor         | 0.3   | ET_BASE:270             | Minimum denial-decay multiplier |
| idleProtect         | 0.05  | ET_BASE:270             | Trust-earned idle multiplier |
| idleUntrusted       | 1.0   | ET_BASE:270             | Untrusted idle multiplier |
| grace               | 20    | ET_BASE:269             | Chain epochs of decay protection from `memory.created_epoch` |
| trustMinServes      | 1     | ET_BASE:271             | Lifetime memory-level serves required for trustEarned gate |
| trustMaxRate        | 0.30  | ET_BASE:271             | Lifetime denial_rate ceiling for trustEarned gate |
| retrievalThreshold  | 1500  | simulate() arg:251      | bps; archive trigger threshold (= 0.15 fractional) |

All 11 are exposed as governance Params fields in `proto/wevibe/memory/v1/params.proto`.
Sim source: `ET_BASE` constant in `wevibe-sim/ranking-fix.js:267–272`.

**Cross-references.**
- D-4.1: KeywordWeight primitive (unchanged; canonical formula reads
  memory-level counters, not per-keyword).
- D-4.4: Archive trigger predicate semantics (this section is the canonical
  reference).
- D-4.6: 7-state lifecycle (archive transition fires as specified; intermediate
  states unchanged).
- D-13.9: Pre-MVP wipe acceptable for the new `serve_count_total` /
  `denial_count_total` fields.

**Implementation source.** CO-031 implements this against
`wevibe-chain/x/memory/keeper/lifecycle.go`, `x/memory/keeper/msg_server.go`,
`x/memory/types/params.go`, and `proto/wevibe/memory/v1/params.proto` (with
proto regen via Docker-pinned `make proto-gen` per R-PROTO-REGEN).

---

### D-4.3: Per-Event Gas Model

**Decision:** Each retrieval event (serve or denial) is one chain transaction. The chain applies the keyword weight changes synchronously.

**Why:** Predictable per-event gas cost is easier to reason about than batch-style amortization. Per-event commitment also makes the chain the immediate source of truth — no batching window during which hub and chain disagree.

---

### D-4.4: Archived When All Keywords Below Retrieval Threshold

**Decision:** Memories transition to `ARCHIVED` when every keyword's weight has decayed at or below the retrieval threshold (default 1500 bps per D-4.2). The `DORMANT` state is not used.

**Why:** `DORMANT` was a holdover from the per-memory confidence design that implied "recoverable if usage returns." Under Earned Trust (D-4.2), a memory whose every keyword has fallen below the retrieval threshold has comprehensively failed to demonstrate value — either by accumulating denials, by never being served, or both. Recovery is not the right path. `ARCHIVED` memories are excluded from all queries.

The threshold-based archive (vs requiring weights to hit exactly zero) is intentional: a weight just above zero is functionally useless for retrieval but adds noise to ranking. Archiving at the retrieval cutoff aligns the chain's terminal state with the retrieval engine's effective availability.

**Predicate semantics (clarified 2026-05-28 via GO-005).** "Every keyword
weight ≤ retrievalThreshold" means **ALL** keywords, not ANY. The predicate
in Go is functionally:

    allBelow := true
    for _, kw := range memory.Keywords {
      if parseWeight(kw.Weight).GT(retrievalThreshold) {
        allBelow = false
        break
      }
    }
    if allBelow { memory.State = MEMORY_STATE_ARCHIVED }

Equivalent to JavaScript `memory.Keywords.every(k => k.weight <=
retrievalThreshold)`. Sim source: `wevibe-sim/ranking-fix.js:198`. Worker
Option B from GO-005 confirmed by Walter on 2026-05-28.

See D-4.2 Implementation Clarifications for the full archive transition
context (end-of-epoch placement, terminal nature, `archived_epoch` audit
field).

---

### D-4.5: Per-Retrieval Chain TX (Gas Model)

**Decision:** Each retrieval event (serve or denial) is one chain transaction. The chain applies the keyword weight changes synchronously.

**Why:** Predictable per-event gas is preferable to batching for three reasons:

1. **Serves were already chain TXs in the prior design.** Only denials add new TX volume. Switching to batching would introduce additional state complexity (tracking pending serve events, batch windows, rollbacks) for marginal gas savings that don't apply to the majority of retrieval events.

2. **Immediate source of truth.** Per-event commitment makes the chain the immediate source of truth — no batching window during which hub and chain disagree on memory health. This matters for orgs making real-time decisions based on memory health.

3. **Predictability for org budget planning.** The cost formula is transparent: `(daily_serves × serve_gas) + (daily_denials × denial_gas) = daily_chain_cost`. Orgs can project costs accurately. Batching would introduce variable batch sizes that make cost estimation harder and create periods where hub state diverges from chain state.

The org cost estimation formula: `(daily_serves × serve_gas) + (daily_denials × denial_gas) = daily_chain_cost`. Predictability for org budget planning outweighs marginal savings of batching.

---

### D-4.6: Lifecycle State Simplification (7-State Set)

**Decision:** The memory lifecycle state set is reduced to 7 states: `pending`, `pending_keyword`, `pending_chain`, `denied`, `committed`, `archived`, `reported_deleted`. DORMANT, DEGRADED, and STABLE are not used.

**Why:** With `confidence_bps` removed (D-4.1), intermediate health states have no signal to track. A memory is either committed and live, archived (all keyword weights below the retrieval threshold per D-4.4), or removed via report. Pre-commit states (`pending`, `pending_keyword`, `pending_chain`, `denied`) capture the moderation pipeline. The three removed states described gradient health that no longer exists in the model — keeping them around would be theater. Simpler state machines have fewer bugs.

The 7 states map to two locations:
- **Hub PostgreSQL only (pre-commit):** `pending`, `pending_keyword`, `pending_chain`, `denied`
- **Hub PostgreSQL + Qdrant (post-commit):** `committed`, `archived`, `reported_deleted`

Only `committed` memories appear in Qdrant and are returned during retrieval.

---

### D-4.7: Epoch Terminology Split (Rotation vs Chain)

**Decision:** "Epoch" has two distinct meanings in the WeVibe system; the documentation disambiguates them as **rotation epoch** and **chain epoch**.

- **Rotation epoch:** PRE crypto rotation unit. Tied to member removal events. No fixed duration. Stored as monotonic counter per org. Generated via Umbral sidecar; produces new epoch keypair and new kfrags for remaining members.
- **Chain epoch:** Time-based window for emissions, idle decay, and bootstrap grace. Fixed duration (configured at chain initialization in `x/epochs` module; queryable via `wevibed query epochs epoch-info wevibe_epoch`).

**Why:** Conflating event-driven crypto rotation with time-based emission/decay epochs caused real confusion (manager-architect conversation, 2026-05-18). The two are independent — an org may rotate frequently while chain epochs tick steadily, or vice versa. An org with no member removals has 0 rotation epochs but hundreds of chain epochs. Disambiguation prevents future readers from making wrong assumptions about durations or triggers.

The bootstrap grace period (D-4.2) is measured in **chain epochs**, not rotation epochs. With a 12-hour chain epoch, the 20-epoch grace ≈ 10 days. With a 24-hour chain epoch, ≈ 20 days. The actual duration depends on the chain's epoch configuration at initialization.

---

## 5. Memory Types & Extraction

### D-5.1: Single Memory Type with Internal Do/DND Distinction

**Decision:** Memories carry one type: `memory` (the only type). Within each memory, two fields quantify the content:
- `implement` (required) — what TO do and how, describes the correct pattern
- `dnd` (optional, can be null) — what NOT to do and why

The `dnd` field being null or non-null is the signal used for the consumer content filter (D-11.5). Memories with a non-null `dnd` field are DNDs (do-not-do warnings / negative signals); memories with a null `dnd` field are pure implementations.

**Why single type with internal DND flag (replaces D-5.1 "Two Memory Types Only"):**

The previous design had two memory types at the protocol level: `correct_implementation` and `negative_signal`. This was wrong — it created a false dichotomy where every memory had to be classified as one or the other, when in reality a single memory can contain both positive guidance (what to do) and negative warnings (what not to do).

1. **Knowledge is not mutually exclusive.** A memory about PostgreSQL connection pooling naturally contains both "set `max_connections` to match your PgBouncer pool size" (implement) AND "do not set `max_connections` above 100 on t2.micro or you will exhaust RAM" (dnd). Forcing this into a single type forced contributors to choose which aspect to emphasize, discarding the other.

2. **The content filter works at the field level, not the type level.** The consumer's content filter (`[DNDs only]` vs `[Implementations + DNDs]`, D-11.5) filters on the presence of a non-null `dnd` field — a more granular and semantically correct filter than binary type membership. A memory can be both; the consumer chooses which aspect to see.

3. **Extraction produces cleaner output.** When the LLM knows it should emit both `implement` and `dnd` as independent fields within a single memory, it produces more complete guidance. The old model forced a binary choice that diluted both types.

4. **Moderation is simpler.** Moderators review one type (`memory`) rather than classifying into one of two types. The `dnd` field's presence is a signal, not a type classification decision.

5. **Chain state is simpler.** One `MemoryType` enum with one variant. No type proliferation risk. The `dnd` field lives inside the encrypted blob — the chain stores the blob, not its internal field structure.

**Memory content schema (inside encrypted blob):**
```json
{
  "implement": "string (required) — what to do and how",
  "dnd": "string (optional, can be null) — what not to do and why",
  "context": "string — environment, versions, conditions",
  "stack": ["technology1", "technology2"],
  "preference_confidence": 0.0-1.0
}
```

**Risk appetite filtering logic:**
- `lowest` → only memories where `dnd !== null` (DND warnings only)
- `neutral` → all memories regardless of `dnd` value

---

### D-5.2: Preference Detection Is a Flag, Not a Filter

**Decision:** Extracted memories carry a `preference_confidence` score (0.0–1.0):
- 0.0: Clearly factual/verifiable
- 0.2–0.3: Organizational convention stated as fact (valid memory)
- 0.5: Ambiguous
- 0.7–0.8: Likely preference
- 1.0: Pure preference

The moderator sees the flag and decides whether to approve. The system does not auto-reject high-preference memories.

**Why:** Auto-filtering would discard valid memories that happen to phrase organizational conventions as facts (e.g., "we use Postgres for our user store" reads as preference but is operationally correct). A continuous scale lets moderators calibrate per-org and provides leaders with aggregate stats for extraction quality assessment.

---

### D-5.3: Contributors Do Not Handle Keywords

**Decision:** Keywords are assigned during moderation (approval time) by moderators selecting from the org's existing vocabulary. Contributors submit raw insights only.

**Why:** Letting contributors freely assign keywords causes vocabulary drift — duplicates, synonyms, typos, and personal terminology accumulate. Centralizing keyword assignment with moderators ensures the vocabulary stays curated and consistent. New keywords proposed during extraction require moderator/leader approval before joining the vocabulary.

---

### D-5.4: Vocabulary-Constrained Keyword Extraction

**Decision:** The LLM extraction pass selects keywords from the org's existing vocabulary. New keyword suggestions are flagged for moderator/leader approval.

**Why:**
- Prevents keyword drift over time
- Forces the org to deliberately expand its taxonomy
- Keeps retrieval matching predictable (the query side uses the same vocabulary)
- Per-memory keyword weights must sum to 1.0 (probability distribution), enforcing meaningful keyword selection rather than tag-everything

---

### D-5.4a: Keyword Scores Sum to 1.0 — Probability Distribution, Not Independent Weights

**Decision:** Per-memory keyword weights are normalized to sum to 1.0, forming a probability distribution over the memory's topics. Each keyword's weight represents the fraction of the memory's semantic identity attributable to that keyword.

**Why:** CO-236 originally specified independent 0.0–1.0 weights per keyword. Sprint 24 changed this to scores summing to 1.0. The probability distribution semantics are critical:

1. **Ranking semantics.** A probability distribution lets the retrieval ranker compute weighted similarity properly: `(query_keyword_weight × memory_keyword_weight)` products only make sense when both are normalized. Independent weights inflated memories that tagged-everything with trivial weights — a memory with 50 keywords at 0.02 each would dominate retrieval despite having no clear identity.

2. **Forces the LLM (and moderator) to choose what the memory is REALLY about.** With independent weights, the trivial approach is "tag everything that might be relevant." With a probability distribution, the LLM must allocate finite semantic mass — which forces genuine prioritization. This is the same reason probability distributions are used in information retrieval (TF-IDF, BM25) rather than raw term frequency counts.

3. **Implication: max ~20 keywords with meaningful weights, not 50+ tags with trivial weights.** The 1.0 sum cap naturally limits the number of effective keywords. A memory with 20 keywords at 0.05 each has a clear identity; a memory with 100 keywords at 0.01 each does not. Extraction and moderation should enforce this ceiling.

---

### D-5.5: Hub-Side Unicode Sanitization (Flag, Not Filter)

**Decision:** At submission time (before encryption), the hub scans memory content for suspicious Unicode:
- Invisible characters (zero-width chars, bidi overrides)
- Control characters
- Homoglyphs (Cyrillic/Greek look-alikes for Latin)
- Zalgo text (excessive combining marks)

Findings are stored on the submission record as `sanitization_findings`. The scanner does NOT auto-reject — the moderator decides.

**Why:** Character-level attacks (homoglyph spoofing, invisible-char injection) are real threats but have false-positive risk on legitimate multilingual content. Auto-rejection of legitimate multilingual content is worse than flagging for moderator review — false positives would suppress valid non-English content from orgs that work across languages. Flagging without rejecting preserves moderator authority while ensuring no Unicode-based attack gets through unnoticed. The Go-native scanner runs sub-millisecond with no external dependencies.

---

### D-5.6: Extraction Prompt Unified via wevibe-mcp

**Decision:** wevibe-mcp's `extraction.ts` is the canonical extraction engine. The dashboard's `/api/extract` route proxies to wevibe-mcp via dashboard-server.

**Why:** Previously the extraction prompt existed in two places (dashboard + wevibe-mcp), creating drift risk. Centralizing on wevibe-mcp:
- One prompt, one source of truth
- Consistent extraction quality across all submission paths
- Avoids the dashboard needing to bundle LLM client code

---

### D-5.7: Contribution Is Manual and User-Controlled (Privacy Is the Default)

**Decision:** Memory contribution is **fully manual and dashboard-driven**. Coding sessions are captured/stored **locally only**; **nothing leaves the contributor's machine without an explicit submit action**. The flow: dashboard `/sessions` → select a session → click **"Extract Memories"** (local extraction via wevibe-mcp `extraction.ts`, D-5.6, returns candidates for review — does NOT submit) → review candidates → choose a per-memory org destination (D-12.2) → click **"Submit"**. `/my-submissions` tracks status only.

There is **NO automatic mining**, **NO session buffer that auto-processes**, **NO `autoContribute()`**, and **NO agent-invoked `wevibe_contribute` tool**. Privacy is the default, not a toggle.

**Why:** The contributor must control exactly what knowledge leaves their machine. Automatic/background contribution ("sessions mined automatically", "runs invisibly", `autoContribute()` on session exit) is a **pre-pivot model that is false and is removed** — it inverts the core guarantee of the social-first product. Explicit per-memory review + routing is also what makes multi-org submission (D-12.2) and contributor accountability coherent.

**Code reality (verified 2026-06-02):** the contribution path is already user-driven in code (no `autoContribute`, no session buffer, no agent contribute tool; only an explicit dashboard submit sends off-machine). **Remnants to remove (code, matrix-gated):** the plugin sets `WEVIBE_AUTO_CONTRIBUTE=1` (DEAD — nothing reads it) + documents it in its README; and a dead `auto-contribute-consent` stub in `extraction.ts` is never consumed. Both must be deleted (R-LONGEVITY / R-ONE-PATH).

---

### D-5.8: Recall — Client Auto-Queries, Injection Is Gated

**Decision:** Recall is split: the **query side is automatic**, the **injection side is gated**.

1. **Auto-query (no user/agent action).** The local client harvests session signals to build the retrieval query parameters; the plugin auto-submits the query. There is **NO agent-invoked `wevibe_recall` tool** — the agent does not "pull." The MCP builds the query (keyword-weight extraction + a **local** Ollama embedding, model `nomic-embed-text`, 768-dim) and POSTs `{ org_id, agent_pubkey, keyword_weights[], vector[], embedding_model_id, limit, pre_pubkey }` to the hub `POST /v1/orgs/{org}/query`.
2. **Hub retrieval.** Qdrant vector search + keyword-boost ranking + pending-denial decay + new-memory boost + power-law sampling. The hub returns **encrypted** candidates (capsule/cfrag/umbral_ciphertext) + trust signals; the MCP fetches ciphertext and **decrypts locally**.
3. **Gated injection.** Candidates are presented to the consumer with the memory **plus details** before injection: memory text, memory_type, score, matched keywords, wevibe-guard scan result, contributor stats (account age, contributions, serve count, reports upheld, false reports against), memory stats (retrieval/acceptance counts), trust panel. Per the consumer **injection-gate** setting (D-11.5): `[Gated approval]` (DEFAULT) pauses the agent for explicit approval; `[No gated approval]` injects directly. The serve is attested on-chain at injection.

**Content/gate settings:** governed by the 2×2 consumer matrix (D-11.5). Recall low-latency by design.

**Intended vs current — logged gap `GAP-RECALL-HARVEST`:** the **intended** model harvests **live session signals** (recent prompt, edited files, stack) to build the query each time it's needed. **Current code** builds the query only from **project metadata at startup** and fires **once** (the plugin captures `lastPrompt` + edited-file paths via hooks but does NOT use them in the query); `agent_sig` is a literal `"stub"`; and two divergent gate implementations exist (`wevibe-mcp` `server.ts` MCP-elicitation path vs `http-server.ts` `/v1/recall` queue path used by the plugin). These are alpha under-builds tracked for code cleanup (matrix-gated). Forward docs describe the **intended** model, framed as alpha.

**Why:** recall must be frictionless (auto-query, no tool ceremony) but safe (gated injection by default). The agent never silently ingests unreviewed memory unless the consumer opts into `[No gated approval]`.

---

## 6. Moderation & Approval

### D-6.1: Batch Pipeline — Approval Decoupled from Keyword Extraction Decoupled from Chain Commit

**Decision:** Three independent stages, each gated:
1. **Approval** (moderator) — status `pending` → `pending_keyword`. Pure quality stamp.
2. **Keyword extraction** (leader-triggered batch) — status `pending_keyword` → `pending_chain`. LLM extracts keywords from org vocabulary; leader reviews/edits.
3. **Chain commitment** (leader, multi-message TX) — status `pending_chain` → `committed`. Memories enter Qdrant at this point.

**Why:** The three-stage pipeline isolates concerns that have fundamentally different operational profiles:

1. **Batch efficiency.** LLM extraction is expensive — running it per-memory at approval time would saturate GPU/API budgets immediately. Batching dozens of pending memories into a single extraction pass dramatically reduces per-memory cost. Approval must be fast and per-memory; extraction benefits from economies of scale.

2. **Leader control.** The leader is the one held accountable for what enters the chain. They should review extracted keywords before commitment — editing weights, removing low-quality suggestions, or rejecting memories that extraction mangled. This review step is impossible if keywords are committed inline at approval time.

3. **Verification gate.** Hub-side checks (keyword format validation, keyword count limits, weight sum verification) need a deliberate gate before chain submission. Inline approval cannot accommodate this verification step without blocking the moderator. The batch pipeline makes this a deliberate handoff, not an implicit side effect.

4. **Gas optimization.** Multi-message Cosmos TX dramatically reduces per-memory fee at scale. Submitting 50 memories in one multi-message TX costs roughly the same as submitting 2-3 individual TXs. Per-memory approval-time chain submission would be prohibitively expensive — each TX has fixed overhead (byteize, signature verification) that becomes the dominant cost at small batch sizes.

**Moderator accountability preserved.** Each approved memory carries the moderator's pubkey through to chain commit (D-6.4).

---

### D-6.2: Qdrant Insert at Chain Commit, Not Approval

**Decision:** Memories are added to Qdrant only after chain commitment, not at moderator approval.

**Why:** The chain is authoritative. If a memory exists in Qdrant before chain commit and the commit fails (or is delayed), retrieval would return memories that aren't actually committed on-chain — violating the "chain is source of truth" property. Inserting after chain commit ensures Qdrant only contains memories that are immutably committed.

---

### D-6.3: Quorum Voting Is Hub-Enforced

**Decision:** `required_approvals` (1–10) is stored on the hub and enforced by hub logic. The chain only sees the final approved memory commitment.

**Why:** Operational decisions like quorum thresholds change as orgs scale. Storing them on-chain would require governance transactions for every adjustment. The hub enforces the threshold; the chain records the outcome (approved memory + moderator pubkey). The chain doesn't validate the vote count — it trusts the **leader's on-chain transaction signature** as the approval authority (the leader wallet is the chain authority per D-1.3 and D-S32-CO044-KEY-SEPARATION, submitted via the delegate/authz relay), NOT the hub's key. (Pre-CO-044 dogfood, the hub key signed approvals directly; that global-authority shortcut is removed by D-S32-CO044-KEY-SEPARATION.)

Note: Changes to `required_approvals` require wallet signature (D-1.3) because the threshold is security-critical.

---

### D-6.4: Moderator Accountability Chain

**Decision:** The moderator's pubkey is recorded on every approved submission and carried through every subsequent state transition (`pending_keyword → pending_chain → committed`) and onto the final on-chain memory commitment.

**Why:** Approval is the moment a moderator's judgment becomes part of the org's knowledge base. Tying their identity to the memory permanently makes moderation a reputation-bearing activity. Moderators cannot hide behind anonymous approvals — every memory they approve carries their cryptographic signature on-chain.

Combined with D-7.6 (sword cuts both ways), this means moderators carry the same accountability burden as contributors and reporters:
- Contributors are accountable for the memories they submit
- Reporters are accountable for the reports they file (dismissed count, malicious dismissal consequences)
- Moderators are accountable for the approvals they sign

The moderator pubkey is embedded in the `pending_keyword` status entry and propagates through the batch pipeline to chain commit, ensuring the accountability chain is unbroken from approval to on-chain permanence.

---

### D-6.5: Undo-Approve Replaces memory_type_override

**Decision:** Memory type is set at extraction and stands. The `memory_type_override` field at approval time is removed entirely. To preserve moderator correction capability without re-inserting dual-classification paths, an **undo-approve** action reverts a memory from `pending_keyword` back to `pending`, with server-enforced race condition prevention.

**Undo-approve rules:**
- Valid ONLY while status is `pending_keyword`
- Once the leader has moved the memory to `pending_chain` via batch submission, undo is no longer available — the leader must use "remove from batch" instead
- Handler performs atomic `UPDATE ... WHERE status = 'pending_keyword'`; if status has already moved, zero rows are updated and a 409 is returned
- Audit log entry recorded for every undo

**Why:** Override added dual classification paths (extracted type + moderator override), introduced race conditions (what if leader started batch extraction during override?), and expanded UI surface for marginal benefit. The cleaner solution: if the moderator wants to reclassify, they undo the approval and the memory re-enters extraction with the corrected type. The race condition is prevented at the database level — if the leader has already moved the memory to `pending_chain`, undo returns 409 and the leader must use "remove from batch" instead. Server-side enforcement means no client coordination is needed.

---

### D-6.6: MaxBatchMemories = 500

**Decision:** Hub-side constant `MaxBatchMemories = 500` enforced when leaders trigger batch chain submission. Exceeding the cap returns a clear 400 error; the leader must split the batch.

**Why:** Chain block has 21MB max_bytes (no gas limit; block size is binding). Each memory's batch messages (`MsgSubmitCommitment` + `MsgApproveMemory` + `MsgIncrementContribution`) total ~3-5 KB. With 20% safety margin (block headers, evidence, other TXs), usable space is ~16.8 MB → theoretical max ~4,200 memories. 500 leaves massive headroom AND gives leaders a clean operational number to plan around. Capping below theoretical max means orgs can't accidentally produce blocks so large they choke validators. Future increase is a hub config change, no chain consensus change required.

---

### D-6.7: Moderator Quorum Is the Leader's Choice (Two Approval Models)

**Decision:** Every org operates under one of two approval models, configurable per-org by the leader:

- **Model A — Leader as sole approver.** No moderators appointed. `required_approvals = 1`. The leader reviews and approves all memories directly. This is the default for new orgs.
- **Model B — Leader + moderator quorum.** Leader appoints moderators (`/members`) and configures `required_approvals` (1-10, via `/settings`). Moderators cast approval votes; once the threshold is met, the memory transitions to `pending_keyword`.

In both models, the **leader is the on-chain TX signer** for batch chain commits (D-1.3). Moderator approvals are operational signals visible to the leader, not chain-level authority.

**Why both models exist:**

- **Solo-led and small expert orgs benefit from Model A.** A leader with deep domain expertise reviewing every memory themselves is the simplest, fastest, highest-quality path. Forcing a moderator layer on small orgs adds friction without benefit.
- **Larger orgs and orgs with multiple contributors benefit from Model B.** When the leader cannot personally review every submission, delegating to trusted moderators distributes the review load. The leader still controls who is a moderator and still signs chain commits.
- **The leader can always override.** Even in Model B, the leader has all moderator capabilities. They can approve or reject any submission directly without waiting for quorum. Moderators are advisory; the leader is sovereign.

**Why the leader is always the on-chain TX signer:**

- **Single-actor accountability for chain state.** The leader's wallet signature is the public attribution for every on-chain commit. Adding moderator signatures to the chain TX would either (a) require chain-side validation of vote process (which violates D-6.3 — operational decisions stay off-chain) or (b) create co-attestation that implicates moderators in commits they didn't directly sign for.
- **Moderator accountability remains org-local.** Moderator vote history is tracked in the hub's `vote_records` table. Orgs that want to surface moderator accountability can do so via their own UX. The chain does not need to enforce it.
- **Consistent with D-1.3.** Security-critical operations (chain commits, leadership transfer, org closure) require the wallet signature. Approval model configuration also requires wallet signature for changes to `required_approvals`.

**Why no auto-quorum-from-reputation:**

- Auto-quorum (e.g., "any 3 moderators with rep > X can approve") was considered and deferred. It introduces a runtime authorization model where the leader's discretion is mediated by chain state. The current model — leader explicitly appoints moderators and sets the threshold — is simpler and gives leaders direct control.

**Implementation status:** Already implemented. `required_approvals` is stored on the org config, defaults to 1, and is configurable from `/settings`. The leader-as-sole-approver path works today. This decision documents the canonical UX framing rather than introducing new behavior.

**Reference:** MASTER.md §2 Leader — UX Flow: Memory Approval Model — Leader's Choice (added by DMO-027).

---

## 7. Reports & Accountability

### D-7.1: Reports Do NOT Auto-Blacklist

**Decision:** When a consumer reports a memory, it continues to appear in their recall queue (and others') until moderator resolution. Only **denial** triggers local blacklisting.

**Why:** A denial is a personal preference ("I don't want to see this"). A report is a claim requiring investigation ("this needs to be reviewed"). If reports auto-blacklisted, a single consumer could remove memories from their own view, bypassing the quorum process. The blacklist is for personal preference; the report queue is for community moderation.

---

### D-7.2: Reporter Identity Always Linked

**Decision:** Every report carries the reporter's pubkey, wallet address, and signature. Moderators see this when reviewing reports.

**Why:**
- **Deters mass-reporting attacks.** Every report is tied to a real wallet with reputation history.
- **Gives moderators context.** A report from a long-standing contributor with zero dismissed reports carries more weight than one from a new member with 5 dismissed reports.
- **Same trust-through-transparency principle as the contributor social graph.**

---

### D-7.3: Trial Members Cannot Report

**Decision:** Only paid members can submit reports. Trial members see the Report button greyed out in the plugin.

**Why:** Trial accounts are free and disposable. Without economic stake, there is no deterrent to report spam. Paid membership (with wallet-linked identity and dismissed-count tracking) provides the accountability layer.

---

### D-7.4: Report Quorum Is Hub-Enforced, Chain Records Outcome

**Decision:** `report_vote_threshold` (1–10, default 1) is stored on the hub and enforced by hub logic. The chain only records the final upheld report commitment (`MsgReportMemory`).

**Why:** Same rationale as D-6.3 (approval quorum). Operational thresholds change frequently; chain governance is too heavy for every adjustment. The chain records the outcome.

Note: Changes to `report_vote_threshold` require wallet signature (D-1.3).

---

### D-7.5: Upheld Reports Require Leader Wallet TX (Two-Step Commit)

**Decision:** When report votes meet the threshold, the report enters intermediate status `upheld_pending_tx`. The leader must explicitly sign `MsgReportMemory` with their wallet (not delegate key) to commit it on-chain. Only after the chain TX confirms does the hub delete the memory from Qdrant.

**Why:**
- **The on-chain commitment is permanent and public.** The contributor's wallet is forever linked to the deleted memory in the chain record. Only the org leader — authenticated via wallet signature — can trigger this consequential action.
- **Atomic with chain TX.** If the chain TX fails, the report remains `upheld_pending_tx` and the memory stays in Qdrant. Nothing changes until the TX confirms.
- **Intermediate status separates quorum from commitment.** Quorum is reached automatically when votes accumulate; commitment is a deliberate leader action. This gives the leader a final review opportunity.

---

### D-7.6: The Sword Cuts Both Ways

**Decision:** Accountability is symmetric:
- **Upheld report:** Contributor's wallet linked on-chain to the deleted memory record. Public record on their social graph.
- **Dismissed report:** Reporter's `dismissed_reports_count` incremented on their member record.
- **Dismissed as malicious:** Leader can remove the reporter from the org (their stake is kept by the org).

**Why:** A one-sided accountability system invites abuse. By making both reporting and being-reported carry consequences, the system disincentivizes both bad memories and bad reports. Each org decides independently how to act on these signals — other orgs see the social graph but make their own judgments.

---

## 8. Emissions & Payouts

### D-8.1: Pay Per Approved Memory, Not Per Serve

**Decision:** Contributor payouts are calculated by counting **approved memories** per contributor per epoch, not serve counts.

**Why:**
- **Serves are gameable** (a malicious consumer could spam retrievals to inflate a contributor's payout).
- **Approved memories represent durable value.** A memory that passes moderation is what the org actually wants to reward.
- **Tier caps prevent dominance.** `MaxContributionsPerEpoch` caps the per-contributor count at each tier; overflow rolls to the next tier, so prolific contributors don't sweep all rewards.

---

### D-8.2: Qualification Gate via `min_contributions_per_epoch`

**Decision:** Orgs can set `min_contributions_per_epoch` — contributors must have at least this many approved memories in the epoch to qualify for payout. Default = 0 (any contribution qualifies).

**Why:** Some orgs want to reward only sustained contribution, not one-off submissions. The default of 0 keeps the system open; leaders who want a higher bar can set it explicitly.

---

### D-8.3: `payout_per_memory`, Not `payout_per_serve`

**Decision:** Renamed from `payout_per_serve` to `payout_per_memory` to reflect the actual payout basis (D-8.1).

**Why:** Naming should reflect semantics. The old name implied serves were the payout unit, leading to confusion. The rename eliminates the serve-count assumption.

---

## 9. Qdrant & Vector Search

### D-9.1: Qdrant Stores Chain-Authoritative Data

**Decision:** Qdrant payload mirrors chain state:
- Vector (768-dim, cosine)
- Keywords (restored after a brief misstep that stripped them)
- Per-keyword weights
- Lifecycle state

**Why:**
- **Keywords are needed for retrieval matching.** Stripping them broke the keyword-boost ranking.
- **Lifecycle state enables filtering.** ARCHIVED memories are excluded from all queries; DORMANT (now unused, see D-4.4) was excluded by default.
- **Mirroring chain state** ensures retrieval ranking uses the same data the chain validates.

The earlier decision to strip keywords from Qdrant for "security" was architecturally incorrect — the threat (Qdrant compromise) is better addressed via API key auth + noise injection (D-9.2), not by removing the data that makes retrieval work.

---

### D-9.2: Phase 1 Qdrant Hardening — Noise Injection + API Key Auth [NOISE INJECTION SUPERSEDED by D-9.5]

> **[SUPERSEDED IN PART by D-9.5, 2026-06-01]** Stored-vector Gaussian noise injection is **DISABLED** (default σ=0; `retrieval.go:221`). It was inherited Echo code with no rationale and cost ~20pp good-memory recall; D-9.5 removed it. The API-key auth and internal-network mitigations below REMAIN ACTIVE.

**Decision:** Phase 1 mitigations against Qdrant embedding inversion attacks:
- **Gaussian noise injection (σ=0.1)** at storage time — ~~reduces inversion accuracy ~40% while preserving >95% recall~~ **[DISABLED — see D-9.5; default σ=0]**
- **Qdrant API key authentication** — required on all requests, rotated per epoch
- **Internal-network deployment** — Qdrant on same host as hub, no external network exposure

**Why:** Published research (Morris et al. 2023, Huang et al. ACL 2024) demonstrates embedding inversion is real. Phase 1 mitigations are stopgaps until encrypted vector search (e.g., IronCore `ironcore-alloy`) is evaluated and integrated.

---

### D-9.3: Enriched Query Ranking

**Decision:** Qdrant queries combine multiple ranking signals:
- Vector similarity (cosine)
- Lifecycle filter (exclude ARCHIVED always)
- Keyword overlap boost (overlap × 0.1)
- Per-keyword weight ranking (memories with stronger weights on matching keywords rank higher)

Position assignment within the result set is governed by D-9.4 (probabilistic exploration).

**Why:** Raw vector similarity isn't enough — keyword weights add health-aware ranking, ensuring decayed memories rank lower than well-validated ones for the same query.

---

### D-9.4: Probabilistic Exploration in Result Positions 2..N

**Decision:** The retrieval engine assigns result positions in two phases:

1. **Position 1** — strictly the highest-scoring candidate. Deterministic. This is the system's strongest answer; predictability matters for top-1 quality.
2. **Positions 2..N** — sampled probabilistically from remaining candidates using softmax weighting: `weight_i = (score_i / max_score)^(1 / temperature)`. Default `temperature = 0.7`.

The candidate pool is the standard D-9.3 ranked output (vector + keyword + weight). The change is *how* positions below #1 are filled.

**Why:**

Pure deterministic top-N ranking creates a **ranking-loss death spiral** that idle-decay cannot fix:

1. Two memories match a query. One has slightly higher weight. It wins, gets served, gets a serve boost, increases its lead.
2. The loser does not get served, does not get a boost, gets idle-decayed (per D-4.2 untrusted path).
3. Next query touching the same keywords: same outcome.
4. Eventually the loser archives, even though it was a perfectly good memory — it just lost the first ranking battle.

This is **independent of the decay formula**. Even with Earned Trust (D-4.2), 23% of good memories archive falsely in sparse-query orgs (Cold Storage scenario: 200 memories, 8 queries/epoch) because the deterministic ranking deprives them of the serves they need to earn the trust gate.

Probabilistic exploration in positions 2..N solves this without compromising top-1 quality:

- The user always gets the system's best answer at position 1
- Lower-ranked memories get occasional chances to be served
- Memories that are actually good win those chances and accumulate trust
- Memories that are not good lose serves to denial rate and archive

**Empirical results (Cold Storage scenario):**

| Engine | Good Memory False-Archive Rate |
|---|---|
| Deterministic top-N + D-4.2 (current chain) | 60.5% |
| Deterministic top-N + Earned Trust (D-4.2 new) | 23.6% |
| **Probabilistic positions 2..N + Earned Trust** | **16.4%** |

Across all scenarios, probabilistic exploration lifts worst-case decoupling gap from 61.8pp to 68.2pp without lowering average gap.

**Why this is a hub-side change, not a chain change:**

The chain produces the authoritative keyword weights (D-4.2). The retrieval engine consumes those weights and produces a ranked result set. The position-assignment policy is purely a query-engine concern — the chain has no visibility into which position a memory was served in. No new chain state, no new transactions, no consensus implications.

**Implementation lives in `wevibe-hub/internal/retrieval`** (Qdrant query handler). Temperature is a hub config parameter, not a chain parameter.

**Why not just adjust α/β scoring weights?**

Adjusting scoring weights (Section 3.5: `γ × keyword_boost`, `δ × vector_score`) changes how candidates are ordered but the order is still deterministic. Same memory wins every time. The death spiral persists. Probabilistic sampling is what breaks the loop.

**Optional companion: new-memory score boost.** For memories with age < `grace + boostWindow` chain epochs, multiply the score by `(1 + boostMult × (1 - age/boostWindow))`. Default: `boostMult = 0.5`, `boostWindow = 30 epochs`. This is a stabilizer that helps new memories cross the trust threshold in growth-heavy orgs. Empirically smaller effect than the probabilistic sampling itself but composes cleanly.

#### D-9.4 — Implementation Clarifications (2026-05-28 GO-005 synthesis)

The original D-9.4 entry described the position-2..N sampler as
"softmax-sampled (temperature 0.7)." GO-005 sim-semantics extraction
(2026-05-28) confirmed that the empirical 79.5pp decoupling-gap result was
produced by a **tempered power-law sampler**, not a softmax. There is no
`Math.exp` in `wevibe-sim/ranking-fix.js`. This subsection codifies the
actual mechanism. Where the original D-9.4 prose and this subsection
disagree, this subsection wins.

**Source of truth.** `wevibe-sim/ranking-fix.js`, functions
`probabilisticRanking` (lines 93–111), `weightedSampleWithoutReplacement`
(lines 73–90), `queryScore` (lines 58–71), `rawScore` (lines 51–55).
Implementation must reproduce sim behavior.

**Query-time score (per candidate memory M, query keyword set Q, epoch e):**

    rawScore = sum over k in M.Keywords where k.id ∈ Q of k.weight

    if rawScore == 0:
      candidate dropped (queryScore = 0, filtered out)

    if new-memory boost is enabled AND (e - M.created_epoch) < (grace + boostWindow):
      age      = e - M.created_epoch
      window   = grace + boostWindow                        // canonical: 20 + 30 = 50
      boost    = max(0, 1 - age / window)                   // linear: age=0 → 1.0, age=window → 0
      queryScore = rawScore * (1 + boostMult * boost)       // peak at age 0: × (1 + boostMult) = ×1.5
    else:
      queryScore = rawScore

The new-memory boost decays linearly from `age=0` (not from `age=grace`).
The total boost window equals `grace + boostWindow`. Initial peak multiplier
at age 0 is `1 + boostMult = 1.5` for canonical parameters. A 25-epoch-old
memory receives `1 + 0.5 * (1 - 25/50) = 1.25` multiplier. A memory aged
`grace + boostWindow` or older receives no boost.

**Position assignment:**

    sorted = candidates sorted by queryScore descending
    if servePer == 1 OR sorted.length == 1:
      return sorted[:servePer]

    position 1 = sorted[0]                                  // STRICT argmax, never sampled
    rest       = sorted[1:]
    maxS       = rest[0].queryScore

    for each candidate i in rest:
      weighted_i = (rest_i.queryScore / maxS) ^ (1 / max(0.01, temperature))

    sampled    = weighted_sample_without_replacement(weighted, servePer - 1)
    finalRest  = rest filtered to sampled members, preserving rest's original score-descending order

    return [position1, ...finalRest]

This is a **tempered power-law sampler**:

    weight_i = (score_i / score_max) ^ (1 / T)

Lower temperature → exponent grows → distribution concentrates on top
candidates. Higher temperature → exponent shrinks toward 1 → distribution
flattens. At T → 0, only the top candidate has nonzero weight (effectively
strict). At T = ∞, the distribution approaches uniform.

This is **not** softmax. Softmax computes `exp(score / T)`; the power law
computes `(score / score_max) ^ (1 / T)`. The two have different sampling
behavior for the same T. The empirical 79.5pp result was measured with the
power law; an actual softmax implementation would produce different
empirical outcomes and would not honor the sprint's empirical contract.

**Sampling is without replacement.** `weightedSampleWithoutReplacement`
recomputes the total weight after each draw (`pool.reduce((s, x) => s + x.s, 0)`
inside the per-draw loop). Sim source: `ranking-fix.js:78` inside the
`for (let i = 0; i < k && pool.length > 0; i++)` loop.

**Ordering within positions 2..N.** The sampler chooses which candidates
fill positions 2..N, but their final order within those positions preserves
the original score-descending order of `rest` — not the draw order. Sim
source: `ranking-fix.js:108–109`:

    const sampledIds = new Set(sampled.map(x => x.m.id));
    const finalRest  = rest.filter(x => sampledIds.has(x.m.id));

This means within positions 2..N, higher-score sampled candidates appear
before lower-score sampled candidates.

**Locked parameter values (hub config, env-var-changeable):**

| Parameter   | Env var                          | Value | Source (ranking-fix.js)        | Notes |
|-------------|----------------------------------|-------|--------------------------------|-------|
| temperature | RETRIEVAL_TEMPERATURE            | 0.7   | QS3b winner:296                | Power-law exponent = 1/T ≈ 1.43 |
| boostMult   | RETRIEVAL_NEW_MEM_BOOST_MULT     | 0.5   | QS3b winner:296                | Peak boost = ×(1 + 0.5) = ×1.5 at age 0 |
| boostWindow | RETRIEVAL_NEW_MEM_BOOST_WINDOW   | 30    | QS3b winner:296                | Chain epochs past `grace`; total window = grace + boostWindow = 50 |

**Determinism.** RNG must be injectable for testing. Tests inject a
fixed-seed PRNG (`rand.New(rand.NewSource(seed))` in Go); production seeds
from `time.Now().UnixNano()` (or equivalent non-deterministic source).

**Strict top-1 invariant.** Position 1 is always the argmax of queryScore.
Tests asserting on position 1 remain deterministic regardless of RNG seed.
Tests asserting on positions 2..N MUST seed the RNG.

**Hub scope (resolves GO-005 Q2.1).** The hub does NOT recompute Earned Trust at query time. The sim's `rawScore` reads pre-decayed weights stored
by the chain — `k.weight` is whatever the chain last wrote. The hub mirrors
chain-settled weights into Qdrant per D-9.1 / D-9.3 and queries them. Per
D-9.4 implementation:

- The hub sums matched keyword weights at query time (`rawScore`).
- The hub applies the new-memory boost based on `memory.created_epoch`
  (multiplicative on `rawScore`).
- The hub applies position-1 strict + power-law-sample positions 2..N.

The hub does NOT:

- Re-derive Earned Trust at query time.
- Track `m.serves` / `m.denials` in its mirror.
- Apply idle decay at query time.
- Apply any decay arithmetic at query time beyond the existing pending-denial
  optimistic ledger (`applyPendingDenialDecay` in
  `internal/retrieval/retrieval.go`), which is a separate concern not
  modeled by the sim and out of scope for D-9.4.

**Cross-references.**
- D-9.1 / D-9.2 / D-9.3: Qdrant payload, hardening, enriched ranking
  (unchanged; D-9.4 reads the same payload).
- D-4.2 + D-4.2 Implementation Clarifications: chain-side weight evolution
  that produces the values D-9.4 reads.
- D-13.9: pre-MVP state pattern (no new chain state introduced by D-9.4;
  hub-only change).

**Implementation source.** CO-032 implements this against
`wevibe-server/wevibe-hub/internal/retrieval/retrieval.go`,
`internal/config/config.go`, `cmd/wevibe-hub/main.go`, and
`wevibe-server/docker-compose.yml`. CO-032 also verifies the Qdrant payload's
"first appearance" epoch semantics (whether `epoch_id` represents creation
or approval) and, if necessary, adds a `created_epoch` payload field to
match `m.created_epoch` in the sim.

---

## 10. Operational Procedures & Recovery

### D-10.1: Epoch SK Compromise Is Accepted Risk

**Decision:** If the epoch secret key is compromised (e.g., 2-of-3 Shamir shareholders collude), the affected epoch is considered burned. The architecture does not attempt instant re-encryption of all memories.

**Why this is acceptable:**

Under PRE with proper key hygiene, the epoch SK never persists on any machine after kfrag generation:
1. Leader generates epoch keypair in memory
2. Generates kfrags for all members (seconds)
3. Shamir-splits the epoch SK into 3 shares
4. Distributes shares to leader, hub, recovery holder
5. Destroys the original epoch SK

Epoch SK compromise therefore requires either:
- **Governance-level attack** (2-of-3 shareholders collude maliciously), OR
- **Leader machine compromise during the brief kfrag generation window**

This is categorically harder than the "removed member with cached keys" attack that PRE was designed to solve.

**Why instant 1M-memory re-encryption is not feasible (or needed):**

For 1M memories, re-encryption would require 1M on-chain transactions — ~100 blocks of dedicated chain throughput plus massive gas costs. More fundamentally:

> **You cannot change the key that encrypted data without re-encrypting the data.** This is a cryptographic constraint, not an engineering limitation.

Solutions like KDF-based key derivation that promised "rotate the nonce, all keys change" were evaluated and found impossible: the existing ciphertext blobs were encrypted with the OLD content key. Changing the derivation function doesn't decrypt old data with new keys.

**What ships:**
- PRE as designed — solves the real threat (removed member exfiltration)
- Shamir 2-of-3 for disaster recovery — provides governance-gated reconstruction
- On-chain epoch status field (`active | compromised | rotated`) — immediate signal for the hub
- Operational runbook for background re-encryption (deferred; see ARCH-G8 / OQ-6 in gap log)

**What does NOT ship:**
- Instant 1M re-encryption capability — not feasible, not needed for actual threat model
- VRF-based rotation nonce — interesting for Phase 3 hardening but not blocking
- KDF-based key derivation — fundamental crypto constraint prevents this

---

### D-10.2: Recovery Path Hardened Against Operational Drift

**Decision:** The PRE recovery path has structural barriers against routine use:
- Requires 2-of-3 Shamir shareholders to cooperate
- Requires on-chain multi-sig transaction to authorize
- Recovery code path NOT in the hub binary (separate `wevibe-recover` CLI tool)
- Recovery events logged on-chain — permanently visible to all org members

**Why:** "Emergency-only cryptographic paths never stay emergency-only" is a well-known operational anti-pattern. By making the recovery procedure deliberately inconvenient and publicly auditable, the design prevents it from being adopted as a routine workaround.

---

### D-10.3: VRF for Rotation Nonce Deferred to Phase 3

**Decision:** Verifiable Random Function (VRF) for rotation nonce generation is architecturally sound but not in Phase 1/2 scope.

**Why:** VRF ensures the rotation nonce is **unpredictable** to attackers who might pre-compute re-encryption tables. This becomes relevant only if background re-encryption is operationalized at scale (Phase 3). For Phase 1/2, the existing key generation entropy is sufficient.

---

### D-10.4: Starter Packs Deferred to Sponsoring Org

**Decision:** Vocabulary starter packs ("web-backend", "mobile", "devops" templates) are not built into the WeVibe platform. New orgs start with empty vocabulary; leaders add keywords manually via the dashboard.

**Why:** Vocabulary curation is an org responsibility, not a platform responsibility. The WeVibe-sponsored org will create starter packs as community contributions when the chain is live. The dashboard's `/keywords` page already provides full CRUD for keyword management — no platform-level feature is missing.

---

## 11. Architecture Tradeoffs (Documented Acceptance)

These are tradeoffs that have been evaluated and accepted as inherent properties of the system rather than gaps to fix.

---

### D-11.1: Confidence System Creates Tension with Revocation Security

**Tradeoff:** The keyword weight system rewards retrieval frequency. High-weight memories are the most-retrieved. They are therefore the most likely to exist in plaintext on multiple endpoints. **The most valuable organizational knowledge has the weakest effective revocation.**

**Why accepted:** This is a fundamental tension in collaborative knowledge systems — it exists in Google Drive, Confluence, SharePoint, and every other shared knowledge platform. It is not unique to WeVibe. Mitigations (anomaly detection on serve patterns, additional controls on high-weight memories) can be added but cannot eliminate the underlying tradeoff.

---

### D-11.2: 1-Block Authorization Lag

**Tradeoff:** Between `MsgRemoveMember` submission and hub recognizing removal, there is a 1-block lag (≈6 seconds with CometBFT instant finality). During this window, a removed member can retrieve at their rate limit (≈0.69 retrievals).

**Why accepted:**
- Hub verifies membership on-chain via gRPC for every retrieval (not cached) — minimum possible lag
- On-chain rate limiting caps retrievals during the window
- The blast radius (≈1 memory retrieved during a 6-second window) is acceptable given the cost of eliminating it entirely

---

### D-11.3: Horizontal Scaling Attack — Graceful Degradation

**Tradeoff:** An attacker who compromises many endpoints (33+) can defeat per-endpoint rate limits.

**Why accepted:** The degradation is graceful:

| Compromised Endpoints | Pre-PRE System | PRE System |
|---------------------|----------------|------------|
| 1 (removed member) | Total epoch breach | Bounded to previously-retrieved memories |
| 1 (active member) | Total epoch breach | Rate-limited, detectable via anomaly |
| 5 (multiple) | Total epoch breach | 5× rate-limited, auditable |
| 33+ (org compromise) | Total breach | Total breach (but with audit trail) |

At the "33+ endpoints compromised" threshold, the attacker has effectively compromised the org itself — no architectural defense at the cryptographic layer can prevent this.

---

### D-11.4: No Concrete Alternative to Solution C Proposed

**Decision (informational):** During the Sprint 21 architecture debate, challenges were raised to Solution C (PRE + Hybrid Threshold Recovery) but no concrete alternative architecture was proposed that satisfies the stated goals better.

**Why documented:** Solution C remains the least-bad viable option. Future architectural challenges should propose specific alternatives with concrete tradeoffs, not abstract objections.

---

### D-11.5: Consumer Settings — Content Filter + Injection Gate (Client-Side)

**Decision:** The term **"risk appetite" is RETIRED.** The consumer has a **2×2 client-side settings matrix**, labeled by the toggles:

- **Content filter:** `[Implementations + DNDs]` | `[DNDs only]`. Filters on the presence of a non-null `dnd` field (D-5.1). `[DNDs only]` surfaces only negative-signal / do-not-do guardrails; `[Implementations + DNDs]` surfaces all memories.
- **Injection gate:** `[Gated approval]` | `[No gated approval]`. `[Gated approval]` pauses the agent for explicit user approval before a recalled memory is injected (the exclusive pop-up + details, D-5.8); `[No gated approval]` injects directly without per-memory approval.

**Default = `[Implementations + DNDs]` + `[Gated approval]`.** Both settings are enforced **client-side** (plugin + wevibe-mcp); the hub does NOT filter by consumer preference — it returns all relevant memories, and the client applies the content filter and gate after retrieval, before injection.

**Why:**
- The hub doesn't need to know consumer preferences. Hub-side filtering would couple the preference to authenticated state and force appetite metadata into every retrieval request.
- Client-side filtering makes changing a setting instant (no hub round-trip).
- `[Gated approval]` as the default keeps the human-in-the-loop invariant: the agent never silently ingests recalled memory unless the consumer explicitly opts into `[No gated approval]`.

**Code mapping (RENAME pending — code change is matrix-gated):** the content filter currently exists as `risk_appetite` (`lowest` = DNDs-only, `neutral` = both) in `wevibe-mcp/src/risk-appetite.ts` + the `set_risk_appetite` MCP tool + the plugin's `getRiskAppetite()` reading `~/.wevibe/plugin-config.json`. The gate currently exists as the `WEVIBE_ALLOW_UNREVIEWED` env flag. Both must be renamed to the toggle labels above, and **the gate DEFAULT must become `[Gated approval]`** — the plugin currently force-sets `WEVIBE_ALLOW_UNREVIEWED=1` (= No-gated), which is the WRONG default and will be corrected. The duplicate read logic across plugin + wevibe-mcp (separate runtimes) remains intentional.

---

## 12. Sprint 25 Architecture: Multi-Org, Discovery, Profiles, Activity

### D-12.1: Multi-Org Isolation — Hybrid Architecture

**Decision:** The hub uses hybrid isolation:
- Logical isolation in PostgreSQL via org_id foreign keys on every memory-related table (members, pending_submissions, kfrags, etc.)
- Physical isolation in Qdrant via per-org collections (one Qdrant collection per org, named org_{orgID}_memories)
- Authorization enforcement at every hub endpoint via middleware that verifies the requester's membership in the target org before any read/write
- The hub is open-source. Anyone can run their own hub. For now, wevibe-infra runs the shared hub instance that hosts multiple orgs.

**Why:**
- End goal: users belong to multiple orgs and receive memories from all of them. Cross-contamination must be impossible by construction, not by query-time discipline.
- Logical isolation in PostgreSQL is necessary but not sufficient — relying on WHERE org_id = ? everywhere is exactly the kind of "remember to filter" code that grows authz holes over time. Defense in depth requires the storage layer to enforce isolation.
- Per-org Qdrant collections eliminate the entire class of "forgot to filter by org_id" bugs in the retrieval path. A query against org_acme_memories cannot return memories from org_globex_memories no matter what filter the hub passes.
- Why not per-org hub instances? Operational cost. Running N hubs for N orgs means N PostgreSQL instances, N TLS certs, N deployment pipelines. Logical isolation at the hub layer + physical at the Qdrant layer gives strong security with manageable ops.
- Open-source hub means orgs CAN run their own if they want full physical isolation. That's the escape hatch for high-security teams.

**Task A Note:** Current Qdrant state is (B) filtered isolation — single shared collection `wevibe_memories` with org_id filter on every query. Implementation requires migrating to per-org collections named `org_{orgID}_memories`.

---

### D-12.2: Per-Memory Org Destination in Extraction UI

**Decision:** When a contributor extracts memories from a coding session, each extracted memory gets its own org destination dropdown. The system suggests a recommended org based on the memory's tech stack tags matched against each org's domain expertise. The contributor can change the destination per memory.

**Why:**
- A single session often produces memories that belong to different orgs. A debugging session may yield a React state-management insight (belongs to the frontend org) AND a Postgres index tip (belongs to the backend org). Forcing the contributor to choose a single org per session would lose half the value.
- Recommendation reduces friction. Most memories will go to the obvious org; the dropdown is for the cases where the contributor disagreed with the auto-suggestion.
- Multi-org submission is the end goal anyway. Designing the UI around single-org-per-session would force a rewrite later.
- Note: Multi-org submission requires that the contributor has membership and active kfrags in every destination org. The dropdown shows only orgs the contributor belongs to.

---

### D-12.3: Cross-Org Retrieval Deferred to Post-Sprint 25

**Decision:** Sprint 25 designs all systems aware of the multi-org end-state but does not ship cross-org retrieval at recall time. For now, the active org is set per-coding-session (mechanism TBD post-Sprint 25). The plugin queries one org per session.

**Why:** Designing for the multi-org end-state avoids the "retrofit single-org into multi-org" rewrite. Cross-org retrieval requires solving: how does the plugin decide which org to query per session? How does the user signal their intent? What does recall look like when the user belongs to 5 orgs? These UX and protocol questions are deferred to post-Sprint 25 while the infrastructure (per-org collections, org-aware submission pipeline) ships in Sprint 25.

---

### D-12.4: Consumer Profile Page

**Decision:** Two profile surfaces: /profile (current user's own profile, editable: avatar, display name) and /u/:wallet_address (public profile of any user, read-only). Profile contents: Avatar + configurable display name + wallet address (always shown, truncated), Aggregate reputation (sum across orgs) and per-org reputation breakdown, List of orgs with role in each (member / moderator / leader), Total memories contributed and total serves received across all orgs, Joined-WeVibe date, Last on-chain activity (links to chain explorer). Future: linked socials (verified). Profile is public by default. Wallet address is the canonical username. Display name is a label that can be changed anytime — uniqueness not enforced.

**Why:**
- Public profiles enable discovery and reputation signaling across org boundaries. A user's reputation in one org is visible to another.
- Wallet address as canonical username is consistent with D-1.1 (wallet is primary identity). Display name is purely cosmetic and carries no system-level identity weight.
- Aggregate + per-org reputation breakdown gives both a quick summary and org-specific detail. Users can signal expertise in specific domains.
- Public by default aligns with the open knowledge graph goal — hiding profiles would undermine the discovery surface that makes WeVibe valuable.

---

### D-12.5: Plugin ↔ wevibe-mcp Transport — HTTP API, Loopback-Only, No Auth (Sprint 25)

**Decision:** Replace the current subprocess-based JSON-RPC interface (where the plugin spawns retrieve-cli.ts) with an HTTP API on wevibe-mcp, bound exclusively to 127.0.0.1. No authentication in Sprint 25. wevibe-mcp port (default 4450) is configurable in plugin settings. This decision creates a follow-up D-12.5a: introduce per-session token authentication before any pre-alpha or external testing. **Closed companion: D-12.5a per-session token auth shipped in CO-260 (Sprint 26).**

**Why:**
- **Subprocess removal:** Spawning a Node.js process per retrieval is slow (~200ms cold start) and doesn't scale. An HTTP API lets the plugin maintain a persistent connection.
- **Loopback-only:** 127.0.0.1 binding means wevibe-mcp is never network-reachable from other machines. Even if an attacker runs code on the user's machine, they can't reach wevibe-mcp remotely.
- **No auth in Sprint 25:** Adding auth requires session management, token issuance, and plugin integration — scope creep that delays the Sprint 25 pipeline. Sprint 25 is closed internal dogfood (OpenCode users on the project lead's machine). The threat model during dogfood doesn't justify the complexity.
- **Auth before alpha:** D-12.5a is already logged. Before any external user runs the plugin, per-session token auth closes the loopback-only gap (in case someone runs the plugin on a shared machine).

---

### D-12.5a: Per-Session Boot Token Authentication (Closed by CO-260)

**Decision:** Plugin↔wevibe-mcp transport uses Bearer token auth. Token is generated once on wevibe-mcp startup (32 random bytes, hex-encoded, 64 chars), written to `~/.wevibe/mcp-session-token` with mode 0600, and held in memory for the lifetime of the wevibe-mcp process. Token rotates only when wevibe-mcp restarts. All HTTP endpoints (`/v1/health`, `/v1/recall`, `/v1/serves`, `/v1/reports`) require the token via `Authorization: Bearer <token>` header. Token comparison is constant-time.

**Why:**
- **Closes the loopback-only gap from D-12.5.** Loopback-only binding prevents network attackers from reaching wevibe-mcp, but doesn't prevent other local processes on the same machine. A boot token raises the bar: any local process trying to impersonate the plugin must also be able to read `~/.wevibe/mcp-session-token` (mode 0600).
- **Mode 0600 is the tightest user-space permission.** A process running as the same user can still read the file. Defense-in-depth: if any local process has that capability, it can also typically dump wevibe-mcp's memory or read keytar. The token is not load-bearing security; it's a meaningful additional check.
- **Boot-lifetime token simplest viable model.** Handshake tokens with TTL (Option B) add refresh logic for marginal benefit at single-user single-plugin scale. Delegate-key-signed requests (Option C) would require giving the plugin signing authority, which breaks the "wevibe-mcp owns the keys" boundary from MASTER.md "Cross-Cutting: wevibe-mcp as Local Backend."
- **Token regenerates on every wevibe-mcp restart.** No persistence across restarts means leaked tokens are bounded by wevibe-mcp uptime.

**What does NOT change:** wevibe-mcp still binds to 127.0.0.1 only (D-12.5). The token is a layer on top of loopback isolation, not a replacement for it. Pre-public-testnet hardening (transport encryption, per-request nonces, etc.) is out of scope and tracked separately if it becomes relevant.

**Implementation:** `wevibe-mcp/src/session-token.ts` (CO-260 Task A).

---

### D-12.6: Plugin Failure UX — Auto-Start First, Plugin-Specific Fallback

**Decision:** When the plugin needs to call wevibe-mcp: Plugin attempts to auto-start wevibe-mcp via known binary path (~/.wevibe/bin/wevibe-mcp or $(which wevibe-mcp)). If auto-start succeeds, proceed silently. If auto-start fails, fall back to a plugin-specific notification: OpenCode (terminal-based, current dogfood target): print structured error to stderr with copy-pasteable start command. Future GUI plugins (Claude Code GUI, Cline, etc.): toaster notification with "Start wevibe-mcp" CTA. If wevibe-mcp is reachable but the hub is unreachable, the plugin shows a transient warning ("WeVibe hub unreachable — retrying") and uses exponential backoff on retries.

**Why:**
- Auto-start eliminates the "you forgot to start wevibe-mcp" friction entirely for the happy path. Most dogfood sessions will never see an error message.
- Terminal plugins (OpenCode) and GUI plugins have different notification capabilities. The fallback is plugin-specific, not a one-size-fits-all toast.
- Exponential backoff on hub unreachability prevents the plugin from spamming the hub when it's temporarily down.
- Copy-pasteable start command in the error message lowers friction for the recovery action — the user types one command and continues.

---

### D-12.7: Org Discovery — Public Web + In-Dashboard

**Decision:** Org discovery exists at two surfaces: wevibe.network/orgs (public, browsable without a wallet. Marketing surface and pre-signup org evaluation) and /discover inside the authenticated dashboard (same data, but the org card has an active "Request to Join" CTA). Orgs cannot opt out of public discovery. Visibility is part of the deal — an org that wants to stay private should rely on invite-only join policy (D-12.8), not on hiding from discovery. Discovery shows: name + domain expertise + member count + description + tech stack tags + reputation tier payouts + trial info + join policy + recent activity (memory count, last batch commit timestamp linked to chain explorer). Search/filter: free text, domain filter, tech stack filter, "accepting members" toggle, sort by newest/largest/most-active/highest-payouts.

**Why:**
- Public discovery is the acquisition funnel. Future org members discover WeVibe through the public org directory. Hiding orgs from public discovery would break this.
- Opt-out would create a two-tier discovery system — messy for a first-release org directory. Invite-only (D-12.8) handles privacy without a visibility toggle.
- Rich discovery data (tech stack, payout tiers, activity) helps prospective members make informed decisions before requesting to join.
- In-dashboard discovery surfaces the same data but with an active CTA — the flow from discovery to join request is seamless.

---

### D-12.8: Join Requests — Zero Friction

**Decision:** Join request flow: User clicks "Request to Join" on an org's discovery page. Request is submitted with no form fields — the user's public profile (D-12.4) is the application. Org leader OR moderator (per-org config decides which) reviews the request in the dashboard. Reviewer approves (member added) or denies with optional free-text reason. Denied requesters see the reason in their own dashboard. Denied requesters have a cooldown period before re-requesting (default 7 days, leader-configurable). No auto-approve based on reputation in v1.

**Why:**
- Zero-friction requests maximize conversion. A form with fields creates abandonment. "Click to request" removes all friction from the happy path.
- The public profile IS the application — reputation, contribution history, and org memberships are all visible to the reviewer. No need to ask for information that's already public.
- Reviewer-controlled admission (leader or moderator) respects org autonomy. Some orgs are selective; some are open. The org decides.
- Denial reason + cooldown prevent request spam and bad-faith repeat requests. The reason tells the denied user what to improve; the cooldown prevents harassment.
- No auto-approve in v1: reputation-based auto-approve was considered and deferred. The risk of sybil attacks via high-reputation sock puppets is non-trivial. Manual review is the safe default.

---

### D-12.9: Activity Feed — Realtime WebSocket, All-Orgs Aggregated

**Decision:** Activity feed surfaces: Top-bar notification bell with unread count (Slack/GitHub style). Dedicated /activity page with full history and filtering. Aggregates across ALL orgs the user belongs to (no per-org silos). Transport: WebSocket push for realtime updates. Polling fallback for clients that lose WS connectivity. Event categories: Contribution (memory approved, memory rejected), Earning (payout received, tier upgraded), Recognition (memory reached serve milestone), Moderation (report filed, report resolved, member removed), Membership (joined org, left org), Reports (memory reported, report upheld).

**Why:**
- All-orgs aggregation is critical. Users who belong to multiple orgs need a unified view of activity across their entire WeVibe presence, not N separate dashboards.
- WebSocket push is the right transport for realtime. Polling is the fallback for reliability when WS isn't available (firewalls, brief disconnections).
- Notification bell with unread count follows established UX patterns (Slack, GitHub). Users already know this interaction model.
- Event categories are broad enough to capture everything meaningful without explosion. Each category maps to a specific WeVibe action and produces a specific notification type.

---

### D-12.10: Solo Dogfood — Mechanical Validation, Not Multi-User Testing

**Decision:** Sprint 25 dogfood is performed by a single user (the project lead) acting as contributor, moderator, AND leader. Goal: validate the pipeline mechanically end-to-end and surface UX friction. Anti-pattern to avoid: rubber-stamping own memories. Discipline mechanism: 24h+ cool-down between memory submission and moderation. This is a process rule, NOT enforced by code. Test boundary: solo dogfood validates the pipeline. It does NOT validate cross-user dynamics (joins, reports, multi-moderator quorum, social-graph behavior). Those require alpha multi-user testing post-Sprint 25.

**Why:**
- Solo dogfood catches mechanical failures (pipeline breaks, encryption errors, Qdrant indexing bugs) without needing a team. The goal is pipeline validation, not social dynamics testing.
- Rubber-stamping is the real risk. A single user approving their own memories provides zero quality assurance. The 24h cooldown forces a pause that breaks the self-approval loop.
- Cross-user dynamics (join flows, reports between users, multi-moderator quorum) cannot be tested solo. These require alpha with real external users post-Sprint 25.
- Process rule vs code enforcement: enforcing the cooldown in code would require tracking "last submission time" and blocking moderation during the window — complexity that doesn't belong in Sprint 25. The discipline is social, not technical.

---

## 13. Sprint 25 Architecture: Chain Hardening — Accountability + Social Graph Foundation

### D-13.1: Moderator Pubkey Persistence on Memory

**Decision:** All moderators who voted to approve a memory are stored permanently on `StoredMemoryCommitment` as `approvers []string`. The previous single-string `approver` field is removed. `MsgApproveMemory` carries `approvers[]` plus `committing_leader_pubkey`.

**D-CO030-APPROVERS:** The `approvers` repeated field in `MsgApproveMemory` has been deleted (CO-030). Co-attestation was removed in the Pattern B redesign. Proto field retained temporarily for backward compatibility through Sprint 31; deleted in Sprint 32 with chain wipe per D-13.9.

**Why:**
- **Multi-mod quorum accountability requires preserving all participants.** For orgs with `required_approvals > 1`, only recording the final approver who triggered the chain commit lets the other quorum members escape accountability if a memory later turns out to be harmful. The whole point of the moderator social graph showing "moderator X approved Y memories that were later upheld-reported" depends on every approver being on-chain.
- **Single-mod orgs are not affected.** With `required_approvals = 1`, the array contains one entry. No semantic change for them.
- **Chain doesn't validate the array.** The hub enforces the vote threshold (D-6.3). The chain records the array as-is. Trust is "the leader signed the TX, so they vouched for the array."
- **Committing leader is the on-chain TX signer.** They sign the batch commit with their wallet. Recording them separately from approvers preserves the distinction between "who voted approve" (operational) and "who chose to commit this batch to chain" (authoritative).

---

### D-13.2: Upheld Report Plaintext + Ciphertext + Capsule Triplet

**Status:** Verification anchor mechanism LOCKED as of 2026-05-27 via DMO-029. The mechanism is the contributor-signed canonical body containing plaintext_hash, salt, and ciphertext_hash — see Section 16 (D-VR-1 through D-VR-8). Implementation ships in the Sprint 31 implementation CO with chain wipe per Walter directive. The prior ZK pathway locked by DMO-028 (D-VE-1 through D-VE-10) is superseded; see Section 14 (superseded).

**Decision (locked):** When a report is upheld and committed on-chain via `MsgReportMemory`, the chain stores:
- `plaintext` (raw memory content, max 4096 bytes)
- `ciphertext` (AEAD-encrypted blob, max 8192 bytes)
- `capsule` (wrapped DEK sealed to moderator pubkey)
- `plaintext_hash` (sha256(salt || plaintext), 32 bytes)
- `salt` (32-byte random per submission)
- `plaintext_oversized` flag

The `plaintext_hash + salt` pair is the verification anchor. It is bound to the actual ciphertext content via the contributor's signature over the canonical body, which includes `plaintext_hash`, `salt`, `ciphertext_hash`, and `wrapped_dek_hash` jointly. The signature is verified at the chain's `MsgApproveMemory` commit and stored on-chain alongside the commitment. See Section 16 (D-VR-1 through D-VR-8) for the binding mechanism.

For memories exceeding the 4KB plaintext cap, `plaintext_oversized=true` and the plaintext/ciphertext/capsule fields are empty. The full plaintext is published off-chain (hub stores it permanently) and verified against the on-chain `plaintext_hash` + `salt` via standard sha256 check.

**Why the full set is stored on-chain:**

- **Cryptographic verifiability against a rogue leader.** Without ciphertext + capsule on-chain, a malicious leader could "uphold" a fabricated report by publishing fake plaintext to discredit a contributor. Storing the full set means anyone can demand the leader prove decryption matches.
- **The contributor-signed canonical body closes the contributor+leader collusion gap.** The plaintext_hash alone (without binding) could be poisoned at submit time by a captured contributor signing a decoy hash. The contributor's signature over the joint canonical body (`plaintext_hash`, `salt`, `ciphertext_hash`) binds the hash to the exact ciphertext at submit time, so the decoy attack is no longer viable (see D-VR-1 through D-VR-8). NOTE: this rationale was previously stated in terms of a ZK proof; ZK/SP1 is removed (D-VR-8) — the binding is now the contributor signature, not a zero-knowledge proof.
- **4KB cap is economically considered.** Upheld reports are rare and consequential — paying 10-40× normal approval gas is acceptable. 4KB covers ~95% of memories based on typical extraction outputs.
- **Oversized fallback preserves the property.** For large memories, the on-chain hash is evidence of deletion. Anyone with the off-chain plaintext can verify against it via standard sha256.

**`VerifyUpheldReport` is a query, not a write.** No gas cost for verification — anyone can audit any leader at any time. Verification reconstructs the canonical memory body from on-chain fields (plaintext, ciphertext, capsule, plaintext_hash, salt), recomputes the sha256 hash, and verifies the contributor signature. This replaces the SP1 verifier approach; canonical-body verification is defined in D-VR-1 through D-VR-8.

---

### D-13.3: Hub-Side Manipulation Alarm via BlockResults

**Decision:** The check-and-balance against leader manipulation of approval or report TXs is NOT chain-side runtime validation. It is a hub-side event log + moderator notification:

- Hub `ChainWatcher` subscribes to CometBFT BlockResults (not a PostgreSQL table), parses relevant TX events, writes events idempotently with restart-safety via a `watcher_state` table.
- Note: `chain_commit_events` PostgreSQL table was dropped Sprint 31 (CO-030). Hub now reads directly from CometBFT block events via BlockResults.
- For approval-overturn case (memory previously approved → later upheld-reported): notify the ORIGINAL approving moderators that their approval was overturned.
- Dashboard exposes side-by-side view: chain record vs hub's vote_records history. Visible discrepancy = the manipulation alarm.

**Why:**
- **The chain cannot enforce vote-process integrity at runtime.** It can't verify "did mod X actually vote approve, or did the leader fabricate the array?" because the vote process is hub-side operational state. Adding chain validation would require pushing all hub vote events to chain, which violates D-6.3 (operational decisions stay off-chain) and explodes gas costs.
- **Provenance + witnesses + alarm is the correct three-layer model.** Chain provides immutable provenance (the TX with the array is forever on-chain, attributed to the leader's wallet signature). Hub provides procedural witnesses (the vote_records history is the operational truth). Moderators provide the human alarm (they see a notification, check the chain record against their own knowledge, and raise a public flag if they were impersonated).
- **The leader's wallet signature is the accountability anchor.** A leader who fabricates a TX has signed their own indictment. The chain record is forever attributed to them. If moderators raise an alarm and the org community confirms, the leader's `StoredLeaderProfile` shows the fraudulent action permanently.
- **No runtime acceptance criteria possible.** The chain trusts the leader's signature because the alternative (chain validates vote process) requires chain-side knowledge of operational state. The hub-side alarm is the cheapest viable substitute.
- **Notifications are best-effort.** If the hub is down or the notification subsystem fails, the chain record is still on-chain and queryable. Moderators can audit retroactively. The notification is the early-warning system, not the only line of defense.

---

### D-13.4: Social Graph Data On-Chain, Display Layer Separate

> **[CANON CROSS-REFERENCE]** D-SG-2 defines the current social-graph read contract (open-source, forkable RPC display layer over chain data); this entry is retained as Sprint-25 foundation context.

**Decision:** The social graph is split into three layers:

| Layer | Lives In | Contains | Owner |
|---|---|---|---|
| Immutable provenance | wevibe-chain | Aggregates, indices, on-chain events with wallet pubkeys | Chain (cryptographic) |
| Operational queue | wevibe-hub | Pending submissions, votes, batches, BlockResults (hub reads from CometBFT block events) | Hub operator (WeVibe-hosted OR self-hosted) |
| Display layer | Social Graph Service | Wallet → display name + avatar + bio + linked socials (future) | Separate Docker container, separate VPS |

The Social Graph Service is a separate scoped CO (not part of CO-245). Both WeVibe-hosted hubs and self-hosted hubs consume it for human-readable names. Mandatory display name registration before a user can be accepted into moderator role (enforced at the hub-side "accept moderator role" handler — also part of the future Social Graph Service CO).

**Why:**
- **Open-source hub portability requires decoupling the display layer.** The hub is open-source (D-12.1). Anyone can run their own. If the hub owned the wallet→name mapping, every self-hosted hub would have its own disconnected naming database. The Social Graph Service is the centralized human-readable surface that all hubs query.
- **Chain stores immutable identity (wallet); display layer stores mutable label (name).** This is the same model as ENS/SNS. Wallets are forever; names are decoration.
- **Social Graph Service grows into a product surface.** Linked socials (verified Twitter/GitHub), avatar storage, future on-chain provenance proofs (D-13.8 future evolution) all belong there. Keeping it separate from the hub prevents hub bloat and enables the social graph to evolve independently.
- **Why deployed as Docker container, then VPS:** During Sprint 25 dogfood, the Social Graph Service doesn't exist yet — dashboard shows truncated wallet addresses like `wevibe1abc...xyz`. Acceptable because Walter is the only user. Pre-alpha, the service ships as Docker for local test, then VPS for production. The two deployment targets enable testing without committing to infrastructure cost.
- **Hubs sync display names from Social Graph Service.** Hub caches names for performance but the Social Graph Service is authoritative. Sync mechanism (poll vs push) is deferred to the Social Graph Service CO.

---

### D-13.5: Reputation Active at Genesis, Additive-Only

**Decision:**
- `DefaultParams.Active = true` (was `false`). Reputation is on from chain genesis.
- `Activate(ctx)` and `Deactivate(ctx)` keeper methods retained but reframed as governance-callable emergency pause, not the normal startup path.
- Reputation accumulation is purely additive in Sprint 25. The only downward pressure is the `RecordBan` event from upheld reports (which sets a flag, not a decrement).
- No time-based decay of reputation.

**Why:**
- **Activate-at-genesis matches Sprint 25's goal: use reputation in payouts.** F8 wires the rep tier lookup to use actual `total_approved_memories`. If reputation defaulted to inactive, the chain would launch with reputation off and payouts would silently use tier=0 for everyone — defeating the entire meritocratic emission model.
- **Activate/Deactivate retained for runtime emergency.** If a reputation bug is discovered post-launch, the chain operator wants to freeze reputation without a chain restart. Keeping the methods as governance-callable preserves the safety brake.
- **No time-based decay because count-based reputation cannot decay coherently.** "Reputation = number of approved memories" is a count. You didn't un-contribute. Decaying a count is incoherent semantically.
- **Decay only makes sense for quality-signal reputation.** Future evolution: reputation will incorporate commitment proofs (which model solved which problem, how many turns, output quality). Quality signals CAN decay because they reflect ongoing demonstration of skill — old quality is less valuable than recent quality. Until commitment proofs ship, no decay infrastructure is needed.
- **`RecordBan` is the only downward pressure.** When a memory is upheld-reported, the contributor's record is marked. This is a flag, not a decrement — the count of approved memories is unchanged, but the count of upheld reports increments. Together they tell the full story.

---

### D-13.6: Memory State Cleanup — 7-State Lifecycle Locked at Code Level

**Decision:** Remove `DORMANT`, `DEGRADED`, and `STABLE` from the `MemoryState` enum in `proto/wevibe/memory/v1/state.proto`. The locked enum values are:
- `MEMORY_STATE_UNSPECIFIED = 0`
- `MEMORY_STATE_PENDING = 1`
- `MEMORY_STATE_PENDING_KEYWORD = 2`
- `MEMORY_STATE_PENDING_CHAIN = 3`
- `MEMORY_STATE_COMMITTED = 4`
- `MEMORY_STATE_DENIED = 5`
- `MEMORY_STATE_ARCHIVED = 6`
- `MEMORY_STATE_REPORTED_DELETED = 7`

`isDecayEligible(state)` returns true only for `MEMORY_STATE_COMMITTED`. All other states are non-decay-eligible (terminal or pre-commit).

**Why:**
- **The docs and chain code have been out of sync since CO-242.** D-4.6 (Sprint 24) killed STABLE/DEGRADED/DORMANT in favor of the 7-state lifecycle. The MASTER.md was updated. The chain proto was not. CO-244 audit surfaced this gap.
- **Dead enum values create silent footguns.** Code paths that check `state == Dormant` look meaningful but execute against a state that the system never enters. Removing them prevents future contributors from believing those states are real.
- **Decay only applies to COMMITTED.** Pre-commit states (PENDING, PENDING_KEYWORD, PENDING_CHAIN) have no keyword weights yet. Terminal states (DENIED, ARCHIVED, REPORTED_DELETED) are out of the active retrieval pool. The new `isDecayEligible` matches the actual lifecycle.
- **R-LONGEVITY: clean up at the root.** Comments like "// DORMANT deprecated, do not use" are R-LONGEVITY violations. The fix is to delete the values entirely. Chain wipe (D-13.9) makes this safe.

---

### D-13.7: Cross-Module Event Wiring

**Decision:** Memory module's `ApproveMemory` and `ReportMemory` keeper methods, upon successful state transition, atomically call into:
- **org keeper** — bump aggregate counters (`total_committed_memories`, `total_upheld_reports`, `total_epoch_rotations`, `last_activity_epoch`)
- **reputation keeper — contributor profile** — bump `total_approved_memories`, `total_reports_upheld_against`, `serves_received`, `denials_received`
- **reputation keeper — moderator profile** — for each pubkey in `approvers[]`: bump `total_approvals`, append to `approved_memory_hashes[]` (bounded), bump `approvals_later_upheld_count` on upheld report

**D-CO030-APPROVERS:** The `approvers` repeated field in `MsgSubmitMemoryBatch` has been deleted (CO-030). Co-attestation was removed in the Pattern B redesign. See D-CO030-APPROVERS note in D-13.1.
- **reputation keeper — leader profile** — bump `total_chain_commits_signed` on approval; `total_upheld_reports_committed` on upheld report; append epoch rotations

Cross-module dependencies are explicit: memory module's keeper struct holds `orgKeeper` and `reputationKeeper` interfaces. App-level wiring in `app/app.go` injects these at keeper construction.

**Why:**
- **The social graph IS the join of cross-module signals.** A contributor's reputation is built from memory approvals (memory module) + serves received (serve module) + reports filed against them (memory module). A moderator's profile is built from approvals (memory module) + upheld-overturns (memory module). A leader's profile is built from chain commits (memory module) + epoch rotations (org module). Decoupling these signals into per-module silos and joining at query time would require expensive cross-module event scans.
- **Atomic updates prevent drift.** If memory is approved on-chain but org aggregate counter increment fails, the social graph diverges from the truth. Atomic in-handler updates ensure the chain state is always internally consistent.
- **Tight coupling is acceptable because the data is genuinely interdependent.** Cosmos best-practice generally avoids tight cross-keeper coupling, but the alternative here (events + epoch-end reconciliation) introduces a window of inconsistency for no real benefit.
- **Denormalized counters trade write cost for read efficiency.** Bumping 4-N counters per memory approval is ~2-3× the gas of a normal approval. This is the price of the audit trail. RPC queries for "show me everything about contributor X" become single-key lookups instead of event scans across the chain's history.

---

### D-13.8: Reputation as Tiering Signal — total_approved_memories

**Decision:** In `emissions/keeper/epoch_hooks.go ProcessOrgPayouts`, `getRepTierForContributor` uses `StoredContributorProfile.total_approved_memories` as the reputation input. The previous hardcoded `reputation := 0` is removed.

Future evolution: multi-factor reputation scoring (XP, domain expertise, quality proofs, model commitment hashes) is deferred until the commitment-proof infrastructure exists.

- **CO-244 found rep tiers plumbed but never functionally wired.** The tier configs were stored, the lookup function existed, but it always received `0` — meaning every contributor got the lowest tier. The meritocratic emission model that justifies WeVibe's economic design was functionally inactive.
- **total_approved_memories is the cleanest available signal.** It's a count of accepted memories — the most concrete proof-of-contribution. XP is also tracked but more abstract. Domain expertise is too sparse early. Quality proofs don't exist yet. The count is what we have.
- **Future scoring will be multi-factor.** When commitment proofs land (proving "model X solved problem Y in N turns producing memory Z"), reputation will fold in: quality of memories, difficulty of problems solved, model capability tier, efficiency (turns to solution). For now, the count is a workable proxy.
- **This unlocks tiered payouts immediately.** Once D-13.7's contributor profile aggregates are wiring memory approvals to `total_approved_memories`, the tier lookup produces meaningful tier assignments. Established contributors earn more per memory than fresh joiners — the core economic property of the system.

---

### D-13.9: Chain Wipe Acceptable Pre-MVP

**Decision:** CO-245 ships breaking schema changes (enum overhaul, message signature changes, new types) without migration code. The chain is wiped before integration tests run. Pre-launch state is disposable.

After mainnet launch, schema changes will require Cosmos `x/upgrade` migration paths.

**Why:**
- **Pre-MVP, the only chain state is Walter's dev state.** Zero users, zero production data. Migration code would be infrastructure for protecting data that doesn't exist.
- **R-NO-DB-HACKS explicitly allows clean wipes.** The standing rule reads: "Schema changes at the root. Clean DB wipes acceptable." This is the situation that rule was written for.
- **Migration code is a maintenance liability.** Every migration is forever — it must keep working through future chain upgrades. Writing migration code now means maintaining it through every subsequent schema change. Avoiding it pre-MVP keeps the codebase lean.
- **The line is mainnet launch.** The moment real users have real wallet addresses on real chain state, every schema change requires a migration path. Until then, chain wipes are the cheapest correctness mechanism.
- **Documented so future contributors don't assume migration discipline is loose.** This decision is for THIS sprint's pre-MVP context. It is not a general permission to skip migrations.

---

### D-13.10: Schema Recreation on Every Postgres Volume Init (Pre-MVP Only)

**Decision:** Pre-MVP, the hub's PostgreSQL schema is bootstrapped from a single `schema.sql` file applied at Postgres container init. There is no migration system. Schema changes require:

1. Edit `wevibe-server/db/schema.sql`
2. `docker compose down -v` (wipes the postgres_data volume)
3. `docker compose up` (Postgres re-runs schema.sql on the empty volume)

The hub at startup verifies required tables exist (via `VerifyConnection`) and exits fatally if any are missing. No incremental migration logic exists in the hub. The previous `RunMigrations` function is deleted.

The Ollama service is a documented host exception — it runs as a macOS host app for Metal GPU acceleration. Containers reach Ollama via `host.docker.internal:11434`.

**CO-267 removes the previous temporary wevibe-mcp host exception.** wevibe-mcp now runs as a Docker service (`wevibe-mcp`) built from `wevibe-server/Dockerfile.wevibe-mcp`, with file-backed keystore storage (`${WEVIBE_KEYSTORE_PATH}/keys.json`), Ollama HTTP embeddings (`${OLLAMA_HOST}/api/embeddings`), and pure-JS Argon2 KDF (`@noble/hashes/argon2`).

The umbral-sidecar service (D-2.2) is a Docker service and is NOT a host exception.

**Why:**
- **Pre-MVP, zero users, zero production data.** Migration code would be infrastructure for protecting data that doesn't exist.
- **R-NO-DB-HACKS explicitly allows clean wipes.** The standing rule reads: "Schema changes at the root. Clean DB wipes acceptable." This is the situation that rule was written for.
- **Incremental migrations are themselves a liability.** Every `ALTER TABLE IF NOT EXISTS` is a path that may or may not have run, depending on when the database was first created. Pre-MVP this is acceptable cost; production it is not.
- **One file, one path, one source of truth.** `schema.sql` is the only thing that ever creates tables. No code path duplicates it.
- **Ollama exception is operational, not architectural.** Native Metal GPU on macOS is dramatically faster than container CPU. For solo dogfood, performance > containerization purity. Documented as the only exception.
- **wevibe-mcp is now containerized.** The prior dependency-driven exception was removed by replacing native module blockers with container-safe implementations.

**Locked counterpart:** **D-13.11** captures the carry-forward debt: a real migration system is required before public testnet ships.

---

### D-13.11: Migration System Required Before Public Testnet

**Decision:** Before WeVibe ships a public testnet (and before any external operator runs an open-source hub against persisted data), a proper schema migration system MUST be implemented. The pre-MVP "wipe + recreate" model is unacceptable post-launch because:

- Public testnet users have wallets, reputation, and memory data that cannot be wiped on schema changes
- Self-hosted hub operators (per D-12.1 — hub is open-source) cannot tolerate "every update requires database wipe"
- Public chain commits create accountability records that must survive schema evolution

**Required before testnet:**
- A migration tool (golang-migrate, sqlc-style versioned migrations, or equivalent) with up/down support
- Migration files numbered and ordered
- Hub startup detects schema version and applies pending migrations automatically OR refuses to start if migrations are pending (operator decision)
- Test coverage for migration up/down paths
- Documentation for operators

**Until then:** Every hub deploy requires `docker compose down -v`. This is the cost of pre-MVP velocity, and it is intentional. CO-253 explicitly accepted this tradeoff.

**Tracked as:** GAP-T2 in MASTER.md gap log. Severity: BLOCKING-FOR-PUBLIC-LAUNCH. Must be addressed before any external user creates persistent state in an WeVibe hub.

**Why this is acceptable today:**
- Solo dogfood. One user. Zero persistent state worth preserving.
- All chain state is wiped concurrently per D-13.9 — there is no data to migrate.
- The forcing function for the migration system is "we're about to launch testnet," which we are not.

**Why this must NOT extend to testnet:**
- Public commitment via chain TX creates immutable accountability records (D-13.2)
- A hub that loses moderator vote history because of a schema change cannot be reconciled with the chain
- Self-hosted hub operators cannot tolerate destructive updates
- Pre-public reputation is meaningful (D-13.8 ties payouts to total_approved_memories)

---

### D-13.12: Chain Broadcast via Comet RPC `broadcast_tx_sync`

**Decision:** The hub broadcasts chain transactions via Comet RPC `broadcast_tx_sync` (against the validator's RPC endpoint at `tcp://wevibed:26657`), not via Cosmos SDK gRPC `BroadcastTx`. Fees are calculated as `ceil(gas_limit × min_gas_price)` with `min_gas_price = 0.01 uvibe` and a floor of 2000 uvibe. Transient state-load errors from the local chain runtime (e.g., `version does not exist (latest height: N)` during block turnover) are retried with up to 8 attempts and exponential backoff (400ms × attempt number).

**Why:**
- **Resilience to local dev block-turnover races.** The Cosmos SDK gRPC `BroadcastTx` path queries latest state during broadcast preparation and repeatedly hit "version does not exist" failures on the local single-node chain during block transitions. Comet RPC `broadcast_tx_sync` sidesteps the state-query dependency.
- **Gas-proportional fees are required for multi-message batch TXs.** D-6.6 caps batches at 500 memories per multi-message TX. Static fees that worked for single-message TXs (2000 uvibe) became insufficient at multi-message gas levels (6000+ uvibe), causing `insufficient fees` errors. The `gas × min_gas_price` formula scales correctly with batch size.
- **Local signing preserved.** The change is to the broadcast transport, not to signing — leader wallet signatures, hub delegate signing, and the account-fallback path (loading account from genesis when gRPC account query is transiently unavailable) all remain.

**File:** `wevibe-server/wevibe-hub/internal/chain/broadcast.go`

**Note:** The transient-error retry exists because of the dev-mode chain settings documented at D-13.13 (which itself is GAP-T4). When those settings are fixed for testnet, the retry policy should be revisited — it may be over-broad once the underlying flakiness is gone.

### D-13.13: Chain Pruning + IAVL Fast-Node Disabled for Local Dev (Pre-MVP Only)

**Decision:** `wevibe-chain/scripts/init-chain.sh` writes `pruning = "nothing"` and `iavl-disable-fastnode = true` into the generated `app.toml` for local dev nodes. These settings make `make dogfood` reliable but are NOT acceptable for any persistent or public chain deployment.

**Why this is acceptable pre-MVP:**
- The local chain is wiped on every `docker compose down -v` per D-13.9. Disk-usage concerns of `pruning=nothing` are moot when the chain lives for one test run.
- Disabling IAVL fast-node hurts query latency but the local single-node chain has trivial query load. The benefit (avoiding the block-turnover state-load races that motivated D-13.12) outweighs the cost.
- The init script is a dev convenience. Production validators will write their own `app.toml` and will not run this script.

**Why this is NOT acceptable for testnet or production:**
- `pruning=nothing` retains every historical block forever. At even modest TX volume this grows unbounded.
- IAVL fast-node is a default Cosmos SDK optimization for query performance. Production chains should not disable it.

**Tracked as:** GAP-T4 in MASTER.md. BLOCKING-FOR-PUBLIC-LAUNCH. Coupled to D-13.12's retry policy — when the underlying state-load behavior is fixed, both the dev flags and the retry policy may be revisited.

**File:** `wevibe-chain/scripts/init-chain.sh`

---

### D-13.14: CO-026 Contributor-Signed Plaintext Hash — Reverted

**Status:** REVERTED via DMO-027 + CO-026R (2026-05-27).

**Decision:** The work shipped under CO-026 (contributor-signed `sha256(plaintext)` carried across proto, chain keeper, hub canonical v2, dashboard, MCP, and e2e tests) is reverted in full.

**Why reverted:**

1. **No salt.** CO-026 used `sha256(plaintext)` without a per-submission salt. For low-entropy plaintexts (short memories, common technical advice), this is rainbow-table-vulnerable. A captured hub operator with read access to on-chain hashes could brute-force the original content for many memories.

2. **Wrong cryptographic foundation.** CO-026 was scoped as the first half of a verifiable-encryption design intended to defeat contributor+leader collusion attacks on Pattern B Tier 2 reports. The intended verification mechanism (a ZK proof binding the hash to the on-chain ciphertext) was scoped on the assumption that production used Umbral PRE encapsulation. The CO-027 abort report (2026-05-27) revealed that production uses AEAD + sealed-box on the submission path. The verification design must be rebuilt against the actual primitives.

3. **Incomplete threat coverage.** Even with salt, a hash signed only by the contributor protects against leader hash poisoning but not against contributor+leader collusion (when both are adversaries). The replacement design must address this explicitly — either by binding the hash to the AEAD ciphertext through a verifiable encryption proof, or by accepting the residual risk and documenting it as a known limit of the verification anchor.

**What is preserved from CO-026:**
- The architectural goal: a verification anchor on-chain that binds revealed plaintext in a Tier 2 report to the actual content the contributor submitted, without trusting the leader.
- The 7-state lifecycle and other proto fields not introduced by CO-026.

**What is removed by the revert:**
- `MsgApproveMemory.plaintext_hash` (field 9)
- `MsgReportMemory.plaintext_hash` (field 12)
- `StoredMemoryCommitment.plaintext_hash` (field 15)
- `StoredMemoryReport.plaintext_hash` (field 12)
- DB migration `000005_add_plaintext_hash.up/down.sql`
- `pending_submissions.plaintext_hash` column
- Hub canonical v2 → reverts to canonical v1
- Hub validation of `plaintext_hash` at submit time
- Hub batch chain submit inclusion of `plaintext_hash`
- Dashboard and MCP computation of `plaintext_hash` at submission
- All e2e test assertions involving `plaintext_hash`

**What replaces it:** A re-architected verification anchor design is now locked via DMO-029 (see Section 16, D-VR-1 through D-VR-8). The replacement uses a contributor-signed canonical body containing plaintext_hash, salt, and ciphertext_hash jointly — no zero-knowledge cryptography. DMO-028's ZK pathway lock (D-VE-1 through D-VE-10) was superseded after CO-028 demonstrated that the ZK approach cannot ship on consumer hardware. See D-13.2 for status, Section 14 for the superseded ZK design preserved as historical record, and the implementation CO that follows this DMO for Sprint 31 ship plan.

**Reference:** DMO-027 (this doc revert), CO-026R (companion code revert), CO-027 questions report (2026-05-27, original abort discovery).

---

### D-S29-VECTORS-CLOSED: Sprint 28 Deferred Test Vectors Regenerated [PROCESS]

**Decision:** All 4 vectors deferred from Sprint 28 MO-006 per D-S28-MO006-VECTORS-SCOPE have been regenerated via the REGEN_VECTORS pattern. `wevibe-protocol/test_vectors/` now contains no stale vectors.

**Vectors regenerated:**
- fee_model_hash
- mnemonic_roundtrip
- seal_open_envelope
- shamir_roundtrip

**Provenance:** Each vector file carries a `regenerated_by: "WeVibe-CO-006"` top-level field per R-VECTORS-PROVENANCE.

**Regen method:** `REGEN_VECTORS=1 cargo test test_<vector>_vectors` — SDK test scaffolding added/extended in `wevibe-sdk/crates/wevibe-sdk-core/tests/crypto_tests.rs`.

**Note:** The 4 vectors happened to already be correct — regeneration produced byte-identical output. The stale designation in REGEN-PENDING.md was conservative; the vectors were not actually stale.

**Closes:** D-S28-VECTORS-SPRINT29-TICKET.

---

## 15. Sprint 29 Chain Foundation Decisions

### D-S29-SDK-V053: Cosmos SDK v0.53.5 is the Canonical SDK Line [ALPHA - FOUNDATION]

**Decision:** wevibe-chain pins `github.com/cosmos/cosmos-sdk v0.53.5`.

**Why v0.53 over v0.54:**
- v0.54.x is an experimental branch with internal store/v2 baseapp adoption that broke compatibility with `cosmossdk.io/x/upgrade` (all published versions). Verified via CO-005-evidence-A and CO-007a-evidence-C.
- v0.53.x is the production-grade line used by live Cosmos chains. Supports the modular `cosmossdk.io/x/*` package ecosystem including x/upgrade v0.2.0.
- Cosmos SDK main branch has reverted store/v2 in baseapp, suggesting v0.54.x is a dead intermediate branch not on the convergence path.
- Echo originally selected v0.54.2 in May 2026 as "latest at time of adoption" without recognizing it as the experimental branch. CO-008 corrects this.

**Closes:** D-S29-STORE-V2-REVERT-PLANNED (superseded by SDK-level downgrade).

---

### D-S29-COMETBFT-V038: CometBFT v0.38.20 [ALPHA - FOUNDATION]

**Decision:** wevibe-chain pins `github.com/cometbft/cometbft v0.38.20`.

**Why:**
- Pinned by Cosmos SDK v0.53.5 (D-S29-SDK-V053).
- Compatible with `cosmossdk.io/x/upgrade v0.2.0` (pinned v0.38.17, forward-compatible within the v0.38.x line).
- CometBFT v0.39.x is paired with SDK v0.54.x; once we leave v0.54.x we leave v0.39.x.

---

### D-S29-LOG-V1: cosmossdk.io/log v1 is Canonical [ALPHA - FOUNDATION]

**Decision:** wevibe-chain uses `cosmossdk.io/log v1.6.1`. `cosmossdk.io/log/v2` is forbidden in custom keeper code.

**Why:**
- SDK v0.53.5 baseapp uses log v1; log/v2 is paired with v0.54.x.
- Echo originally imported log/v2 in custom keepers when on v0.54.2. CO-008 aligns to v1.

---

### D-S29-PREBLOCKER-COMPLIANCE: SetOrderPreBlockers Wired [ALPHA - FOUNDATION]

**Decision:** `app/app.go` calls `app.ModuleManager.SetOrderPreBlockers(authtypes.ModuleName)` per Cosmos SDK v0.53.5 UPGRADING.md mandatory requirement.

**Why:**
- Sprint 28 chain wiring under v0.54.2 was missing this call. v0.53 makes this requirement explicit.
- Future upgrades that need additional preblockers (e.g. x/upgrade module preblocker) add to this ordered list, not replace it.

---

### D-S29-CHAIN-LATEST-IS-RISKY: SDK Version Selection Policy [PERMANENT - PROCESS]

**Decision:** wevibe-chain pins to actively maintained stable Cosmos SDK lines (v0.53.x for the foreseeable future), not "latest at time of decision."

**Why:**
- Echo's adoption of v0.54.2 in May 2026 inherited an experimental branch in disguise.
- Cosmos SDK release tags do not clearly distinguish "stable production line" from "experimental refactor in progress" - version numbers alone are insufficient signal.
- Future SDK bumps require an explicit CO with rationale: which release line, why it is the production-grade choice, what ecosystem support looks like, and what migration path exists for chains already on the previous line.

**Reference:** CO-008 implementation report.

---

### D-S29-IAVL-QUERY-BUG-KNOWN: IAVL State Queries Broken — Resolved by D-S29-CHAIN-RESTART-FOUNDATION

**Decision:** All ABCI state queries failed on wevibe-chain with `"version does not exist"`. Root cause was the empty-IAVL-tree problem: WeVibe modules implement AppModule but NOT appmodule.HasGenesis, so ModuleManager.InitGenesis skips their genesis handlers. Their KV stores received zero writes, causing ErrVersionDoesNotExist on LoadVersion for every state query.

The fix (D-S29-CHAIN-RESTART-FOUNDATION, CO-005d) writes a sentinel marker to every mounted KV store in InitChainer. This ensures every IAVL tree has version history.

**Resolution:** CO-005d wrote the marker-write. CO-010 verified all standard module state queries now work on fresh genesis chains (5/5 passed: bank balances, distribution params, upgrade plan, slashing params, staking validators). GAP-CHAIN-20 is CLOSED.

**Discovery:** CO-005b upgrade verification (2026-05-23).

**Reference:** CO-010 implementation report.

---

### D-S29-CHAIN-RESTART-FOUNDATION

**Decision:** wevibe-chain's InitChainer writes a sentinel marker key (4-byte 0xFF prefix) to every mounted KV store after ModuleManager.InitGenesis. Without this, IAVL's empty-tree optimization skips persistence for stores backing modules that don't implement appmodule.HasGenesis, causing ErrVersionDoesNotExist on any restart. Discovered via CO-005c; fixed in CO-005d.

---

### D-S29-INITCHAINER-VERSION-MAP

**Decision:** wevibe-chain's InitChainer calls UpgradeKeeper.SetModuleVersionMap(ctx, ModuleManager.GetVersionMap()) after ModuleManager.InitGenesis returns. This is required for any manually-wired (non-depinject) Cosmos SDK chain. Without it, x/upgrade's ApplyUpgrade reads an empty fromVM, RunMigrations treats every module as new, and re-runs InitGenesis on already-initialized state — panicking on distribution's balance invariant check. The canonical guidance is at cosmossdk.io/x/upgrade@v0.2.0/module.go:130-131. Discovered via CO-005c-resume; fixed in CO-005e.

---

### D-S29-HUB-SEQUENCE-RACE [SUPERSEDED by D-S32-CO044-PER-ORG-BROADCAST]

**Decision:** wevibe-hub's broadcast.go cross-checks queried account sequence against a local post-broadcast cache (max-of-two). Without this, successive broadcasts within one CometBFT block window race and the second is rejected with sequence mismatch. Known limitation: max-of-two doesn't handle rejected broadcasts; proper fix is in-flight counter pattern (documented in code). Discovered via CO-005d dogfood; fixed in CO-005d Stage 9.

**Superseded:** CO-044 replaces the single shared `submitter` account (the root cause this hack
worked around) with a per-org signer registry, each org with its own account and sequence. The
max-of-two cache is removed. See D-S32-CO044-PER-ORG-BROADCAST.

---

### D-S29-UPGRADE-STORE-LOADER

**Decision:** UpgradeStoreLoader wiring (ReadUpgradeInfoFromDisk + SetStoreLoader) is canonical and stays. For empty StoreUpgrades{} it falls through to DefaultStoreLoader and is functionally a no-op. Original claim that wiring it fixes the restart panic was wrong; the actual restart fix is D-S29-CHAIN-RESTART-FOUNDATION. Wired in CO-005c.

---

### D-S29-UPGRADE-VERIFIED

**Decision:** wevibe-chain's x/upgrade flow verified end-to-end: governance proposal → halt at upgrade height → binary swap → state load across upgrade boundary → ApplyUpgrade fires → RunMigrations completes with populated fromVM → block production resumes. Verified via CO-005e Stage 3 on fresh wevibe-upgrade-v3-* fixtures. Closes GAP-CHAIN-3 / D-14.8.

---

### D-S29-AUDIT-BEFORE-FIX

**Decision:** Process discipline: when iterative bug-discovery on the same gap exceeds 3 R-ABORT cycles, Manager must step back and dispatch a canonical-reference audit before the next fix attempt. Continuing piecemeal fixes beyond 3 iterations is a manager failure mode. Established after CO-005c-resume (the 5th iteration on GAP-CHAIN-3); CO-005e is the corrective audit-first approach.

---

### D-S29-DOGFOOD-IS-LOAD-BEARING

**Decision:** Sprint 29 discovered 3 alpha-blocking bugs (chain restart, hub broadcast race, x/upgrade wiring incompleteness) that unit tests did not catch. R-DOGFOOD-FROZEN is non-negotiable. Reinforced; never relax.

---

### D-S29-THROUGHPUT-DEFERRED

**Decision:** The per-event chain TX model for serves and denials (D-4.3, D-4.5) is retained for pre-alpha. A post-alpha architectural review will evaluate batch settlement when multi-user load arrives.

**Why retained now:**
- Solo-dogfood scale (~10 serves/day, ~1 denial/day) generates negligible chain load
- Per-event TX gives immediate source-of-truth consistency between hub and chain (no batching window)
- The model is simple, debugged, and already exercised by CO-005e's end-to-end upgrade test
- Redesigning the settlement model during chain foundation hardening would introduce scope creep into a sprint focused on getting basics working

**Why review is needed post-alpha:**
- At multi-org production scale (100+ orgs, 1000+ daily serves), per-event TX volume will dominate block space
- Hub-side batch aggregation with periodic cryptographic settlement would reduce chain load by 10-100x
- This is the same pattern used by production L2s and rollup-style systems
- The hub already has the trust model (D-3.1) to perform aggregation — it's not a new trust assumption

**Review trigger:** When daily serve+denial TX count exceeds 50% of single-validator block capacity, or when the second external org onboards — whichever comes first.

**What does NOT change:** D-4.3 and D-4.5 remain locked. The per-event model is the current implementation. The post-alpha review may propose amendments to those decisions via a formal CO with Walter approval.

---

## 16. Pattern B Tier 2 Verification Anchor (Current Design)

This section replaces Section 14 (D-VE-1 through D-VE-10, superseded). The decisions here define the verification anchor for Pattern B Tier 2 public report escalation using contributor signatures over a redesigned canonical body. No zero-knowledge cryptography.

**Implementation status:** Design LOCKED. Implementation CO follows immediately; chain wipe is part of the implementation. The anchor (D-VR-1 through D-VR-8) is implemented and live. The Tier 2 escalation loop that consumes the anchor — reporter-signed public exposure under a chain-enforced response-window timer — is DESIGNED, NOT BUILT; see D-VR-9 and GAP-TIER2-EXPOSE.

---

### D-VR-1: Signed Canonical Body Is the Verification Anchor

**Decision:** Pattern B Tier 2 uses a contributor-signed canonical body whose signed fields include `plaintext_hash`, `salt`, and `ciphertext_hash`. At Tier 2 escalation, the chain verifies (a) that the reporter's revealed (plaintext, salt) produces the on-chain plaintext_hash via sha256, and (b) that the contributor's on-chain signature is valid over the canonical body containing those fields. Together these two checks bind the contributor to the specific plaintext and ciphertext that was committed.

**Why signatures rather than ZK:**

The ZK pathway (Section 14, superseded) proved a relationship between plaintext_hash, salt, and ciphertext at memory commit time. A signed canonical body that includes plaintext_hash, salt, and ciphertext_hash as signed fields proves the same relationship — the contributor cryptographically attests to all three values jointly. The attacks defeated are identical:

- Leader substitutes ciphertext between submit and chain commit → defeated, signature binds ciphertext_hash
- Contributor claims different plaintext at Tier 2 → defeated, signature binds plaintext_hash and the hash is salted
- Contributor + leader collusion to commit poisoned content with mismatched hash → defeated, contributor signs all three fields jointly so any mismatch invalidates the signature

The single attack the ZK pathway claimed advantage on — "what if the contributor lies about the relationship between plaintext and ciphertext while still producing a valid signature" — was a phantom. A signature over `(plaintext_hash, salt, ciphertext_hash)` is itself the proof of the relationship. There is no additional relationship a ZK proof would establish that a joint signature does not.

The operational delta is enormous: zero ZK proving time (vs 45 s on consumer hardware), zero proving memory (vs 16.6 GB), zero prover-service dependency, zero zkVM audit, zero new container.

**Reference:** GO-001 gather report (2026-05-27), CO-028 feasibility spike (2026-05-27), 2026-05-27 design session.

---

### D-VR-2: The Contributor Is the Sole Signer of the Verification Anchor

**Decision:** Only the contributor signs the canonical body that produces the verification anchor. Neither the moderator nor the leader signs over plaintext_hash, salt, or ciphertext_hash. The leader's wallet signs the chain TX (per D-1.3), but the leader's signature is on the transaction envelope, not on the inner canonical body. The contributor's signature on the canonical body travels through the hub and lands on chain inside the `MsgApproveMemory` payload.

**Why the leader does not sign the anchor:**

If the leader signed the anchor, then a captured leader could refuse to sign honest memories or sign fabricated memories. The point of the anchor is to defeat captured-leader scenarios — the leader must be cryptographically removed from the verification chain. The leader's only role is operational: they decide which memories to batch and they pay gas to commit. The cryptographic binding is purely contributor-to-content.

**Why moderators do not sign the anchor:**

Moderators are already accountable through D-6.4 (their pubkey is recorded on every approved memory). Adding moderator signatures over the verification anchor would conflate two distinct accountability flows (quality review vs content provenance). Moderators can be captured the same way leaders can; the anchor must be robust against that.

**Reference:** GO-001 Task C findings (signature persistence path), Walter directive 2026-05-27.

---

### D-VR-3: Canonical Body Field Set Overhauled In Place

**Decision:** The canonical body tagged `wevibe.submit_memory.v1` is overhauled in place to include the new fields. No `v2` tag, no parallel-tag dual handling, no migration path. Pre-MVP per Walter directive.

**Current canonical body fields (six fields):**

```
wevibe.submit_memory.v1
contributor_pubkey:<hex>
epoch_id:<int>
memory_type:memory
org_id:<string>
submission_hash:<hex>
```

**Replacement canonical body fields (nine fields, alphabetically sorted after domain tag):**

```
wevibe.submit_memory.v1
ciphertext_hash:<hex>
contributor_pubkey:<hex>
epoch_id:<int>
memory_type:memory
org_id:<string>
plaintext_hash:<hex>
salt:<hex>
submission_hash:<hex>
wrapped_dek_hash:<hex>
```

Where:
- `plaintext_hash = sha256(salt || plaintext)` — 32 bytes hex-encoded
- `salt = random_32_bytes` — 32 bytes hex-encoded
- `ciphertext_hash = sha256(ciphertext)` — 32 bytes hex-encoded
- `wrapped_dek_hash = sha256(wrapped_dek_mod)` — 32 bytes hex-encoded
- `submission_hash` retained as `sha256(ciphertext || wrapped_dek_mod)` for backward semantic compatibility with current retrieval-path code paths that already reference it

**Why all three new fields and not just plaintext_hash:**

The minimum sufficient field for the verification anchor is `plaintext_hash` (with salt). But binding `ciphertext_hash` separately defends against ciphertext substitution attacks that `submission_hash` alone is weaker against (because `submission_hash` is the combined hash of ciphertext concatenated with wrapped_dek_mod — splitting them into separate hashes makes each independently verifiable). `wrapped_dek_hash` is added for symmetry and to support the WrappedDekEnc on-chain forwarding (D-VR-5) — the chain can independently verify the wrapped DEK against this hash.

**Why no version bump:**

Walter directive: pre-MVP, no users, no migration path required. The implementation CO wipes the chain, wipes hub PostgreSQL via existing migration discipline (D-13.10 → D-13.11), and replaces the canonical body in place across MCP, Dashboard, and Hub. No backwards compatibility, no v1/v2 dual handling, no graceful degradation. R-ONE-PATH, R-OVERHAUL.

**Reference:** GO-001 Task B/C/D findings (three independent canonical-body implementations exist), Walter directive 2026-05-27.

---

### D-VR-4: Contributor Signature Persisted On-Chain Alongside Commitment

**Decision:** `MsgApproveMemory` carries a new `contributor_sig` field (bytes). `StoredMemoryCommitment` carries a new `contributor_sig` field (bytes). The contributor's signature over the canonical body is forwarded to chain by the leader's batch commit and stored permanently alongside the commitment.

**Why on-chain rather than hub-only:**

The current production code stores `contributor_sig` only in hub PostgreSQL (`pending_submissions.contributor_sig`, `rotation_buffer.contributor_sig`). At Tier 2 escalation, the verifier would have to trust the hub to produce the signature — but the entire point of Tier 2 is to operate when the hub or org is captured. The signature MUST be on chain for Tier 2 verification to be trustworthy.

**On-chain bytes cost:**

Ed25519 signatures are 64 bytes. Per-memory overhead is negligible compared to the encrypted blob and the keyword index. The chain pays this cost once at commit time; it pays back at every Tier 2 verification. Verification is on-demand and rare; storage is permanent. The trade is correct.

**Why the existing hub-side persistence (`pending_submissions.contributor_sig`) is retained:**

It is needed during the moderation pipeline (pre-commit) when the chain has no record yet. Once the batch commit confirms, the chain record becomes authoritative. The hub-side row remains for operational lookup but is no longer the sole record.

**Reference:** GO-001 Task C Q6 findings.

---

### D-VR-5: WrappedDekEnc Must Be Forwarded to Chain

**Decision:** `handlers/moderation.go:762` (the leader batch chain-submit path) must populate `BatchMemory.WrappedDekEnc` from `pending_submissions.wrapped_dek_mod` before constructing `MsgApproveMemory`. The chain's `StoredMemoryCommitment.wrapped_dek_enc` MUST be non-empty for all memories committed via this path.

**Why this is part of the verification anchor design and not a separate bug:**

The verification anchor at Tier 2 verifies the contributor's signature against a canonical body reconstructed from on-chain state. The canonical body includes `wrapped_dek_hash = sha256(wrapped_dek_mod)`. If `wrapped_dek_mod` is not on chain (currently `WrappedDekEnc` is left nil per GO-001 Task C Q5), the canonical body cannot be reconstructed from on-chain state alone. The verifier would need to query the hub for the missing bytes — which means the hub is in the trust path again, which means Tier 2 verification is not robust against hub capture.

Forwarding WrappedDekEnc to chain closes this gap. The chain now holds every byte the contributor signed over, plus the signature, plus the salt and hashes. Verification is fully chain-local.

**Reference:** GO-001 Task C Q5 findings — "WrappedDekEnc gap: contributor's wrapped_dek_mod is preserved in postgres but never forwarded to chain."

---

### D-VR-6: Rotation Buffer Path Must Verify Signatures Before Persisting

**Decision:** `internal/orgs/orgs.go:125` (`BufferSubmission`) MUST call `verify.RequestSignature` before writing `contributor_sig` to `rotation_buffer`. The flush path that copies `rotation_buffer` rows into `pending_submissions` MUST also verify the signature (not trust the buffered row).

**Why this is part of the verification anchor design and not a separate bug:**

The rotation buffer is the path through which submissions land during an epoch rotation. If unverified signatures can enter the buffer, then after flush, the hub's `pending_submissions` table contains rows with `contributor_sig` values that nothing has ever verified. The leader's batch commit reads from `pending_submissions` and forwards the signature to chain. The chain has no way to detect that the signature was never verified by the hub. A malicious hub operator (or compromised hub process) could inject arbitrary signatures.

Verifying at the buffer write closes this. The verification path is uniform: every persistence point that holds a `contributor_sig` value has verified that signature against the corresponding canonical body before storing it.

**Reference:** GO-001 Task C OPEN ITEMS — "Rotation-buffer path is SIGNATURE-UNVERIFIED at intake."

---

### D-VR-7: Dashboard Must Not Send Plaintext to Hub

**Decision:** `wevibe-server/wevibe-dashboard/lib/wevibe-submit.ts` MUST NOT include `plaintext` in the submit payload to the hub. The dashboard's submit flow is overhauled to match the MCP's flow: encrypt locally, sign canonical body locally, submit only `(ciphertext, wrapped_dek_mod, plaintext_hash, salt, ciphertext_hash, contributor_pubkey, contributor_sig, …)` to the hub. The plaintext is never sent over the network.

**Why this is part of the verification anchor design and not a separate bug:**

The verification anchor's threat model assumes the hub does not see plaintext. The MCP client honors this; the dashboard violates it (GO-001 Task D OPEN ITEMS). If the dashboard sends plaintext, then:

- The hub's `sanitization_findings` scan operates on plaintext (which is the only reason the dashboard currently sends it — for hub-side scanning).
- A honest-but-curious hub sees every dashboard-submitted memory in cleartext.
- A captured hub harvests plaintext from every dashboard submission.

The sanitization scan must move client-side (run inside the dashboard before encryption) or be deferred (run by the moderator at decryption time). Either path is acceptable. Sending plaintext to the hub is not.

**Reference:** GO-001 Task D OPEN ITEMS — "Dashboard payload includes plaintext as cleartext over TLS to the hub; MCP does not."

---

### D-VR-8: No ZK Cryptography in Production Path

**Decision:** No production component (chain, hub, dashboard, MCP, plugin, sidecar) depends on a zero-knowledge proof system. SP1, zkVM, Groth16, the previously-scoped `wevibe-prover` local service, the previously-scoped `wevibe-veproof-sidecar`, and the previously-scoped chain-side `wevibe-veproof-verifier-service` are removed from the architecture. The `spike-aead-ve/` workspace remains as historical record but is not committed to any production repo (R-ISOLATED-WORKSPACE) and produces no shipping artifact.

**Why explicitly documented as a non-decision:**

DMO-028 locked the ZK pathway. CO-028 spike validated and then unblocked walking away from it. Without explicit documentation that ZK is out, future contributors reading the spike's positive feasibility numbers might re-propose the pathway. This decision exists to prevent that — the verification anchor is signatures, full stop, and the spike's "GO-WITH-RESERVATIONS" is decisively superseded.

**Reference:** Walter directive 2026-05-27 ("we redesign everything"), CO-028 feasibility report Section 8 (memory ceiling observations), this DMO.

---

### D-VR-9: Tier 2 Public Expose Loop — Reporter-Signed, Chain-Timed, Chain-Verified (DESIGNED, NOT BUILT)

**Status:** DESIGNED. NOT BUILT. Tracked as GAP-TIER2-EXPOSE.

**Context.** The verification anchor (D-VR-1 through D-VR-8) is implemented and live: the contributor signs a canonical body binding `plaintext_hash`, `salt`, and `ciphertext_hash`; the chain verifies that signature at commit; and the fields are stored on-chain so a revealed memory can be checked against the anchor. The anchor is the FOUNDATION for Tier 2, not Tier 2 itself. The user-facing accountability loop built on it — a reporter-driven public exposure that bypasses a captured or absent leader, under a chain-enforced timer — is not implemented. Today the only on-chain report message is leader-gated (signer must equal the org leader wallet, per D-S32-CO044 key separation); there is no on-chain response-window timer; the reveal hash check runs only as the off-chain `VerifyUpheldReport` query rather than as an enforced write gate; and the reported-memory state transition (`MEMORY_STATE_REPORTED_DELETED`) is never set. Net effect: the chain does not yet deliver censorship-resistant Tier 2 — a captured leader can suppress any report.

**Decision (target architecture).** Tier 2 is a two-transaction, reporter-signed, chain-enforced loop:

1. **File (reporter-signed, no reveal).** The reporter signs an on-chain report naming the memory and reason. This starts a chain-enforced response window (epoch-based deadline) and is a permanent, unsuppressable record. No plaintext is revealed.
2. **Window.** The org resolves through the Tier 1 path (uphold → memory removed from retrieval, deleted terminal state), which closes the window. A dismissal does not close it.
3. **Expose (reporter-signed, gated, final).** If the window elapses unresolved, the chain unlocks the reporter's expose transaction. The reporter reveals `(plaintext, salt)`; the chain ENFORCES `sha256(salt || plaintext) == on-chain plaintext_hash` and the contributor signature over the canonical body before transitioning the memory to an exposed/deleted terminal state. After exposure the leader may publish exactly one signed dispute counter-statement.

The chain is the sole enforcer: it owns the timer, gatekeeps the expose, and verifies the reveal. The reporter signs with their own key and pays their own gas; this does not weaken D-S32-CO044 (the reporter's key is neither the serving key nor the leader wallet, and the new messages are explicitly NOT on the leader-only list). The reveal is placed last because chain data is public and revealing the plaintext is the irreversible step.

**Launch broadcast + threat model.** The MCP/plugin client ships a two-entry broadcast path (primary + retry fallback) for the report/expose transactions. At launch both entries are the WeVibe-operated hub relay; the two-slot design is forward-compatible so that when orgs run their own relays, slot 1 becomes the org relay and slot 2 remains the WeVibe relay as the backstop that bypasses a blocking org. The launch threat model is "captured org, trusted WeVibe backstop" — it defends against a future captured org-run relay, NOT against WeVibe-operated infrastructure itself. Trustless resistance independent of WeVibe infrastructure, org-run relays, and multi-org subscription are roadmap. At launch a user subscribes to exactly one org.

**Open parameters (NOT yet locked):** reporter gas model (self-fund vs fee-grant vs bond); response-window length; expose-signing surface (wallet vs delegate); whether every report is filed on-chain immediately vs hub-first escalation; oversized-plaintext reveal handling; whether the chain gate requires roster membership only (paid status remains hub-side).

**Build surface:** new reporter-signed chain messages (file + expose) with the reporter as signer; a `ReportWindow` state + epoch-based deadline checked in the existing `AfterEpochEnd` hook; on-chain reveal verification (porting the existing anchor check into an enforced write); a report-filed event for moderator notification; new chain queries for report windows / reports-by-reporter; relay whitelist entry + reporter authz grant; MCP two-entry broadcast + retry + local report state; plugin report action; dashboard Reports tab (failed / pending / expose / resolved). Also corrects two latent defects: an upheld report must blacklist the memory and must NOT increment the contributor's approved-memory count.

**Supersedes:** the implicit assumption (whitepaper §5.5/§5.6/§8) that Tier 2 escalation is live. Amends the operative reading of D-7.4/D-7.5 — the chain gains a reporter-driven bypass, not only a leader-records-outcome path.

---

### D-S32-CO044-APPROVEMEM-SOFTFAIL — Contributor sig check soft-fails in batch approval

**Decision:** The contributor signature verification inside `ApproveMemory` (chain keeper) returns
success without committing the individual memory when the signature is invalid. The memory stays
in `pending` state. The batch tx (up to 500 memories per D-6.6) is NOT rolled back.

**Why soft-fail:** Approvals commit as atomic multi-message batches. A hard error on one bad
contributor sig would roll back all 499 other valid approvals in the same tx. The hub pre-verifies
contributor signatures at submit time (D-VR-5, D-VR-6), so an on-chain sig failure means genuine
corruption between hub and chain (tampering, relay corruption, bug) — not normal operation.

**Detection:** The leader can detect a dropped memory by comparing batch count vs committed count.
A future CO may add a chain event (`EventApprovalSigFailed`) so the hub watcher can surface the
drop as a leader notification. The silently-dropped memory remains in `pending` state and can be
re-submitted.

**Why not hard-error:** The cost (499 good approvals rolled back) exceeds the benefit (immediate
error signal) given that the hub pre-verification makes this path fire only on genuine faults.

---

## 17. Sprint 30 Deferred Decisions

### D-2026-05-25-B: Leader Activity Aggregation — Deferred

**Status:** DEFERRED

**Extends:** D-2026-05-25-A

**Context:**
- Sprint 30 CO-013 removed `leader_last_chain_commit_at` from `GetOrg` because it was misleading.
- The removed field was org-level `last_chain_submission_at`, not per-leader activity.
- Walter deferred final definition of "leader last active" until leader action taxonomy is finalized.
- Candidate leader actions include denial batch submissions, memory approvals, user invites, report submissions, credit top-ups, org config changes, plus TBD actions.

**Decision:**
- Per-leader RPC-queryable activity aggregation is deferred to a future sprint.
- This deferral is intentional to avoid rework before the leader action taxonomy is locked.
- BlockResults (hub reads from CometBFT block events) already provides raw `committing_leader_pubkey` data; the deferred scope is the query/API surface and formal action definition.

---

## 18. Sprint 32 Decisions: Memory Decay Activation + Tokenomics

This section records decisions from Sprint 32 ("Memory Decay: Earned Trust + Probabilistic Retrieval"). CO-040 delivered genesis activation + epoch-hook resilience and, in doing so, surfaced the true root cause of the Sprint-31 "zero decay" symptom (D-S32-CACHEKV-ITER). The tokenomics overhaul and the cachekv fix are rolled into CO-041.

### D-S32-EMISSION-POOL-GENESIS

**Decision:** The emissions module seeds an `EmissionPool` at chain genesis. The pool is derived from `emissions/types.DefaultParams()` via `DefaultEmissionPool()` (single source of truth), and `init-chain.sh` makes the genesis key present by seeding `app_state.emissions = {}` (the module's `InitGenesis` fills the pool from `DefaultParams` when none is supplied). Implemented in CO-040.

**Why:** Before CO-040 no pool was written at genesis, so the emissions epoch hook logged "no emission pool found" every epoch and never minted. The previous design assumed a module `DefaultGenesis` would auto-seed it, but `wevibed init` builds genesis.json from `app.ModuleBasics` (encoding.go), which contains ONLY the SDK modules — the custom WeVibe modules are absent, so their `app_state` keys never appeared and `ModuleManager.InitGenesis` skipped any module whose genesis data is nil. Seeding the key in `init-chain.sh` plus implementing `module.HasGenesis` (D-S32-HASGENESIS-CUSTOM-MODULES) is the load-bearing combination.

---

### D-S32-HASGENESIS-CUSTOM-MODULES

**Decision:** Custom WeVibe modules that require genesis state implement `cosmos-sdk/types/module.HasGenesis` (the legacy stateful genesis interface): `DefaultGenesis(codec.JSONCodec)`, `ValidateGenesis(...)`, `InitGenesis(sdk.Context, codec.JSONCodec, json.RawMessage)`, `ExportGenesis(...)`. CO-040 wired this for `x/emissions` and `x/reputation`. Both modules were already present in `app.go SetOrderInitGenesis`, so no ordering change was needed. Genesis state for these Go structs is marshaled with `encoding/json` (they are not proto messages); the `codec.JSONCodec` argument is unused.

**Why:** The SDK `ModuleManager.InitGenesis` dispatches on `appmodule.HasGenesis` → `module.HasGenesis` → `module.HasABCIGenesis`. The previous modules implemented only the `appmodule.AppModule` marker, so their genesis path was silently skipped (the root cause noted in app.go's CO-005d comment). `module.HasGenesis` is the minimal correct interface for a non-validator module and is fully dispatched by both `ModuleManager` (via `m.Modules[name]`) and `BasicManager` (via `coreAppModuleBasicAdaptor`, which forwards to `HasGenesisBasics`).

**Known follow-up:** Because `app.ModuleBasics` (encoding.go) does not contain the custom modules, `wevibed init` does not auto-write their default genesis; `init-chain.sh` jq-seeds the keys instead. A future cleanup could add custom-module basics to `ModuleBasics` to make `wevibed init` self-seeding and drop the jq seeds. Not done now (out of CO-040 scope).

---

### D-S32-REPUTATION-DEFAULTGENESIS-ACTIVE

**Decision:** `reputation/types.DefaultGenesis()` returns `Active: true`, and reputation `InitGenesis` persists both the active flag and `DefaultParams()`. Implemented in CO-040 (GAP-REP-1).

**Why:** D-13.5 already set `DefaultParams.Active = true`, but the module's `DefaultGenesis` returned `Active: false`, so the chain launched with reputation inert despite the params claiming otherwise — payouts would silently use tier=0 for everyone. This decision aligns `DefaultGenesis` with `DefaultParams` and D-13.5. Active state is an explicit genesis decision, so `init-chain.sh` seeds `app_state.reputation = {"active": true}` rather than relying on a zero-value default.

---

### D-S32-EPOCH-HOOK-RESILIENCE (R-EPOCH-HOOK-RESILIENCE)

**Decision:** No epoch hook (`AfterEpochEnd`/`BeforeEpochStart`) may return a non-nil error for a recoverable condition. All recoverable failures (missing data, empty iteration, no qualifying contributors) log a warning and return nil. Each step within a hook is independent: a failure in one step must not skip the others (e.g. a `setCurrentEpoch` failure must not prevent `ApplyEpochDecay`). Implemented in CO-040 for `x/memory` and verified for `x/emissions`.

**Why:** The Cosmos SDK epoch dispatcher runs all epoch hooks inside one cached-write batch and discards ALL cached writes for the batch if any hook returns a non-nil error. A recoverable error in one hook would therefore silently roll back unrelated successful work (e.g. `ApplyEpochDecay`'s weight changes) — the mechanism originally suspected for the Sprint-31 "zero decay." The resilience change is also what made the real cause (D-S32-CACHEKV-ITER) visible in logs instead of being silently rolled back.

---

### D-S32-CACHEKV-ITER (R-CACHEKV-ITER) — root cause of "zero decay" [LOAD-BEARING]

**Decision:** Keeper iteration code MUST NOT use the post-loop pattern
`for iter.Valid() { … }; if err := iter.Error(); err != nil { return err }`, and MUST NOT write to or delete from a store while iterating that same store. The corrected patterns are: (1) rely on the `Valid()` loop to terminate (a genuine parent-store error panics via `assertValid()` inside `Next()/Key()/Value()`); (2) collect-then-mutate for any iterate-and-modify path (gather keys/values, `Close()`, then mutate). New iterations must follow this from the start. Discovered in CO-040; the systemic sweep + fix is scoped to CO-041.

**Why:** `cosmossdk.io/store@v1.1.2 cachekv/internal/mergeiterator.go` defines
```go
func (iter *cacheMergeIterator) Error() error {
    if !iter.Valid() { return errors.New("invalid cacheMergeIterator") }
    return nil
}
```
On a **cache-wrapped** KV store — which is the store every BeginBlock / epoch-hook / branched-tx path uses — `iter.Error()` returns a NON-NIL error at NORMAL end-of-iteration (the iterator is exhausted, so `!Valid()`). Direct IAVL stores (used by unit tests via `rootmulti`) return nil at end, which is why the entire test suite was green while the live chain failed. The WeVibe keepers added `iter.Error()` failure checks at 24 sites across 10 keeper files (emissions, memory, org, reputation); under the epoch hook this turned every iteration into a false failure:
```
INF failed to get orgs error="invalid cacheMergeIterator"
WRN epoch hook: apply epoch decay failed ... error="iterate approved memories: invalid cacheMergeIterator"
WRN epoch hook: check epoch expiry failed ... error="iterate validity metadata: invalid cacheMergeIterator"
```
So `ApplyEpochDecay` never ran on the live chain; the only decay observed in CO-040's smoke came from the event-time serve/denial path (single-key, no iteration). Additionally, `ApplyEpochDecay` (lifecycle.go) and `CheckEpochExpiry` (validity.go) write/delete while iterating the same prefix, which is independently unsafe on cachekv.

**Why it surfaced in CO-040, not earlier:** the emissions epoch hook previously returned early at "no emission pool found" BEFORE reaching `GetAllOrgs`. Seeding the pool (D-S32-EMISSION-POOL-GENESIS) advanced execution past the mint step and exposed the next broken iteration. The bug is pre-existing; CO-040 made it observable.

**Process lesson:** unit tests over a direct IAVL store do NOT exercise `cacheMergeIterator`. Any test that must validate epoch-hook / BeginBlock iteration behavior MUST wrap the store in `cachekv.NewStore` (regression test mandated in CO-041 Task A).

---

### D-S32-TOKENOMICS-LOCKED — 32-Year Emission Schedule [SCHEDULED: CO-041]

> **[CANON CROSS-REFERENCE]** D-ECON-CANON consolidates payout-source and serve/retrieval-exclusion rules; this entry remains the locked schedule constants.

**Decision (locked constants; implementation in CO-041):**
```
Total supply:           1,000,000,000 VIBE   (1,000,000,000,000,000 uvibe; 10^6 uvibe per VIBE)
Foundation genesis:     10%  = 100,000,000 VIBE    (unlocked at genesis)
Validator genesis:      1%   = 10,000,000 VIBE     (to docker validator)
Contributor 32yr pool:  1%/yr × 32yr = 320,000,000 VIBE
Validator 32yr pool:    remainder = 570,000,000 VIBE (emitted over 32 years)
Contributor emission:   10,000,000 VIBE/year cap, split evenly among qualifying contributors
Qualifying:             ≥ contributor_qualify_threshold approved memories network-wide per epoch (default 1)
Rollover:               global bucket — if nobody qualifies the epoch budget rolls forward; the integer
                        remainder of an even split also carries forward (no token loss)
Schedule length:        11,680 epochs (32 × 365)
```
New emissions params: `total_supply_uvibe`, `validator_emission_pool_uvibe`, `contributor_annual_cap_uvibe`, `schedule_duration_days`, `contributor_qualify_threshold`. New `StoredEmissionPool` fields: `validator_pool_remaining_uvibe`, `contributor_pool_remaining_uvibe`, `contributor_rollover_uvibe`, `start_epoch`, `total_epochs_elapsed`. `DefaultParams()` remains the single source of truth for schedule constants; `init-chain.sh` seeds the full 32-year pool at genesis.

**Why:** This is the real economic model deferred out of CO-040 (originally "CO-040b"). It replaces the flat `daily_mint` placeholder with a validator pool + contributor pool emitted over 32 years, a per-year contributor cap, and a global rollover so unclaimed contributor budget is preserved rather than burned. Per-epoch: validator emission = `validator_pool_remaining / remaining_epochs`; contributor budget = `min(contributor_pool_remaining / remaining_epochs, annual_cap / epochs_per_year)`.

**Cascade (mapped in CO-041):** proto regen → keys.go conversion helpers + `DefaultParams`/`DefaultEmissionPool` → `GetEmissionPool`/`SetEmissionPool`/`InitGenesis`/`ExportGenesis` → epoch emission logic → contributor distribution, which queries memory network-wide by epoch and therefore DEPENDS on the D-S32-CACHEKV-ITER fix landing first.

---

### D-S32-CONTRIBUTOR-ATTRIBUTION — Address Persisted Through Memory State [SCHEDULED: CO-041]

**Decision (implementation in CO-041):** Persist the contributor's address through the memory lifecycle and derive serve attribution from the authoritative stored record:
- Add `contributor_address` to `StoredPendingCommitment` (field 8) and `StoredMemoryCommitment` (field 23).
- `SubmitCommitment` persists `msg.contributor_wallet` into pending; `ApproveMemory` copies it into the committed record.
- Serve attribution: `x/serve` `RecordServe` credits the served memory's `contributor_address` (looked up from `x/memory`) instead of trusting the serve payload's `contributor_wallet`.
- Add `x/memory` `GetContributorsWithApprovalsInEpoch(ctx, epoch)` (distinct qualifying contributor addresses network-wide) consumed by the emissions contributor distribution.

**Why:** Contributor emissions (D-S32-TOKENOMICS-LOCKED) must credit the real author network-wide, and the serve payload's wallet is consumer-supplied and therefore untrustworthy. The committed memory record is the chain-authoritative source. This closes the attribution gap between "who is paid" and "who actually contributed the served memory."

---

## Sprint 32+ — CO-044: Multi-Org Broadcasting, Least-Privilege Key Separation, Gas Faucet

This section records the locked architecture for hub multi-tenancy and per-org key security.
It is the canonical end-goal for org key custody and on-chain authority. Where earlier decisions
described the dogfood single-account / hub-as-global-authority model, those are superseded here.

### D-S32-CO044-KEY-SEPARATION — Two on-chain authorities per org [LOAD-BEARING]

**Decision:** An org has exactly two on-chain signing authorities, with disjoint powers:

1. **Org serving key** — per-org, HUB-HELD. HD-derived from the hub master mnemonic at
   `m/44'/118'/0'/0/{account_index}`, where `account_index` is a DB-assigned monotonic value from a
   Postgres sequence (`org_chain_accounts.account_index`, UNIQUE; NEVER hash-derived — collision
   risk). It signs **serves and denials ONLY** (`MsgSubmitServeBatch`, `MsgSubmitDenialBatch`). It
   is registered on-chain and is leader-revocable (D-S32-CO044-SERVING-KEY-REVOCATION).

2. **Leader wallet** — held by the leader/org, NOT the hub. Multisig-capable (cosmos k-of-n). It is
   the on-chain authority for **all org decisions**: `MsgApproveMemory`, `MsgSubmitCommitment`,
   `MsgReportMemory`, `MsgRegisterOrg`, and org governance (`MsgRotateEpoch`,
   `MsgTransferLeadership`, member ops, `MsgSetOrgConfig`, `MsgSetRepTiers`, `MsgSetServingKey`).
   Org-decision txs are SUBMITTED THROUGH the hub's delegate/authz relay (`internal/relay`); the hub
   relays leader-signed bytes and NEVER signs org decisions with its own key.

**Hub validation only (off-chain, unchanged):** quorum thresholds (`required_approvals`),
membership/moderator roles, and vote process remain hub-enforced operational state (D-6.3, D-6.4).

**Blast radius (the spine of CO-044):** if the hub is compromised and a per-org serving key is
stolen, the attacker can do exactly two things: (a) submit serve/denial batches (accepted only when
the tx signer equals the org's currently-registered serving key), and (b) drain that key's gas.
The attacker CANNOT approve memories, commit, file reports, register/govern orgs, or act for any
other org — all of those require the leader wallet, which the hub does not hold.

**Why:** The dogfood hub signed every org's every tx with one global "submitter" key, making the hub
an implicit global authority and serializing all orgs behind one account sequence
(D-S29-HUB-SEQUENCE-RACE). Splitting authority (a) eliminates the global-authority blast radius and
(b) gives each org an independent account/sequence, enabling true multi-org concurrency.

---

### D-S32-CO044-LEADER-DUAL-PATH — Two leader signing paths [EXPLICIT R-ONE-PATH EXCEPTION]

> **[UPDATE 2026-06-05]** The CO-214 client delegate-key + hub relay implementation described in this historical section is **REMOVED** and superseded by amendment 13 (**D-ECON-STORAGE-MARKET**). Current path: leaders/members sign chain txs wallet-direct (Keplr/Leap + CosmJS `directBroadcast`), and the hub records data only.

**Decision:** The leader may authorize org-decision txs via EITHER of two paths, both supported
simultaneously. This is a deliberate, manager-directed exception to R-ONE-PATH for this order:

1. **Delegate path** — the leader's wallet grants cosmos `authz` to a delegate session key; the
   delegate key signs a `MsgExec`-wrapped org-decision msg with `granter = leader wallet`; the hub
   relays it (`internal/relay/relay.go`, `Delegate <sig>` scheme, granter-must-match check). For
   high-volume routine signing (e.g. batch approvals).
2. **Non-delegate (direct) path** — the leader's wallet signs the org-decision tx directly (no
   delegate key); the hub relays the signed bytes.

In both paths the leader wallet is the on-chain authority and the hub is pure transport + fee
relay. The operations enumerated in **D-1.3** (report commitment, leadership transfer, org closure,
`required_approvals`/`report_vote_threshold` changes) MUST use the direct wallet path — they are too
consequential for a delegate key. Both paths support a multisig leader wallet natively.

**Why:** Different orgs have different security postures. Routine, high-frequency approvals benefit
from a delegate session key (no wallet popup per action, D-1.1/D-1.3); consequential or
security-critical actions, and orgs that decline delegate keys entirely, sign directly with the
(possibly multisig) wallet. Supporting both is worth the two-path cost; the alternative (forcing one)
either weakens routine UX or weakens security-critical actions.

---

### D-S32-CO044-SERVING-KEY-REVOCATION — Leader-revocable serving key + accepted residual risk

**Decision:** The org's serving-key address is stored on-chain (`StoredOrg.hub_serving_address`,
populated from `MsgRegisterOrg.hub_serving_key`) and is rotatable/revocable by the leader via a
leader-authorized `MsgSetServingKey`. `SubmitServeBatch`/`SubmitDenialBatch` are accepted ONLY if the
tx signer equals the org's currently-registered serving key.

**Accepted residual risk (documented):** serve/denial CONTENT forged by a live, not-yet-revoked
serving key — and any token emissions those forged serves trigger before revocation — is NOT
prevented in this order. There is no per-serve consumer proof and no rollback engine. Containment is
leader revocation of the serving key.

**Why not rollback / why not per-serve proofs (now):** A forged serve fans out into memory decay
weight (frozen CO-042 model), reputation, epoch stats, and — at epoch rotation — minted, distributed
emissions (`x/emissions` `MintDailyEmission`/`DistributePayout`). Tokens minted and withdrawn before
detection cannot be clawed back, so a serve-attestation rollback cannot restore correctness; a
cross-module compensation engine is disproportionate and still incomplete. Per-serve consumer proofs
would prevent forgery cryptographically but touch the consumer/MCP retrieval + nullifier + SDK
surface; an emissions settle/clawback window would touch the FROZEN CO-042 decay/emissions surface
(R-DECAY-FROZEN). Both are deferred. Revocation bounds the damage at the right cost.

---

### D-S32-CO044-GAS-FAUCET — Standalone faucet service funds per-org accounts

**Decision:** A new top-level `faucet/` standalone Go service holds one funded chain account and
exposes `POST /v1/fund { address, amount }` (bank-send uvibe; idempotency + rate limiting; returns
tx hash) plus a health endpoint, with serialized single-account nonce management. The hub calls it to
fund a new/low per-org serving account; if the faucet is unavailable the hub operation is a HARD
ERROR (no fallback). The faucet account is funded from the WeVibe validator account at chain init via
a validator→faucet bank send in `scripts/init-chain.sh`; the faucet address is deterministic.

**Why:** Per-org accounts (D-S32-CO044-KEY-SEPARATION) must exist and hold gas on-chain before they
can pay for `RegisterOrg`/serve/denial txs. A faucet is the deterministic way to create+fund them
without minting from a chain module. The validator holds the 1% genesis allocation
(D-S32-TOKENOMICS-LOCKED) and forwards uvibe to the faucet.

---

### D-S32-CO044-REGISTERORG-FLOW — Leader signs RegisterOrg after self-funding via faucet

**Decision:** The hub NEVER signs `MsgRegisterOrg` or any other org decision. The org creation
flow is: (1) leader connects wallet in dashboard, (2) dashboard calls faucet `POST /v1/fund`
to fund the leader's wallet with gas, (3) dashboard calls hub `POST /v1/orgs` which creates the
org in PostgreSQL, derives the per-org serving key, funds it via faucet, and returns the
`hub_serving_key_address`, (4) dashboard builds `MsgRegisterOrg` with the leader's wallet as
signer and the hub-returned serving key address, (5) leader signs with Keplr/Leap, (6) dashboard
relays the signed tx through the hub relay endpoint.

**Why no bootstrap exception:** A bootstrap exception ("hub signs RegisterOrg because the leader
wallet has no gas yet") would make the hub an org-decision signer for one operation, violating
D-S32-CO044-KEY-SEPARATION. The faucet exists precisely to solve the gas-bootstrapping problem.
The leader funds their own wallet before signing. One path, no exceptions (R-ONE-PATH).

---

### D-S32-CO044-PER-ORG-BROADCAST — Per-org signer registry [SUPERSEDES D-S29-HUB-SEQUENCE-RACE]

**Decision:** The hub replaces the single `submitter` with a per-org signer registry
(`map[orgID]*orgSigner`, each holding its own keyring uid, address, account number, next sequence,
and its OWN mutex). `BroadcastMsgsForOrg(ctx, orgID, msgs)` acquires that org's mutex, ensures the
account is funded (balance check → faucet top-up; hard error if faucet fails), builds/simulates/signs
with that org's sequence, broadcasts, increments on success, and resyncs-from-chain + retries once on
sequence error. Different orgs broadcast in PARALLEL; one org serializes on its own mutex. The serve
relay worker pool is restored to >1 (safe because the `serveRelayQueued` dedup guarantees one worker
per org and each worker drives a distinct account).

Serves/denials keep BLOCKING commit (`broadcast_tx_commit`) to preserve the CO-042 decay
settle-window invariant (drain-pacing relies on relay broadcasts being committed when
`pending_total` hits 0). Intra-org sync-broadcast pipelining is separately gated and MUST NOT ship
without a fresh decay-gate validation.

**Why:** The single shared account sequence (D-S29-HUB-SEQUENCE-RACE) made concurrent broadcasts
collide (`incorrect account sequence`) and forced the relay pool to 1 worker, making multi-org
hosting impossible. Per-org accounts give each org an independent sequence — the correct fix the
max-of-two cache only approximated.

---

### D-S32-CO047-SUBSCRIPTION-CREDITS — Credits are a hub-internal subscription gate [SUPERSEDES the per-query deduction in D-3.1 / GAP-O6]

> **⚠️ AMENDED (Walter, 2026-06-07) — ADMISSION IS DECOUPLED FROM CREDITS.** The clause below
> ("A member is admitted by subscribing … `Subscribe` … debits `SubscriptionCost`") is **OVERRIDDEN**.
> Member acceptance must NOT touch hub credits. **Accepting a member is an on-chain `MsgAddMember`
> (leader-signed), gated solely by the org having VIBE to cover gas** (paid via the org-account
> feegrant — see `D-S32-ORG-FEEGRANT-ALL`). Hub credits gate **recall only** (`membership_active`,
> read at `retrieval.go`), never admission and never contribution (contribution is free + wallet-free
> per `D-IDENTITY-PROGRESSIVE-CUSTODY` / `D-LEADER-SOLE-SIGNER`). Implemented: the `billing.Subscribe`
> call + 402 was removed from `ApproveJoinRequest` and `InviteMember`; a freshly-admitted member is
> `active=true, membership_active=false` (admitted, recall not yet enabled). `billing.Subscribe`
> survives as the recall-activation primitive for a future explicit "enable recall" surface (NOT
> bundled into admission). The "org pool seeded from `fee_model.monthly_credits`" provisioning remains
> for the recall/demand-leg path. Map: `wevibe-meta/workspace/reports/gather-org-tx-key-gas-map.md`.

**Decision:** Hub credits are an internal PostgreSQL accounting pool per org (`org_credits.balance`,
hub-only, never on-chain). The org pool is seeded at creation from `fee_model.monthly_credits`
(`ProvisionOrgLedger`, recorded as a `subscription_grant` txn) and topped up via `TopUp`. A member is
admitted by subscribing: `Subscribe(orgID, memberPubkey)` atomically debits `SubscriptionCost` (=10)
from the org pool, sets `members.membership_active = TRUE`, and records a `subscription` txn;
insufficient balance → explicit `ErrInsufficientCredits` (HTTP 402), member row kept inactive. Recall
access for NON-TRIAL members is gated SOLELY on `membership_active`; trial members remain governed by
the orthogonal trial path (expiry + daily limit). The obsolete per-query `DeductQueryCredit` /
`QueryCost` path (1 credit per query, from the pre-pivot hub-memory model) is DELETED — there is no
per-query credit metering.

**Why:** Memories moved from the hub to the chain (the Sprint-28→32 pivot); the consumer no longer
"pays per query for a hub lookup." Access is a subscription a member buys from their org; the org's
prepaid credit pool funds admissions. The old per-query deduction seeded balances at 0 and violated
`CHECK (balance >= 0)` on the first query — it was both the wrong model and a live defect (660
constraint violations observed pre-fix). Gating on a boolean `membership_active` is the one-path
expression of "subscribed or not." (Landed: CO-047.)

---

### D-S32-ORG-FEEGRANT-ALL — Org account is the gas payer for ALL org-purpose on-chain txs [REFINES D-ECON-STORAGE-MARKET amendments 10–11]

**Decision (Walter, 2026-06-07):** The **org's on-chain account** (`org.AccountAddress`, capitalized by
the 50% acquisition retain + top-ups) is the **single gas source for every org-purpose transaction**,
via an on-chain **feegrant**. The **leader wallet signs** (authority) but never personally pays gas for
org work. The **one gate is the org VIBE balance** — a dry org account ⇒ org-purpose txs fail ("org needs
VIBE"). Realized by a `feegrant.AllowedMsgAllowance` granted org-account → leader-wallet at
`RegisterOrg` (and re-granted to the new leader on `TransferLeadership`), **scoped to the leader's
org-purpose msg type URLs** so org funds cannot pay for the leader's unrelated personal txs:
`x/org` MsgAddMember/RemoveMember/UpdateMemberRole/SetOrgConfig/SetServingKey/SetServingInfo/
RotateEpoch/TransferLeadership/CloseOrg/GrantTrialAllowance + `x/memory`
SubmitCommitment/ApproveMemory/ReportMemory. The dashboard sets `Fee.Granter = org account` on
`directBroadcast`. This mirrors the pre-existing serving-key feegrant (org account → `HubServingAddress`,
scoped to serve/deny batches).

**Boundaries:**
- **`MsgRegisterOrg` is the sole leader-personal cost** — there is no org account yet at creation; the
  leader's wallet pays the acquisition (50% burn / 50% → new org account) + that tx's gas. This is the
  leader's slot bond / skin-in-the-game.
- **Personal economic txs stay on the user's own wallet:** `MsgClaimContributorReward` and
  `MsgMigrateIdentity` are individual actions, NOT org operations — org funds do not cover them.
- **The three keys, delineated:** *leader wallet* (secp256k1) = authority/signer for org-auxiliary +
  adjudication + contribution-commit; *serving key* (secp256k1, `HubServingAddress`) = serve/deny batch
  signer (org-account feegrant); *handshake key* (ed25519, `hub_response_pubkey`) = HTTP response signing
  (`X-Hub-Signature`), off-chain, no gas, no chain identity.

**Why:** Keeps the leader's economic burden on the *slot* (acquisition + self-assessed-value rent +
forfeiture), not on per-action gas — a leader running a busy org shouldn't bleed personal VIBE per
member add / approval. Concentrating org gas on the org account makes "org VIBE balance" the single,
legible solvency gate and keeps key responsibilities clean (the recurring maintenance-nightmare risk
is muddy signer/payer coupling). Scoped allowance prevents the org account from being drained for
non-org txs. Full inventory + rationale: `wevibe-meta/workspace/reports/gather-org-tx-key-gas-map.md`.

**Implemented (not yet committed):** chain `grantLeaderFeegrant` (`x/org/keeper/serving_feegrant.go`,
called in `RegisterOrg`/`TransferLeadership`); dashboard `directBroadcast(…, feeGranter?)` +
`getOrgAccountAddress`; approve flow pays via org account. **Follow-ups:** (1) sweep the remaining
dashboard org-tx callers (member remove, role update, approve memory, report, config, serving, epoch)
to pass the org-account granter; (2) leadership-transfer does NOT revoke the old leader's allowance
(SDK feegrant keeper exposes no public revoke) — inert because every org msg gates on
`signer == current LeaderWalletAddress` and the allowance is msg-scoped, so a former leader can neither
pass auth nor use it for non-org txs; (3) a future explicit "enable recall" surface to set
`membership_active` (decoupled from admission). **Open inconsistency to unify:** two leader-auth
patterns coexist — address-check (`signer == LeaderWalletAddress`) vs role-check (`IsLeader`) — across
different x/org messages; they can diverge and should be unified.

---

### D-S32-CO047-TWO-KEY-GAS — Two hub-held per-org chain keys in the headless/dogfood model [REFINES D-S32-CO044-KEY-SEPARATION]

**Decision:** Each org has TWO HD-derived chain keys (distinct account indices from the hub master
mnemonic), BOTH faucet-funded at creation: `org-serving-{orgID}` signs serves/denials
(`MsgSubmitServeBatch` / `MsgSubmitDenialBatch`; registered on-chain as `HubServingAddress`) and
`org-leader-{orgID}` signs org decisions (`MsgSubmitCommitment` / `MsgApproveMemory` /
`MsgRegisterOrg`; registered on-chain as `LeaderWallet`). `org_chain_accounts` is role-keyed
(PK `(org_id, key_role)`, `key_role IN ('serving','leader')`). The chain already separates the two
authorities (`requireServingKeySigner` == `GetServingAddress`, `requireLeaderWallet` ==
`GetLeaderWallet`), so no chain change was needed — the prior conflation (hub passed the serving key
as BOTH arguments to `RegisterOrgOnChain`) was purely a hub bug.

**Why:** D-S32-CO044-KEY-SEPARATION envisions the leader wallet held OFF-hub (Keplr) signing via
relay. The headless dogfood/replay harness has no wallet, so in that environment the "leader
authority" is a second hub-held key — but it remains a DISTINCT, independently-fundable,
independently-revocable account from the serving key, preserving the blast-radius property (a stolen
serving key can only submit serve/denial batches + drain its own gas; it cannot approve, commit,
report, or register). The full off-hub leader wallet remains the production target; this is the
dogfood realization of the same separation, not a contradiction. (Landed: CO-047.)

---

### D-S32-CO048-EPOCH-COST — Block production was NOT regressed; the consensus floor is the SDK-default 5s `timeout_commit` [CORRECTED — CO-048 NO-OP; supersedes the original falsified mechanism]

**Decision (corrected by CO-048 measurement):** There is NO epoch-hook block-production regression.
Block interval is a flat ~5.01s set by the Cosmos SDK's default `timeout_commit = 5s`: cosmos-sdk
v0.53.5 `server/util.go:252-253` overrides cometbft v0.38.20's 1s default to 5s when the chain leaves
`TimeoutCommit` at the cometbft default, and `init-chain.sh` does not touch it. Under load (≈800
memories, 4× HEAVY) the per-epoch `AfterEpochEnd` work is negligible against that floor: x/memory decay
loop ~4-9ms, merkle ~2-6ms, emissions mint/payouts ~0-3ms, and total app `FinalizeBlock` execution
~12-31ms — i.e. tens of ms against a 5000ms commit floor.

**The "~15s" was a measurement artifact, not block time (Lesson 9).** It is the empirical-replay
harness's per-epoch CYCLE wall-clock (`traffic + drainServeRelay + waitEpochAdvance`), which spans
N blocks × 5s: STEADY ≈3 blocks ≈15s, HEAVY ≈5 blocks ≈25s, BOOTSTRAP ≈2 blocks ≈10s. It is throughput
pacing at a 5s cadence, not a block-production regression.

**Consequence:** the original entry's mechanism — a "~1s cometbft floor" that the epoch hook inflated to
~15s via redundant `approved/` keyspace walks plus a 2s fast-stack feedback loop — is FALSIFIED. The
optimizations it proposed (collapse `approved/` walks, scope the Merkle root, bulk-load per-epoch
serve/denial data) were ABANDONED under R-MEASURE-FIRST: they optimize a non-bottleneck (~tens of ms)
and would be churn against a 5s floor (R-LONGEVITY). CO-048 shipped NO code; `wevibe-chain` is pristine
at `8c92385`. Full verbatim evidence (block intervals, FinalizeBlock exec, hook timings, config) is in
`wevibe-meta/workspace/reports/CO-048-implementation-report.txt`.

**Still binding (independent of the falsified mechanism):**
- FORBIDDEN to mask cost by lengthening `WEVIBE_EPOCH_DURATION_SECONDS`, changing `IdleDecaySettleEpochs`
  (untouched at 5), or bounding / paginating / skipping per-epoch decay or emissions work — all change
  the decay/gate timeline and a half-settled chain is unacceptable (R-DECAY-FROZEN).
- IF a sub-5s block target is ever desired, it is a `timeout_commit` CONFIG decision (override
  `Consensus.TimeoutCommit` in `initCometBFTConfig()`), explicitly weighed against consensus stability —
  NOT an epoch-hook optimization.

**Why the original was wrong:** it measured the harness per-epoch cycle metric and mislabeled it as the
block interval, and it missed the SDK's 1s→5s `timeout_commit` override. CO-048's instrumentation
corrected both. (Scoped: CO-048 — closed NO-OP.)

---

### D-S32-CO048-MEMORY-TYPE-IMPL — Finish the half-applied single-memory-type migration to match D-5.1

**Decision:** Complete the D-5.1 single-memory-type migration end-to-end: (a) collapse the chain
`memory/v1` proto enum to a single `MEMORY_TYPE_MEMORY` (drop `MEMORY_TYPE_CORRECT_IMPLEMENTATION` and
`MEMORY_TYPE_NEGATIVE_SIGNAL`), regenerated via `make proto-gen` (R-PROTO-REGEN — Docker, no hand-edit
of `*.pb.go`); update `x/memory/keeper/msg_server.go` `canonicalMemoryType` / `ValidMemoryType`
accordingly. (b) Hub `internal/chain/submit.go` maps protocol `"memory"` → `MEMORY_TYPE_MEMORY`
(removing the stale `CORRECT_IMPLEMENTATION` mapping that surfaced
`memory_type=MEMORY_TYPE_CORRECT_IMPLEMENTATION` in watcher bookkeeping logs); `query.go` to the single
type. (c) MCP/plugin perform risk-appetite filtering on the `dnd` FIELD (`lowest` = `dnd != null` only;
`neutral` = all) — not on a type dichotomy — and extraction emits independent `implement` + `dnd`
fields; drop the dual `MemoryType` from MCP `types.ts` and dashboard `wevibe-submit.ts`. Canonical
signing already collapses to `"memory"`, so this changes NO signed bytes and does not touch decay.

**Why:** D-5.1 locked the single-type + internal `dnd` model, but the migration was only half-applied
(protocol layer + canonicalization migrated; chain enum + hub mapping + MCP filtering still carried the
obsolete dual-type model). The on-chain `CORRECT_IMPLEMENTATION` label and the MCP's type-level (rather
than `dnd`-field-level) filtering are the visible symptoms. This closes the gap so the code matches the
already-locked decision. (Scoped: CO-048; discovered during CO-047 execution.)

---

### D-9.5: Retrieval Fidelity — Remove Inherited Vector Noise + Scale Recall Depth (2026-06-01, Walter-approved)

**Decision (hub-side only — NO decay/chain change, R-DECAY-FROZEN untouched):**
1. **Stored-vector Gaussian noise is OFF by default (`σ = 0`).** The hub previously injected
   `N(0, 0.1·‖v‖)` into every memory embedding at upsert (`wevibe-hub/internal/retrieval/retrieval.go`
   `injectGaussianNoise`). This was **inherited verbatim from the Echo migration (commit MO-003)** with
   no rationale, no config knob, and no DECISIONS basis. It asymmetrically corrupted recall (stored side
   noised, query side clean).
2. **Vector recall depth scales with the org, capped at 5000** (was a fixed `30`). The vector DB returns
   `min(recallDepth, active)`, so below the cap the engine effectively ranks *all* of an org's active
   memories; the cap is the compute safety valve.

Both are env-configurable (`RETRIEVAL_VECTOR_NOISE_SIGMA`, `RETRIEVAL_RECALL_DEPTH`); defaults now `0` / `5000`.

**Why (empirical — measure-first, R-26):** Earned-Trust decay was proven equal to the canonical sim, yet the
live chain false-archived ~25% of good memories in the easy "steady" regime (gate-cell1: good-survival 0.7528,
gap 75.28pp vs sim 95.45pp; Δ −20.17pp, FAILING the |Δgap|≤5pp tokenomics-match). Diagnosis (firsthand chart of
retrieval.go) localized the loss to recall fidelity, NOT decay. The measure-first cell (steady/seed42/300ep,
σ=0 + depth=5000, all else identical) closed it: **good-survival 0.9438, gap 94.38pp, Δ −1.07pp → PASS.**
~95% of the 20pp (19.10pp) was these two fixable knobs; only ~1.07pp residual is the irreducible
"embedding-cosine ≠ exact-keyword-set-overlap" effect (to be modeled by the calibrated sim, see sim purpose).
D-9.4 probabilistic exploration + new-mem boost is already implemented on this path and is NOT the gap source.

**COMPUTE CAVEAT (do not forget — the scale at which this punishes us):** per-query cost is ~linear in the
number of candidates examined (vector-DB payload deserialization + keyword scoring + sort + probabilistic
sample, all ∝ recall depth), multiplied by query frequency. At alpha scale (hundreds to a few thousand active
memories/org) this is negligible — dominated by block production, not search (proven: steady ~100 and heavy
~1,900 memories both ~15s/epoch). **It begins to punish us when an org's ACTIVE memory count reaches the tens
of thousands**, where per-query latency + memory dominate. The `5000` cap covers every closed-alpha org and the
brutal heavy gate cell (~1,900). **Before any org approaches ~5,000 active memories we MUST instrument query
latency vs active-set size and likely move to a two-stage approximate-prefilter → rerank** rather than simply
raising the cap. This is the documented compute ceiling.

**Status:** implemented in the wevibe-server working tree (config.go, retrieval.go, docker-compose.yml);
**commit pending the full 4-seed × 3-regime matrix passing with these defaults.**

---

### D-RELAY-THROUGHPUT: Hub→Chain Serve Relay — Throughput From Batch Size, Not Concurrency (2026-06-02, Walter-approved)

**Decision (hub-side only — NO decay/chain change, R-DECAY-FROZEN untouched):** the hub→chain
serve/denial relay is overhauled to lift its measured throughput floor while preserving 100% landing
and coin-efficiency. Three coupled levers, all in `wevibe-hub`:

1. **Throughput comes from BATCH SIZE.** A relay pass for an org packs *multiple epochs'*
   `MsgSubmitServeBatch` / `MsgSubmitDenialBatch` messages into *one* multi-message commit-mode tx
   (`relayPendingEventsByOrg` in `internal/api/handlers/serves.go`; `SubmitRelayBatch` +
   `BuildServeBatchMsg`/`BuildDenialBatchMsg` in `internal/chain/submit.go`). Previously the relay sent
   one tx per epoch per kind, each blocking a full ~5s `broadcast_tx_commit` — sustaining only
   **~0.2 tx/sec** (measured 2026-06-02). One ~5s block-wait now amortizes across many epochs.
2. **Reliability + coin-efficiency come from SERIALIZED, just-in-time simulation** — *not* concurrency.
   One submitter per org-signer (`orgSigner.mu` already serializes); the tx is simulated immediately
   before broadcast against committed state (no concurrent writer), so the simulate-vs-execute race is
   absent and the gas buffer is dropped from **2× → 1.4× (7/5)** (`simulateBufferMultiplier*` in
   `internal/chain/broadcast.go`). `simulate-retry` is retained purely as a rare safety net.
3. **Confirmation model = commit-mode on fat batches (option A).** We keep `broadcast_tx_commit` so the
   "submitted == executed" invariant holds; the throughput win is entirely from batch size. The async
   `broadcast_tx_sync` + watcher model (option B) is explicitly DEFERRED unless production load demands it.

**Invariants preserved (LOAD-BEARING):**
- **Ordering:** within a single multi-msg tx the chain executes messages in order, so the relay assembles
  them per-epoch-ascending as serve-msg-then-denial-msg. This keeps the existing rule (for any epoch E the
  serve attestation lands before its denial; serveEpoch ≤ denialEpoch) and makes the whole pass *atomic*.
- **Bounded blast radius:** a `maxRelayMsgsPerTx = 200` cap flushes at epoch boundaries; one bad entry can
  only fail a bounded tx, and an epoch's serve+denial pair is never split across txs.
- **Single-flight:** the existing `serveRelayQueued`/`serveRelayMu` machinery guarantees one relay pass per
  org at a time; the multi-msg path relies on it. **No new concurrency is introduced.**

**Why serialized, NOT concurrent (institutional memory — the CO-042 scar; do NOT re-litigate):** a prior
*concurrent/batched* submit attempt (`simulate-buffer`, 2× buffer, no confirm) failed under load — per-tx
gas was simulated against state snapshot S, but by `DeliverTx` time other in-flight txs had mutated the
store, so write costs drifted **above** the estimate → out-of-gas panics *even at 2×*. The serialized
commit-mode relay was the defensive retreat for 100% landing. **The root cause was CONCURRENCY, not gas.**
This decision keeps serialization and gets throughput from batch size instead — so the 1.4× buffer is safe
*only because* submission is serialized. Any future re-introduction of concurrent submission re-opens the
gas-storm and is forbidden without a fresh Walter-approved order.

**Why this is also the legitimate test speedup (NOT a measurement hack):** the replay gate is a
pipeline-timing integration test; its result is sensitive to serve-landing-vs-epoch alignment (changing
block time moved steady-42 91pp→78.6pp — see Lesson 27). The harness waits for the relay to fully drain
(`pending_total==0`) **before** advancing the (time-based) epoch, so a faster *drain* changes wall-clock
only, not attribution — unlike a block-time change. **Block time and epoch duration MUST NOT change.**

**Mandatory validation:** after implementation, re-run the steady gate seeds and confirm the
good-survival gap stays in the canonical band before trusting any re-run.

**Validation result (2026-06-02, steady seed=42, 300-epoch cell, early-stop at e150):**
**PASS.** chain.gap=97.75, sim.gap=95.45, **delta=+2.30** (well inside ±5pp); good-survival 97.75%,
bad-persistence 0.00%; **every checkpoint flagged ON-TRACK** (the opposite of the block-time-hack
corruption, which dropped good-survival and flagged "archiving too fast"). The result moved ~+7pp vs
the pre-fix full run (chain.gap ~91pp, delta −4.44) but in the SAFE direction: |delta| *shrank*
(−4.44 → +2.30, a tighter match to the canonical sim) and good-survival went UP, not down — consistent
with the cleaner serialized just-in-time-simulate landing serve attestations reliably in the correct
epoch so fewer *good* memories falsely decay. NOTE: early-stop fired mid-gentle-decline (100→98.9→97.8),
so 97.75 slightly *over*-states the converged value; a full 300-epoch run would settle a touch lower
(nearer the original low-90s) — still a PASS, good-survival never degraded.

**Throughput note (HONEST — the win is PRODUCTION-only, NOT a test speedup):** the validation run still
measured **~0.2 tx/sec, ~1.0 tx/block, ~9.7s relay drain** — UNCHANGED. This is expected per §10 of the
design: the gate harness drains the relay to `pending_total==0` *before* advancing each epoch, so only
~one epoch (~1 serve + ~1 denial) is ever pending and the cross-epoch fat-batch lever never fires. The
batching win materializes only where serves accumulate continuously across epochs (production), not in
this drain-then-advance test. The slow heavy gate cells therefore remain slow under this harness; the
production throughput improvement is real but unmeasured by the gate.

**Status:** IMPLEMENTED + VALIDATED (steady-42 PASS, measurement-neutral, correctness slightly improved).
Closes GAP-RELAY-THROUGHPUT. Commit pending.

---

## 19. Social Graph Attribution, Economy Canon, and Attestation Roadmap

### D-SG-1: Serve/Retrieval Attribution Is a Social Signal, Not Economic

**Decision:**
- When a memory is served (retrieved + injected and recorded via the on-chain serve batch), it increments aggregate served counters for BOTH the contributing author and the org.
- Serve counters are aggregate-only. Individual memories are NOT collectible per-card social objects.
- Serve/retrieval counts are SOCIAL status/reputation signal only and MUST NEVER drive VIBE payout. The serve-attribution code is RETAINED and reclassified as social (not stripped).
- Source of truth is the chain (immutable serve/denial batches). The social graph displays chain truth.
- **Cross-reference:** Supersedes any prior serve/retrieval-based reward economics; see D-SG-3 (badges) and D-ECON-CANON (economy lock below).

---

### D-SG-2: Social Graph Is an Open-Source Display Layer over Chain RPC

**Decision:**
- The social graph is an OPEN-SOURCE, forkable, self-hostable DISPLAY client. It reads chain state via RPC and renders PUBLIC user profiles. Anyone may host/enhance their own version.
- Profiles are PUBLIC and show serve counts (contributor + org), reputation, and badges with a PER-ORG breakdown.
- There is NO cross-org leaderboard/ranking.
- Layering is explicit: Chain (source of truth) -> RPC (read contract) -> social graph (display). The chain exposes raw counts/inputs; the social graph renders.

---

### D-SG-3: Badges (Gamified Status, No Reward)

**Decision:**
- Badge families:
  - **serve-milestone:** your memories served N times.
  - **rarity-tier:** per-memory keyword supply/demand, computed once at commit and frozen ON-CHAIN (see GAP-RARITY-1).
  - **contribution-volume:** approved-memory count.
- Badges are earned PER-ORG (e.g., "Legendary in OrgX"); profile display includes a per-org breakdown. No cross-org leaderboard.
- Criteria location:
  - **On-chain:** rarity tier.
  - **Canonical spec:** serve-milestone and contribution-volume criteria/thresholds, computed by the reference open-source social graph from chain RPC inputs.
- Canonical-spec thresholds keep badge tiers (e.g., "Legendary") consistent across forked social-graph implementations.
- Badges are STATUS-ONLY: no VIBE reward, no emissions, and no rep-tier payout coupling.

**ALPHA / PUBLIC-TESTNET SPEC (`⚠ PROVISIONAL-ALPHA` — RETUNE BEFORE MAINNET):**

Locked 2026-06-02 (Walter). These thresholds are tuned for low testnet volume so badges are actually earnable. Mainnet volume will be orders of magnitude higher; EVERY number below must be revisited before mainnet. Status-only, per-org, no cross-org leaderboard (D-SG-2).

- **Computation model (alpha):** ALL THREE families are computed READ-TIME by the reference open-source social-graph display client from chain RPC inputs (per D-SG-2 "display client over RPC"). This SUPERSEDES, for alpha, the earlier "rarity tier frozen on-chain at commit" — on-chain freeze of rarity is deferred to a mainnet refinement (GAP-RARITY-1). No on-chain badge entity exists today (rarity exists only as a numeric `rarity_multiplier` in emissions `WorkScore`).

- **Serve-milestone (per org)** — from `GetContributorProfile.serve_count` (per-org serves of the contributor's memories):
  | Tier | Serves |
  |---|---|
  | Bronze | 10 |
  | Silver | 50 |
  | Gold | 250 |
  | Platinum | 1,000 |

- **Contribution-volume (per org)** — from `GetContributorProfile.memory_count` (approved memories):
  | Tier | Approved memories |
  |---|---|
  | Contributor | 1 |
  | Builder | 10 |
  | Maintainer | 25 |
  | Pillar | 100 |

- **Rarity-tier (per org)** — per-memory; a contributor's rarity badge = their highest-rarity memory in the org. Simplified alpha supply/demand rule (full per-keyword transform is the mainnet GAP-RARITY-1 refinement): for each keyword `k` on a memory, `supply_k` = # approved memories in the org carrying `k`; `demand_k` = total serves on memories carrying `k`. A memory qualifies on keyword `k` only if `supply_k ≤ 3` AND `demand_k ≥ 10`. Tier set by `demand_k`:
  | Tier | demand_k (serves), with supply_k ≤ 3 |
  |---|---|
  | Uncommon | 10 |
  | Rare | 30 |
  | Legendary | 100 |

---

### D-ECON-CANON: Canonical VIBE Economy (Consolidation Lock)

> **⚠️ OPEN — CUSTODY MODEL REOPENED (Walter, 2026-06).** This entry still describes leader revenue settling to an **org treasury** withdrawn via `MsgWithdrawTreasury`, but amendment 10 of `D-ECON-STORAGE-MARKET` already **removed** `Treasury`/`MsgWithdrawTreasury` (R-OVERHAUL), and the code has no withdrawal path. Walter's standing position: **WeVibe must be non-custodial** — it holds no user/org funds and needs only a burn-enforcing payment **router**; enforcement implies gating org subscriptions through that router, which conflicts with the hub currently managing subscriptions/payments. The custody question (non-custodial router-to-leader-wallet vs. network-held per-org account), the contributor-claim and leader-withdrawal mechanisms (wallet-link precondition + reentrancy + double-claim guards), and the hub-vs-chain ownership of subscriptions are **OPEN pending Walter resolution**. See WHITEPAPER §13 + §10.5. Do not treat the treasury/`MsgWithdrawTreasury` language below as binding until reconciled.

**Decision:**
- Contributor VIBE payout is CONTRIBUTION-ONLY: paid per APPROVED MEMORY, gated by a NETWORK-set qualification threshold (not org-set).
- There is NO retrieval/serve-based contributor payout (anti-game).
- Validators/stakers earn emissions. LEADERS earn NO emissions.
- Serve/retrieval counts are excluded from ALL VIBE flows (social-only; see D-SG-1).
- Leader revenue comes from the org demand leg: members pay the org in VIBE for recall access, settling to the org treasury; the leader withdraws revenue (`MsgWithdrawTreasury`).
- The org's access/payment MODEL is LEADER-CONFIGURED, not protocol-mandated. The leader sets pricing (market-driven) and the access model — including whether/how contributors pay to recall vs are earn-only — via the HUB ACCOUNTING layer (the CO-047 `org_credits` ledger). The protocol does NOT fix a single subscription cadence or price.
- Subscription pricing is LEADER-SET / market-driven (orgs compete on price vs memory quality).
- Protocol economic rule on this leg: a SMALL PROTOCOL BURN is taken from org subscription revenue at on-chain settlement; the remainder accrues to the leader treasury. This burn is the deflationary sink that closes the loop.
- Moderator compensation is LEADER-DISCRETIONARY from the org treasury (`MsgWithdrawTreasury`); there is no protocol-enforced revenue split.
- CANONICAL CLOSED LOOP: emission -> contributors (contribution-only) + validators/stakers (mint/sell) -> users buy VIBE -> users pay orgs (hub-accounted, leader-set model & price) -> small protocol burn + remainder to org treasury -> leader -> leader pays moderators -> stake/secure -> repeat.
- IMPLEMENTATION STATUS: DECIDED, not yet built. CO-047 `org_credits` is the hub skeleton (currently non-VIBE, leader-seeded); wiring VIBE subscriptions -> treasury and the protocol burn is a future hub+chain order.
- Org creation BURNS VIBE (deflationary sink): `x/org` `ComputeBurnPrice` -> `BurnCoins`.

**CODE-REMNANT FLAGS (documentation only):** cleanup targets for a future chain order (NOT changed by this doc pass; verify before acting):
1. Contributor payout is currently sourced from the ORG TREASURY (`x/emissions/keeper/epoch_hooks.go` ~line 147, `DebitTreasury`), whereas canonical source is the network contributor-emission pool.
2. Operator/leader emission machinery (`OperatorShare`, `DistributeOperatorRewards`/`MsgDistributeOperatorRewards`, `opreward/`, `ComputeWorkScore`) is dead and contradicts "leaders earn no emissions"; remove in cleanup.
3. The retrieval term in `WorkScore` (`x/emissions/types/keys.go` ~line 48, `retrievalScore`) must be reclassified social, not economic. The serve-attribution code itself is KEPT, not stripped.

---

### D-ATTEST-ROADMAP: Future Pluggable Attestation (Post-Mainnet Roadmap)

**Decision:**
- Post-mainnet roadmap: evolve optional whitepaper §3.10 Session Attestation + §3.11 Two-Layer Difficulty Scoring into a PLUGGABLE attestation framework.
- Separate components plug into the chain to validate session claims (cryptographically OR via API), such as: "user X using LLM model Y took N turns to solve problem Z."
- How attestation enhances the economic and/or social-graph layers is UNDETERMINED (TBD).
- Status is POST-MAINNET. Infra is not there yet.
- This is a MAJOR roadmap item and must remain in canonical docs (do not drop it).

---

### D-ECON-STORAGE-MARKET: Storage-Priced Org Economy (extends + amends D-ECON-CANON)

**Context.** The network's metered cost is STORAGE (validators store memories perpetually). The economy is architected around pricing storage directly, with leader accountability flowing from it. Approved by Walter through design dialogue (Sprint 32). Full spec: `wevibe-meta/workspace/reports/design-storage-market-economy.md`.

**Decision (one self-regulating storage market):**

1. **Slots & identity [SUPERSEDES the pending `D-ORG-ID-DERIVED`].** `org_id` is a REGISTRY-ALLOCATED PERMANENT SLOT identifier (monotonic index), leader-INDEPENDENT, surviving leadership transfer/resale. NOT derived from any wallet (the prior sha256-of-creator scheme is dropped — it could not survive resale). Hard scarcity cap, governance param: **32 alpha / 320 testnet / 3200 mainnet**. Org name, domain (= domain-of-expertise, `D-ORG-DOMAIN-EXPERTISE` RETAINED), and description are mutable any time by the leader.

2. **Primary allocation.** Ascending price per subsequent slot until the cap is filled; proceeds BURNED. Replaces the `x/org` `DynamicPrice` singleton and the dynamic-burn-price line of `D-ECON-CANON` (the burn now occurs via the auction). Testnet: low floor (near-free) so the mechanism is exercised without real bidders.

3. **Secondary allocation.** Abandoned/closed/lapsed org → timeout → slot listed for DESCENDING (Dutch) resale; price falls until bought; proceeds BURNED. `org_id` persists; name/domain/leader swap (old leader removed, new added).

4. **Rent (Harberger-style).** Leader posts a SELF-ASSESSED value V; rent = `r × V`; ANYONE can FORCE-BUY at V during a bid window. Chosen over right-of-first-refusal (which chills bidding) and over flat fees; the force-buy-at-V removes grief-bidding (a grief buy at V actually transfers the org). `r` is a governance param.

5. **On-chain router (demand leg) [moves D-ECON-CANON's demand leg on-chain].** Membership payment hits a consensus-level router that BURNS `max(n%, floor)` and forwards the remainder to the leader treasury. `max` not `min` (a cap would discount the highest-throughput/highest-cost orgs — regressive). `n` and `floor` are governance params. The hub `org_credits` ledger (`D-S32-CO047`) becomes a MIRROR of chain state, not the source of truth. Leader treasury (kept) funds moderators (discretionary) + memory deposits.

6. **Per-memory storage deposit + keeper-prune.** Each committed memory carries a VIBE DEPOSIT that decays as storage rent and is KEEPER-CLAIMABLE as a deletion bounty when exhausted/abandoned. Prune DELETES the ciphertext blob but PERMANENTLY RETAINS accountability metadata (content hash, salt, ciphertext_hash, wrapped_dek_hash, contributor_sig, contributor+leader pubkeys) — a rugger cannot destroy Tier-2 evidence by abandoning. A memory under an OPEN REPORT has its deposit FROZEN (unprunable until resolved). Leader has first-dibs clawback after a ≥1-week window, with a LONG-TAIL ESCROW fraction held against late exposure. BUILT for production; deposit param ~0 on testnet (prune inert there — acceptable: bandwidth caps the rate, testnet is wiped). ONE PATH: deposit-funded prune, token value differs by network — no throwaway testnet-only sweep.

7. **Anti-DDoS split.** Faucet tokens are free, so a token-cost deposit deters nothing on testnet. Therefore the (currently dead) `x/bandwidth` memory-cap is WIRED into `SubmitCommitment` as the TESTNET rate-limit guard; the per-memory deposit is the MAINNET economic anti-spam; `x/bandwidth` is DELETED at mainnet launch (deposit takes over). The serve-cap stays live throughout. [Deferred mainnet consideration: a wealthy attacker can pay deposits to flood at rate — revisit full deletion vs keep-as-throughput-cap.]

8. **Accountability (objective vs subjective decomposition).** Objective violations (anchor mismatch, report-suppression past window, deposit exhaustion/abandonment) are DETERMINISTIC chain predicates — no vote, collusion-proof. `x/gov` is PARAMETERS ONLY, NEVER case adjudication. The subjective "is this malicious?" residue is adjudicated by a KLEROS-STYLE staked random jury + appeal ladder (loser-pays, Schelling point, exponential appeal cost), layered in PRE-MAINNET — NOT an alpha blocker (alpha relies on the 32 curated slots + public Tier-2 exposure + slot-forfeiture). SLASH = SLOT FORFEITURE (slot → re-auction; deposits → reporter + harmed contributors, making detection +EV). The scarce, network-appreciating slot IS the leader's skin-in-the-game. GOVERNANCE-COLLUSION BACKSTOP: minimal subjective surface + permanent public vote attribution (collusion self-incriminates) + exit/fork as the final check (per WHITEPAPER §5.7/§9.7 "no tribunal; the public judges"). A captured chain is answered by exit/fork, never a better ballot.

**Cleanup folded into this work (confirms D-ECON-CANON CODE-REMNANT FLAGS):** remove the org-treasury contributor-payout subsystem (`ProcessOrgPayouts` + `RepTier`/`PayoutPerMemory` + the `DebitTreasury` payout path); contributor payout = flat even split from the network pool (`MintDailyEmission`). Remove dead `WorkScore`/retrieval term + operator/validator reward distribution + stale emissions params. Neuter `x/attestation` to disabled-but-wired (`D-ATTEST-ROADMAP` insurance). Fix genesis (`HasGenesis` + seed `app_state.org`). Close the leader-signer security holes on `MsgAddMember`/`MsgRemoveMember`/`MsgRotateEpoch`. Remove dead `x/org` fields + dead `x/memory` lifecycle states + the approval double-write.

**Foundation requirement.** All leader accountability requires the LEADER'S OWN WALLET as the on-chain authority (`D-S32-CO044-REGISTERORG-FLOW`): the dashboard builds `MsgRegisterOrg` (+ slot acquisition + deposit funding), the leader signs with Keplr, and the dashboard BROADCASTS DIRECTLY to chain RPC via CosmJS; the hub NEVER signs OR relays `MsgRegisterOrg` (see amendment 13). Until this lands there is no leader identity to bond, slash, or hold accountable.

**AMENDMENTS (Sprint 32 — Walter, post-design; folded after the chain-economy build began; supersede the conflicting lines above where noted):**

9. **Org purchase split = 50% BURN / 50% to the org's on-chain account** (NOT 100% burn). Rationale: the leader now also pays gas + per-memory storage deposits, so a 100% entry burn is too harsh; the retained half capitalizes the org's operating account (amendment 10). Applies to primary allocation (item 2) and the secondary/Dutch resale (item 3): the burned half is the sink; the retained half seeds the acquired org's account.

10. **Org on-chain account = a REAL, PUBLIC, per-org bech32 address [SUPERSEDES the bookkeeping-integer `Treasury`].** The current `x/org` `Treasury{Balance string}` integer ledger + the shared `"org"` module-account coin pool + `FundTreasury`/`WithdrawTreasury`/`DebitTreasury` are OVERHAULED (R-OVERHAUL, no shim) into a genuine per-org account that HOLDS real `uvibe`. The address is DETERMINISTIC and module-controlled (derived from the permanent slot `org_id`, so it is leader-INDEPENDENT and survives transfer/resale like the slot); because `x/org` already holds the `FeegrantKeeper` and `BankKeeper`, the module issues grants and moves funds from this account internally — no private key, no leader signature for the grant. This account:
    - RECEIVES the 50% retained on org purchase (amendment 9) + the demand-leg router remainder (item 5 — the "leader treasury" now IS this org account) + leader top-ups (amendment 12).
    - PAYS the org's own chain gas FIRST (e.g. memory submissions); the hub self-funded key is the BACKSTOP only.
    - Is the feegrant GRANTER (amendment 11) and is PUBLICLY queryable (amendment 12).

11. **Feegrant graph — org account feegrants the org's LEADER-WHITELISTED serving address, restricted to serve/deny. [REFINED for self-hostable hubs — supersedes the earlier "single global hub key" framing.]** The hub infra is OPEN-SOURCE and any org must be able to self-host its own hub at any time. Therefore the serving address is a PER-ORG, LEADER-CONTROLLED, CHANGEABLE whitelist (the existing `org.HubServingAddress`) — the single address authorized to submit that org's serve/deny batches:
    - **Default at creation:** the canonical hosted hub's key (the dashboard fills it into `MsgRegisterOrg.hub_serving_key` from hub config) — the easy path works with zero setup. The canonical hub still uses ONE key for all orgs it hosts; that key is simply the DEFAULT each org whitelists, NOT a hard-wired global.
    - **Self-host anytime:** the leader broadcasts a leader-signed `MsgSetServingKey(new_address)` to re-point the org at their OWN hub's key. Exposed in dashboard Settings → "Hub Infrastructure" (show current address + change). DELETE the per-org HD-derived serving keys + their N-way faucet funding (`org-serving-{orgID}` via `accounts.go` + `broadcast.go` auto-faucet).
    - **Feegrant tracks the current whitelist:** the org's on-chain account (granter) grants the CURRENT serving address (grantee) an `AllowedMsgAllowance` wrapping a `BasicAllowance`, `AllowedMessages` = EXACTLY the serve/deny type URLs. The grant is created at `RegisterOrg` and RE-GRANTED inside `MsgSetServingKey` (revoke old grantee → grant new) — all internal to x/org via its `FeegrantKeeper` (no leader signature for the grant itself; the leader only signs the `MsgSetServingKey`). The leader is trusted for their own org; the allowance is capped to serve/deny msgs + the org-account balance, so a bad whitelist cannot drain beyond serve/deny fees.
    - **No grace window (Walter):** switching the serving address immediately invalidates in-flight batches signed by the OLD key (chain auth checks `signer == org.HubServingAddress`); the new hub simply re-submits. Self-host migration is deliberate + low-frequency.
    - When the hub broadcasts a serve/deny batch it sets `Fee.Granter = <orgAccount>`; the chain feegrant ante (`DeductFeeDecorator`) verifies the allowance + deducts from the org account. `x/feegrant` + `x/authz` already wired in `app.go`. [The membership-submission feegrant leg is deferred; the serve/deny grant is the locked first leg.]

12. **Public balance query + leader top-up to the org address.** Expose a chain balance query for the org's on-chain address (REST gateway, per `D-SG-2`), surfaced THROUGH the social graph as a PUBLIC fundedness metric (the social graph reads the chain per `D-SG-1/2/3`; the hub stays hub-only). MOVE the leader top-up flow to fund this PUBLIC org address (replacing the private serving-key top-up). Rationale: a public, well-funded org account is a trust/credibility signal a prospective member or contributor can gauge before deciding to join or contribute.

13. **DELETE all hub-as-relay paths for leader-signed txs [SUPERSEDES the CO-044 dashboard-relays line + the CO-214 delegate path].** The hub RECORDS DATA ONLY — it MUST NOT relay/broadcast leader-signed txs. ALL leader signing goes dashboard (CosmJS) → chain RPC DIRECTLY. REMOVE: the `internal/relay` package (`RelayBroadcast`, `POST /v1/relay/broadcast`, `handleDirectRelay`/`handleDelegateRelay`, the canonical-body validators, `granter_fields.go`), the CO-214 delegate-key machinery (`delegate_keys` table + `VerifyDelegateSignature`/`ExtractInnerGranter` + the `internal/members` insert/select), and the hub's HD-derived per-org LEADER key (`org-leader-{orgID}`) + its faucet funding. The dashboard-facing org-mutation handlers that currently broadcast via the hub leader key (`POST /v1/orgs` `RegisterOrgOnChain`, join-approval/invite `MsgAddMember`, role-change `MsgUpdateMemberRole`) move to dashboard-direct CosmJS signing. The hub KEEPS its keypair ONLY for serve/deny (the single serving key of amendment 11). Big security win: no leader authority transits the VPS, no hub passing txs (Lesson 35).

**IMPLEMENTATION STATUS:** DECIDED; build IN PROGRESS. Committed: Wave 1 — `wevibe-chain 0eb9988` (neuter x/attestation, `D-ATTEST-ROADMAP`) + `9bd601b` (remove `ProcessOrgPayouts`); Wave 2 — `739411b` (leader-wallet signer on AddMember/RemoveMember/RotateEpoch, GAP-SEC-1) + `e95f086` (memory single-write). Remaining build partitioned: proto-sediment excise (dead `WorkScore`/operator-reward msgs + orphaned `RepTier`/`DebitTreasury`) → x/org genesis (seed slot-registry params) → O1 x/org slot registry/identity → O2 x/org rent/router + 50/50 split + per-org account (amendment 10) + feegrant grant (amendment 11) + unify all org-decision msgs to the leader-wallet signer → M x/memory deposit/prune + report-state + wire bandwidth cap → Phase H hub/dashboard wallet-signing + DELETE relay (amendment 13) + public balance query/top-up (amendment 12) → Phase U dashboard create-org slot-acquisition UX.

---

## Identity, Onboarding & Accountability (Sprint 32 — locked via Walter design dialogue)

> These decisions were locked through an extended design dialogue (Sprint 32) and **supersede the wallet-primary identity model** of D-1.1/D-1.4 for onboarding. Rationale brief: `wevibe-meta/workspace/reports/design-identity-onboarding-migration.md` + `gather-wallet-switching-map.md`. Status: **partially BUILT** — the seed-based client identity is implemented (plugin keychain + biometric + BIP39 ID-8a/b/c; dashboard passkey-wrapped foundation ID-1..7) and the cross-surface carrier architecture is RESOLVED (see `D-IDENTITY-CARRIER` below, which also resolves the former open `GAP-IDENTITY-DUAL-PATH`). Onboarding UI + hub blob carrier + provisioning + convergence remain to build.

### D-IDENTITY-PROGRESSIVE-CUSTODY: Passkey-first, self-custodial, wallet-optional identity

**Decision.** A WeVibe identity is a **client-held account key created at first run with no wallet and no seed phrase**, protected by a **passkey (WebAuthn / Face ID / fingerprint)**. Custody is a progressive ladder; the wallet is an optional upgrade, never an entry requirement.

1. **Creation (zero-friction).** On first install the client generates the account keypair (the Ed25519 hub-auth identity + the secp256k1 PRE identity) locally and **wraps it under a passkey-derived secret**. The user signs in with a biometric prompt — no wallet, no seed phrase. The plugin/MCP already does on-device key generation; the dashboard is brought onto the **same** client-key scheme (ending the prior plugin-random vs dashboard-wallet-derived divergence — the two must mint the **same** identity for a given user).
2. **Non-custodial protection.** The key is encrypted under the passkey; the resulting **ciphertext may be backed up to the hub, but the hub can NEVER read or sign with it.** OAuth/email, if offered, is **only** an optional secondary recovery wrap — **never custody, never the signing authority.** (The earlier "hub holds the keypair behind OAuth" idea was explicitly rejected: it makes the hub a honeypot, contradicts the open-source/self-hostable hub model, and reverses the trust boundary.)
3. **Portability.** Syncable passkeys (iCloud Keychain / Google Password Manager) carry the identity across the user's own devices; combined with the hub-held ciphertext, the identity is **not tied to one computer.** Honest trade-off: passkey sync leans on the OS vendor (same trust as a password manager) for the *convenience* tier; the wallet upgrade (below) is the sovereignty escape hatch.
4. **Recovery.** A passkey-only identity with no backup is one dead device from loss. Onboarding MUST surface a gentle, early "secure/back up your account" path (recovery file, OAuth/email wrap, or "link a wallet"). This is a required design item, not optional polish.
5. **Wallet upgrade = LINK via staged handover, NOT big-bang migration.** A Cosmos wallet (Keplr) is an optional later step. Because the chain is **append-only**, prior history (membership, reputation, contributor-signed anchors) keyed to the original pubkey **cannot be rewritten** — so the wallet is **linked** (it becomes the canonical chain authority + funding anchor, reusing the existing `x/authz MsgGrant` mechanism of D-1.2), while the underlying account/history is preserved. The cutover is modeled as a **staged handover** (epoch-rotation-style: old authority valid until the new one has provably bootstrapped, then a clean flip) — never a single all-or-nothing tx. The user never faces a "migrate or lose your progress" moment. *(This resolves the earlier "Model D — Keplr becomes canonical" vs "Model L — link" debate: the wallet is canonical for authority/identity-display; the key/history is preserved via link + staged handover.)*
6. **What actually needs a wallet (and nothing more).** Browsing, requesting to join, being a member, and **contributing** all work on the passkey identity alone (the hub keys members by pubkey; `wallet_address` is nullable; join requires only a pubkey; the leader does the on-chain commit). A wallet becomes necessary only to **claim accrued VIBE rewards** and to pay any mainnet per-memory fee. Rewards accrue to a claim-later balance until a wallet is linked — this is the natural upgrade carrot, not a tollgate.

**Why.** The product target is normie vibe coders: a usable account in seconds with no crypto ceremony, keys that are not naked on disk and not stranded on one machine, and a self-custodial model that never makes the hub a custodian. This is the professional embedded-wallet pattern (passkey-wrapped client key, optional external-wallet link) adapted to WeVibe's self-hostable, anti-custody ethos.

### D-IDENTITY-CARRIER: "One seed, many wrappers" — how the identity flows across surfaces (resolves GAP-IDENTITY-DUAL-PATH)

**Decision.** The cross-surface carry problem (the plugin and the browser are sandboxed and cannot read each other's stored key) is solved by treating the **32-byte ed-seed as the single canonical identity (the DEK)** and giving each surface its own independent **wrapper (KEK) around its own copy** of that seed. No surface mints a second identity; they all wrap the same seed. (Rationale brief + research synthesis: `wevibe-meta/workspace/reports/design-identity-carrier.md`.)

1. **Wrappers:** (i) **browser passkey** — `KEK = HKDF-SHA256(PRF_output, perUserSalt, info="wevibe-seed-kek-v1")`, seed wrapped AES-256-GCM, ciphertext uploaded to the hub; syncable passkeys (iCloud/Google) make the PRF output reproducible on the user's other devices → automatic multi-device. (ii) **macOS plugin** — Touch-ID-gated OS keychain (local cache, offline-capable). (iii) **Win/Linux plugin** — OS keychain-at-rest, seed set via one-time transfer or BIP39. (iv) **BIP39 phrase** — the universal, no-dependency **break-glass / recovery floor**. (v) **wallet** — unchanged: the canonical authority/funding layer LINKED on top, key preserved, never re-keyed (per D-IDENTITY-PROGRESSIVE-CUSTODY §5).
2. **Hub stays non-custodial.** The hub stores the pubkey + N opaque wrapped-seed blobs (one per KEK) and can read **none** of them — no key-server, no OAuth-gated release server. This is the deciding advantage over any OAuth/threshold carrier.
3. **Safe by construction (envelope encryption + independent recovery).** Per the WebAuthn-PRF E2EE guidance (Cappalli): never derive the seed directly from a single passkey; always envelope-encrypt with one KEK per authenticator; always HKDF the PRF output with a non-empty `info`; never a static global salt; and **always keep an independent recovery path** — which the BIP39 floor provides, placing this design in the explicitly-legitimate category (cf. Bitwarden/1Password).
4. **Plugin is NOT a passkey client.** It is provisioned once via a low-friction transfer (loopback handshake / QR / one-time code) — or BIP39 — then caches the seed locally and signs offline. (Making Node a passkey client is macOS-only + heavy and is rejected.)
5. **Convergence — canonical first run.** First surface to create an identity is canonical; every later surface **adopts** the existing seed, never minting a new one. Auto within an ecosystem (synced passkey `credentialId` matches a hub blob); cross-ecosystem requires active adoption (transfer/phrase). An **optional email/OAuth login may serve as a non-custodial ACCOUNT ANCHOR** (a grouping key for cross-ecosystem blob discovery) — anchor only, never custody, never the carrier (consistent with D-SYBIL-MEMBERSHIP-GATED).
6. **OAuth is NOT the carrier and NOT primary.** "Sign in with X" is OIDC = a token, not key material; no mainstream IdP exposes a per-user secret. A frictionless self-custodial OAuth carrier would require an OAuth-gated threshold-share server (a central role that complicates the forkable/self-hostable hub), so it is held in reserve as an OPTIONAL later add-on built on **WeVibe's own Shamir** (`splitSecret`/`reconstructSecret`), share folded into the self-hostable hub — explicitly NOT a managed provider (Web3Auth/Privy/Magic/Turnkey all add a hosted dependency that breaks self-hostability and the space is consolidating). **D-IDENTITY-PROGRESSIVE-CUSTODY is therefore UNCHANGED — OAuth stays optional / recovery-or-anchor / non-custodial.**
7. **PRF is enhancement-with-fallback, never a hard dependency.** Good on Apple/Android, fragile on Windows Hello, unreliable over hybrid/QR transport — so every flow MUST have a BIP39 / transfer-code fallback.
8. **Headless contexts (e.g. the docker MCP) are just another at-rest backend:** a provisioned seed cached to the existing file-based AES-GCM (machine-seed-encrypted) keystore with an app passphrase — no biometrics. (Resolves deploy-blocker B2: keychain-only at-rest must not be assumed.)

**Why.** A synced passkey's WebAuthn-PRF output is reproducible across the user's devices, so a hub-stored PRF-wrapped-seed ciphertext gives **frictionless, multi-device, self-custodial carry with a non-custodial hub** — no managed third party, no custody trade-off, no locked-decision change. It is the textbook envelope-encryption pattern, it salvages the entire built identity stack (the seed model, derivation, keychain-at-rest, biometric gating, passkey wrapping, wallet-link all stand — only wrappers/UI/provisioning/convergence remain to build), and it keeps OAuth as a clean optional progressive-custody add-on rather than a rewrite.

### D-SYBIL-MEMBERSHIP-GATED: Sybil resistance lives at membership + economics, not identity creation

**Decision.** Free keypair creation **cannot** be prevented (it is offline math) — this is an accepted, documented gap. Sybil resistance is therefore located where it can bite:
- A membership-less identity **can do nothing** (no contribution, no recall, no earnings) — so abundant free identities are inert.
- **Org membership is the scarce, gated thing**, controlled by the human leader (invitation / approval). The leader is the bouncer. (Aligns with WHITEPAPER §9.1 "invitation-only membership.")
- **Reputation is the soft stake** for passkey-only users: it is destroyed on boot, so a sleeper who built standing loses it and restarts at zero under a fresh identity that must be re-approved.
- The **only hard monetary bond in the system is the leader's slot** (`D-ECON-STORAGE-MARKET`: scarce, capped, auction-burned, slot-forfeiture-on-malice). **There is no per-contributor bond** (see D-LEADER-SOLE-SIGNER).
- Cheap anti-whack-a-mole friction is permitted at the *join door* (post-boot cooldown, invite codes, or OAuth/email purely as a one-human≈one-applicant signal — not as custody), never as a wallet/payment gate on the free tier.

**Why.** You can't tax identity creation without a scarce resource, and taxing it would break zero-friction onboarding. Making identities free-but-inert and pushing the gate to leader-approved membership + the leader's economic slot puts the cost exactly where harm can occur.

### D-LEADER-SOLE-SIGNER: The leader is the sole signer and sole bond for all published content — NO contributor certification, ever

**Decision (HARD INVARIANT).** Every memory an org publishes is committed by the **leader's wallet signature**, backed by the leader's slot. This is permanent and not to be relaxed:
- **No contributor-certification / auto-approve / "trusted contributor skips review" tier — ever.** Contributors are **free, unbonded, and always leader-gated**: everything a contributor submits sits behind mandatory human review before it can be committed or served.
- **Moderators delegate review LABOR, never the signature.** A leader may appoint moderators to vouch operationally (Model B), but the **leader remains the sole on-chain signer** and sole accountable party. "On the hook for everything" means the leader *signs for* (takes responsibility for) everything — not that the leader personally *reads* everything.
- **No per-contributor bond** is needed precisely because a contributor can never publish unilaterally; their worst case is wasting reviewer time (handled by rate-limits + boot + reputation-burn + leader-gated joins), never poisoning the network.

**Why.** WeVibe's entire anti-capture model rests on **one accountable signer per org**: the public, prospective members, and the exit/escalation machinery all read an org as the leader's signed, slot-backed curatorial judgment (WHITEPAPER §9.7, "the leader bears sole signature," §5.3 co-attestation removed). A certification tier would (a) re-introduce the multi-signer co-attestation the model deliberately deleted, (b) swap the leader's large slot-bond for a small contributor deposit (a downgrade in collateral per item), (c) add a new unilateral-publish attack path, and (d) require a subjective slashing judge (a new capture vector). The scaling objection ("review won't scale") is answered by delegating review *labor* to moderators while the leader still signs batches — not by removing the gate. Walter stress-tested this and challenged for a counterexample; none survived.

### D-SINGLE-ORG: One leader = one org, one member = one org (alpha)

**Decision.** For alpha, a leader administers exactly **one** org and a member belongs to exactly **one** org. This is the intended starting shape, with **easy gates to open it up** (multi-org membership) once the system proves stable. Crucially, the identity layer is **already multi-org-ready** — membership is keyed by `(org_id, pubkey)` — so expanding later is "allow >1 membership row," not an identity-model change.

**Why.** A single, clean accountability and UX surface for alpha (one org context, one role, no org-switching complexity); the constraint also sharpens the security story (with identity = the stable passkey key rather than the wallet, a wallet/account change doesn't change *who you are*, so wallet-switch staleness largely evaporates).

### D-FE-VIEW-STATE: Derived view-state machine + standing FE/UX principles

**Decision.** Dashboard role-gated rendering derives from a **single source of truth** view-state enum (`INITIALIZING | NO_WALLET | CONNECTED_NO_ORG | CONNECTED_{LEADER|MODERATOR|CONTRIBUTOR|MEMBER}`) computed from identity + org-context — never a hand-maintained parallel copy. Identity is **hoisted into one provider** read once app-wide (no N-per-consumer storage reads), which also listens for identity/account changes so no surface renders stale data after a switch. Standing UX principles: **progressive disclosure** (reveal a control/help only when needed), **derive-don't-duplicate**, **clean dead code as touched**, **render performance** (managed polling, no effect cascades). Identity + wallet are coupled; alpha is **one wallet → one role** (no multi-role complexity). *(The specific two-level nav layout is being iterated live with Walter and is intentionally NOT frozen here.)*

**Why.** The pre-redesign dashboard duplicated role/state logic across many pages and went stale on wallet/account change (no identity-change listener; orgs loaded once on mount). A derived machine + a single identity provider fixes both correctness (no drift, no stale render) and performance, and gives a clean substrate for the role-based nav.

### D-STORAGE-DECENTRALIZE-ROADMAP: Decentralized blob storage is a post-MVP roadmap item (integrate, don't rebuild)

**Decision (ROADMAP — not this sprint).** WeVibe's encrypted memory blobs live on the single hub's Qdrant today. The long-term target is to move the blob-storage layer onto a **decentralized storage network** (Walrus-style: blockchain as control plane, staked/slashable storage committee, storage-resource market, availability challenges) so the data plane is as credibly-neutral and forkable as the protocol claims. WeVibe's locked `D-ECON-STORAGE-MARKET` (per-memory deposit, keeper-prune, slot scarcity) is effectively a single-operator, lightweight version of that model. **Pragmatic constraint: integrate an existing DA/storage network behind the unchanged Umbral-KMS overlay — do NOT re-implement an erasure-coding storage protocol.** Orthogonal to the identity sprint; explicitly deferred to post-alpha.

**Why.** A single-hub store is a single point of storage custody and censorship for the very data the whitepaper promises no one can suppress. Decentralizing it closes that gap — but it is large, post-MVP, and best satisfied by integration, not a from-scratch build.

### D-REPUTATION-KEYED-BY-PUBKEY: Reputation AND earnings are keyed by the passkey pubkey until a deliberate, dual-signed migration

**Decision (DESIGN-LOCKED — NOT yet built).** A contributor's reputation/XP **and earnings** are keyed by their **passkey Ed25519 pubkey**, not by a wallet, and stay that way until the user performs a deliberate migration (`D-MIGRATION-ONCHAIN-ALIAS`). Connecting a wallet does **not**, by itself, move or re-key any reputation. Consequences:
- **Memory-contribution XP is already keyed by the passkey pubkey on-chain** (`x/memory ApproveMemory → reputationKeeper.IncrementContribution(contributor)`), so wallet-free contributors already earn and display contribution reputation on My Org today.
- **Serve XP MUST also key by the passkey pubkey.** The current `x/serve` path keys serve XP by `ContributorAddress` (the wallet) and SKIPS it when no wallet is linked, so wallet-free users earn no serve XP — a bug against the wallet-free invariant. The fix keys serve XP by the passkey pubkey (no proto change).
- **The hub `retrieval/stats.go GetContributorStats` MUST NOT silently prefer a linked wallet address over the pubkey.** Preferring the wallet strands pubkey-earned reputation the instant a user links a wallet; the stats path becomes **alias-aware** (resolve via the migration alias) instead of wallet-preferring.
- **Earnings accrue on-chain to the passkey identity too — not just XP.** `x/emissions` currently credits the wallet (`contributor_address`); it must credit the passkey `contributor_id` recorded on the committed memory (the memory already records both). **Pre-migration earnings are a claim-later accrual record keyed by the pubkey, NOT spendable coins** (a passkey pubkey is not a funded cosmos account); they become withdrawable only after migration (`D-MIGRATION-ONCHAIN-ALIAS`). This folds the contributor withdrawal-claim path (SEC-FLAG-4 / GAP-ECON-BUILD).

**Why.** Identity is the passkey (`D-IDENTITY-PROGRESSIVE-CUSTODY`); a wallet is an optional earnings/authority upgrade, not identity. Reputation is the soft stake (`D-SYBIL-MEMBERSHIP-GATED`) and must accrue from the moment of contribution, with no wallet, and must never silently vanish when a wallet is later linked.

### D-MIGRATION-ONCHAIN-ALIAS: Pubkey→wallet reputation migration is an on-chain, dual-signed alias gated by memory contribution

**Decision (DESIGN-LOCKED — NOT yet built).** Moving reputation from a passkey pubkey onto a wallet address is a deliberate, explicit **migration**, modeled as an **on-chain alias** (passkey pubkey → wallet address):
- **On-chain, not hub-DB.** The alias and its completion flag live on the **chain** (the social-data source of truth, per `D-SG-2` / `D-13.4`). A hub-DB approach is **REJECTED** — the hub recording XP/aliases is an easy gaming surface and reverses the trust boundary.
- **Hub triggers/records; the chain enforces.** The hub is responsible for recording/relaying the alias, but the migration is **gated by / enforced through memory contribution** — the contributor's memory-contribution signature trail is the closest-to-enforcement anchor proving the pubkey is genuinely theirs.
- **DUAL-SIGNED (passkey + wallet).** The alias tx MUST be signed by BOTH the passkey identity AND the wallet. This is the security anchor preventing anyone from pointing a wallet at someone else's pubkey to claim their reputation.
- **Flag name = `is_migrated`** (on-chain): `false` before migration, `true` after.
- **Append-only.** Pre-migration history keyed to the pubkey is preserved and resolved via the alias after migration (the chain is append-only; nothing is rewritten). Social-graph reputation reads stay keyed by pubkey and resolve via the alias once `is_migrated=true`.
- There is **no migration flag or ed25519→wallet mapping on chain today** — this is entirely unbuilt. The prior belief that a `migration_completed` flag already existed on chain was false.
- **`is_migrated` unlocks WITHDRAWAL; link ≠ migrate.** Linking a wallet only puts it on file (authority/bond). Migration is the separate, deliberate, dual-signed act that unlocks withdrawal of the accrued passkey-keyed earnings to the wallet and flips the UI to wallet-only. A user may link a wallet (e.g. to lead) WITHOUT migrating their contributor identity — so the linked-but-not-migrated state is real (see `D-IDENTITY-UI-STATE-MACHINE`).
- **Wallet-immediately users auto-migrate.** A user who connects a wallet before ever using the passkey identity has zero accrued passkey history, so the memory-contribution gate is vacuous (nothing to steal, nothing to move) and migration auto-completes at link time — they never see the dual-render state. The passkey identity is still minted (there is NO second "wallet-as-identity" path — that would fork the model).
- **No hub-DB authority for the non-migrated state.** The non-migrated render state is served from the CHAIN keyed by the passkey pubkey; the hub MAY cache it for render speed but is never the source of truth. The earlier "maybe a hub-DB component" idea is explicitly rejected — once earnings are on-chain it buys nothing and reintroduces a gaming surface + a chain↔DB drift bug class.
- **Earnings-secured nudge.** Because real economic value now accrues to a passkey identity, a lost passkey with no backup loses unclaimed earnings. The earnings UI surfaces a "secure your earnings" prompt (BIP39 break-glass / link a wallet) once a non-trivial balance accrues — gated on having something to lose, so zero-friction is preserved otherwise.
- **Dual-sign construction.** The alias tx MUST bind both keys: each signs a message containing the OTHER's pubkey + a nonce, atomic in one tx, to prevent replay/splice attacks.

**Why.** Reputation must be portable onto the canonical wallet authority when a user upgrades, but only through an explicit, cryptographically anchored act. Dual-signing + memory-contribution gating make rep theft impossible; keeping the alias on-chain keeps the social data on its source of truth and out of a gameable hub table.

### D-IDENTITY-UI-STATE-MACHINE: Dashboard identity rendering derives from {wallet connected?} × {biometric present?} × {is_migrated?} [extends D-FE-VIEW-STATE]

**Decision (DESIGN-LOCKED — NOT yet built).** The dashboard derives identity rendering from three checks: (1) is a wallet connected? (2) is a platform passkey/biometric present (macOS Touch ID / Windows Hello)? (3) `is_migrated`? Rendering:
- A **non-migrated passkey account WITH a wallet connected is its own state** → **DUAL-RENDER**: show the passkey-keyed reputation AND the wallet together. (The user linked a wallet for earnings but has not migrated their reputation onto it.)
- Once `is_migrated = true` → render **ONLY the wallet identity** (reputation now resolves to the wallet via the alias).
- A pre-wallet "dual chain+DB" rendering is NOT needed: reputation is already on-chain by pubkey, so it is chain-only — pubkey-keyed before migration, wallet-keyed (via alias) after.
- **All three states are served from the chain** (keyed by passkey pubkey pre-migration; resolved to the wallet via the alias post-migration). The hub may cache for render speed but is not authoritative — there is no hub-DB identity/earnings store (`D-MIGRATION-ONCHAIN-ALIAS`).

**Why.** The link-but-not-migrated window is a real, distinct user state that the old binary wallet/no-wallet view-state machine (`D-FE-VIEW-STATE`) cannot express; rendering it explicitly prevents both stranded-rep confusion and premature wallet-only display.

### D-LEADER-REQUIRES-WALLET: Leader/moderator ⟹ has a wallet (hard invariant)

**Decision (HARD INVARIANT — DESIGN-LOCKED).** Any **leader or moderator MUST have a linked wallet** — there is no edge case of a leader or moderator without a wallet, across all accounts. The identity state space is {has wallet?} × {is_migrated?}; **leader/moderator can never occupy the no-wallet quadrant.** Leaders sign and bond with the wallet (`D-LEADER-SOLE-SIGNER`); moderators act within the leader's accountability orbit. Pure contributors and consumers may remain wallet-free indefinitely.

**Why.** The whole accountability model rests on a slot-bonded, wallet-signing leader; a privileged role without a wallet would have no bond and no on-chain signing authority, breaking the invariant that published content is always leader-wallet-signed.

### D-WALLET-SWITCH-LEADER-MISMATCH: Switching the Keplr account is not switching the user; warn only on leader mismatch

**Decision (DESIGN-LOCKED).** Because identity is the passkey (not the wallet), switching the active Keplr account does **not** change who the user is. The dashboard topbar shows the **stored linked `walletAddress`**, not a live Keplr read, so a Keplr switch + reload does not change the displayed user — this is correct behavior, not a bug. The generic "wallet changed" modal is low-stakes and is **REFRAMED to a leader-mismatch warning**: if the active Keplr account ≠ this org's leader wallet, warn that leader txs will fail until the user switches Keplr back. For non-leaders the warning is irrelevant and is not shown.

**Why.** A "your wallet changed, identity updated" framing is actively misleading under passkey identity. The only case where the active Keplr account actually matters is a leader about to sign a tx that will fail against the wrong account — so that is the only case worth a warning.

### D-E2E-BROWSER-PROFILES: Multi-user E2E testing uses separate browser profiles, not Keplr wallet switching

**Decision (DESIGN-LOCKED — testing).** Because a Keplr switch does not change the passkey identity, multi-user end-to-end testing MUST use **separate browser profiles** (each with its own passkey identity, optionally its own wallet), **not** wallet switching within a single browser. The earlier "swap wallet A↔B in one browser" plan cannot work under the passkey-identity model.

**Why.** Each browser profile is one passkey identity = one user; this is the only way to exercise true multi-user flows (contributor → consumer recall, leader → member) given identity is decoupled from the wallet.

---

## Recall UX & Model-Attested Provenance (Sprint 32 — locked via Walter design dialogue)

> Locked through a Sprint-32 design dialogue covering recall popup-fatigue UX and LLM/model provenance. Status: **DECIDED, design-only — NOT yet built.** The attestation tier is explicitly an OPTIONAL, off-chain, production toggle and is **NOT** wired into `x/attestation` (which stays disabled-but-wired per `D-ATTEST-ROADMAP`). Primitive reference: OpenRouter / Phala / RedPill GPU-TEE confidential inference (Intel TDX + NVIDIA H100 Confidential Computing).

### D-RECALL-GATE-ON-RISK: Risk-triggered injection gate + batched quality signal [EXTENDS D-11.5]

**Context.** At a healthy, mature org a typical coding session can surface on the order of tens of recall candidates. A per-memory blocking approval modal for every candidate causes popup fatigue, which pushes users to blind-accept. Blind-accept is doubly harmful: (a) it turns the human safety gate into theater — a claimed-but-rubber-stamped defense layer is worse than none — and (b) it poisons Earned Trust (D-4.2): serve attestations stop meaning "useful" and degrade into "retrieved and not bothered-with," starving the very signal the decay model depends on.

**Decision.** The injection gate gains a THIRD mode between D-11.5's `[Gated approval]` and `[No gated approval]`:
1. **`[Gated on risk]`** — auto-inject + attest the safe majority (high-confidence, well-served, trusted-contributor, wevibe-guard-clean memories) with NO modal; a blocking pop-up is raised ONLY when a candidate earns it: a wevibe-guard flag, a new/low-trust contributor, or low confidence. A popup then signals "this one needs your eyes," restoring the gate's honesty and collapsing fatigue.
2. **The two jobs of the old single modal are SPLIT.** The SAFETY gate (block-before-inject) is reserved for risk-flagged candidates only. The QUALITY signal (attest / deny) moves OFF the critical path into a low-friction **batch review tray** the consumer clears at a natural break (end of task/session). The agent is not interrupted mid-flow to harvest a signal that need not be synchronous.
3. `[Gated approval]` (always-modal) and `[No gated approval]` (never-modal) remain available; `[Gated on risk]` is the fatigue-aware middle path and the intended default direction.
4. **Blind-accepting negative signals (DNDs) is permitted** as a safety choice — a "don't do this" guardrail cannot inject malicious code into the agent the way an implementation memory can — BUT wevibe-guard MUST still scan a DND's text (a malicious DND can carry a prompt-injection payload). The scanner is never skipped; only the human modal is.

**Why.** Error costs are asymmetric: a false-accept (injecting a bad memory) is potentially high-cost; a false-reject (skipping a good one) is cheap. Spend the consumer's scarce attention only where false-accept risk is real. This is progressive disclosure (D-FE-VIEW-STATE) applied to the recall gate, and it protects the Earned-Trust signal (D-4.2).

### D-FAVORITE-PINNED-REFERENCE: A "favorite" is a pinned reference, never a local plaintext snapshot

**Decision.** When a consumer "favorites" a memory it is a **pinned reference** — "always re-fetch and rank this high for me," served through the normal live recall path. It is explicitly NOT a persisted local plaintext copy.

**Why.** A local plaintext snapshot would quietly break three subsystems:
1. **Revocation hole.** The security model assumes decrypted plaintext is local and EPHEMERAL (the hub never persists it). A snapshot is persistent plaintext on disk that survives the user being booted, epoch rotation, or org closure — defeating membership revocation.
2. **Staleness fork.** If the org later decays/denies/supersedes/deletes the memory (e.g. the advice went wrong after a version bump), a frozen local copy keeps serving the stale, now-wrong version forever. A product whose pitch is "current verified memory vs. stale generic knowledge" must not ship a silent stale-copy mechanism.
3. **Attribution/decay blind spot.** A favorited memory used constantly would never re-serve through the loop, so the contributor stops earning attribution and the memory looks unused on-chain while it is a workhorse — distorting Earned Trust (D-4.2) and reputation.
A pinned reference keeps revocation, decay, re-guard, and serve attribution all working.

### D-ATTEST-TEE-TIER: Optional TEE model-attestation as a provenance tier (off-chain, production toggle) [EXTENDS D-ATTEST-ROADMAP]

**Primitive.** OpenRouter / Phala / RedPill GPU-TEE confidential inference exposes, per request, a signature over `H(request) ‖ H(response)` by an enclave-held key, plus an attestation report (Intel TDX + NVIDIA quotes) binding that key to the loaded MODEL MEASUREMENT — a hash of the model code/weights, i.e. the actual checkpoint, not the marketing name. Verifiable with standard tooling against Intel DCAP + NVIDIA's attestation service.

**Decision.** Add `tee-attested` as an OPTIONAL provenance tier — a "Tier 0" above the roadmap's CommitLLM and proxy tiers (`provenance` enum becomes `tee-attested | commitllm | proxy-attested | self-declared | unattested`). Locked properties:
1. **Optional, opt-in, production toggle — NEVER a contribution gate.** Requiring attested-SOTA would exclude the local/smaller-model users who are WeVibe's core audience (contradicts D-IDENTITY-PROGRESSIVE-CUSTODY and the mission). It is a prior/badge/ranking input only.
2. **Verification is OFF-CHAIN.** The chain cannot verify TDX/NVIDIA quotes in consensus. An off-chain WeVibe attestation-verifier (or the MCP client) checks the quotes, confirms the measurement, validates per-turn signatures, and emits a compact signed assertion. The RAW bundle is retained off-chain so anyone can RE-VERIFY later — the verifier is a convenience, not a trusted oracle.
3. **`certified_model` certifies SESSION PROVENANCE, not memory text.** Extraction transforms the session and the human edits before submit, so the tag means "this memory was DERIVED from a session in which model Y produced the work," never "model Y wrote this memory text." A `derivation` flag records `verbatim | edited-after-extraction` (see D-EXTRACTION-MODEL-STANDARD).
4. **Identity binding is mandatory.** The contributor binds the attested session to their WeVibe pubkey (embed pubkey/nonce in the session so the attested request hash commits to "session belongs to X"; the contributor signs the bundle). Without this, attested sessions are buyable/transferable and the gamification surface is forgeable.
5. **Turn count is anti-FABRICATION, not anti-PADDING.** Attested N-turns proves N genuine inferences on model Y happened (defeats pasted fake transcripts) but does not prevent padding with junk turns. Because the signature binds content hashes, retain the transcript so padding is visible to the scorer/leader. Treat N as a difficulty-scoring input, never a raw effort score.
6. **Store the MEASUREMENT, not the name** (checkpoint hash) so "certified Model Y" cannot silently drift as a provider updates weights.
7. **Per-field graded provenance — no single binary "certified ✓" badge** (avoid false rigor; every surface shows the weakest grade in the chain — see D-GAMIFICATION-PROVENANCE).
8. **Different privacy posture — disclose it.** Certified mode routes session content through a TEE provider (adds Intel/NVIDIA/Phala to the trust root); it is not the local-only path. The toggle copy must say so.
9. **Keep it pluggable + re-verifiable; do NOT make it a hard dependency.** A Phala Cloud API vulnerability was disclosed 2026-06-01 and TEEs have side-channel history. Build behind the pluggable adapter D-ATTEST-ROADMAP envisions, with a measurement allowlist (reproducible-build registry) and the retained bundle, so the claim survives the provider's API changing or being compromised. (Same discipline as CO-028's SP1 ZK path: technically achievable, operationally unshippable.)
10. **Do NOT couple `tee-attested` to VIBE.** Attestation proves the model, not difficulty — paying more for certified-SOTA incentivizes SOTA-laundering trivial work. Prior/badge/ranking input only (consistent with D-ECON-CANON).

**Phasing.** **Phase 0 (now, where built):** off-chain metadata + verifier + retained bundle, surfaced as a production toggle; ZERO chain changes; `x/attestation` stays disabled-but-wired. **Phase 1 (later, only when non-experimental):** anchor `H(bundle)` + provenance enum + measurement on-chain via `x/attestation`, giving the economic/reputation layer a trustless input.

### D-EXTRACTION-MODEL-STANDARD: Standardize (and optionally attest) the extraction model — the controllable half

**Context.** "Weaker models surface weaker memories" has two distinct heads: the PRODUCTION model (did the coding, in the user's suite, not controlled by WeVibe — and per difficulty logic a weak-model/hard-problem session may be the MOST valuable), and the EXTRACTION model (distills the session into the memory, runs in WeVibe's own pipeline — the part WeVibe controls, and where weak models do the most damage).

**Decision.**
1. **Pin/own the extraction model.** Memory extraction ("Extract Memories", D-12.2) runs through a known, pinned model (or a small pinned set) regardless of the user's coding model — removing weak-extractor variance at the source, with no attestation required.
2. **Optionally attest extraction** (run it in a TEE per D-ATTEST-TEE-TIER) so EVERY certified memory carries a baseline "extracted by model E" provenance even when the production model is unattested.
3. **`derivation` flag** is computed by comparing `H(extraction_output)` to `H(submitted)`: `verbatim` if unchanged, `edited-after-extraction` if the human edited it. Surfaced to the leader (and downstream) so a hand-edited memory is never dressed as fully model-derived.

**Why.** The highest-leverage, cheapest standardization win is the half WeVibe controls. Production-model attestation is an optional bonus; extraction standardization is the controllable baseline.

### D-GAMIFICATION-PROVENANCE: "user X / model Y / N turns / problem Z / L negatives" is a per-field graded composite [under D-SG]

**Decision.** The session/contribution gamification claim is NOT one attested fact — it is a composite of fields with DIFFERENT provenance grades, and each field carries its own grade:
- **user X** — contributor pubkey: native on-chain.
- **model Y** — TEE attestation measurement: cryptographically attested (attested slice only).
- **N turns** — count of signed turns: attested-real but not un-padded (D-ATTEST-TEE-TIER §5).
- **problem Z** — memory keywords/title/session-summary: DESCRIPTIVE/semantic, NOT cryptographically attestable; never presented as "proven."
- **L negative signals** — denials from the serve/deny loop (D-4.2): native on-chain and LAGGING (zero at publish; accrues later). Surface as a served:denied RATIO / "challenged L times and survived," never a raw scarlet-letter count that discourages contribution.

**Placement.** The composite is SOCIAL/display data → rendered by the social-graph layer (D-SG-1/2/3) reading the chain as source of truth; the hub stays hub-only. It is display/badge-only and MUST NOT drive VIBE (D-ECON-CANON). When two-layer difficulty scoring (WHITEPAPER §3.11) is built, `model Y` + `N turns` + `difficulty(Z)` are its trustworthy inputs — TEE attestation is the missing input that makes difficulty scoring farming-resistant.

**Why.** A single "certified" badge over a composite where only some fields are attested is a lie of omission; per-field grading keeps the surface honest and prevents attested fields from lending false credibility to the unattestable ones.

## 20. Chain-Resolved Hub Endpoints + Plugin Onboarding

> DECIDED this session. `D-CHAIN-RESOLVED-HUB-ENDPOINT` / `D-HUB-RESPONSE-SIGNED` / `D-HUB-ENDPOINT-CHANGE-TOAST` are DESIGN-ONLY (multi-repo build not started: wevibe-chain proto + leader Msg, wevibe-protocol contract, wevibe-mcp resolution, wevibe-dashboard leader UI). `D-SIDECAR-PLUGIN-OWNS-STATE` + `D-PLUGIN-ONBOARDING-HOOK` are BUILT + committed in wevibe-mcp (6ceac5d..264a29f).

### D-CHAIN-RESOLVED-HUB-ENDPOINT: The chain is the org directory; the plugin resolves hub URLs from chain (consumer never configures a URL)

**Decision.** There is exactly one canonical WeVibe chain = the network source of truth (consistent with `D-12.1`: one chain, many self-hostable hubs). The chain's public RPC is the single stable trust anchor (built-in default, env-overridable for testnet/self-host). The chain is the org directory: add a leader-configurable on-chain org field `hub_endpoints` — an **ordered list of 1 to 3 network URLs** (redundancy/failover) — distinct from the existing `StoredOrg.hub_serving_address` in section 18 amendment 11 (leader-signed `MsgSetServingKey`, self-host-anytime, feegrant tracks the current whitelist). The split is locked: `hub_serving_address` = AUTH (the Cosmos key authorized to submit org serve/deny txs); `hub_endpoints` = TRANSPORT (where contributors/consumers reach the org's hub). Both are leader-controlled, on-chain, and set by leader-signed message flow (extend `MsgSetServingKey` → `MsgSetServingInfo`, or fold into `MsgSetOrgConfig`). The Msg handler validates 1–3 well-formed URLs (max 3 is a chain-enforced bound on state + client failover cost). Proto changes are regenerated via Docker `make proto-gen` and never hand-edit `.pb.go`.

**Failover (redundancy).** The list is priority-ordered: the plugin uses `hub_endpoints[0]` and falls through to the next on connection failure, health-check failure, OR response-signature-verification failure (a primary that fails verification is treated as unhealthy and skipped — this is also the resilience path if a primary is compromised). All endpoints front the SAME single on-chain `hub_serving_address` — they are REPLICAS of the org's one hub identity, not independent operators (the chain's serve/deny auth is a single address per §18 amendment 11). So every endpoint's responses verify against that one key, and redundancy does NOT weaken `D-HUB-RESPONSE-SIGNED`. The active endpoint is cached in the sidecar and re-evaluated per session / on failure.

Resolution path is one-way and biometric-free: chain → RPC → plugin org query → resolve/check endpoints → update local config/sidecar if changed → done. This runs once per session start (org details are public, so no identity load; consistent with lazy boot). The plugin reads cached `org_id`s from its sidecar, queries chain RPC for each org's `hub_endpoints`, and rewrites local config + sidecar only when needed. Consumers/contributors never see or type a hub URL. Per-org plugin-sidecar state is keyed by globally unique chain `org_id` (stable across a leader hub migration, so endpoint re-pointing does not change `org_id`): `orgs: { "weorg-N": { hubEndpoints: [...], activeHubEndpoint, hubServingAddress, hubResponsePubkey, joinedAt, lastExtractedAt } }`. Alpha (`D-SINGLE-ORG`) fills one entry; the shape stays forward-compatible.

**Why.** Self-hosting must remain first-class (`D-12.1`) without pushing infra configuration onto end users. Resolving transport from the same chain source already used for org details + serving auth keeps one canonical path and avoids introducing a separate directory service every self-hosted hub would need to synchronize (same shared-surface rationale as `D-13.4`). Allowing up to 3 ordered endpoints gives a leader simple redundancy/failover (a single hub outage does not cut the org's members off from recall) without a new mechanism — it is still just transport URLs fronting the one on-chain serving identity; the 3-cap bounds chain state and client failover cost.

### D-HUB-RESPONSE-SIGNED: Hubs sign responses with a dedicated on-chain ed25519 key; the plugin verifies

**Decision.** Every hub response is signed with a **dedicated ed25519 response-signing key**, and the plugin verifies each response signature (`X-Hub-Signature`, ed25519 over a canonical digest of the response body) against the org's on-chain **`hub_response_pubkey`** — a new StoredOrg field (hex ed25519 public key) published by the leader via `MsgSetServingInfo` alongside `hub_endpoints`. The dashboard auto-fills it from the hub's advertised key for the canonical/default case; a self-hosting leader supplies their own hub's response pubkey. The signing/verification contract is published in `wevibe-protocol` so every self-hostable hub conforms to one interoperable wire contract.

**Why a dedicated ed25519 key (not the secp256k1 serving key).** Three reasons settled with the gathered facts: (1) the plugin already has ed25519 `verify()` wired (the secp256k1 serving key would force secp256k1 recover + cosmos-address derivation on the client); (2) publishing the pubkey on-chain is a clean trust anchor — no pubkey-recovery and no trusting the hub to report its own key; (3) **key separation** — the response-signing hot key is distinct from the funded secp256k1 serving/tx wallet (`hub_serving_address`), so a leaked response key cannot drain funds or submit serve/deny batches. `hub_serving_address` stays the serve/deny AUTH key; `hub_response_pubkey` is the response-INTEGRITY key. This closes transport-level endpoint-swap / MITM before content checks: a hostile endpoint cannot impersonate the authorized hub without the on-chain-published response key.

**Security / damage-minimization model (shared rationale; hub = UNTRUSTED transport).**
1. Endpoint changes are leader-signed, on-chain, auditable, and revertible (not a silent server flip).
2. Content integrity is endpoint-independent: Umbral end-to-end + consumer-side guard (`D-RECALL-GATE-ON-RISK`) + provenance. A malicious endpoint can withhold, stale, or attempt-inject, but cannot forge content past consumer crypto + guard.
3. Blast radius is per-org: a compromised org endpoint degrades only that org's recall for that consumer, never identity/keys/other orgs.
4. The canonical chain RPC is the only default trust anchor.
5. Hub-response signing (`D-HUB-RESPONSE-SIGNED`) defeats transport-level impersonation.

**Worst case.** If an org/hub is fully compromised, impact is degraded/poisoned recall for that one org; it is caught by guard + crypto and reverted on-chain. It is never key/identity loss and never cross-org contamination.

### D-HUB-ENDPOINT-CHANGE-TOAST: Passive one-time toast when an org's endpoint changes

**Decision.** If a resolved org `hub_endpoints` list changes between sessions, the plugin switches automatically and surfaces a one-time passive toast ("Org X moved its hub"). Otherwise resolution stays silent and requires no user action.

**Why.** This preserves zero-config UX while still giving the consumer a visible signal if a leader-key-authorized re-point occurs.

### D-SIDECAR-PLUGIN-OWNS-STATE: The plugin is the sole writer of the local identity sidecar (`adoptedAt`/`extractedAt`)

**Decision.** Only the plugin (Node, on-device) writes `~/.wevibe/identity.json`; the browser dashboard is sandboxed and cannot touch local files. The plugin stamps `adoptedAt` when it performs local pair/adopt. It learns `extractedAt` by querying the org's serving hub, preferring reuse of an existing per-contributor memory query ("has this pubkey submitted >=1 memory for this org") over inventing a new endpoint.

**Why.** One physical writer prevents split-brain. Hub/chain remain the shared rendezvous for browser-side events, while the plugin owns the device-local state file.

### D-PLUGIN-ONBOARDING-HOOK: opencode TUI onboarding + lazy identity + zero-config installer

**Decision.** This behavior is locked as BUILT in `wevibe-mcp` (commits `6ceac5d..264a29f`):
1. opencode 1.16 user-loadable TUI plugin (`~/.config/opencode/tui.json`) with first-run no-identity flow: `DialogConfirm` ("create identity now?") → Touch ID → `DialogAlert` congrats. Identity creation is step 1 only; rewards come only from joining an org + contributing on dashboard (never from identity creation/click).
2. Session-count nudge: at `N >= 3` distinct sessions (24h cooldown), open `app.wevibe.network` to join/contribute (adopted-aware copy). Slash commands: `/wevibe-setup`, `/wevibe-connect`, `/wevibe-status`.
3. Lazy identity boot: no biometric/keychain load at boot. Boot does only `initCrypto` + non-prompting existence check; membership sync + pre-pubkey registration defer to first use.
4. Non-secret sidecar `~/.wevibe/identity.json` stores public keys + `createdAt`/`adoptedAt`/`extractedAt`/`lastPairingId` + per-org `orgs` map, enabling identity-state reporting and endpoint resolution with no biometric prompt.
5. `wevibe-admin install-opencode` / `uninstall-opencode`: idempotent cross-OS installer that resolves config dir, copies the TUI plugin, merges `tui.json` plugin entry + `opencode.json` `mcp.wevibe` entry, preserves other keys, and refuses to clobber commented JSONC.
6. Identity is seed-based (32-byte Ed25519 seed in OS keychain, biometric-gated on macOS) with BIP39 recovery/pairing; dashboard↔plugin reconcile to one pubkey via short pairing code (both directions) or phrase.

**Why.** This gets a normie vibe-coder from zero to a self-custodial identity in seconds, then to dashboard join/contribute, with no crypto ceremony and no boot-time biometric surprises — aligned with `D-IDENTITY-PROGRESSIVE-CUSTODY` and with contribution accountability remaining leader-gated (`D-LEADER-SOLE-SIGNER`).

## 21. Memory-Extraction Scope: Object Shape, On-Chain Extraction Profile, Preference & Derivation (Sprint-alpha-final — Walter-locked 2026-06-07)

> Locked via Walter design dialogue + a 4-repo build (chain/hub/mcp/dashboard), R-29-verified, UNCOMMITTED at
> time of writing. Reports: `wevibe-meta/workspace/reports/{impl-chain-extraction-profile, impl-mcp-extraction-implement-dnd, impl-hub-preference-carry, impl-dash-memory-object-preference, impl-dash-extraction-profile-editor}.txt`
> and the design + gap docs `design-extraction-prompt-primitive.md` / `gather-memory-extract-gap-surface.md`.

### D-MEMORY-OBJECT-IMPLEMENT-DND: The extraction memory object is implement/dnd (closes the insight/avoid drift)

**Decision.** The contributor extraction object is locked to the D-5.1 schema: `implement` (REQUIRED — what TO do
and how), `dnd` (nullable — what NOT to do and why), `context`, `stack`, `preference_confidence`, plus an
engine-stamped `extraction_hash`. The legacy `insight`/`avoid` field names are renamed end-to-end (MCP
`extraction.ts`, dashboard `session-types.ts` + `sessions/page.tsx` + `/api/extract` validation). The do/do-not
signal is the **`dnd` field's presence** (not a memory type — `memory_type` is already single-valued
`MEMORY_TYPE_MEMORY` everywhere, so D-S32-CO048's chain-enum collapse was already complete; only the field rename
+ field-level filter remained). **Why:** the as-built extraction violated the already-locked D-5.1 / D-S32-CO048;
this is the gating fix to "produce the correct memory object."

### D-EXTRACTION-PROFILE-ONCHAIN: The org's extraction profile lives fully on-chain; the contributor imports it at extract time

**Decision (Walter FORK B + C, "full chain", "on chain now").** Each org carries a leader-configurable
**Extraction Profile** as on-chain state (`x/org` `StoredExtractionProfile { org_id, profile_version,
extraction_model, num_ctx, system_prompt, output_schema, domain_framing, exemplars[], constraints,
updated_at_height }`), set by a leader-signed `MsgSetExtractionProfile` (auth `signer == LeaderWalletAddress`,
mirroring `MsgSetServingInfo`; keeper bumps `profile_version`), and read over REST at
`/wevibe/org/v1/extraction-profile/{org_id}` (auto gRPC-gateway). The FULL profile — including exemplars — is
on-chain, **size-capped in ValidateBasic** (system_prompt ≤ 8192 B, output_schema ≤ 4096, domain_framing ≤ 1024,
constraints ≤ 4096, extraction_model ≤ 256, ≤ 5 exemplars each ≤ 4096, num_ctx ≤ 131072, **total string bytes ≤
16384**, `ErrExtractionProfileTooLarge`). Genesis default = empty.
- **Resolution path (FORK C): chain is SoT, read directly.** The dashboard `/api/extract` fetches the active org's
  profile from chain REST and forwards `system_prompt`→`prompt`, `num_ctx`, `extraction_model`→`model` to the
  MCP `/v1/extract` (which now accepts `prompt`/`num_ctx` alongside `model`/`ollama_url`). The hub stores NOTHING
  for the profile (mirrors D-CHAIN-RESOLVED-HUB-ENDPOINT, one-path). On no-profile/fetch-failure the MCP falls
  back to its `DEFAULT_EXTRACTION_PROMPT` (the shipped sane default; env `WEVIBE_EXTRACTION_PROMPT` is the only
  other fallback).
- **Leader editor** lives in dashboard Settings (leader-gated), broadcasting `MsgSetExtractionProfile` via
  `directBroadcast` with the org-account feegrant (the msg URL is on the leader feegrant allowlist), with
  client-side cap validation mirroring the chain.

**Why.** The extraction prompt is the highest-leverage, WeVibe-controllable lever on recall efficacy
(D-EXTRACTION-MODEL-STANDARD "the controllable half"): it shapes the embedding, the moderation keyword pass, and
therefore whether a memory ever earns Earned Trust (D-4.2) instead of idle-decaying. Putting it on-chain makes the
org's stated quality bar the tamper-evident, forkable source of truth a contributor imports and an auditor can
verify (attestation pillar).

### D-PREFERENCE-QUALITY-SCORE-CARRIED: preference_confidence is a carried quality-guidance score (hub-side), surfaced before chain commit [extends D-5.2]

**Decision (Walter FORK A).** `preference_confidence` (0.0 factual … 1.0 pure opinion) is upgraded from a dead
extraction-only flag to a **quality-guidance score that renders low-quality memory** and is **carried forward**:
MCP extraction → dashboard review card (a "likely/possible preference · low quality" badge guides the contributor)
→ submit payload → hub `pending_submissions.preference_confidence` → returned in the chain-submit listing →
surfaced to the leader at chain-submit (the threshold UI >0.8/>0.5 already existed and is now fed) **to guide how
memories are committed.** It is **HUB-SIDE ONLY** — NOT on-chain, NOT in the signed canonical body, NOT in
`MsgApproveMemory`. It is a pre-chain decision aid (advisory), not a verification anchor. **Why.** High preference
= subjective/low-quality; the leader (sole signer, D-LEADER-SOLE-SIGNER) should see it before committing. As an
advisory aid it does not belong on the immutable signed record.

### D-DERIVATION-FLAG: verbatim | edited-after-extraction, computed client-side, carried hub-side [implements D-EXTRACTION-MODEL-STANDARD §3]

**Decision (Walter FORK D).** Each memory carries `derivation ∈ {verbatim, edited-after-extraction}` =
`H(extraction_output)` vs `H(submitted)`. The MCP stamps each candidate with `extraction_hash` =
`sha256(JSON.stringify({implement,context,dnd,stack}` with sorted keys, stack order preserved`))`; the dashboard
recomputes the byte-identical hash (Web Crypto, same construction) at submit and sets `derivation`. It is carried
HUB-SIDE (`pending_submissions.derivation`, returned in the chain-submit listing) and shown to the leader as an
"edited after extraction" badge — so a hand-edited memory is never dressed as fully model-derived. NOT on-chain,
NOT signed. **Why.** Completes the attestation half alongside the on-chain profile + pinned extraction model; cheap
(hash compare) and honest about provenance.

**Cross-references.** D-5.1 (memory schema), D-5.2 (preference flag, now a carried score), D-5.6 (extraction unified
via MCP), D-EXTRACTION-MODEL-STANDARD (pin model + derivation), D-S32-CO048-MEMORY-TYPE-IMPL (single type — chain
collapse already done), D-CHAIN-RESOLVED-HUB-ENDPOINT (chain-as-directory pattern reused for the profile),
D-S32-ORG-FEEGRANT-ALL (the new MsgSetExtractionProfile rides the org-account feegrant). Pre-MVP wipe per D-13.9
covers the new chain field + hub columns.

---

*End of DECISIONS.md*
