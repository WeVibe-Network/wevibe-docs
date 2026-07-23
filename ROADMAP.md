<!-- ============================================================= -->
<!-- ROADMAP.md                                                    -->
<!-- WeVibe Network — Roadmap (deferred / forward-looking)         -->
<!-- Companion to WP-DESIGN-SPEC.md. Prose lifted from the         -->
<!-- whitepaper's removed forward-looking + economic sections.     -->
<!-- ============================================================= -->

# WeVibe — Roadmap

Capabilities designed but deferred beyond the alpha whitepaper. The token-economic model is not solidified until public mainnet.

> This document collects the forward-looking capabilities and the token-economic model that were carved out of `WP-DESIGN-SPEC.md`. The alpha whitepaper describes the shipping product; everything below is *planned* / deferred. Designer directive tokens (`[[TABLE n]]`, `[[CODE n]]`, `[[DIAGRAM n]]`, `[[CALLOUT]]`) travel with the content they annotate.

---

## 1. Token & Economic Model (not solidified until public mainnet)

*Planned. The entire token-economic model below is deferred; none of it is solidified until public mainnet, and it is not part of the alpha product.*

A single token, **VIBE**, is used for staking, organization-slot acquisition (auction burns), per-memory storage deposits, demand-leg router settlement, and contributor payouts.

**Organization slots (scarce, capped, auctioned).** Capacity is a hard-capped set of registry-allocated slots (governance param: 32 alpha / 320 testnet / 3200 mainnet). Primary allocation is an ascending price per subsequent slot until the cap fills; a freed slot (abandoned/closed/lapsed) is re-homed by a descending (Dutch) resale. The acquisition payment is split 50/50 — half burned, half capitalizing the organization's own on-chain account. The slot `org_id` is permanent and leader-independent. Scarcity plus the burn is the network-level anti-spam.

**Self-assessed-value rent (Harberger-style).** A leader posts a self-assessed value V for their slot, pays rent `r × V` per period, and anyone may force-buy the slot at V during a bid window. This keeps slots in productive hands without right-of-first-refusal chilling or grief-bidding. Non-payment/abandonment frees the slot back to Dutch resale and marks the organization dormant.

[[CALLOUT: KEY — "Per-memory storage deposit is a liveness function, not a participation cost"]]
Each committed memory carries a small VIBE deposit that decays as storage rent and is keeper-claimable as a deletion bounty when exhausted or abandoned; pruning deletes the ciphertext blob but permanently retains accountability metadata. Its purpose is storage hygiene and liveness: it lets keepers reap dead memories and reclaim perpetual storage, and it provides the path to prune an organization's memories if the leader goes AWOL, so the network is not stuck storing orphaned blobs forever. It is explicitly **not** a cost on contributing or a contributor bond — contributors pay nothing; the deposit is funded from the organization side and prices the real (perpetual storage) externality where it occurs. A memory under an open report has its deposit frozen (unprunable until resolved). It is parameterized near zero on testnet, where `x/bandwidth` is the rate-limit guard instead.

**Demand-leg router.** Members' recall-access payments settle through an on-chain router whose one job is to enforce the protocol burn `max(n%, floor)` and route the remainder in-transaction to the leader's wallet (the network holds no revenue account). `n`, `floor`, `r`, and the slot cap are governance params.

**Contributor payouts (contribution-only).** Contributors are paid per **approved memory** from the network contributor-emission budget, gated by a network-set qualification threshold. There is no payout per serve or retrieval. Reputation tiers may scale payout-per-approved-memory, but retrieval counts are excluded from VIBE flows.

### 1.1 Emission Schedule (Locked)

The protocol mints VIBE on a fixed **32-year schedule** from genesis.

[[TABLE 1 — "VIBE allocation"]]

| Allocation | Amount | Notes |
|---|---|---|
| Total supply | 1,000,000,000 VIBE | 10⁶ uvibe per VIBE |
| Foundation (genesis) | 10% = 100,000,000 VIBE | Unlocked at genesis |
| Validator (genesis) | 1% = 10,000,000 VIBE | Docker validator self-delegation |
| Validator 32-yr pool | 570,000,000 VIBE | Emitted linearly over the schedule |
| Contributor 32-yr pool | 320,000,000 VIBE | 1%/yr, capped at 10,000,000 VIBE/year |

