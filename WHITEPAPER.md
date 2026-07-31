# WeVibe Network

## Verified Memory for Coding Agents — Owned by No One, Killable by No One

*A censorship-resistant knowledge network where hard-won engineering fixes are encrypted on-chain, curated by accountable human experts, and delivered straight into any coding agent — readable by no one in the middle, deletable by no one at all.*

**Architecture & Design Specification · July 2026 · Classification: Public**

---

## Abstract

WeVibe is a memory layer for AI coding agents that makes shared memory usable across trust boundaries. It delivers version-exact fixes and negative knowledge as attributed memories: each item is tied to its author, its organization, and a curation decision.

Memories are extracted from real sessions and encrypted before submission; contribution requires explicit user actions. Approved memories are committed as encrypted chain state that validators replicate without plaintext access. At recall, candidates are decrypted locally, scanned by the local sanitization pipeline, and presented through the plugin-controlled injection gate — every recalled memory passes human eyes before it enters an agent's context.

Where work admits an executable definition of done, memories additionally carry goal-sealed verification receipts: a goal, a test predicate, and a starting-state hash sealed before the work begins, graded openly into tiers at recall, and never used to gate contribution.

A memory's continued standing in the corpus is decided by evidence, not by a formula frozen into consensus. The chain records observations — a memory was served, a memory was blocked, an episode that used a memory subsequently resolved or did not — as immutable, content-free, consumer-signed events. Standing is then computed from that log at the edge, by a published and versioned policy whose hash is anchored on-chain. Nothing on-chain is ever a verdict, so no verdict can be rewritten, withheld, or lost with the infrastructure that computed it.

Curation is accountable without a platform tribunal: each organization's leader is the sole chain publisher for that organization's commits. Consumers can file wallet-signed on-chain reports that a captured organization cannot suppress; dismissal or inaction unlocks a reporter-signed public plaintext reveal verified against a contributor-signed anchor.

Confidentiality is stated narrowly and precisely: the hub that coordinates retrieval cannot decrypt stored memory ciphertext under the Umbral proxy re-encryption relation. This is not a claim that the hub learns nothing; for retrieval the hub holds embeddings and label metadata that constitute a disclosed semantic shadow, with operational mitigations rather than a zero-knowledge index.

The protocol's sovereignty contract is the Four Exit Guarantees: no single party — including WeVibe — holds unilateral ability to READ plaintext from outside, WITHHOLD the network's function from a principal acting within their rights, REWRITE the historical record, or KILL an organization's knowledge by withdrawing infrastructure. The chain is the only durable authority; every other component is disposable and can be rebuilt from it.

> **Nothing enters an agent's context without human eyes on it first.**

This document is the architecture contract: the normative specification the network is built and audited against.

---

## 1. Overview

### 1.1 What WeVibe Is

WeVibe is a memory layer for AI coding agents. A local plugin and MCP path retrieve relevant community memory, present each candidate to the user, and inject only approved memory. A memory is one atomic insight: what to do, when it applies, and what not to do.

Memories are extracted from real sessions, encrypted before submission, reviewed by accountable humans, and committed as attributed encrypted chain state. What happens to them afterwards — whether they keep surfacing, or quietly stop — is decided by what the network observes about their use, recorded as evidence and evaluated openly.

### 1.2 How It Works at a Glance

**Figure 1 — The WeVibe loop.** A single horizontal loop, four stages left to right, with a return arrow to the start.

- **Stage 1 — Contribute.** A developer extracts a memory from a real session and encrypts it on their own machine.
- **Stage 2 — Curate.** An organization's expert leader reviews it and signs it onto the chain.
- **Stage 3 — Recall.** Another developer's agent surfaces it, they approve it, and it enters the session.
- **Stage 4 — Answer.** What happened next is recorded: the work resolved, or it did not. That observation is the memory's standing.
- Return arrow from Stage 4 back to Stage 1: "Every use is attributed back to the author; every outcome is evidence about the memory."
- Centre of the loop: "Encrypted, on-chain, owned by no one."

*Figure 1. Contribution, curation, recall and outcome form one loop; attribution closes it and evidence keeps it honest.*

### 1.3 Who It Is For

| Role | What they do | What they need |
|---|---|---|
| **Developer** | Codes with an agent, recalls community memory, and chooses what to contribute. | A passkey (Face ID / fingerprint). No wallet, no seed phrase. |
| **Organization leader** | A domain expert who curates memory quality and signs every commit. | Holds the organization's scarce slot. |
| **Validator** | Runs consensus and stores all chain state, including encrypted memories. | Runs a node. |
| **WeVibe (the protocol)** | Open-source software. No company sits in the middle; the protocol depends on no single entity. | Nothing — it is code. |

*Table 1. The four roles. The developer is the protagonist; the organization is the container.*

---

## 2. Design Philosophy

### 2.1 The Problem

Shared agent memory becomes valuable only when it crosses trust boundaries. A memory you did not personally witness is an untrusted write from a stranger. It is usable only if six questions can be answered: **authenticity, correctness, safety, confidentiality, permanence, and sovereignty.**

Developers solve hard problems with agents every day, yet sessions repeatedly reset. Version-exact fixes and negative knowledge are rediscovered, repaid for, and lost in private transcripts.

This is most visible on local or smaller models: they generate code, but they do not carry a stack's lived edge cases by default. The scarce input is verified memory with provenance, not more generic text.

Developer channels are now saturated with plausible AI-generated content that misses decisive details. Provenance — who wrote it, who curated it, and under what accountability path — becomes the scarce input.

Prior shared-knowledge systems lived in company databases with kill switches. Knowledge that should outlive its platform needs infrastructure that cannot be unilaterally withdrawn.

Cross-run agreement is expensive and repeatedly repurchased; verified shared memory amortizes that cost across the network.

And a shared corpus that only grows is not a corpus. Knowledge goes stale, and some of it was wrong when it was written. A network that admits strangers' contributions needs a principled answer to what leaves, not only to what enters.

### 2.2 The Six Questions of Shared-Memory Trust

The rest of this specification is structured as explicit answers to the six trust questions:

1. **Authenticity** — is this really from who it claims? → contributor signatures + wallet identity; the contributor-signed anchor (§7.4) and the provenance/two-key model (§8). The remaining gap — proving the *session* itself — is stated openly in §8.11.
2. **Correctness** — is it right? → accountable human curation by the organization leader (§7); goal-sealed trajectory verification where the work admits an executable check (§2.7, §8.12); and, after the fact, outcome evidence: what happened in the sessions that used it (§7.7, §7.8). Human judgment admits it; recorded outcomes decide whether it stays.
3. **Safety** — will it harm my agent? → local wevibe-guard sanitization plus a mandatory human injection gate (§4.7, §2.5 / §5.2).
4. **Confidentiality** — who can read it? → Umbral proxy re-encryption; the hub cannot decrypt stored ciphertext, with the semantic-shadow boundary stated openly (§4.5).
5. **Permanence** — will it survive? → approved memories are encrypted chain state replicated by validators (§3.2 / §9.1). Nothing is ever deleted; standing decides visibility, not existence (§7.7).
6. **Sovereignty** — can any single party read, withhold, rewrite, or kill it? → the Four Exit Guarantees; the chain is the durable authority and every other component is disposable (§2.3, §11).

**Landscape: memory vs. shared memory you can trust.** Most shipping "memory" systems are personal or single-organization stores; useful inside one trust domain, but they do not address the requirements that appear when memory is shared across strangers — especially provenance/authenticity, a human gate before injection, and any mechanism for retiring knowledge that turned out to be wrong. The scorecard below is factual, drawn from public product documentation.

| System | Auth | Correct | Safety | Confid | Perm | Sovereign | Human-gate | Retirement |
|---|---|---|---|---|---|---|---|---|
| ChatGPT Memory (OpenAI) [16] | Not addressed | Partial | Not addressed | Not addressed (host-readable) | Not addressed | Not addressed | Auto-inject (no gate) | User deletion only |
| Claude Memory + Projects (Anthropic) [17] | Not addressed | Partial | Not addressed | Not addressed (host-readable) | Not addressed | Not addressed | Auto-inject (no gate) | User deletion only |
| Gemini Personal Intelligence (Google) [18] | Not addressed | Partial | Not addressed | Not addressed (host-readable) | Not addressed | Not addressed | Auto-inject (no gate) | User deletion only |
| GitHub Copilot Memory [19] | Not addressed | Partial | Partial | Not addressed (host-readable) | Not addressed | Not addressed | Auto-inject (no gate) | User deletion only |
| Cursor / Windsurf / Cline / Roo memory-bank [20] | Not addressed | Not addressed | Not addressed | User-edits-own-only (local files) | Partial | Partial | User-edits-own-only | Manual file edit |
| Mem0 [21] | Not addressed | Partial | Not addressed | Not addressed (host-readable, self-hostable) | Partial | Not addressed | Auto-inject, developer-configurable | Developer-configured expiry |
| Letta / MemGPT [22] | Not addressed | Not addressed | Not addressed | Not addressed (host-readable, self-hostable) | Partial (self-host) | Not addressed | Auto-inject (no gate) | Context-eviction heuristics |
| Zep / Graphiti [23] | Not addressed | Partial | Not addressed | Not addressed (host-readable) | Partial | Not addressed | Auto-inject (no gate) | Temporal invalidation |
| LangMem / LangGraph memory [24] | Not addressed | Partial | Not addressed | Not addressed (host-readable, pluggable storage) | Partial | Not addressed | Auto-inject or retrieval-tool | Developer-configured |
| RAG-over-wiki (Notion/Confluence + vector DB) [25] | Not addressed | Not addressed | Not addressed | Not addressed (host-readable) | Not addressed | Not addressed | Retrieval-only | Source-document edit |
| Recall Network [26] | Solved (on-chain signed actions) | Partial (staking/reputation curation) | Unknown | Not addressed (public on-chain) | Solved (on-chain, replicated) | Partial | Auto-inject (no human gate) | Not addressed |
| Ceramic / ComposeDB [27] | Solved (DID-signed streams) | Not addressed | Not addressed | Not addressed (streams broadly readable) | Solved (decentralized) | Partial | Not addressed (data layer) | Stream update only |
| Arweave + AO [28] | Not addressed | Not addressed | Not addressed | Not addressed (public/permanent by default) | Solved (endowment model) | Partial | Not addressed (storage layer) | None (permanent by design) |
| GAIA / GaiaNet [29] | Partial (node = owner-controlled) | Not addressed | Not addressed | Partial (single-owner node) | Partial (node-dependent) | Partial | Not addressed (single-node retrieval) | Not addressed |

*Table 19. Landscape as of July 2026; scored from public product documentation (see §12). Axis values are reported facts, not editorial ranking.*

The structural result is stable across categories: established systems are single-trust-domain. Either one vendor holds readable plaintext, or "self-hosting" relocates the single trusted reader to one operator — so "sharing" is relocated centralization, not decentralization. Team/workspace sharing is not admission of a stranger's contribution with independent verification. Decentralized efforts have advanced authenticity and permanence, but not the safety problem of stranger-contributed plausible-but-wrong or malicious memory. Every system above with an injection step auto-injects: none places a human between a stranger's contributed memory and another party's agent context — the No-Blind-Injection requirement this paper treats as non-negotiable.

And on the final axis, the pattern is starker still. Where retirement exists at all it is either the owner deleting their own item, a developer-configured expiry, or a heuristic about context size. No system in this landscape retires a shared memory on the evidence of whether it worked for the people who used it — because no system in this landscape has that evidence, or a place to put it that its operator cannot edit.

### 2.3 The Inversion

Two decades of file-sharing networks proved a property that no legal or infrastructural pressure has reversed: when content is replicated across many independent machines and the index is owned by no one, distribution cannot be stopped. That is exactly the property durable knowledge sharing needs — and exactly what no knowledge platform has ever had.

But that lineage carried two fatal absences. It moved content *without the consent of its creators*, and it *stripped provenance* — the one thing that makes technical knowledge trustworthy. WeVibe is the inversion: keep the unstoppability, and restore consent and attribution.

