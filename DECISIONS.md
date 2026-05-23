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

**New decisions in this update:** D-13.1 (Moderator Pubkey Persistence on Memory), D-13.2 (Upheld Report Plaintext + Ciphertext + Capsule Triplet), D-13.3 (Hub-Side Manipulation Alarm via chain_commit_events), D-13.4 (Social Graph Data On-Chain, Display Layer Separate), D-13.5 (Reputation Active at Genesis, Additive-Only), D-13.6 (Memory State Cleanup — 7-State Lifecycle Locked at Code Level), D-13.7 (Cross-Module Event Wiring), D-13.8 (Reputation as Tiering Signal — total_approved_memories), D-13.9 (Chain Wipe Acceptable Pre-MVP), D-13.12 (Chain Broadcast via Comet RPC), D-13.13 (Chain Pruning + IAVL Dev Settings); updates to D-2.2 (Umbral container) and D-13.10 (only host exception is Ollama).

---

## 1. Identity & Wallet Architecture

### D-1.1: Cosmos Wallet Is Primary Identity

**Decision:** Every user's primary identity is a Cosmos-compatible wallet (Keplr, Leap, etc.). All other keys derive from or are authorized by this wallet.

**Why:** The wallet connects users to the chain economy (emissions, treasury, reputation) and provides the signing key for all authenticated operations. By establishing the wallet as primary identity, every action in the ecosystem has a single root of trust. Without wallet integration, there is no chain identity, no treasury interaction, no emission payouts, and no reputation tracking tied to a real wallet.

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

**Decision:** The PRE encryption identity (used for Umbral re-encryption decryption) is a BIP-32-derived child key from the Cosmos wallet, with a distinct derivation path from the transaction signing key.

```
Cosmos wallet key (secp256k1, BIP-32 path m/44'/118'/0'/0/0)
    ├── Transaction signing (delegated via MsgGrant)
    └── BIP-32 derived child key ("wevibe-pre-identity/v1")
            └── PRE encryption identity (Umbral SecretKey)
```

**Why:** Coupling financial identity (Cosmos wallet) with confidential memory retrieval (PRE encryption) increases blast radius. Standard BIP-32 derivation paths separate the two: compromise of the PRE identity key does not expose the wallet key. This keeps encryption identity tied to the user's chain identity while providing cryptographic isolation between roles.

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
- **Policy enforcement engine** — rate limits, credit deduction, audit logging

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

### D-4.2: Dual-Vector Decay Model

**Decision:** Keyword weights move along two independent vectors per epoch:
- **Idle decay** (50 bps/epoch) — applied when memory had 0 serves AND 0 denials
- **Negative-signal decay** (500 bps per denial event) — applied per denial TX
- **Serve boost** (100 bps per serve, capped at 5 serves/epoch) — applied per serve TX
- **Bootstrap grace** (14 epochs) — no decay during grace period; boost still applies

Denial decay is applied first, then serve boost — net effect of both signals visible in the final weight.

**Why:**
- **Denials must take precedence.** A memory being actively reported as wrong is a stronger signal than serves; we want the negative signal to dominate.
- **Boost still applies during grace.** New memories need a fair chance to accumulate positive signal; punishing them during their initial epochs would suppress new knowledge.
- **Idle ≠ denied.** A memory with no signal at all this epoch isn't necessarily wrong — it just wasn't queried. Slow idle decay handles that case without penalizing it as harshly as denial.
- **Memories archive when ALL keyword weights = 0.** Terminal state is reached when every topic has fully decayed.

**Note on bootstrap grace duration:** The 14-epoch bootstrap grace period is measured in **chain epochs** (D-4.7), not wall time. The actual wall-clock duration depends on the chain's epoch configuration at initialization (query via `wevibed query epochs epoch-info wevibe_epoch`). With a 12-hour chain epoch, 14 epochs ≈ 7 days. With a 24-hour chain epoch, 14 epochs ≈ 2 weeks. The D-4.2 comment "~2 weeks at 12h/epoch" assumes 12h epochs — verify against actual chain configuration.

---