*Caption:* Table 1. The locked VIBE emission allocation.

[[CODE 1 — per-epoch emission]]
```
validator_emission = validator_pool_remaining / remaining_epochs
contributor_budget = min(contributor_pool_remaining / remaining_epochs,
                         annual_cap / epochs_per_year)

schedule_length = 11,680 epochs   (32 × 365)
```

- **Qualifying contributors** are those with at least `contributor_qualify_threshold` (default 1) approved memories network-wide in the epoch. The contributor budget is split evenly among them.
- **Global rollover.** If no one qualifies in an epoch, that epoch's contributor budget rolls forward; the integer remainder of an even split also carries forward, so no VIBE is burned by rounding.
- **Genesis seeding.** The emission pool (`validator_pool_remaining_uvibe`, `contributor_pool_remaining_uvibe`, `contributor_rollover_uvibe`, `start_epoch`) is written at chain genesis. Emission and decay both run in the chain's epoch-end hook.
- **Contributor attribution.** Emissions credit the *authoritative* contributor recorded on the committed memory (`contributor_address`), not a consumer-supplied serve payload field. The same address drives serve attribution.
- **Claim path.** Contributor rewards accrue on-chain to a claim-later balance keyed to the passkey identity. They become withdrawable to a wallet only after the contributor links a Cosmos wallet **and** performs the explicit dual-signed on-chain migration (`is_migrated`; link ≠ migrate; §8.3). Privileged capabilities are the exception to wallet-optionality: a leader always has a linked wallet, and members granted `can_moderate` likewise require one (they sign their advisory votes), since the leader is the sole wallet-signing bond for published content. The mint+claim transfer path carries a reentrancy guard and a double-claim/duplication guard as consensus-level invariants.

### 1.2 Validator Economics

Validators earn standard Cosmos SDK staking rewards for running consensus, supplemented by the linear validator-pool emission (§1.1). They store all encrypted memories as part of chain state — this is not separate "operator work," it is inherent to running a node. No separate storage challenges are needed.

### 1.3 Demand-Leg Economics (Non-Custodial Router)

Serve/retrieval attribution is social, not economic: serve counts drive public profiles and badges but do not trigger VIBE payout. Economic demand is the organization access leg:

1. Users buy VIBE and pay organizations for recall access.
2. The access/payment model and pricing are leader-set; payments settle through the on-chain demand-leg router (the hub `org_credits` ledger is a mirror of chain state, never the source of truth).
3. The router burns `max(n%, floor)` of each payment (governance params) — the deflationary sink.
4. The remainder is the leader's revenue, routed in-transaction directly to the leader's wallet. There is no network-held revenue account. The per-organization on-chain account is the operating account only (gas, feegrants, storage deposits, the 50% acquisition-retain capitalization, voluntary top-ups); it never accumulates member revenue.
5. The leader compensates members holding `can_moderate` at discretion from that revenue (their own wallet); there is no protocol-enforced split.

**Router enforcement.** Organization recall access is gated through the router: the hub's `membership_active` flag is set for non-trial members only by the hub's chain watcher upon a confirmed router payment event; `org_credits`/subscription state is a strict mirror of chain payment events. Trial members remain on the orthogonal trial path.

[[DIAGRAM 1 — "The canonical closed loop"]]
[[DESIGN NOTE]] Draw as a circular flow (economic cycle). Use the exact node
labels; the burn is a branch that exits the loop.

Cycle nodes, clockwise:
1. "Emission → contributors (contribution-only; claim after wallet-link + migration) + validators/stakers (mint/sell)."
2. "Users buy VIBE."
3. "Users pay organizations (leader-set model & price)."
4. "Router: burn `max(n%, floor)` + remainder to leader." (The burn portion exits the loop to a "🔥 burned" sink node.)
5. "Leader pays members holding `can_moderate`."
6. "Stake / secure." → arrow back to node 1.

*Caption:* Diagram 1. Leaders earn no emissions, there is no per-serve royalty, and there is no protocol-enforced split for members holding `can_moderate`. WeVibe is non-custodial — p2p payments + memory storage + reputation, holding no user or organization funds. The only consensus-level economic infrastructure required here is the payment router that enforces the burn and routes the remainder in-transaction to the leader's wallet.