- **Consent is structural.** Nothing leaves a contributor's machine without two explicit actions — Extract, then Submit (§6.4). Sessions stay local by default. *(Narrowed: this holds for the local/P1 path and for all plaintext; but a closed-weight P2 `PROVIDER_WITNESSED` session and remote extraction route transcript slices to the user-chosen provider, and the P2 blind witness learns encrypted-traffic metadata — size, turn timing/count, provider endpoint/routing, session shape — never plaintext. This must be disclosed plainly; see §8.11.)*
- **Provenance is the payload.** Every memory carries contributor signature, organization curation, and its own evidence history.
- **Unstoppability is enforced as four exit guarantees**, defined below.

> **The Four Exit Guarantees**
>
> No single party — including WeVibe-the-company, any hub operator, or any organization leader — may hold the unilateral ability to:
>
> 1. **READ** a member's memory plaintext from the outside;
> 2. **WITHHOLD** the network's function from a principal acting within their rights;
> 3. **REWRITE** the historical record; or
> 4. **KILL** an organization's knowledge or a contributor's standing by withdrawing infrastructure.
>
> The chain is the only durable authority; every other component must be disposable and reconstructible from chain state plus member-held keys.

### 2.4 The Organization Model

Organizations are domain-expert-run memory collections. Leaders define standards, manage membership, and publish with sole chain signature authority.

| Attribute | Description |
|---|---|
| Membership roster | Managed by the leader. |
| Role hierarchy | Leader, Reviewer, Member. |
| Commitment standards | What counts as high-quality memory in this organization. |
| Domain focus | The organization's subject and coverage map. |
| Operating policies | Contribution/review cadence and recall access. |

*Table 2. An organization is a collaboration container with its own standards.*

Public plaintext keywords are discovery labels (`redis`, `solana`, `django`) and are intentionally non-secret metadata (§4.6). They are labels and retrieval bonus terms only. They carry no weight in consensus and no role in whether a memory survives.

> **Personal memory vs shared organization memory.** Shared organization memory is the curated, encrypted, chain-anchored corpus. Personal memory is a bounded local pull layer and sits outside the chain-rebuildable contract.

### 2.5 Human-in-the-Loop: The Curator Workbench and the Plugin Gate

WeVibe is a curator workbench, not an autonomous ranking machine. Curators process retrieval frequency, denials, staleness, and drift signals; they decide what enters and stays in the corpus.

On the consumer side, one invariant governs everything: **every memory passes through human eyes before it enters an agent's context.**

During a coding session the plugin harvests local signals, auto-queries organization memory through the hub, decrypts candidates locally, scans them with wevibe-guard, and presents an approval gate:

**Figure 2 — The memory injection request.**

> **Memory Injection Request**
>
> *"Redis cluster-node-timeout must be set to 15000ms when running behind AWS NLB with cross-AZ failover…"*
>
> - Contributor : `wevibe1x7k…f3q2`
> - Wallet age : 8 months
> - Rep score : 347 (Tier 3)
> - Serves : 214 across 12 orgs
> - Domain : redis, kubernetes, aws
> - Verification : `T2 — test-backed`
>
> Detections: `[url: aws.amazon.com]`
>
> **✓ Accept** · **✗ Deny** · **⊘ Block** · **⚑ Report**

*Figure 2. The plugin approval gate. The consumer sees the memory, the security flags, and who wrote it before deciding. The Verification row renders only for memories carrying tier T1 or above (§8.12).*

| Button | Effect | Signal emitted |
|---|---|---|
| **Accept** | Memory is injected into agent context. A serve event is signed and queued to the chain. | A public, content-free serve event. No per-serve payout. A serve alone is not evidence that the memory helped (§7.8). |
| **Deny** | Memory is blocked for this session/context only. | None. A neutral "not what I need right now" signal — not a corpus down-vote; no chain event. |
| **Block** | Memory enters the consumer's permanent personal blacklist. | A signed block event: the corpus-level negative judgment, and the only one a human issues deliberately. |
| **Report** | Memory is reported on-chain against the contributor, with the reporter's wallet, and enters the organization's accountability path (§7.5–§7.7). The memory keeps serving until the report resolves. | A wallet-gated, gas-paid on-chain accountability event. |

*Table 3. Reports are the high-friction accountability primitive; Block is the low-friction negative judgment. The Deny/Block split is load-bearing: Deny says nothing about quality. Neither button carries the main correctness signal — that is the outcome of the work itself (§7.8).*

Two properties make the gate trustworthy. **No plugin installed means no memory injection path** — the MCP server has no route to force memory into an agent's context without the plugin front end. And injection is **once at acceptance, not per turn**: recall is queried on each user prompt, but an accepted memory is placed once, at a stable early position immediately after the system instructions, within a fixed token budget over a hub-ranked top-K set, and preserved verbatim across context compaction rather than summarized through it. A relevance floor and a surface budget keep the injected volume small; returning nothing is a healthy, designed outcome.

### 2.6 Protocol, Not Platform

WeVibe is a protocol with open, auditable surfaces. The chain, hub, local client, and plugin each perform one narrow role.

| # | Capability | Description |
|---|---|---|
| 1 | On-chain encrypted storage + provenance | Memories are committed as encrypted blobs with attribution metadata. |
| 2 | Human-gated delivery | The plugin is the mandatory approval path before any memory enters agent context. |
| 3 | A public evidence log | Serves, blocks, outcomes and related observations are recorded as immutable, content-free, consumer-signed events (§9.5). |
| 4 | Open standing computation | A memory's standing is a deterministic function of the public evidence log and a published, version-anchored policy — recomputable by anyone (§7.7). |
| 5 | Domain-expert governance | Leaders and reviewers curate memory quality inside each organization. |
| 6 | Suppression-proof accountability | Consumer reports and public escalation broadcast directly to the chain; no WeVibe infrastructure sits between a reporter and the record (§7.5). |
| 7 | Coordination layer | wevibe-hub runs hosted coordination, accounting, and retrieval; it is never a plaintext memory oracle and never an authority on standing. |
| 8 | Local retrieval edge + sanitization | Decryption, guardrails, and injection run close to the user. |

*Table 4. Eight protocol capabilities, split across four cooperating components.*

### 2.7 Verification by Pre-Commitment: The Verification Goal

Human review and reputation are judgment. Judgment scales poorly across trust boundaries, and machine judgment fails in a specific, measured way: in judge-gated workflow-memory systems [35], roughly half of certified inductions have been found to originate from failed trajectories [34]. In a trustless network the failure is structural, not statistical — the judge runs on the contributor's machine, so a verdict from the contributor's own model is self-attestation with extra steps. WeVibe therefore assigns the extraction LLM a narrow role: it drafts and annotates candidate memories; it never gates them.

Where the work admits an executable definition of "done," WeVibe replaces the judge with a fact, established in three moves:

1. **Seal the target before the work.** At session start the plugin seals the goal, an executable test predicate that defines completion, and a hash of the starting working tree — signed and timestamped before the outcome exists.
2. **Record the trajectory as a hash chain.** Working-tree states are hashed and chained across sessions until the sealed predicate passes.
3. **Let the predicate speak.** The sealed test's exit status — not a model's opinion, not the contributor's claim — determines whether the goal was met.

A target fixed before the answer existed cannot be retrofitted to whatever was produced. And fabrication buys nothing: to forge a passing receipt, a contributor must make the pre-sealed predicate actually pass — which is the work itself. Verification cost collapses onto honest cost.

> In a trustless setting, the only thing a stranger can verify later is something you locked in before you knew the answer.

> **The Verification Goal.** For every memory whose claim can be checked by execution, WeVibe's goal is to carry a receipt a stranger can check: sealed before the outcome existed, graded openly by tier, and used only as a label — never as a contribution gate. Facts replace judgment wherever a fact can exist; judgment — leader review, the human gate, reputation — covers everything facts cannot reach.

The mechanism, its receipts, its multi-session semantics, and its honest boundaries are specified in §8.12.

### 2.8 Evidence, Not Verdicts

The same discipline applies to what happens to a memory after it is committed.

A shared corpus needs an answer to decay: some memories are wrong, some go stale, and a network that only accumulates becomes a landfill with provenance. The obvious design is to put a trust score on the chain and a formula in consensus that moves it. WeVibe deliberately does not do this, for three reasons.

**A number in consensus is a verdict nobody can revisit.** A trust score committed on-chain is a judgment about a memory rendered by a formula that, once in consensus, can only be changed by governance. If the formula is wrong — and formulas about human behaviour usually are, at first — the error is not a bug to fix but an unamendable fact about everyone's corpus.

**Consensus should record what happened, not what it means.** A serve is an observation. A block is an observation. A test suite going from red to green after a memory was injected is an observation. Whether those observations add up to "this memory is trustworthy" is an interpretation, and interpretations improve. Separating the two lets the record be permanent and the interpretation be revisable.

**Judgment placed at the edge can be checked; judgment placed in a service cannot.** If a coordination service simply told clients what a memory's standing was, that service would be trusted. Instead standing is a deterministic pure function of a public log and a published policy, so any client, auditor or replacement service recomputes it independently and catches a lying one.

> **Evidence, not verdicts.** The chain stores observations with provenance, never conclusions. Standing — whether a memory should surface, and where — is computed from that log by an open, versioned policy whose hash is anchored on-chain. The record is immutable; the interpretation is improvable; neither is owned.

The consequences run through the rest of this document. Nothing is deleted, so KILL has nothing to grip (§7.7). No standing is written, so REWRITE has nothing to alter (§10.7). Anyone can recompute, so no operator can quietly decide what a stranger's knowledge is worth (§10.9). And because the policy is code rather than consensus, the network can get better at judging memory without ever asking anyone to trust a new judge.

---

## 3. System Architecture

### 3.1 Entities and Roles

Let **L** be leaders, **O** organizations, **D** reviewers, **C** contributing members, and **R** read-only members. Each organization *o* ∈ O has a leader *l(o)* ∈ L who controls membership and configuration.

Participants hold Ed25519 identity keys plus X25519 encryption keys. Contributor public keys are on-chain; evidence events and reputation aggregates are keyed to those identities.

| Role | Permissions | Appointed by |
|---|---|---|
| **Leader** | Full organization control: roster management, epoch rotation, key custody, reviewer appointment, label-taxonomy management, slot/rent/revenue management, standing-policy version selection (§10.9). | Self (slot acquisition). |
| **Reviewer** | Review pending memories; approve/deny; decrypt pending submissions via SK_mod(e). | Leader. |
| **Member** | Submit memories (within per-epoch bandwidth caps); retrieve approved content; view own pending submissions. | Leader (via invitation). |

*Table 5. All roles require epoch-specific encryption keys for content access; the leader distributes these through sealed envelope key exchange (Appendix A).*

### 3.2 The Three Software Pieces

**Diagram 1 — System components and trust.**

- **wevibe-chain** — *Source of truth.* Sees ciphertext, never plaintext. Holds encrypted memories, contributor anchors, and the evidence log.
- **wevibe-hub** — *Coordination + retrieval.* Disposable. Never a plaintext oracle, and never an authority on standing — it computes standing the same way any other party can.
- **MCP server + plugin** (inside the Local machine boundary) — *Safety + approval + injection.* The only place plaintext lives at recall.
- Arrows: plugin ⇄ hub, "encrypted candidates + query vectors"; hub ⇄ chain, "derived from chain state"; plugin ⇄ chain, "signed evidence events / RPC."

*Diagram 1. Three software pieces over one chain. Plaintext is confined to the local machine; authority is confined to the chain.*

### 3.3 Organization Lifecycle

**Creation.** The leader acquires a governance-capped slot (hard cap: 32 alpha / 320 testnet / 3200 mainnet), signs `MsgRegisterOrg` from their own wallet, creates `K_master`, and derives epoch-0 keys and moderation keys.

**Operation.** Members join by leader approval. The leader distributes sealed key envelopes for the current epoch.

**Rotation.** Removing a member sets `rotation_pending` and requires epoch advancement.