### D-4.3: Per-Event Gas Model

**Decision:** Each retrieval event (serve or denial) is one chain transaction. The chain applies the keyword weight changes synchronously.

**Why:** Predictable per-event gas cost is easier to reason about than batch-style amortization. Per-event commitment also makes the chain the immediate source of truth — no batching window during which hub and chain disagree.

---

### D-4.4: Archived at Confidence Zero, Not Dormant

**Decision:** Memories with all keyword weights at 0 transition directly to `ARCHIVED`. The `DORMANT` state is not used.

**Why:** `DORMANT` was a holdover from the per-memory confidence design that implied "recoverable if usage returns." Under keyword weights, full decay across all keywords means the memory has been comprehensively rejected — recovery is not the right path. `ARCHIVED` memories are excluded from all queries.

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

**Why:** With `confidence_bps` removed (D-4.1), intermediate health states have no signal to track. A memory is either committed and live, archived (all keyword weights at 0), or removed via report. Pre-commit states (`pending`, `pending_keyword`, `pending_chain`, `denied`) capture the moderation pipeline. The three removed states described gradient health that no longer exists in the model — keeping them around would be theater. Simpler state machines have fewer bugs.

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

The bootstrap grace period (D-4.2) is measured in **chain epochs**, not rotation epochs. The "~2 weeks" estimate is `14 × chain_epoch_duration` — the actual duration depends on the chain's epoch configuration at initialization.

---

## 5. Memory Types & Extraction

### D-5.1: Two Memory Types Only

**Decision:** Memories carry one of two types: `correct_implementation` or `negative_signal`. There is no "preference" type.

**Why:** Type proliferation creates ambiguity at extraction and moderation time. The preference-vs-fact distinction was considered as a third type and rejected for six reasons:

1. **Decay model breakage.** The dual-vector decay model treats serves as positive signal and denials as negative signal. If "preference" were a third type, denials on preferences would mean "I disagree" — not "this is wrong" — which corrupts the keyword weight signal entirely. The decay model depends on denials representing factual inaccuracy, not opinion divergence.

2. **wevibe-guard scanning limitations.** The guard was designed to detect credential leakage, injection patterns, and homoglyph attacks. It cannot meaningfully scan soft/opinion content for threats it wasn't built to detect. Classifying preferences as a separate type would require the guard to make judgments about opinion vs fact — a task beyond its threat model.

3. **Keyword classification breakdown.** Soft terms like "we prefer functional style" or "I like error-first callbacks" don't map cleanly to vocabulary keywords. Forcing a type label on content that doesn't naturally decompose into topic keywords creates false precision in retrieval matching.

4. **Moderation burden.** Preferences are the highest-friction category for moderators because right/wrong judgment simply doesn't apply. Auto-classifying as a third type would push this friction onto moderators who would need to constantly override the classification.

5. **False-positive rate.** Preference detection has a 15-30% false-positive rate. Auto-classifying as a type would cause too many valid factual memories to be miscategorized as opinion.

6. **The flag is sufficient.** The continuous `preference_confidence` score (D-5.2) handles this signal without requiring a third type. Moderators see the flag and decide whether to approve — the system never auto-rejects based on it.

Memory type is stored on-chain (chain is authority) and the moderator can override at approval time.

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

**Why:** Operational decisions like quorum thresholds change as orgs scale. Storing them on-chain would require governance transactions for every adjustment. The hub enforces the threshold; the chain records the outcome (approved memory + moderator pubkey). The chain doesn't need to validate the vote count — it trusts the hub's approval signature.

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

### D-9.2: Phase 1 Qdrant Hardening — Noise Injection + API Key Auth

**Decision:** Phase 1 mitigations against Qdrant embedding inversion attacks:
- **Gaussian noise injection (σ=0.1)** at storage time — reduces inversion accuracy ~40% while preserving >95% recall
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

**Why:** Raw vector similarity isn't enough — keyword weights add health-aware ranking, ensuring decayed memories rank lower than well-validated ones for the same query.

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

### D-11.5: Risk Appetite Is Client-Side Filter