### 1.4 Passkey → Wallet Migration

Reputation and earnings accrue under the contributor's passkey-derived identity from the moment they contribute — no wallet required (this passkey-keyed identity model is alpha; see WP-DESIGN-SPEC §8.3). Carrying reputation and earnings onto a wallet is a deliberate, explicit **migration**: an on-chain, dual-signed alias (passkey pubkey → wallet address), gated by the contributor's own memory-contribution trail and recorded with an `is_migrated` flag. Until migration, reputation stays keyed to the passkey pubkey and rewards sit in a claim-later balance; after migration reputation resolves to the wallet via the alias and rewards become withdrawable. Linking a wallet is not the same as migrating (link ≠ migrate). The chain is append-only, so no prior history is rewritten.

---

## 2. Session Attestation (Roadmap, Post-Mainnet)

Sessions produce memories. Without provenance attestation, a contributor could paste fabricated "coding sessions" into the extraction pipeline and farm reputation. Everything downstream — difficulty scoring, quality grading, reputation — depends on knowing the session actually happened.

**Current posture.** Organization-leader curation is the trust layer, and a real one: every published memory carries a leader's bonded signature. A generalized cryptographic attestation rail is post-mainnet roadmap.

**Roadmap direction.** A post-mainnet pluggable attestation framework. Separate attestation components plug into the chain and validate session claims either cryptographically or via API-backed trust services. The target claim shape is explicit session provenance: *"user X using LLM model Y took N turns to solve problem Z."* Post-mainnet, the on-chain `x/attestation` socket generalizes the provenance grades into a single typed proof artifact — `{ proof_type, trust_label, … }` where `proof_type` is one of `tee_receipt`, `zktls_proof`, or `zkml_proof` — each verified off-chain before any on-chain anchoring.

**What an attestation receipt is — and what it reveals.** The core artifact is a signed receipt: an ordered event log binding the hash of each request and response to a hardware-attested workload, signed by a key that the enclave's attestation report vouches for. Receipts carry hashes, never session text — verifying one proves that an attested workload processed exactly those bytes, without disclosing the bytes to the verifier. Verification is programmatic (signature and quote-chain checks any client or hub can run, with no curator in the loop) and the retained proof bundle keeps the claim re-verifiable after the fact. This shape is not speculative: commercial confidential-inference gateways ship per-response signed receipts with exactly these properties today [30][31].

[[TABLE 2 — "Attestation tiers (grounded against the July 2026 landscape)"]]

| Tier | Design | What it binds | Availability |
|---|---|---|---|
| **Tier 0 — Confidential inference (model-attested)** | GPU-TEE confidential inference (Intel TDX + NVIDIA Confidential Computing) signs each request/response inside the enclave; the attestation report binds the signing key to the measured **workload** — the deployed image digest, which pins the exact model weights when images are digest-pinned and reproducible. | Exact bytes + workload/weights measurement + contributor identity | **Shipped today** on commercial gateways for **open-weight models** [31]. Closed-weight coverage requires vendor-operated attested inference: industry precedents exist (Apple Private Cloud Compute, Google Private AI Compute [32][33]) but serve those vendors' own privacy products and issue no third-party receipts. |
| **Tier 1 — CommitLLM cryptographic receipts** | Commit-and-audit protocol for open-weight inference (Lambda Class, MIT-licensed) [12]. | Exact bytes + public checkpoint | Open-weight only; the verifier must hold the checkpoint. |
| **Tier 2 — Attested-gateway receipts** | An independently attested TEE *gateway* signs a receipt over the exact request/response bytes exchanged with a closed-model provider. Supersedes the earlier "WeVibe-controlled proxy" seed design: the gateway's own code is hardware-attested and publicly verifiable, so no trust in WeVibe or any single operator is required. | Exact bytes + channel; model identity is the provider's **self-reported label**, not a measurement | **Shipped today** on commercial gateways for closed frontier models [31]. Defeats fabricated transcripts; does not prove which weights ran. |
| **zkTLS (alternative closed-model path)** | Verifiable HTTPS-session proof (TLSNotary / MPC-TLS): proves the exact bytes of a genuine provider session with selective disclosure, requiring no provider cooperation. | Exact bytes + channel + self-reported label | Technically proven; production-blocked on MPC sent-side cost ceilings and TLS 1.3 support [30]. TLSNotary's 2026 proxy mode trades the MPC step for a network-path assumption at far lower cost and may clear the ceiling. |