| Step | Action |
|---|---|
| 1 | **Removal triggers `rotation_pending`.** The chain marks the organization pending rotation; the removed member's envelope is deleted. |
| 2 | **New submissions are buffered.** Contributors can still submit, but submissions enter a hub-side rotation buffer — not admitted to the chain, not indexed for retrieval, not assigned a final epoch. |
| 3 | **Leader completes rotation.** The leader derives new epoch keys from K_master via HKDF, generates a new moderation keypair SK_mod(e+1)/PK_mod(e+1), and re-seals envelopes for all remaining members. |
| 4 | **Buffer finalizes.** After rotation completes, buffered submissions are released to the chain under the new epoch. |
| 5 | **Grace-period escalation.** If rotation is not completed within 72 hours, the organization's submission bandwidth is suspended until it is. |

*Table 6. Removing a member advances the epoch. Rotation provides forward secrecy only — a removed member retains previously distributed epoch keys and can still decrypt content from their membership period.*

> **Wallet-free contributor onboarding.** A contributor can create an account with a passkey (Face ID / fingerprint), install the plugin, and join an organization without a wallet. Contribution remains explicit and dashboard-driven with two consent gates: **Extract** then **Submit**. Evidence events are signed with the member's per-organization key and carried under the organization feegrant, so a consumer never needs a wallet or gas to be counted. A wallet is optional later for rewards and mainnet fees; the leader still requires a wallet to register and hold the organization slot.

Chain state is authoritative for membership transitions; the hub mirrors chain events.

### 3.4 Tool Surface

The plugin registers operational tools in the coding agent. Recall is automatic in the plugin path (no agent recall tool call); contribution submission stays in dashboard flows.

| Tool | Purpose |
|---|---|
| `setup_org` | First-run organization bootstrap when the local stack has no membership. Guides the create/join flow and local setup handoff. |
| `wevibe_status` | Show organization membership and runtime status so the user can verify local setup and connectivity. |
| Consumer-settings tools | Configure the content filter: `[Implementations + DNDs]` or `[DNDs only]`. Default: `[Implementations + DNDs]`. |

*Table 7. Agent-visible tools. The MCP server backend handles recall mechanics — query construction, local embedding/decryption, guard scanning, and packaging candidates for the plugin gate — and is not directly callable by the agent. It never auto-submits contributions.*

---

## 4. Cryptographic Architecture

### 4.1 Threat Model

| Adversary | What they can see | What they must not be able to do |
|---|---|---|
| **Chain validators** | Ciphertext, organization IDs, contributor public keys, evidence events. | Read memory content. |
| **Unauthorized external observers** | Public chain state (no epoch keys). | Decrypt content. |
| **Removed members** | Content from their membership period (they held epoch keys). | Decrypt content created after removal (forward secrecy). |
| **Memory poisoning via recalled content** | — (a malicious member submits an indirect prompt injection that passes review). | Reach an agent's context unseen. The human gate is the final defense: the reviewer and consumer see every memory, plus the contributor's reputation and wallet age. |
| **Receipt forger / predicate gamer** | Their own seals, chains, and receipts (produced on hardware they control). | Pass a fabricated or hollow receipt off as strong verification. Tier semantics cap the claim (§8.12); a forged receipt against a sealed predicate becomes a provable, reportable lie on reveal-and-replay. |
| **Evidence forger** | Their own machine's signing keys and local session state. | Manufacture standing for their own memories. Events are signed per-consumer and deduplicated on-chain; self-authored uses are discounted (§8.7); outcome events are welded to session identity and evidence references (§7.8). |

*Table 8. The six adversary classes the system defends against.*

> **Out of scope (explicitly not defended).** The system does **not** protect against: a compromised active member who leaks epoch keys or decrypted content; metadata inference from on-chain patterns (organization sizes, submission frequency, evidence patterns); compromised reviewer endpoints; or semantic payloads encoded in natural-language prose. It also does not prevent fabrication of T1–T2 verification receipts on contributor hardware, nor trivially weak sealed predicates; both are bounded by label semantics, curation, and dispute-path replay rather than cryptography (§8.12). Evidence events are content-free but not pattern-free: an observer can see that some pseudonymous member used and did or did not succeed with a given memory, and dense evidence streams are a linkage surface (§9.5).

### 4.2 Epoch-Based Key Hierarchy

Per-epoch keys derive from organization `K_master` via HKDF. The leader derives keys locally and never transmits epoch secret keys to the hub. Full derivation formulas and invariants are in Appendix A (CODE 1).

### 4.3 Memory Encryption and Moderation Keys

Each memory uses a random per-memory DEK. The DEK is initially sealed to the moderation public key for review, then re-wrapped to the epoch encryption key on approval. Full procedures are in Appendix A (CODE 2 and CODE 3).

### 4.4 Key Distribution: Sealed Envelopes

Epoch keys are distributed to members via sealed envelopes (`seal_to_pubkey`), with custody and recovery constraints anchored at the organization level. Full envelope procedure and recovery model are in Appendix A (CODE 4).

### 4.5 Hub Confidentiality: Why the Hub Cannot Decrypt

A natural objection: if an authorized consumer can retrieve plaintext, surely a captured hub could too. This section states the guarantee narrowly and then states the boundary.

**The claim, stated narrowly.** The hub **cannot decrypt** memory content, including under a fully malicious-hub model. This is not a claim that the hub learns nothing.

> The hub is a postal sorting office that re-addresses a sealed envelope so a different recipient's key opens it — without ever being able to open it itself.

**The confidentiality core (Umbral proxy re-encryption over secp256k1).** Umbral layers verifiability and threshold-splitting on top; the core is the ElGamal-style re-encryption relation below. Let `G` be the curve generator and `n` the curve order.

```
Keys
  org (delegator)     secret a = epoch_sk       public  A = a·G = umbral_pk
  member (recipient)  secret b = receiving_sk   public  B = b·G = pre_pubkey

Encrypt   (client-side, under the org public key A)
  the per-memory DEK is sealed in a capsule; the capsule public part is a point (E+V)
  sealed key  K = KDF( a·(E+V) )              ← recovering K the direct way needs a

Re-encryption key   (kfrag — minted ONLY by the leader, REQUIRES the secret a)
  rk ≈ a·b⁻¹  (mod n)                          ← reveals neither a nor b (discrete log);
                                                 cannot decrypt by itself

Re-encrypt   (the hub's ONLY crypto op: capsule + kfrag, NO secret key)
  cfrag = rk·(E+V) = (a·b⁻¹)·(E+V)

Decrypt   (client-side, on the member's device, REQUIRES the secret b)
  b·cfrag = b·(a·b⁻¹)·(E+V) = a·(E+V) → K = KDF( a·(E+V) )
  then AES-decrypt the memory body with the recovered DEK
```

- **Encryption** seals the per-memory DEK under the organization public key `A`; recovering the sealed key `K = KDF(a·(E+V))` the direct way requires the organization secret `a = epoch_sk`, which the hub never holds.
- **The kfrag** encodes `rk ≈ a·b⁻¹` and can be minted only by a party holding the delegating secret `a` — the leader. Recovering `a` or `b` from `rk` is the discrete-logarithm problem, and `rk` decrypts nothing on its own.
- **Re-encryption** is the hub's *only* cryptographic operation: given the capsule and a leader-minted kfrag, and no secret key, it computes `cfrag = rk·(E+V)`.
- **Decryption** happens only on the member's device, which multiplies the cfrag by its own secret scalar `b` to recover `K`, hence the DEK.

> **The punchline.** The hub can compute `a·b⁻¹·(E+V)` but needs `a·(E+V)`. The one missing operation is a single multiplication by `b`, the member's secret scalar, which the hub is *structurally* never given. This is geometry, not policy: no configuration, key rotation, or privileged mode grants the hub `b`. Everything the hub holds — `{capsule, cfrag, ciphertext, kfrag, A, B}` — is computationally independent of the plaintext without `b`. This is exactly Umbral's IND-PRE-CCA security guarantee [15].

**Attacks a malicious hub might attempt.**

- *"The hub forges its own kfrag toward a key it controls."* Fails. Minting a kfrag requires the delegating secret `a = epoch_sk`, which the hub never holds. The hub can only *apply* leader-minted kfrags, and those are minted toward the registered public keys of authorized members — never toward a hub-controlled key.
- *"The hub colludes with an authorized member."* Conceded openly. An authorized member decrypts with their own secret `b` and could leak the plaintext. That is authorized-insider abuse, not a cryptographic break, and it is already out of scope in the threat model (§4.1). The hub as a standalone party still cannot read.
- *"The hub lies about standing to bury a memory."* Fails as a durable attack. Standing is a deterministic function of a public log and a published policy version, so any client or auditor recomputes it and detects the divergence (§7.7). The hub can degrade its own answers; it cannot make its answers authoritative.

> **The honest boundary (what this does NOT cover).**
> Two statements must not be conflated:
> - **TRUE (proven above):** the hub cannot **decrypt** your memory content.
> - **FALSE (never claimed):** the hub learns **nothing** about your content.
>
> For search, the hub holds — in Qdrant — clean float32 embeddings plus plaintext label metadata: a disclosed, lossy, realistically-invertible **semantic shadow** of each memory (§4.6, §9.3). Published embedding-inversion research [13][14] shows approximate content recovery from clean embeddings is realistic. The mitigations here are **operational, not cryptographic**: Qdrant API auth, internal-network deployment, per-organization collection isolation, and signed responses. Encrypted vector search is the documented evaluation trigger — formally evaluated when an organization requests confidentiality-sensitive hosting, or when public-testnet launch planning begins, whichever comes first. WeVibe makes **no** claim of a zero-knowledge index or a content-confidential hub. "Cannot decrypt" and "learns nothing" are different guarantees, and only the first is made.

Two further notes. **The consumer-side injection gate is live and enforced today** — a blocking, fail-closed human-approval step (Accept/Deny/Block/Report, §2.5) that injects only human-approved memories; benchmark/test mode outside the shared-memory safety contract may auto-approve for evaluation only. **Key locality:** the hub never receives `epoch_sk`; only the epoch public key and finished kfrags cross the wire, and the confidentiality proof rests on this locality holding cleanly.

### 4.6 Metadata Visibility Model

WeVibe organizations are public developer communities. On-chain metadata is intentionally public for discovery and reputation.

| On-chain (public by design) | Local to the MCP/plugin (the hub never sees these) |
|---|---|
| Organization IDs and topic tags; contributor public keys; encrypted memory blobs; plaintext discovery labels; submission timestamps; memory sizes; epoch boundaries; the evidence log (serve, block, outcome and related events — §9.5); reputation aggregates; bandwidth consumption; quarantine state; report flags; the anchored standing-policy version hash. | Decrypted memory plaintext; local wevibe-guard and blacklist state; session context profiles; the content of a session beyond the observations the consumer chooses to sign. |

*Table 9. Embedding vectors and label metadata live in the hub's Qdrant (§9.3); the hub stores ciphertext and vectors but never decrypts. Standing appears nowhere in this table because standing is never stored — it is computed (§7.7).*

### 4.7 Defense-in-Depth: The Memory Sanitization Pipeline

Sanitization is canonical here and referenced elsewhere. Decryption, scanning, approval, and injection all run locally.

**Diagram 2 — The sanitization pipeline.** Two columns: submission time (steps 1–5) and recall time (steps 6–15).

*Submission time (before on-chain storage):*
1. **wevibe-guard scan.** Rule-based detection (YARA-X + regex) for injection patterns, credential detection, and exfiltration matching. Advisory: it warns but does not block. The human reviewer is the security boundary.
2. **Text sanitization suite.** WeVibe memories are text, so the sanitizer works on the text directly rather than rendering images. It normalizes and flags the injection vectors that are invisible to the human eye: zero-width and unusual-whitespace characters, emoji-pattern "sleeper" payloads (variation-selector and emoji-tag encodings), homoglyph substitutions, bidirectional and directional-override sequences, invisible Unicode tag characters (U+E0000 block), and mathematical-alphanumeric injection (U+1D400–U+1D7FF).
3. **Encryption.** Memory encrypted with a per-memory DEK; the DEK sealed to the moderation public key.
4. **Human review.** The reviewer decrypts locally, reads plaintext, runs a steganography scan, and approves or denies.
5. **On-chain submission.** The approved memory (ciphertext + wrapped DEK + metadata) goes on-chain and is mirrored to hub storage for retrieval. The organization pays the submission cost.