**Decision:** Consumer risk appetite (`lowest | neutral`) is enforced client-side in both the plugin AND wevibe-mcp (via separate read implementations of the same `~/.wevibe/plugin-config.json`). The hub does NOT filter by risk appetite — it returns all relevant memories. The filter is applied after retrieval, before the approval UI.

**Why:**
- The hub doesn't need to know consumer preferences. Filter logic on the hub would mean every retrieval request includes appetite metadata, hub stores it, hub applies it — coupling the consumer's preference setting to authenticated state.
- Client-side filtering also means changing the setting is instant (no hub-state round-trip).
- The duplication of read logic (plugin + wevibe-mcp) is intentional: they're separate runtimes. Reconciling them via a shared library would mean compiling wevibe-mcp's TypeScript into the plugin or making the plugin a child process of wevibe-mcp — both worse than the small duplication of a `getRiskAppetite()` helper.

**Accepted tradeoff:** Both the plugin (`wevibe-opencode-plugin/plugins/wevibe-plugin.ts`) and wevibe-mcp (`wevibe-mcp/src/risk-appetite.ts`) implement their own `getRiskAppetite()` function reading `~/.wevibe/plugin-config.json`. This is the intended design, not a bug to fix by extracting shared code.

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

**Why:**
- **Multi-mod quorum accountability requires preserving all participants.** For orgs with `required_approvals > 1`, only recording the final approver who triggered the chain commit lets the other quorum members escape accountability if a memory later turns out to be harmful. The whole point of the moderator social graph showing "moderator X approved Y memories that were later upheld-reported" depends on every approver being on-chain.
- **Single-mod orgs are not affected.** With `required_approvals = 1`, the array contains one entry. No semantic change for them.
- **Chain doesn't validate the array.** The hub enforces the vote threshold (D-6.3). The chain records the array as-is. Trust is "the leader signed the TX, so they vouched for the array."
- **Committing leader is the on-chain TX signer.** They sign the batch commit with their wallet. Recording them separately from approvers preserves the distinction between "who voted approve" (operational) and "who chose to commit this batch to chain" (authoritative).

---

### D-13.2: Upheld Report Plaintext + Ciphertext + Capsule Triplet

**Decision:** When a report is upheld and committed on-chain via `MsgReportMemory`, the chain stores all three of:
- `plaintext` (raw memory content, max 4096 bytes)
- `ciphertext` (Umbral-encrypted blob, max 8192 bytes)
- `capsule` (Umbral capsule, ~200 bytes)
- `plaintext_hash` (SHA-256 of plaintext, always present)
- `plaintext_oversized` flag

For memories exceeding the 4KB plaintext cap, `plaintext_oversized=true` and the plaintext/ciphertext/capsule fields are empty — only `plaintext_hash` is stored. The full plaintext is published off-chain (hub stores it permanently) and verified against the on-chain hash.

A new gRPC query `VerifyUpheldReport(memory_hash) → {plaintext, ciphertext, capsule, plaintext_hash, plaintext_oversized}` enables anyone to challenge a leader's upheld-report TX by:
1. Computing `sha256(plaintext)` from returned plaintext
2. Verifying it matches returned `plaintext_hash`
3. Independently decrypting `ciphertext` (if they have access via PRE) and verifying it produces the same plaintext

**Why:**
- **The check-and-balance against a rogue leader needs cryptographic verifiability.** Without ciphertext+capsule on-chain, a malicious leader could "uphold" a fabricated report by publishing fake plaintext to discredit a contributor. Storing the triplet means anyone can demand the leader prove decryption matches.
- **Plaintext alone is insufficient.** It's just a string the leader claims is what was decrypted. The ciphertext+capsule are the cryptographic evidence that the published plaintext is indeed what the contributor originally submitted and the moderators approved.
- **4KB cap is economically considered.** At Cosmos gas costs (~200 gas per byte), a 4KB plaintext + 8KB ciphertext + 200B capsule + metadata = ~1-2M gas per upheld report. Upheld reports are rare and consequential — paying 10-40× normal approval gas is acceptable. 4KB covers ~95% of memories based on typical extraction outputs.
- **Oversized fallback preserves the property.** Even for large memories, the on-chain hash is evidence of deletion. Anyone with the off-chain plaintext can verify against it. The leader cannot fabricate; they can only commit.
- **`VerifyUpheldReport` is a query, not a write.** No gas cost for verification — anyone can audit any leader at any time.