*Caption:* Table 2. Attestation tiers against the July 2026 landscape. Strength is graded per field; no tier proves memory truth.

**Pathway consolidation (2026-07-23 — D-PROVENANCE-ADMISSIBILITY-2026-07-23).** These tiers resolve to exactly two admissible pathways: **P1 `ATTESTED_EXECUTION`** (Tier 0 + Tier 1 — open-weight execution with verifiable workload/model identity; CommitLLM-class local verifiable execution is the recorded gap for local users) and **P2 `PROVIDER_WITNESSED`** (Tier 2 attested-gateway + the zkTLS path — closed-weight sessions captured through a blind-witness TLS/transport commitment; the hub is the initial witness). `self-declared`/`unattested` are absence states, never served. P2 proves the provider endpoint + committed bytes + provider-reported label, **not hidden weights**; major closed-weight providers expose no cryptographic hidden-weight proof to ordinary API consumers. Admissible provenance is a distinct SERVING gate (contribution/receipts stay ungated) and requires BOTH a producer-session leg and an extraction-session leg (§2.1).

**Benchmark attestation contract (ratified 2026-07-23).** The benchmark path now runs a mock attestation adapter that preserves the same logical receipt envelope and off-chain verifier boundary as production; only the trust root changes (the benchmark harness as a trusted test oracle instead of TEE attestation). Benchmark outputs are explicitly labeled `bench-mock/self-declared`, are never surfaced as a live attestation tier, and are never presented as cryptographic proof (R-PUBLIC-PROOF-GATE; D-MOCK-ATTESTATION-CONTRACT).

### 2.1 The Folded Pipeline: From Attested Sessions to Certified Memories

A session receipt alone certifies the session, not the memory: extraction and human editing sit between them. The folded pipeline closes that gap by chaining a second receipt over the extraction step itself. The extraction model runs on attested inference; its receipt binds the session transcript (hash-linked to the session receipts) and the pinned extraction prompt as input, and the memory candidates as output. A verifier can then confirm, over hashes alone, that a submitted memory is the **verbatim output of a pinned extractor applied to a genuine session owned by the contributor**.

The result is a per-field provenance ladder, surfaced as graded labels and never as a single "certified" badge:

`pipeline-attested (verbatim)` → `session-attested, edited after extraction` → `attested-gateway session` → `self-declared`

An edited memory is not penalized — review and editing are first-class consent steps — it simply carries the grade matching what can be proven. The folded pipeline requires an attestable extractor: open-weight today; closed extractors join if and when a vendor ships attested inference.

**Per-memory producer-model provenance (ratified 2026-07-23).** The ladder now carries an immutable per-memory producer-model stamp (canonical producer identity for the source-session model) plus attestation reference at commit time; the hub materializes and indexes these fields as rebuildable derived state (D-PRODUCER-MODEL-PROVENANCE).

[[CALLOUT: WARNING — "What attestation does and does not prove"]]
Attestation is anti-**fabrication**, not a truth oracle. A full receipt chain proves the session genuinely ran on the attested model, belonged to the contributor, and — folded — that the memory is the extractor's verbatim output. It does **not** prove the session was substantive (padding with junk turns remains possible; turn counts are scoring inputs, never raw effort), that the session was not deliberately staged, or that the memory's guidance is correct. Correctness remains the curation layer's job. Producing receipts also carries a disclosed privacy tradeoff: attested sessions route through confidential-inference providers rather than the local-only path, adding the hardware vendors to the trust root — though the receipts themselves carry only hashes, never session text. Attestation stays optional, opt-in, never a contribution gate, and never coupled to token economics — and every tier is carried as a pluggable, re-verifiable adapter, never a hard dependency on any single enclave vendor (TEEs have a side-channel history; the retained bundle keeps claims verifiable even if a provider's API changes or is compromised).