*Recall time (before and after delivery):*
6. **Hub candidate query.** The MCP/plugin posts the local query vector to hub retrieval and receives memory IDs, metadata, and ranking detail.
7. **Ciphertext fetch + local decryption.** The MCP/plugin fetches each candidate's ciphertext and decrypts it locally through the Umbral sidecar.
8. **Blacklist filter.** The MCP/plugin checks the local blacklist and chain quarantine flags.
9. **wevibe-guard scan.** The same scan on the decrypted memory, catching payloads not detectable at approval time (new rules since approval).
10. **Text sanitization suite.** The same text normalization and invisible-character detection, applied to the decrypted memory.
11. **Artifact extraction and egress flagging.** Typed extraction of URLs, bare domains, IPv4 addresses, shell commands, package-install commands, and config directives. Every network-resolvable token is flagged for the reviewer (advisory — the human gate decides).
12. **Plugin approval gate.** The plugin renders the approval UI with guard detections and contributor trust signals (public key, wallet age, reputation score, serve count, domain expertise). The user sees the memory, the flags, and who wrote it, and decides.
13. **Serve event.** On Accept, the consumer's per-organization key signs a content-free serve event (Ed25519, offline — no wallet, no gas); the batch is carried by the organization serving key under feegrant. The chain verifies each entry's signature, recomputes the dedup fingerprint itself, and counts only verified entries (§8.3).
14. **Context injection.** Approved memories are formatted as `context:\n{memory content}` and injected once, at a stable early position, within the token budget (§2.5).
15. **Outcome observation.** The plugin watches the work that follows and, if the episode reaches an observable conclusion, signs an outcome event binding that memory's use to what happened (§7.8). If no conclusion is observed within the window, nothing is claimed.

*Diagram 2. Sanitization runs twice — once before storage, once before delivery — because rules improve over time. Observation runs once more, after delivery, because whether a memory helped is only knowable afterwards.*

| Catches | Does NOT catch |
|---|---|
| Rule-based prompt injections (YARA-X + regex); credential leakage (AWS keys, API tokens, passwords, connection strings); invisible-to-the-eye Unicode steganography — zero-width and unusual-whitespace characters, homoglyph substitutions, bidirectional and directional-override sequences, invisible Unicode tag characters (U+E0000 block), and mathematical-alphanumeric injection (U+1D400–U+1D7FF); emoji-pattern sleeper-prompt payloads; Base64-encoded injections; external URL injection (scheme-ful and scheme-less); bare hostname references (any TLD); IPv4 literal references; malicious dependency injection; config-directive injection; shell pipe-to-execution attacks; previously-rejected memories (local blacklist + chain quarantine). | Semantic payloads encoded in natural-language prose (mitigated by human review and contributor reputation). Technically-plausible but subtly wrong recommendations (mitigated by reviewer domain expertise, contributor reputation visibility, and — over time — accumulated outcome evidence, §7.8). |

*Table 10. The pipeline hardens against mechanical attacks; human judgment covers the semantic residue in the moment, and recorded outcomes cover it over time.*

---

## 5. Retrieval and Recall

Retrieval is hub-served; plaintext handling remains local. The hub retrieval plane (Qdrant index + ranking) is derived and rebuildable from chain state plus organization keys.

### 5.1 The Retrieval Pipeline

**Context profiling.** Session context (intent, stack, dependencies, errors, files) is harvested locally and used for pre-filtering.

**Label extraction.** Contributor extraction proposes discovery labels; leaders curate taxonomy at batch stage. Labels are retrieval bonus terms and discovery aids, never hard gates, and never inputs to standing.

**Retrieval representation (situation-centric card, symmetric embedding).** Memory and query are embedded from deterministic cards with matched `nomic-embed-text` prefixes (`search_document:` and `search_query:`). Memory vectors are produced at approval on a plaintext-capable client path, not by the hub.

**Semantic embedding.** Query vectors are computed locally via Ollama and posted to hub retrieval.

> **Atomic memory format.** Each memory is a single, self-contained technical insight with four fields:
> - **implement** — the fix itself: specific technical knowledge in 1–2 sentences, with exact values.
> - **context** — the environment, versions, and conditions where it applies.
> - **dnd** — negative knowledge: what *not* to do, and why.
> - **stack** — the specific technologies involved.
>
> This atomic form is preserved end-to-end: every extraction consumer, benchmark harnesses included, operates on individual memory objects. Collapsing multiple atomic memories into a single text blob is a conformance violation.

**Retrieval scoring.**

```
label_boost   = Σ(query_weight_i × memory_label_weight_i)
relevance     = vector_score + min(γ × label_boost, δ × vector_score)
final_score   = f(relevance, standing)

Defaults: γ = 0.1, δ = 0.15
```