---

### D-13.3: Hub-Side Manipulation Alarm via chain_commit_events

**Decision:** The check-and-balance against leader manipulation of approval or report TXs is NOT chain-side runtime validation. It is a hub-side event log + moderator notification:

- New `chain_commit_events` PostgreSQL table on every hub instance records every chain TX involving moderators (`MsgApproveMemory` and `MsgReportMemory`).
- Hub `ChainWatcher` subscribes to CometBFT blocks, parses relevant TXs, writes events idempotently with restart-safety via a `watcher_state` table.
- For each moderator listed in `approvers[]` (approval) or `upholding_moderators[]` (upheld report): emit notification through D-12.9 activity feed channel.
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

**Decision:** The social graph is split into three layers:

| Layer | Lives In | Contains | Owner |
|---|---|---|---|
| Immutable provenance | wevibe-chain | Aggregates, indices, on-chain events with wallet pubkeys | Chain (cryptographic) |
| Operational queue | wevibe-hub | Pending submissions, votes, batches, chain_commit_events | Hub operator (WeVibe-hosted OR self-hosted) |
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

## 14. Sprint 29 Chain Foundation Decisions

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

### D-S29-IAVL-QUERY-BUG-KNOWN: IAVL State Queries Broken — Workaround Documented [ALPHA — MUST FIX]

**Decision:** All ABCI state queries fail on wevibe-chain with `"version does not exist"`. Root cause is in the app's multistore/IAVL configuration. The chain produces blocks and processes TXs normally. Workaround: use TX submission commands + HTTP RPC status + TX-index queries (`query tx <hash>`) instead of state queries.

**Discovery:** CO-005b upgrade verification (2026-05-23). Reproduced across multiple fresh chain initializations in Docker.

**Impact:** CLI `query` commands unusable. Blocks validator operations, dashboard state queries, hub gRPC queries. Does NOT block consensus, TX processing, or upgrade mechanism (proven by CO-005b).

**Status:** Tracked as GAP-CHAIN-20. Requires dedicated diagnostic CO before alpha.

---

### D-S29-CHAIN-RESTART-FOUNDATION

**Decision:** wevibe-chain's InitChainer writes a sentinel marker key (4-byte 0xFF prefix) to every mounted KV store after ModuleManager.InitGenesis. Without this, IAVL's empty-tree optimization skips persistence for stores backing modules that don't implement appmodule.HasGenesis, causing ErrVersionDoesNotExist on any restart. Discovered via CO-005c; fixed in CO-005d.

---

### D-S29-INITCHAINER-VERSION-MAP

**Decision:** wevibe-chain's InitChainer calls UpgradeKeeper.SetModuleVersionMap(ctx, ModuleManager.GetVersionMap()) after ModuleManager.InitGenesis returns. This is required for any manually-wired (non-depinject) Cosmos SDK chain. Without it, x/upgrade's ApplyUpgrade reads an empty fromVM, RunMigrations treats every module as new, and re-runs InitGenesis on already-initialized state — panicking on distribution's balance invariant check. The canonical guidance is at cosmossdk.io/x/upgrade@v0.2.0/module.go:130-131. Discovered via CO-005c-resume; fixed in CO-005e.

---

### D-S29-HUB-SEQUENCE-RACE

**Decision:** wevibe-hub's broadcast.go cross-checks queried account sequence against a local post-broadcast cache (max-of-two). Without this, successive broadcasts within one CometBFT block window race and the second is rejected with sequence mismatch. Known limitation: max-of-two doesn't handle rejected broadcasts; proper fix is in-flight counter pattern (documented in code). Discovered via CO-005d dogfood; fixed in CO-005d Stage 9.

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

*End of DECISIONS.md*