[[CALLOUT: KEY — "The extraction leg needs its own provenance (the controllable-half claim, corrected 2026-07-23)"]]
"Weaker models surface weaker memories" has two heads: the *production* model in the user's coding suite (uncontrolled — and sometimes the most valuable signal, when a weak model solves a hard problem) and the *extraction* model that distills a session into a memory. **Correction (D-PROVENANCE-ADMISSIBILITY-2026-07-23):** WeVibe does NOT control a small pinned extractor, and extraction is NOT one call. Extraction is post-hoc, user-triggered, and is itself an LLM session — the USER supplies the extraction model/provider/key (remote OpenRouter default or local), one call only if the transcript fits, else overlapping chunk fan-out. Production currently submits `attestation:null` with no session binding. Because the extraction session is a real, user-chosen LLM leg, it needs its OWN admissible provenance (P1 for local via CommitLLM-class verifiable execution; the remote leg witnessed/attested per pathway) and must be welded to the producer trajectory (§2.1 welds a/b/c). Attesting the production model is the bonus on top; attesting the extraction leg is what makes a memory servable end-to-end.

*Bracketed citations in this section resolve in `WP-DESIGN-SPEC.md` §12 References.*

### 2.2 Goal-Sealed Verification Receipts (Roadmap Tiers T3–T4)

Goal-sealed trajectory verification (`WP-DESIGN-SPEC.md` §8.12) grades memories into verification tiers T0–T4. Tiers T0–T2 ship in the alpha as local, contributor-signed receipts; the top two tiers are roadmap-scoped and anchor into this attestation rail:

- **T3 — attested-run receipts.** The GSTV receipts themselves (predicate and ablation) are produced by an attested runtime rather than merely claimed, by committing the goal seal and receipts through the `x/attestation` socket (§2, §8) over the same signed-event-log mechanism used for session receipts. This upgrades a T2 receipt from "checkable on contributor hardware" to "produced by an attested runtime."
- **T4 — stranger convergence.** k independent contributors, each sealing an independent goal against a matching context fingerprint, converge on the same fix. The certification is the agreement itself: a sybil must actually solve the task k times against k pre-fixed predicates. Requires corpus traffic and is inert at cold start.

The full T3 activation criteria (attestation over GSTV artifacts) and the T4 convergence specification (k, context-fingerprint matching, convergence detection) are deferred to a dedicated follow-on order (the GSTV `ROADMAP.md` DMO). This subsection registers them as forward-looking and resolves the T3 reference in `WP-DESIGN-SPEC.md` §8.12.

---

## 3. Two-Layer Difficulty Scoring (Roadmap, Requires Attestation)

Two-layer difficulty scoring is the evolutionary continuation of attestation and its likely first consumer once the pluggable framework exists. The chain carries `difficulty` and `quality` values (1–10) on attested-memory records and maintains a per-contributor difficulty histogram (`x/reputation`); the scoring layers below populate those values automatically once live.

How attested difficulty enhances the economic and/or social-graph layers is intentionally **TBD**. What attestation settles is the *trustworthiness of the inputs*: the model and the turn count become cryptographically grounded. The composite claim "user X using model Y took N turns to solve problem Z and drew L negative signals" is rendered as a **per-field graded provenance** object in the social-graph layer — each field labeled by its own grade (attested model/turns, native-on-chain user/denials, descriptive problem tag) — display/badge-only and never coupled to VIBE.

- **Layer 1 — Structural signal (automated, cheap).** Model-capability coefficient × turn count × (1 + 0.25 × failed alternatives). Computed from session structure without understanding content.
- **Layer 2 — LLM grading (semantic, authoritative).** A separate grading LLM evaluates non-obviousness, specificity, and reasoning progression. Temperature 0, deterministic, hash-seeded. The grade and session hash are committed together.

### 3.1 Capability Eligibility Gate + Soft Origin Prior

Capability direction is now a hard pre-scoring admission gate (ratified 2026-07-23; D-CAPABILITY-ELIGIBILITY; D-CAPABILITY-REGISTRY): producer memories inject only into equal-or-lower-capability consumers, unknown producer/consumer classifications fail closed, and exact-self injection is allowed only when producer identity is proven.

Inside that eligible set, model origin remains a soft retrieval prior only: generic conceptual memories can still be deprioritized while highly specific memories take no penalty. That soft-prior scoring behavior is unchanged, and eligibility itself is never a scoring input or boost.