Vector similarity is primary; label overlap is additive and capped. Standing enters as a separate term computed from the evidence log (§7.7), never as a stored score read from chain. Parameters (`γ`, `δ`, depth, and the standing term's shape) are tuning defaults in the published policy, not protocol constants.

**Blacklist filtering + fetch/decrypt.** Local blacklist and quarantine flags apply before approval. Candidates are fetched as ciphertext and decrypted locally through Umbral.

**Contested-result handling.** When the top two results are near-tied (score gap < 0.20), the shipped default is **deterministic twin-suppression** — surface the clear winner and suppress the near-tied twin (no model call). A non-blocking, model-based rerank via a separately configured endpoint is an optional, deferred enhancement, not part of the shipped path.

### 5.2 Recall Flow (Auto-Query, Plugin-Gated Injection, Observed Outcome)

**Diagram 3 — Recall flow.** Top-to-bottom, three swimlanes: local plugin, hub, local plugin.

1. "Developer works in their coding session."
2. "Plugin harvests local session signals (intent, task, stack, dependencies, errors, files, and live build/test failure signals: failing checks + error strings), feeds fix-iteration need-cards from those live failure signals, and attaches the session to any open goal seal on this repo (§8.12)."
3. "Plugin auto-submits the recall query via local MCP (no agent recall tool call)."
4. Local MCP/plugin: "Build the deterministic need-card → compute query embedding locally via Ollama (`nomic-embed-text`) → POST the query vector to the hub retrieval endpoint."
5. Hub: "Qdrant vector + label-boosted search over hub-hosted embeddings, ranked with standing computed from the evidence log under the anchored policy version → return encrypted candidates + ranking/trust metadata."
6. Local MCP/plugin: "Decrypt candidate ciphertext via the Umbral sidecar → run wevibe-guard + policy checks → apply content filter → render candidate details."
   - Rendered details: memory text, memory_type, relevance and standing, matched labels · wevibe-guard result · contributor stats (account age, contributions, serve count, reports upheld, false reports) · memory evidence summary (serves, blocks, observed outcomes) · report indicators · verification tier + receipts (§8.12) · trust panel.

Mandatory branch (shared/org production memory):
- **[Gated approval]:** explicit user action on every candidate — ACCEPT → inject context + sign a serve event · DENY → block for this session/context (no corpus signal) · BLOCK → personal blacklist + signed block event · REPORT → file on-chain report (`is_reported`); the memory keeps serving until resolved.

Post-injection (no user action required):
- **[Observation]:** the plugin watches the episode the memory was injected into. If it reaches an observable conclusion, an outcome event is signed and broadcast. If it does not, nothing is signed and nothing is claimed (§7.8).

Benchmark/test-only note (outside shared-memory safety contract):
- **Benchmark/test mode:** evaluation mode may auto-approve candidates for throughput testing; never enabled for production shared/org recall.

Terminal node: "Agent continues, with or without memory."

*Diagram 3. Recall is queried on every prompt; the shared-memory production flow uses a single mandatory human gate, with benchmark/test mode explicitly out of contract.*

---

## 6. Local Architecture (MCP Plugin + Sidecars)

### 6.1 Local Footprint and Responsibilities

Local components are the MCP server + plugin, Umbral sidecar, and wevibe-guard. The local path owns decryption, sanitization, approval, injection, and observation.

| Recall-side (local) | Contribution-side (local) |
|---|---|
| 1. Harvest session signals and build the deterministic need-card. | 1. Extract atomic memory candidates from the captured session substrate only when the contributor clicks `/sessions` → **Extract Memories** (client-side candidate generation, no submission yet). |
| 2. Compute the query embedding locally via Ollama (`nomic-embed-text`). | 2. Sanitize, encrypt, and sign submission material. |
| 3. Send the query vector (plus context filters) to the hub retrieval endpoint. | 3. Submit commitment data to the chain (organization pays submission bandwidth), then follow the moderation/finalization flow. |
| 4. Fetch candidate ciphertext (chain is source of truth; hub serves it cached with Umbral PRE materials). | |
| 5. Decrypt locally through the Umbral sidecar. | |
| 6. Run wevibe-guard sanitization/policy checks. | |
| 7. Present the human approval gate and inject only approved context. | |
| 8. Observe the episode that follows and sign an outcome event if it concludes observably (§7.8). | |

*Table 11. Vector retrieval itself is hub-side; the hub stores and serves ciphertext, vectors, and label metadata, and never decrypts plaintext. Observation is local because only the local machine can see what happened.*

**The extraction substrate.** A captured local session is the full plugin event stream: user messages (including repair/feedback turns), assistant text, tool calls with outputs (test runs, builds, and error output), and file-edit events. One deterministic substrate builder assembles that stream and is shared byte-identically by the dashboard Extract path and benchmark harnesses, so benchmark extraction and production extraction run over the same session bytes.

**The observation substrate.** The same event stream is what makes outcome observation possible: deterministic, code-only segmentation partitions it into failure episodes — a failing signal, the edits that followed, and whether that signal later disappeared or persisted. Observation reads those episodes; it does not ask a model for an opinion (§7.8).

### 6.2 Dependencies

The MCP/plugin stack requires Ollama, a local Umbral sidecar, wevibe-guard, and connectivity to hub APIs and chain RPC.

### 6.3 Sync and Bootstrapping

Member setup is key/bootstrap first: join organization, receive sealed envelopes, initialize local services. Recall then runs request/response against hub retrieval; clients do not download full corpora.

### 6.4 Contribution Flow

**Diagram 4 — Contribution flow.** Two explicit consent gates.

1. "Developer opens the dashboard `/sessions` and selects a captured local session (the full event stream, §6.1)."
2. **Gate — click "Extract Memories":** "client-side extraction over that substrate with the contributor's selected model — local or session-matched cloud; returns review candidates; does NOT submit. Candidates from goal-sealed trajectories carry their receipts and provisional tier (§8.12)."
3. "Review/edit candidates and choose a per-memory organization destination."
4. **Gate — click "Submit N":** "explicit off-machine send."
5. Local contribution pipeline:
   1. "Run wevibe-guard scan (advisory) and sanitization steps."
   2. "Generate a fresh DEK, encrypt plaintext, seal to PK_mod(e)."
   3. "Contributor signs the submission hash / canonical body."
   4. "Submit COMMITMENT to the chain (hash, org_id, contributor_pubkey, expiry, size). The organization pays bandwidth for the commitment transaction."
   5. "Deliver the encrypted blob to the reviewer via a temporary off-chain path (local transfer, P2P, or org-hosted mailbox)."
   6. "Return: 'WeVibe: submitted N learning(s). Pending review.'"

*Diagram 4. Two explicit consent points; nothing leaves the machine before Submit.*

### 6.5 Reviewer Flow

Moderation and review run in the hosted dashboard (`wevibe-dashboard`). Reviewers and leaders process pending submissions, then finalize through the hub-to-chain path.

### 6.6 Plugin Architecture

| Agent | Plugin type | Status |
|---|---|---|
| **OpenCode** | JS/TS editor plugin | Reference implementation |

*Table 12. All plugins call the same local MCP server and the same wevibe-guard binary; decryption runs through the local Umbral sidecar; retrieval/search stays in hub APIs.*

### 6.7 Chain-Resolved Hub Endpoints (Zero-Config Transport)

The canonical chain is the organization directory. Each organization publishes ordered `hub_endpoints` (transport URLs) on-chain; `hub_serving_address` is a distinct authorization key.

Clients resolve endpoints from chain RPC at session start, verify hub response signatures against on-chain serving keys, and fail over by priority if a transport fails.

> **Hub transport is untrusted.** Transport is untrusted; response authority is chain-resolved and signature-verified. Ranking is not authority either: a client that wishes to can recompute standing itself from the same public log (§7.7).

---

## 7. Content Review and Accountability

### 7.1 Review Flow

Contributions enter encrypted pre-commit review visible to reviewers and leaders. Pre-commit memories remain off-chain until leader commit; this is both quality control and provenance protection.

> **Moderation is advisory; the leader is the backstop.** Reviewer recommendations are advisory metadata. The leader remains sole publisher: label curation, commit, and terminal deny are leader decisions and leader signatures.

**Diagram 5 — Pre-commit lifecycle.** States left-to-right: `pending_keyword` → `pending_chain` → `committed`.

- `pending_keyword`: "Contributor submits; the memory enters the leader's curation queue with labels already attached."
- `pending_chain`: "The leader curates labels and verifies."
- `committed`: "The leader signs the multi-message batch commit."
- Downward branch from any pre-commit state to `denied`: "A leader denial is the only terminal removal before commit."
- Side annotation: "Advisory votes ride alongside as metadata at any point before commit."

*Diagram 5. The pre-commit lifecycle. Only the leader's signature publishes; only a leader denial removes. After commit, nothing removes — standing decides visibility (§7.7).*

### 7.2 What Review Can and Cannot Catch

| Can catch | Cannot catch |
|---|---|
| Prompt-injection patterns, malicious URLs, credential exfiltration, spam, duplicates, off-topic content, obvious technical errors, stale references, Unicode steganography, memories too generic for the organization's target model/stack. | Subtly incorrect technical guidance that looks plausible; semantically-encoded malicious instructions in natural-language prose; context-dependent correctness. |

*Table 13. What human review catches, and the semantic residue it cannot. The residue is precisely what the evidence log is for: review decides admission at one point in time, and use decides standing continuously afterwards.*

### 7.3 Reviewer Trust Boundary

Reviewers process plaintext on local endpoints; endpoint security and judgment remain part of the boundary. The leader is sole chain signer for commits, report resolutions, and dispute publications.

### 7.4 The Contributor-Signed Canonical Body (Verification Anchor)

Every memory's submit-time canonical body includes three fields that, together with the contributor's signature over the body, form the public-escalation verification anchor:

> **The anchor fields**
> - **plaintext_hash** — `sha256(salt || plaintext)`, computed by the contributor before encryption.
> - **salt** — a fresh 32-byte random value generated per submission.
> - **ciphertext_hash** — `sha256(ciphertext)`, where ciphertext is the AEAD output.

The contributor signs the canonical body with their own key. The canonical body, the signature, and the ciphertext all travel through moderation and land on the chain together. The leader's batch commit includes the contributor's signature; the leader cannot modify the signed fields without invalidating that signature.

This is what makes the public report tier (§7.5) trustworthy without trusting the leader. Any future reveal of plaintext + salt can be verified against the on-chain `plaintext_hash` by a direct SHA-256 check, and the on-chain ciphertext can be verified against `ciphertext_hash`. The leader is removed from the verification chain entirely; a captured leader cannot poison the anchor because they do not sign it. The contributor cannot substitute ciphertext between submit and commit (the signature binds the specific `ciphertext_hash`), and cannot later claim different plaintext at escalation (the `plaintext_hash` binds them, and SHA-256 collision resistance prevents a second matching pair).

> **Why the signature covers all three fields jointly.** A signature over `plaintext_hash` alone — without salt and without ciphertext binding — is vulnerable to contributor-plus-leader collusion: the contributor signs one hash while the leader commits ciphertext encrypting different content, and the asymmetry is undetectable. Binding all three fields in one signature closes the gap. The leader has no signing role in the anchor; the contributor has no opportunity to substitute ciphertext after signing.

### 7.5 The Report Model

Reports are the high-friction accountability primitive: filing with a fixed window, then public reveal only if dismissed or unaddressed.

**Diagram 6 — Report lifecycle and the expose gate.**

*Stage 1 — File (no reveal):*
- "Consumer files from the plugin. Wallet-gated: the reporter signs the report tx from their own wallet and pays gas; their public wallet is recorded on-chain."
- Effect: "Sets `is_reported` (open report stands) and `was_reported` (permanent, never unset)."
- Constraints: "Rate-limited to one report per reporter per 24 hours (block-height scoped)." · "Broadcasts directly to chain RPC; a WeVibe relay is retry-fallback only." · "The memory keeps serving during the window."
- Then: "One-week resolution window."

*Stage 2 — Resolution (org acts), two outcomes:*
- "**Valid** → the leader deletes the ciphertext blob. A minimal on-chain transparency record is kept (reported and upheld, attributed to the contributor). Plaintext is NOT revealed."
- "**Dismissed** → the leader clears the open report with a `clear_report` tx (`report_cleared = true`, `is_reported = false`; `was_reported` stays true)."

*Stage 3 — Expose (gated, final), triggered by window elapse or dismissal:*
- "The reporter's public report unlocks and reveals the memory plaintext, anchored to the contributor-signed hash (§7.4). The block explorer renders it with full provenance — contributor pubkey, leader pubkey, org ID, original commit height, and the reporter's signed reason. One escalation per report; no re-publishing."

*Diagram 6. Revealing is last because it is irreversible. A captured organization cannot both refuse to act and prevent the public record.*

### 7.6 The Expose Gate, Dispute, and Silent Acquiescence

Ordering is deliberate: file without reveal, allow one-week resolution, unlock reveal on dismissal/inaction, then allow one immutable leader counter-statement.

> **Silent acquiescence.** A leader who neither upholds nor dismisses during the window implicitly accepts the claim: reveal unlocks, and the on-chain report remains visible.

### 7.7 Standing: Computed, Not Stored

**WeVibe's dashboard is not the courtroom. The chain is.** WeVibe surfaces do not aggregate or rank report statistics. The chain is publication; public judgment happens externally via explorer URLs.

> There is no tribunal. There is only publication of unforgeable evidence on a chain no one controls.

The same principle governs the quieter, continuous judgment that decides whether a memory keeps surfacing at all.

**What standing is.** Standing is a memory's current claim on a consumer's attention: whether it is eligible to surface, and how strongly it competes when it does. It is not a field. It is the output of a function:

```
standing(memory) = policy_v( events(memory) )
```

where `events(memory)` is that memory's slice of the public, append-only evidence log (§9.5), and `policy_v` is a published, versioned, deterministic function whose hash is anchored on-chain.

**What that buys.**

- **Anyone can check.** Given the public log and the anchored policy version, any client, auditor, or replacement hub computes the same standing, byte for byte. A hub that reports something else is caught by arithmetic, not by trust.
- **The record cannot rot.** Because standing is never written to the chain, there is no stored judgment to become wrong, and no migration that would have to rewrite history to correct it.
- **Policy can improve.** A better interpretation of the same evidence ships as a new anchored version. Old standing remains exactly reproducible under its own version; no history is edited to accommodate the change.
- **Nothing is deleted.** A memory whose standing falls below the surfacing threshold stops competing for attention. It is not erased, not archived out of the chain, and not unrecoverable: if evidence changes, or policy changes, or an organization pins a different version, it returns. Withdrawal of infrastructure cannot destroy it, because the infrastructure never held it.

**What the evidence is.** Three kinds of observation carry most of the weight, and they are deliberately different from one another:

- **A serve** records that a memory was retrieved, cleared the relevance floor, was shown to a human, and was accepted into a context. It is evidence of *relevance*, and by itself that is all it is.
- **A block** records that a human deliberately refused a memory permanently. It is rare, high-intent, and unambiguous — the only negative judgment a person issues on purpose.
- **An outcome** records what happened to the work that used the memory. It is evidence of *efficacy*, and it is the load-bearing signal (§7.8).

A serve alone does not credit a memory. Attention is not endorsement, and a corpus that rewards being looked at will fill with memories that are good at being looked at. Credit for a use is held until an outcome for that use is observed; if none is observed within the policy's window, the use contributes nothing in either direction. This is deliberate: the honest treatment of an unobserved use is silence, not a vote.

> **The honest boundary of standing.** Standing is a claim about evidence, not a claim about truth. A memory with strong standing has been used by people whose work then succeeded; it has not been proven correct, and no amount of accumulated evidence makes it so. Correctness in the strong sense belongs to the goal-sealed receipts of §8.12, to leader curation, and to the consumer reading the memory at the gate. Standing decides what competes for attention; it never decides what is true, and it never overrides the human gate.

**Contextual suppression (designed, disclosure-gated).** A memory is rarely wrong everywhere — it is wrong in a context. The evidence schema therefore reserves the ability for a block event to carry the *situation* in which the memory was refused, so that suppression can be scoped to similar situations rather than applied globally. This capability is specified but not activated: publishing a situation embedding on-chain would widen the disclosed semantic shadow of §4.5 from the memory side to the query side, and that is a confidentiality decision that must be made explicitly and disclosed plainly before the field carries data, not after. Until then, block evidence is scope-free.

### 7.8 The Use Leg: Outcome as Evidence

The signal that decides whether a wrong memory leaves the corpus is not a button. It is what happened next.

**What is observed.** The plugin already segments a session deterministically into failure episodes: a failing signal appears, edits follow, and the signal either disappears or persists. When a memory was injected into an episode, the episode's conclusion is the observation — the work resolved, or it did not. That observation is signed as an outcome event bound to the memory, the episode, and an evidence reference.

**Why it is harvested, not reported.** Asking a developer "did that memory help?" produces a survey, not a measurement: it is answered rarely, by the wrong people, and with the biases of anyone rating a stranger's contribution. The build system already knows. Observation reads the machine's own record of what happened rather than a human's recollection of it, and the human's own report is retained only as a dispute path, not as the data source.

**Why no model judges it.** The observation is produced by deterministic code over the session event stream. A model asked to grade an episode is a judge on the contributor's hardware — the exact failure §2.7 exists to remove. The extraction model drafts; it does not grade. The observer counts; it does not opine.

**Three honest limits, stated plainly.**

- **Attribution.** An episode that resolved with a memory in context did not necessarily resolve *because* of it, and one that failed did not necessarily fail because of it. Episode-level scoping narrows the confounding considerably relative to session-level accounting, but it does not eliminate it. Outcome evidence is a correlation with provenance, not a causal proof; the causal instrument is the ablation receipt of §8.12, which is expensive and available only sometimes.
- **Coverage.** Not every episode reaches an observable conclusion within the window. Uses that do not are recorded as nothing at all — neither positive nor negative. Coverage is therefore a property the network must measure and report, because a corpus whose evidence is thin is a corpus whose standing is weakly grounded, and pretending otherwise would be the same error as crediting unobserved uses.
- **Label accuracy.** The observer can be wrong in both directions: a suite that goes green for unrelated reasons, an episode that resolves outside the observer's view. The accuracy of the observation is an empirical property of the harvester in a given environment, not a constant, and it varies with the task and the model doing the work. It is measured and published per environment rather than assumed.

**Anti-gaming.** Outcome evidence would be worthless if a contributor could manufacture it. Three controls apply, in addition to the general integrity controls of §8.7: outcomes on a contributor's own memories are discounted, as self-serve attribution already is; events are deduplicated on-chain by fingerprint so retries and replays cannot inflate a count; and outcome events reference the evidence that produced them, so a claim of success is checkable against a trajectory rather than asserted.

---

## 8. Provenance, Reputation, and the Social Graph

### 8.1 Reputation Is Provenance Made Visible

Reputation is a presentation layer over signed chain events: what was contributed, where it served, under whose curation, and what happened afterwards.

### 8.2 Serve Attribution Is a Social Signal, Not an Economic One

Serve attribution is public and on-chain, but decoupled from payout.

### 8.3 Identity and Attribution Model

| Key | Role |
|---|---|
| `global_contributor_key` | Public identity for authorship and contributor-profile reputation. |
| `org_serve_key` | Per-organization pseudonymous key used for retrieval, serve, and outcome events. |

*Table 14. The `org_serve_key` proves organization activity and supports deduplication without auto-linking retrieval behavior across organizations. Users can opt in to publicly link selected organization activity to a profile.*

### 8.4 Open-Source Social Graph Client

The social graph is an open-source, forkable client over chain RPC; chain state remains source of truth.

### 8.5 Badge Taxonomy and Display Scope

Badge taxonomy is normative but non-economic. Canonical badge families and display semantics are specified in Appendix B.

### 8.6 Badge Scoping and Canonical Criteria

Badges are earned per organization and optionally summarized on contributor profiles. No global cross-organization leaderboard is published.

### 8.7 Signal Integrity and Anti-Gaming

Integrity controls include on-chain deduplication by fingerprint, per-epoch event caps, self-serve and self-outcome discounting, diminishing returns, and per-entry signature verification. Human review, block events, and the report rail provide additional negative feedback. Because standing is computed rather than stored, an anti-gaming improvement ships as a policy version rather than a chain migration.

### 8.8 Public Discovery Interface (Opt-In)

| Visible to non-members (if public) | Not visible to non-members |
|---|---|
| Organization name, specialization, description, memory count, member count, age, leader identity, total serves, social-badge summary, standing-policy version in force, and the two unfakeable org-health signals below. | Memory content (encrypted on-chain), member identities (privacy-preserving), review history, payout rules. |

*Table 16. Discovery is opt-in per organization.*

> **Two unfakeable org-health signals**
> - **Leader last active.** Timestamp of the latest leader-signed on-chain action.
> - **Voluntary departure rate.** Share of members who left voluntarily over the trailing window.

Discovery surfaces do not aggregate report statistics.

### 8.9 Leader and Member Interfaces

Leader surfaces include review queue, member management, taxonomy, relay status, standing-policy version selection, and report-response controls. Member surfaces include role, contribution/serve summaries, and badge progress.

### 8.10 Reporter's Private View

Each reporter has a private list of their own reports and published escalations.

### 8.11 Session Provenance: The Honest Boundary

Every provenance claim in this section rests on signatures over *submitted content*: the contributor signs the canonical body (§7.4), the leader's bonded signature publishes it, and evidence accrues to it on-chain. That chain of signatures proves **who submitted and who curated**. It does not cryptographically prove that the underlying coding session occurred as described.

> **What provenance does not yet prove.** A contributor could fabricate a plausible "session" and submit memories extracted from it. Today's defenses against this are human and reputational — leader review, the contributor's reputation and wallet age shown at the gate, and the report rail — and they are real, but they are judgment, not proof. Accordingly, the word "verified" anywhere in this document means leader-curated and contributor-signed; it never implies session or model attestation.

The designed answer is a cryptographic session-attestation rail (the `x/attestation` socket, held inactive in current scope). Hardware-attested inference can produce signed attestation receipts that bind a session — and, folded one step further, the extraction that distills it into a memory — to an attested model and the contributor's identity. Such receipts carry only cryptographic hashes, never session text, so any client can verify them without the session being disclosed; commercial confidential-inference gateways ship receipts with exactly these properties today [30][31]. When this rail ships, it arrives as an optional, per-field-graded provenance label — never a contribution gate, and never a single "certified" badge over a claim only partly proven.

**Producer-model provenance and capability admission.** Every committed memory carries an immutable on-chain producer-model stamp plus a session/attestation reference riding the `x/attestation` envelope family. The stamp is immutable fact: canonical producer-model identity for the model that produced the source session. The hub remains rebuildable from chain state and materializes/indexes producer identity, attestation status, capability-registry snapshot ID, and a derived capability tier + uncertainty band.

Capability classification is versioned policy rather than immutable fact: a pinned, dated index snapshot as primary with a coding-specific index as cross-check; tier + uncertainty-band output only (never raw scalar ordering); canonical upstream identity owns classification and aliases inherit with provenance; manual overrides are explicit, expiring, and auditable as supersessions. Missing classification fails closed.

Recall applies capability eligibility before relevance ranking: a memory may inject only from a producer into equal-or-lower-capability consumers; lower→higher injection is forbidden; unknown producer/consumer classification fails closed; exact-self injection is allowed only when producer identity is cryptographically proven. This is an admission-stage pre-scoring filter, never a retrieval-scoring input, and it is distinct from standing (§7.7): eligibility asks whether a memory may be offered at all, standing asks how strongly it competes among those that may.

**Three legs, two pathways, one serving gate.** Admissible provenance resolves to exactly two pathways: **P1 `ATTESTED_EXECUTION`** (open-weight execution with verifiable workload/model identity — pluggable TEE/confidential-compute receipts today, CommitLLM-class local verifiable execution the recorded gap for local users) and **P2 `PROVIDER_WITNESSED`** (closed-weight sessions captured through a blind-witness TLS/transport commitment; the hub is the initial witness; it sees ciphertext + unavoidable metadata only and commits to the provider-authenticated request/response bytes). **P2 proves the real provider endpoint returned the committed bytes and binds the provider-reported model label — it does NOT prove which hidden weights ran.** `SELF_DECLARED`/`UNATTESTED` are absence states, never served. Major closed-weight providers do not expose cryptographic hidden-weight proof to ordinary API consumers; this is the honest limit, not a WeVibe defect.

Provenance admissibility is a distinct **SERVING gate** (moderation labels and contribution stay advisory/ungated; commit does not imply serveable). It is enforced two-tier: a hub pre-scan before ranking and gas-bearing work, then a receiving-client final check immediately before injection — the client wins, and unknown/missing/invalid evidence fails closed.

A servable memory needs admissible evidence for its **producer-session leg** (an ordered event/turn commitment root — versioned hash chain or Merkle accumulator — replacing the inadequate flat session hash) and its **extraction-session leg** (extraction is post-hoc, user-triggered, itself an LLM session — one call if it fits, else overlapping chunk fan-out; never one WeVibe-controlled call), joined by welds that bind producer-evidence→substrate, substrate/chunk-plan→every extraction request, and extraction-responses→the submitted memory.

To these the evidence log adds a third: the **consumer-session use leg** (§7.8). Where the first two legs establish that a memory was honestly derived from a real session, the use leg establishes what happened when a stranger relied on it — bound to the consumer's session identity and evidence reference, signed by the consumer, and carrying the same absence semantics as the others. It is weaker than the producer legs by construction: it attests an observation about work, not a cryptographic property of execution.

**Provenance proves exact inputs, responses, identities, and deterministic derivation; it can never prove the synthesized memory is semantically faithful or correct** — that stays leader review, consumer judgment, accumulated outcome evidence, and objective engineering evidence. Later validators verify these legs; folding/IVC may compress the checks into one succinct proof bound to the trajectory root, but folding compresses existing trust and cannot manufacture missing evidence.

### 8.12 Goal-Sealed Trajectory Verification

Section 8.11 fixed the honest boundary: signatures prove who submitted and who curated, never that the underlying session occurred. This section makes part of that gap irrelevant rather than closing it. Instead of proving the session, WeVibe proves the outcome — against a target fixed before the outcome existed. A memory that turns a pre-sealed check green is real knowledge even if its transcript is theater; for the class of work that admits an executable definition of done, the session-occurrence question dissolves.

**The goal seal.** At session start the plugin reads the project goal asserted in `AGENTS.md`, asks the contributor to confirm an executable test predicate that defines completion — often the currently failing test — and seals both, together with a hash of the starting working tree, under the contributor's signature and a signed timestamp.

```
goal_seal {
  goal_id          : unique identifier; the goal stays open while the predicate is red
  goal_text_hash   : sha256(goal statement as asserted in AGENTS.md at seal time)
  predicate_hash   : sha256(test command || test file contents)
  state0_hash      : sha256(working-tree manifest at seal time)
  repo_binding     : .wevibe/org.json marker reference
  sealed_at        : signed local timestamp
  contributor_sig  : Ed25519 signature over all fields above
  chain_anchor     : optional anchoring transaction hash (hook; unset in MVP)
}
```

Seals are immutable. A wrong goal is never edited; it is abandoned and a new goal sealed, and every seal still predates its own outcome — which is the entire anti-backfill property. Abandoned goals still yield negative receipts.

**The trajectory chain.** The plugin extends its session-signal harvesting (§5.2) by hashing working-tree states into a chained record of diffs, `state_0 → state_n`. The chain belongs to the goal, not the session: sessions attach to open goals ("Continuing goal X?") and continue the chain where the previous session ended, with each session boundary stamped with the predicate's color at that point. External changes between sessions — collaborator pushes, edits made outside the plugin — are recorded as explicit chain gaps: per-memory receipts survive a gap, but the completeness claim over the chain weakens, and the receipt says so.

**Goal close.** When the sealed predicate passes, the goal closes and extraction unlocks over the entire chain, not the final session alone. Necessity is judged against the finish line, not the chapter: a fix from a mid-chain session that ended red can still be the load-bearing fix. Memories submitted before their goal closes carry tier T0 and are receipt-upgraded when it does.

**Diagram 7 — One goal, many sessions.**
- Seal node: "Goal sealed: predicate + state_0, signed, timestamped."
- Session 1: "Config change → predicate: RED."
- Session 2: "Timeout raised → predicate: RED."
- Session 3: "Timeout reverted; pool recycling added → predicate: RED."
- Session 4: "Retry double-release fixed → predicate: GREEN. Goal closes."
- Below-rail (session 3): "Ablation: revert pool recycling from final state → RED ⇒ load-bearing (T2)."
- Below-rail (session 2): "Negative receipt: present at a red state, absent from the final green state (dnd)."
- Terminal: "Extraction unlocks over the whole chain."

*Diagram 7. The seal belongs to the goal, not the session. Receipts are judged against the finish line: a red chapter can still contain the decisive fix.*

**Receipts.** Each extracted memory cites the specific diff in the chain where its insight was earned. Three receipt types attach to that citation:

| Receipt | What it attests | Strength |
|---|---|---|
| **Predicate receipt** | The sealed test passed at `state_n`: exit status, environment fingerprint, and chain reference, signed. | A fact at goal close: the outcome met a definition of done fixed before the outcome existed. |
| **Ablation receipt** | Reverting the memory's cited diff from the final state and rerunning the sealed predicate turns it red. | Necessity: the cited fix was load-bearing for the outcome. Best-effort — revert conflicts and flaky predicates void it; it is a graded bonus, never a rung every memory reaches. |
| **Negative receipt** | The cited change was present at `state_x`, the predicate ran red there, and the change is absent from the final green state. | Deliberately weak: proves "not sufficient as tried, then" — never "wrong." A masked or incomplete idea earns the same receipt as a bad one. |

*Table 21. Receipts are contributor-machine artifacts: checkable claims, not proofs. See the boundary callout below.*

**Failure-episode evidence.** Extractor evidence spans are produced by deterministic, code-only segmentation of the session stream into failure episodes: a failing signal, the edits that followed, and whether that signal later disappeared or persisted. Resolved episodes yield grounded symptom→diff→validation spans; unresolved episodes emit negative candidates ("tried X, symptom Y, unresolved") — **but only within a session that resolved at least one problem. A session that resolves zero problems produces zero memories; extracting from a zero-progress session is a hard integrity failure.** This is the substrate-level slice of receipt semantics — before seals, signatures, tiers, ablation runs, or chain anchors — with attribution bound to the full attempt diff (never a guessed line) and coincidental green flips explicitly disclosed. The same segmentation is what makes the consumer-side use leg observable (§7.8); one mechanism serves contribution and evidence alike.

Leave-one-out ablation has a known blind spot: if either of two surviving changes alone keeps the predicate green, reverting one leaves it green and neither earns the badge, even though together they mattered. Full subset testing is combinatorial and deliberately out of scope.

**Tier grading.** Receipts compose into a per-memory verification tier:

| Tier | Requires | What it says at the gate |
|---|---|---|
| **T0** | Contributor signature over the canonical body (§7.4). | Today's baseline: a signed, leader-curated claim. |
| **T1** | Goal seal + trajectory chain + predicate receipt. | The outcome met a pre-sealed definition of done. |
| **T2** | T1 + ablation receipt. | The cited fix was shown necessary for that outcome. |
| **T3** | T2 + attested-run receipt through the `x/attestation` socket (§8.11). | The receipts themselves were produced by an attested runtime, not merely claimed. |
| **T4** | k independent contributors, k independent seals, matching context fingerprint, convergent fix [36]. | Stranger convergence: the certification is the agreement itself. A sybil must actually solve the task k times against k pre-fixed predicates. Requires corpus traffic; inert at cold start. |

*Table 22. Tiers are per-field, per-memory labels — rendered in review and at the plugin gate — and are never a contribution gate. Exploration, design judgment, and much negative knowledge have no executable predicate and contribute at T0 indefinitely, by design. Tiers do not enter retrieval scoring, and they are distinct from both model-capability tiers (§8.11) and from standing (§7.7): a tier says what was proven at contribution time, standing says what has been observed since, and neither may override the human gate.*

**Surfaces.** The contributor sees one banner at seal time ("Goal locked."), one question when resuming ("Continuing goal X?"), one stamp at session end, and one optional action at extraction ("Prove this memory mattered" — runs the ablation check, costs local compute, skippable). The reviewer queue gains a per-memory receipt strip: goal sealed before work ✓ / predicate passed ✓ / ablation red ✓ — or blank. The consumer gate gains the single Verification row specified in Figure 2. The four-button gate (Table 3) is unchanged, and nothing is ever blocked for lacking a receipt.

> **Why fabrication buys nothing.** A target fixed before the answer existed cannot be retrofitted to whatever was produced. And to forge a green predicate receipt, a contributor must make the pre-sealed predicate actually pass — which is the work itself. Verification cost collapses onto honest cost: proof-of-work for knowledge.

> **The honest boundary (checkable, not unforgeable).** T1–T2 receipts are produced on hardware the contributor controls and can be fabricated. Pre-commitment changes what a fabrication becomes: a forged receipt against a sealed predicate is a provable lie — reveal the sealed inputs, replay the predicate, watch it fail — anchored to the contributor's signature and handled through the same report rail as content disputes (§7.5). Replay requires the contributor's private states, so it bites only through the dispute path: a demanded reveal that is refused reads as silent acquiescence (§7.6), not as automatic detection. Two limits are permanent. A trivially weak sealed predicate yields technically true, practically empty receipts — a tier badge says "met its own pre-set bar," never "the bar was high"; bar quality remains with leader curation and the evidence log (§7.7). And no tier below T3 attests how the receipts were produced.

---

## 9. Storage Architecture

### 9.1 On-Chain Encrypted Memory Storage

Approved memories are encrypted chain state: ciphertext, wrapped DEK, plaintext metadata, and batch Merkle leaves. Validators replicate all memory state.

> **Size economics.** Typical encrypted memory size is 500 bytes–2 KB. At 10,000 memories: ~10–20 MB chain state; at 100,000: ~100–200 MB. Evidence events are far smaller per record but grow with usage rather than with corpus size; §9.5 states the compaction contract that bounds them.

### 9.2 Discovery Labels

Per-memory labels are on-chain metadata: public plaintext terms that support discovery, browsing, and a capped retrieval bonus. Taxonomy management is hub-side but anchored by on-chain `vocab_hash` and `embedding_model_id`.

Labels carry no weights in consensus, participate in no decay arithmetic, and have no bearing on whether a memory survives. A memory found purely by semantic similarity, matching no label at all, is a fully ordinary and fully countable retrieval; earlier designs that required a label intersection to record a use were removed, because the requirement rejected legitimate use and detected nothing.

### 9.3 Semantic Vector Index (Hub Qdrant)

Embeddings are hub-side derived data, rebuildable from chain state plus organization keys. Qdrant holds vectors + label metadata, not plaintext memory bodies; the confidentiality boundary and semantic-shadow concession are defined once in §4.5.

### 9.4 Memory Metadata

| Field | Meaning |
|---|---|
| `contributor_pubkey` | On-chain identity of the contributor. |
| `model_origin` | Contributing model (producer-model stamp, §8.11). |
| `stack_tags` | Freeform technology tags. |
| `labels` | Public plaintext discovery labels (§9.2). Unweighted. |
| `version` | Nullable version string. |
| `source` | `session` \| `doc_import` \| `authored`. |
| `approved` | Boolean moderation state. |
| `is_reported` | An open report currently stands against this memory. |
| `was_reported` | Reported at least once; permanent historical flag. |
| `report_cleared` | The leader dismissed the open report via `clear_report`. |
| `quarantined` | Flag for memories with repeated upheld rejections; retrieval-policy exclusion. |
| `deprecated` | Curator has marked stale. |
| `verification_tier` | T0–T4 receipt tier at commit time; upgradeable on goal close (§8.12). |
| `receipt_refs` | Hashes of the goal seal, predicate receipt, and (if present) ablation receipt. Nullable. |
| `mc_version` | Memory Contract (MC-1) schema version. |

*Table 17. The per-memory metadata record. Note what is absent: there is no trust score, no weight, no standing, and no archive flag. Those are outputs of §7.7, not fields.*

### 9.5 The Evidence Log

The evidence log is the chain's record of what the network observed. It is the only substrate standing is computed from.

**One envelope.** Every event carries `(event_type, org_id, memory_cid, window, signer_pubkey, nonce, [event fields], signature)` — Ed25519 over a canonical body, signed by the consumer's per-organization key. Every event is **content-free** (references and fingerprints only — never plaintext, keys, or memory content), **consumer-signed**, and **directly broadcastable** by the consumer; a WeVibe relay is retry-fallback only, never the required path. The organization serving key signs only the transaction envelope and carries gas under feegrant; it cannot mint event content.

| Event | Records | Notes |
|---|---|---|
| **serve** | A memory was delivered to a consumer and accepted at the gate. | Content-free. No label condition of any kind. |
| **block** | A human permanently refused a memory. | Optionally scoped to the situation of refusal — capability reserved, disclosure-gated (§7.7). |
| **outcome** | The episode a memory was used in resolved, or did not. | The load-bearing signal (§7.8). Bound to episode identity and an evidence reference. |
| **validity-predicate outcome** | A machine-checkable condition on the memory's applicability passed, failed, or could not be evaluated. | Staleness decided by fact rather than by elapsed time. Absence is recorded as absence, never inferred. |
| **cost-to-discover** | What the originating work cost to reach the knowledge: cycles, tool calls, attempts. | Expensive to fabricate, since inflating it means doing the work. |
| **convergence** | Independent sessions arrived at the same knowledge. | The positive counterpart of a dispute; underpins T4 (§8.12). |
| **contest** | A memory is challenged as wrong, with counter-evidence. | Reserved; activation depends on the supersession mechanism. |
| **sponsorship** | A principal stakes on a memory's continued presence in the index. | Reserved; presence only, never position — see §10.9. |

*Table 23. The evidence event families. Reserved families carry defined slots and are not emitted until their dependent mechanisms exist; a reserved slot costs nothing, while a shipped-but-inert event type costs permanent state.*

**The boundary rule.** No event may contain a verdict, a weight, a score, a trust value, an archive flag, or any derived judgment — and none may contain content. An event is an **observation with provenance**. Even an outcome's resolved/unresolved bit is a raw observed fact bound to evidence; what it *means* is policy's business, computed off-chain.

**Growth and compaction.** Evidence accumulates with usage, not with corpus size, so the log grows without a natural bound and naive replay-from-genesis would grow with it. Two mechanisms keep the contract honest without breaking it. Old events may be **compacted** into windowed observation summaries — still observations, still content-free, never conclusions. And a computed standing state may be committed as a **hash anchor** for replay verification without the state itself becoming authoritative: a hash is a commitment to a computation, not a judgment about a memory. Both preserve the property that any party can independently arrive at the same standing from public inputs.

---

## 10. Security Analysis

### 10.1 Sybil Resistance

Free identity creation is not prevented; harm is gated at organization membership. Membership and publication are leader-gated, and contributor reputation is resettable on removal. Evidence events inherit the same gate: only members of an organization can sign events against its memories, and per-epoch caps plus on-chain deduplication bound what any one identity can contribute to standing.

### 10.2 Memory Poisoning

Canonical defenses and limits are specified in §4.7 (sanitization pipeline, Table 10) and §7.2 (review coverage limits). Residual risk remains semantic payloads and plausible-but-wrong guidance. The evidence log narrows the window on the second category specifically: a plausible-but-wrong memory that repeatedly fails to help the work accumulates the evidence of that failure, and standing falls. This is mitigation over time, not prevention at admission, and it is stated as such.

### 10.3 Leader Key Compromise

`K_master` compromise exposes epoch-derived content. Mitigations are recovery phrase handling, encrypted vaulting, threshold recovery, and on-chain serving-key verification.

### 10.4 Chain State Observability

On-chain metadata is public; plaintext and local private state are not. Evidence events add a pattern surface: an observer sees which pseudonymous member used which memory and whether their work resolved. This is disclosed in the threat model (§4.1) and is a linkage consideration for any consumer who publicly links an organization identity to a profile (§8.3).

### 10.5 Network-Level Anti-DDoS

Alpha anti-DDoS is enforced by `x/bandwidth` per-organization per-epoch caps. Mainnet economic anti-spam is roadmap-scoped. Evidence events are rate-bounded by the same per-epoch caps.

### 10.6 Content Suitability Policy

| Suitable | Unsuitable |
|---|---|
| Coding patterns and anti-patterns, architecture lessons, debugging notes, dependency guidance, tool usage, process workflows, version-specific gotchas, negative knowledge. | Credentials/secrets, customer PII, regulated data, legal/HR records, high-sensitivity security-incident details. |

*Table 18. What belongs in a WeVibe memory, and what does not.*

### 10.7 Organization Capture and Public Escalation

A single actor wearing every hat — leader, every member with `can_moderate`, every contributing member — can fully capture an organization's internal governance. Inside the organization, every approval, every report dismissal, and every chain commit can be coordinated. Internal accountability primitives provide no protection against this: the captured operator simply approves their own malicious memories and dismisses every report against them.

The system's security model is therefore not "prevent capture through internal governance." It is:

> Make capture economically unsustainable through transparent on-chain accountability, frictionless exit for members, and a public escalation primitive a captured organization cannot suppress.

> **Four load-bearing properties**
>
> 1. **The chain is the unforgeable audit log.** Every consequential action — memory commit, evidence event, report resolution, dispute publication, member departure — is a signed on-chain transaction. Neither the captured organization, nor WeVibe, nor any platform operator can edit or suppress it after the fact. And because no judgment is written, there is no judgment to falsify: the log records what happened, and what it means is recomputable by anyone.
> 2. **Consumers hold an escalation path the organization cannot close.** A dismissed or unaddressed on-chain report unlocks a reporter-signed public escalation — wallet-gated, gas-paid, revealing plaintext only after the window elapses or the leader dismisses, anchored to the contributor-signed hash the leader cannot poison (§7.4) — and once published it cannot be edited or deleted. The reporter broadcasts directly to chain RPC, with the WeVibe relay as retry fallback only.
> 3. **Exit is unfakeable.** Members leaving voluntarily is a first-class on-chain event. Sybils can be invited and can file frivolous reports, but they cannot fake people walking away. The voluntary-departure-rate signal on discovery (§8.8) lets prospective joiners read the most honest possible signal.
> 4. **Hub compromise is a per-organization degradation event, not a network takeover.** Per-memory Umbral crypto and consumer-side wevibe-guard still gate plaintext and injection, and hub responses must verify against on-chain serving keys. A compromised hub cannot mint standing either — it can only report standing that any client can recompute and reject. It can at worst degrade recall for that one organization; it cannot mint identities, steal contributor keys, or affect other organizations. The endpoint can be rotated on-chain by leader signature.

**The leader bears sole signature.** Co-attestation of additional reviewer public keys on leader-signed chain transactions is explicitly excluded (§7.3). A leader's chain commit binds the leader's wallet only. This concentrates responsibility on the actor who actually signs and prevents implicating advisory reviewers in chain-level decisions they did not authorize.

**Why no platform tribunal.** WeVibe deliberately does not adjudicate published reports. Any in-app judging body is a capture vector — whoever controls the tribunal controls the verdict. The chain publishes the evidence, the block explorer renders it, the reporter shares the URL, and the public judges on its own merits (§7.7).

> **Residual: contributor-leader collusion.** If the contributor who submitted a memory and the leader who committed it are both adversaries, the contributor can sign a false plaintext hash, the leader can commit it, and a public report's plaintext reveal will not match the on-chain hash — making a true report look invalid. The on-chain ciphertext + capsule remain a final backstop: any future key disclosure lets independent parties decrypt and verify after the fact (§7.4). This residual cannot be eliminated at submit time — when the content creator and the content approver are the same adversary, there is no honest party in the verification chain to sign over.

### 10.8 Receipt Integrity and Predicate Gaming

Verification receipts move risk in one direction: they convert unverifiable narrative into checkable claims. Canonical semantics, tiers, and limits are specified in §8.12. Residual risks are three: fabricated T1–T2 receipts, detectable only through dispute-path reveal-and-replay; trivially weak sealed predicates, bounded by "met its own bar" label semantics and leader curation rather than cryptography; and leave-one-out ablation blindness to jointly redundant fixes. None of these widens the injection surface: tiers are labels rendered at the human gate, never bypasses of it, and a memory with no receipt is exactly as admissible as it is today.

### 10.9 Standing-Policy Risk

Moving judgment out of consensus removes a class of risk and introduces a smaller one that must be named.

**The risk.** Standing is computed over the whole log, so a new policy version re-evaluates all history at once. A badly-drafted policy could therefore suppress a large fraction of an organization's corpus in a single deployment — with no vote, no signature, and no warning. Placing that power with whoever operates a deployment pipeline would recreate, in a new location, exactly the concentration the Four Exit Guarantees exist to prevent.

**Three controls.**

1. **Organizations pin their policy version.** The version in force for an organization's corpus is the organization's choice, recorded on-chain and upgraded deliberately. No operator can unilaterally re-judge a stranger's knowledge; a leader who declines an upgrade keeps the interpretation their members already accepted.
2. **Every version is published before it is anchored,** with its behavioural delta against the incumbent measured on a fixed corpus. A policy that changes what survives must say so in advance, in numbers, or it does not ship.
3. **Nothing is destroyed by a policy change.** Because standing is computed and no memory is deleted, a bad policy is reversible by pinning back. The worst case is a period of degraded surfacing, not a loss of knowledge — which is the correct failure direction for a system whose central promise is that knowledge cannot be killed.

**Two open limits, stated rather than hidden.** The accuracy of outcome observation in production is an empirical quantity that varies by environment and must be measured per deployment rather than assumed from design (§7.8). And standing computed from evidence generated under a previous standing shapes what evidence is generated next — a feedback loop that a well-run deployment must measure directly, via randomized exposure and counterfactual comparison, rather than reason about from first principles.

---

## 11. Decentralized Architecture

### 11.1 Chain Architecture

WeVibe runs as a sovereign L1 appchain on Cosmos SDK + CometBFT with deterministic finality and safety-over-liveness behavior. Organization state includes chain-resolved `hub_endpoints`; serving authority is separate (`hub_serving_address`); and the standing-policy version in force is anchored per organization.

### 11.2 The Four Roles in Detail

**Developer.** Onboards by passkey, contributes explicitly, recalls through the mandatory gate, and — passively — supplies the evidence that keeps the corpus honest.

**Organization leader.** Signs org registration from their own wallet, is sole publishing authority for commits, and selects the standing-policy version in force.

**Validator.** Runs consensus and stores encrypted chain state and the evidence log.

**WeVibe (protocol).** Open-source software; no single operator is a required trust anchor, and no operator is an authority on what a memory is worth.

### 11.3 On-Chain Modules

| Module | Responsibility |
|---|---|
| `x/org` | Slot registry, membership, chain-resolved transport/auth fields (`hub_endpoints`, `hub_serving_address`), standing-policy version anchor, feegrant, dormancy/abandonment detection. |
| `x/memory` | Pending commitments, approved encrypted blobs, epoch Merkle roots, contributor-signed anchors, report/quarantine flags. |
| `x/serve` | The evidence log: recording and signature-verification of serve, block, outcome, and related event families; deduplication by fingerprint; self-authored discounting; per-epoch caps; compaction of aged windows. Holds no weights and computes no standing. |
| `x/identity` | Passkey-derived contributor identity and key registry. |
| `x/reputation` | Cross-organization contributor aggregates (contribution counts, serve breadth, domain tags). |
| `x/emissions` | Validator rewards, contributor emissions, schedule controls, claim path. |
| `x/bandwidth` | Per-organization per-epoch flat rate-limit caps in testnet/alpha anti-DDoS mode. |
| `x/attestation` | Session-attestation storage socket (inactive in current scope; §8.11). Reserved anchor point for optional goal-seal and verification-receipt commitments (§8.12). |

*Table 20. Custom modules. Standard SDK modules also in use: `x/staking`, `x/auth`, `x/bank`, `x/gov`, `x/slashing`, `x/distribution`.*

### 11.4 Node Lifecycle (`wevibed`)

`wevibed` follows standard Cosmos bootstrap patterns.

---

## 12. References

1. Song, D., Wagner, D., Perrig, A. (2000). *Practical Techniques for Searches on Encrypted Data.* IEEE S&P.
2. Curtmola, R., et al. (2006). *Searchable Symmetric Encryption.* ACM CCS.
3. Cormack, G. V., et al. (2009). *Reciprocal Rank Fusion.* SIGIR '09.
4. Reimers, N. & Gurevych, I. (2019). *Sentence-BERT.* EMNLP 2019.
5. Kusupati, A., et al. (2022). *Matryoshka Representation Learning.* NeurIPS 2022.
6. Greshake, K., et al. (2023). *Indirect Prompt Injection.* arXiv:2302.12173.
7. OWASP (2025). *Top 10 for LLM Applications.*
8. Sha, F., et al. (2025). *Lessons from Defending Gemini Against Indirect Prompt Injections.* Google DeepMind.
9. Zou, A., et al. (2024). *PoisonedRAG.* arXiv:2402.07867.
10. Google DeepMind (2026). *AI Agent Traps.*
11. *MLS Architecture* (RFC 9420).
12. Lambda Class. *CommitLLM: Cryptographic commit-and-audit for LLM inference.* github.com/lambdaclass/CommitLLM.
13. Morris, J., et al. (2023). *Text Embeddings Reveal (Almost) As Much As Text.* EMNLP 2023.
14. Huang, Y., et al. (2024). *Transferable Embedding Inversion Attacks.* ACL 2024.
15. Núñez, D., et al. (2018). *Umbral: A Threshold Proxy Re-Encryption Scheme.* NuCypher / NICS Lab, Universidad de Málaga.
16. OpenAI. *ChatGPT Memory — Memory FAQ.* (accessed July 2026).
17. Anthropic. *Claude Memory + Projects / Memory Tool.* (accessed July 2026).
18. Google. *Gemini Personal Intelligence / memory.* (accessed July 2026).
19. GitHub. *About GitHub Copilot Memory.* (accessed July 2026).
20. Bhartendu-Kumar et al. *Rules & Memory-Bank pattern (Cursor / Windsurf / Cline / Roo).* (accessed July 2026).
21. Mem0. *Mem0 — memory layer for AI agents.* (accessed July 2026).
22. Letta AI. *Letta (formerly MemGPT).* (accessed July 2026).
23. Zep AI. *Zep / Graphiti — temporal knowledge-graph memory.* arXiv:2501.13956 (accessed July 2026).
24. LangChain. *LangMem / LangGraph long-term memory.* (accessed July 2026).
25. *RAG-over-a-wiki (Notion/Confluence + vector DB).* Widely-used integration pattern (accessed July 2026).
26. Recall Network. *Recall — decentralized intelligence layer.* (accessed July 2026).
27. Ceramic Network. *Ceramic / ComposeDB.* (accessed July 2026).
28. Arweave / AO. *Permanent storage protocol + compute layer.* (accessed July 2026).
29. GaiaNet. *GaiaNet — GenAI agent network (litepaper).* (accessed July 2026).
30. TLSNotary (Privacy Stewards of Ethereum). *TLSNotary — data provenance without compromising privacy.* (accessed July 2026).
31. RedPill / Phala. *Confidential AI inference: attestation reports and signed receipts.* (accessed July 2026).
32. Apple Security Engineering and Architecture. *Private Cloud Compute.* (June 2024; accessed July 2026).
33. Google. *Private AI Compute.* (accessed July 2026).
34. *Are Online Skill and Memory Modules Always Worth Their Tokens? A Budget-Constrained Study of Web Agents.* arXiv:2606.15017 (accessed July 2026).
35. Wang, Z., et al. (2025). *Agent Workflow Memory.* ICML 2025; arXiv:2409.07429.
36. *WISE-Flow: Workflow-Induced Structured Experience for Self-Evolving Conversational Service Agents.* arXiv:2601.08158 (accessed July 2026).

---

## Appendix A — Cryptographic Procedures

Each organization has a master key `K_master`. Per-epoch keys derive from it via HKDF. The leader derives these keys locally.

```
K_enc(e)    = HKDF-SHA256(K_master, info="wevibe-enc-"   || epoch_be_bytes)
K_audit(e)  = HKDF-SHA256(K_master, info="wevibe-audit-" || epoch_be_bytes)
epoch_sk(e) = HKDF-SHA256(K_master, info="wevibe-umbral-epoch-" || epoch_decimal_ascii)[:32]   → Umbral secret key
```

- **K_enc(e)** wraps per-memory DEKs for approved memories in epoch *e*.
- **K_audit(e)** is reserved for audit and receipt verification.
- **epoch_sk(e)** is derived on the leader machine and never transmitted; the hub receives only `umbral_pk` and finished kfrags.

Each memory is encrypted with a fresh DEK.

```
DEK             = random(32)
ciphertext      = AES-256-GCM(DEK, nonce, plaintext_memory)
wrapped_dek_mod = seal_to_pubkey(DEK, PK_mod(e))
```

On approval, the reviewer re-wraps DEK under the epoch encryption key.

```
wrapped_dek_enc = AES-256-GCM(K_enc(e), nonce, DEK)
```

The approved bundle (`ciphertext`, `wrapped_dek_enc`, metadata) is then published on-chain.

```
seal_to_pubkey:
  1. Generate an ephemeral X25519 keypair.
  2. ECDH between the ephemeral private key and the recipient's X25519 public key.
  3. Derive a symmetric key via HKDF (info: "wevibe-envelope-v1").
  4. Encrypt with AES-256-GCM.
  5. Output: ephemeral_pubkey (32 bytes) || nonce (12 bytes) || ciphertext+tag
```

**Custody model invariant.** Each organization's `K_master` is independent random entropy.

**Recovery.** BIP39 24-word recovery phrase; Shamir 2-of-3 threshold recovery; encrypted leader vault (`~/.wevibe/vault.enc`) with Argon2id (t=3, m=64 MB, p=4).

---

## Appendix B — Badge & Reputation Taxonomy

| Family | Basis |
|---|---|
| **Serve-milestone badges** | Thresholds on how often a contributor's memories are served. |
| **Outcome badges** | Thresholds on observed outcomes attributable to a contributor's memories, discounted for self-authored use (§8.7). |
| **Contribution-volume badges** | Thresholds on approved-memory contribution volume. |

*Table 15. Scoping is per-organization, with profile breakdowns instead of a single global ladder. Badge status is strictly non-economic: no reward and no payout coupling.*

Organization-level badge state is canonical. Contributors may optionally summarize across organizations, but WeVibe does not publish a global leaderboard.

---

## Appendix C — The Document Set

WeVibe is a family of interoperable services. This general architecture document captures the shared threat model, encryption design, evidence model, and contributor experience. Deeper, implementation-specific material lives alongside the source for each product: the consensus-layer economics and keeper architecture; the client stack (SDK, MCP server, guard, protocol assets) and plugin UX; the standing-policy implementations and their published version history; and the operational surfaces (hub, dashboard, infrastructure) with their deployment models. Use this document for cross-cutting concerns and the per-product sets for implementation specifics. Forward-looking capabilities and the token-economic model are specified separately in `ROADMAP.md`.

---

*This document is an architecture specification. Nothing in this document constitutes an offer or solicitation of securities.*