---

## 4. Skills & Federation

### 4.1 Task-Context Skills

Skills are curator-defined collections organized by task context — deployment, testing, error-handling — **not** by difficulty level. A skill is a lightweight named set of memories with a description that improves curation and social discoverability inside an organization. Ungrouped memories are allowed, and skill assignment is optional.

### 4.2 Federation (Design Phase)

Federation operates at the skill level. Organizations publish skill packages; receiving organizations set quality thresholds. No individual contributor reputation crosses federation boundaries. Federation is deferred until the alpha curation loop is proven.

---

## 5. Documentation Seeding (Cold-Start)

New organizations import canonical documentation as seed memories (`source: doc_import`), with session contributions extending and replacing the seeds over time.

---

## 6. Badge On-Chain Freeze

Badge rarity tiers are computed read-time by the social-graph client in the alpha (see WP-DESIGN-SPEC §8.5–§8.6). The **on-chain freeze of rarity semantics lands at mainnet** — freezing the rarity computation into chain state so tier labels are fixed at the protocol layer rather than recomputed by each display client.

---

## 7. Additional Editor Integrations

The OpenCode plugin (`wevibe-opencode-plugin`) is the shipped reference implementation. Additional editor integrations are planned:

| Agent | Plugin type | Hook mechanism |
|---|---|---|
| **Claude Code** | Plugin with a `.claude-plugin/plugin.json` manifest | PreToolUse hook; `permissionDecision: "deny"` |
| **Cursor** | Hooks + marketplace plugin | Claude Code hook-format compatibility |
| **Cline** | VS Code extension + hooks | `.clinerules/hooks/`; custom hook system |

All plugins call the same local MCP server and the same wevibe-guard binary (`WEVIBE_GUARD_BIN`); decryption runs through the local Umbral sidecar; retrieval/search stays in hub APIs.

---

## 8. Implementation Phases

[[DESIGN NOTE]] Everything here is forward-looking. Consider a subtle visual
treatment (e.g. a "roadmap" band or muted accent) so a reader never confuses it with
shipped architecture.

[[TABLE 3 — "Phased delivery"]]

| Phase | Theme | Scope |
|---|---|---|
| **I — Alpha** | The knowledge loop + the accountability rail | Working memory contribution/review/retrieval loop with encrypted on-chain storage; organization membership/role flows and leader-signed curation; gated injection (four-button approval) across MCP/plugin entry points with guard/sanitization; consumer-filed on-chain reports, resolution window, public escalation, and direct-broadcast path; chain-resolved hub endpoints with signed responses; serve-attribution counters on-chain for contributor + organization signals. |
| **II — Provenance surfaces + economic settlement** | Make provenance visible and settle economics | Public contributor/organization profile surfaces over chain data; badge surfaces (serve milestones, contribution volume, rarity tiers) in the reference social graph; demand-leg router settlement, claim-later reward path, and slot resale/rent mechanics; curator workflows for direct authoring, documentation seeding, and skill curation. |
| **III — Post-mainnet** | Attestation + federation | Pluggable session-attestation framework (§2), generalizing `x/attestation` to typed proof artifacts; two-layer difficulty scoring (§3) as an optional quality input once attestation is live; skill-level federation with receiver-side quality thresholds and no cross-organization reputation portability. |

*Caption:* Table 3. Phase I exit criteria: a reliable daily loop from session output to approved memory to gated recall, with the report/expose accountability rail live end-to-end and attribution updating public provenance signals.

---

## 9. Network-Governed Fee Policy & Mainnet Capacity (Roadmap, Post-Mainnet)

*Planned. This section roadmaps the move from today's validator-local fee configuration to a network-governed fee policy, and records ballpark planning figures for a filled mainnet. Delivery maps to §8 Phase III (Post-mainnet). The figures below are planning estimates, not a mainnet sizing commitment.*

### 9.1 Fee policy: from validator-local minimum to network-governed (`x/globalfee` vs `x/feemarket`)

**Current posture.** The chain has no on-chain or governance-controlled fee floor. The minimum gas price is purely **validator-local** (`app.toml minimum-gas-prices`), enforced only in each validator's own CheckTx. The live single-node dogfood chain runs at an observed **0.025 uvibe/gas**, and the hub's fee calculator pays that same price with a **2,000 uvibe per-tx floor** (D-13.12; base denom **uvibe**, where 1 VIBE = 1,000,000 uvibe is display-only). On a permissioned single-node chain that one setting is the whole network's floor; across heterogeneous public validators the effective floor is the lowest minimum among proposing validators, so there is no uniform, governable price today. Fees are gas-proportional and load-independent (`max_gas = -1`, so there is no gas auction); congestion surfaces as mempool latency, not higher fees. This price signal is one anti-spam layer, complementary to slot scarcity plus the acquisition burn (§1) and the testnet `x/bandwidth` rate-limit guard (§1) — not a replacement for them.

**Roadmap direction.** Move the minimum gas price out of per-validator config and into a **network-governed fee policy** set by governance, so a public multi-validator chain has one uniform, adjustable floor. There are two mutually-exclusive implementation paths, selected at the decision gate below — **not** both installed at once:

[[TABLE 4 — "Governed-fee options"]]

| Path | Model | When it fits |
|---|---|---|
| **`x/globalfee`** (simpler, static — the near-term path) | A static, governance-set network-wide minimum gas price: one gov param, predictable, and it preserves the hub's fixed-fee assumption. | The permissioned / early-public phase, where load is modest and a stable, cheap floor is all that is needed. |
| **`x/feemarket`** (dynamic, demand-based — a possible later evolution) | An EIP-1559-style base fee that rises and falls with block fullness. | Only if sustained demand-driven congestion later makes a static floor inadequate — a later evolution, not a launch requirement. |

*Caption:* Table 4. Two candidate fee-policy modules. Exactly one is adopted; the other is deferred (see the decision gate).

**Interim (config-only, no chain change).** Independent of module adoption, the near-term floor can be lowered by validator config from 0.025 to a proposed **0.0001 uvibe/gas** — **250× lower** — which assumes the hub's 2,000 uvibe per-tx floor is removed or reduced to match (otherwise the hub floor dominates and the lower price has no effect). That keeps a non-zero token anti-spam signal while making per-op fees dust. It is a policy choice, not a chain-mechanics limit.

**Zero-fee is a failure mode, not an option.** Setting the minimum to 0 removes the only price-based anti-spam signal. Combined with the org serving key's **unlimited, non-expiring feegrant** and **`max_gas = -1`** (no per-block gas cap — blocks are bounded only by their 21 MiB of bytes), a compromised or hostile serving key could spam full blocks for free until revoked. That directly weakens the documented serving-key **containment** property (D-S32-CO044 / D-SERVE-CONSUMER-SIGNED: a stolen serving key's blast radius is "drain its own gas"), which assumes gas costs something. Do not default to zero.

**Decision gate (before any module is wired).** Choose `x/globalfee` vs `x/feemarket` against: expected mainnet load versus the capacity in §9.2; whether static predictability or dynamic congestion-pricing is actually needed; hub compatibility (feemarket's variable base fee must not break the hub's `ceil(gas × price)` + floor calculator); and governance/operational overhead. Adopting either is a **net-new chain change** — no canon decision and no app wiring exists today — requiring: a Walter-approved decision; a coordinated **chain upgrade** (governance upgrade proposal); **ante-handler wiring** of the fee decorator; **tests** (ante unit + fee-floor integration + hub broadcast-compatibility); **validator rollout** (config alignment / upgrade coordination across heterogeneous validators); **observability** (per R-37: log the effective floor and rejected-for-fee counts); and a **security review** of the interaction with the org feegrant and serving-key containment. One-path discipline (R-13): exactly one module is selected and wired, and the other is explicitly deferred — never two active production fee paths.

### 9.2 Mainnet filled-org planning capacity

[[CALLOUT: KEY — "Activity cadence and payload size drive load, not registrations"]]
Throughput at scale is set by how often members act and how large their encrypted payloads are — **not** by the headcount of registered members. The figures below are **ballpark planning estimates** derived from a live-but-idle single-node chain; they require a fresh size-distribution / load benchmark before any real mainnet sizing.

**Model.** A filled mainnet is **3,200 org slots × 1,000 members = 3.2M members** (the mainnet slot cap, §1). The binding resource at capacity is **block bytes**, not gas or fee: `max_gas = -1` (no block gas limit) and `max_bytes = 22,020,096` (**21 MiB/block**) are consensus-verified [measured], and block time is **~5.01s observed** (the CometBFT default; not a configured param). Usable throughput ≈ 21 MiB × 0.9 ÷ 5.01s ≈ **~3.95 MB/s**. The "unit" is one serve entry + one memory commit+approve **pair**. Note that nearly all figures below are **inferred** arithmetic — the per-tx byte sizes are extrapolated, not swept; only the consensus params and the block time are measured.

[[TABLE 5 — "Filled-mainnet capacity (planning estimates)"]]

| Payload assumption | Bytes / pair | Pairs/s (byte ceiling) | Saturation ≈ pairs/member/day (3.2M members) |
|---|---|---|---|
| **Lean ~1.5 KB** (≈320 B serve entry + ~1.5 KB memory) | ~1,820 B | **~2,170** | **~59** |
| Rich ~5 KB (memory-heavy) | ~5,320 B | ~742 | ~20 |

*Caption:* Table 5. Capacity is byte-bound because `max_gas = -1`; payload size sets the ceiling.

**Cadence scenarios** (lean ~1.5 KB column, against ~2,170 pairs/s):

[[TABLE 6 — "Per-member cadence vs byte capacity"]]

| Per-member cadence | Network pairs/s | % of byte capacity | Verdict |
|---|---|---|---|
| 1 / day | 37 | **~1.7%** | trivial |
| 1 / hour | 889 | **~41%** | comfortable |
| 1 / 10 min | 5,333 | **~246%** | over capacity — **not sustainable** (mempool queues) |

*Caption:* Table 6. At 3.2M members, daily and hourly cadences sit far under the byte ceiling; a 10-minute cadence exceeds it.

**Reading it.** At the lean ~1.5 KB payload the chain sustains up to **~59 serve+submit pairs per member per day** before the 21 MiB byte ceiling binds; at a memory-heavy ~5 KB payload that drops to **~20 pairs per member per day**. Daily and hourly cadences sit far under capacity; a 10-minute cadence across all 3.2M members exceeds it. Because load is byte-bound, the levers are payload size and cadence, not member count. **These remain ballpark figures — a fresh across-sizes size-distribution / load benchmark is required before mainnet sizing.** Source analysis: `wevibe-meta/workspace/reports/15-07-26-2114-gas-fee-full-capacity-analysis.md`.

---

## 10. Open Design Questions

[[DESIGN NOTE]] Render as an honest, plainly-styled Q-list. These are deliberately
unresolved; do not dress them up as settled.

- **Earned-Trust settlement lag.** What is the maximum acceptable divergence window between hub-optimistic ranking and chain-settled weights? The decay formula does not change — only event durability/timing semantics are in question (WP-DESIGN-SPEC §7.7).
- **Pending-memory retention window.** How long do pending (unreviewed) commitments stay on-chain before auto-purging — 72 hours? Configurable per organization?
- **On-chain storage limits.** At what scale does on-chain storage warrant tiered pruning beyond keeper reaping — 100K memories? 1M? Chain-state growth and validator hardware are modeled against adoption.
- **Embedding-model evolution.** `nomic-embed-text` is confirmed for now. Future upgrades mean re-embedding all memories — this happens client-side at approval but needs coordination across organization members.
- **Federation rollout scope.** Which minimum skill-package contract ships first (metadata, provenance, quality thresholds) when federation moves from design phase toward implementation?
- **Social-signal integrity for serves.** Serves are social/status signals (not payout inputs), but self-serve and coordinated-serve patterns still need a monitoring and visibility policy.
- **Retained-transcript access control (attestation).** Attestation receipts carry only hashes, but the anti-padding/fidelity judgment (§2, §3) needs the plaintext transcript. Who may read the retained transcript — the scorer, the leader, or only the contributor via selective disclosure — and where it lives, is undecided. It must never reach the hub, which would otherwise become a session oracle.
- **Attested-gateway grade weighting.** How much display/ranking weight a Tier-2 receipt (bytes-bound, model label self-asserted) earns relative to Tier-0 model-attested provenance, without a single badge overstating either.
