# WeVibe Network

**Unstoppable Knowledge, With Provenance**
*A censorship-resistant network where hard-won engineering knowledge is encrypted on-chain, curated by accountable humans, and delivered into any coding agent — readable by no one in the middle, killable by no one at all.*

Draft v2.7 · July 2026 · Architecture Document
Classification: Public

---

## Changelog

| Version | Summary |
|---|---|
| v2.7 | Positioning inversion (DMO-033): knowledge-network thesis leads; reputation reframed as the provenance layer made visible; report/expose accountability loop embedded as core architecture; four-button gate unified across all sections; build-status registers removed from the public document (implementation sequencing lives in engineering canon); abstract rewritten around the exit guarantees. |
| v2.6 | Personal-memory layer scoped: bounded, pull-mode local memory distinct from shared org memory; provider interface open, one default packaged, BYO opt-in. |
| v2.5 | Extraction-provenance honesty pass (DMO-032): trust-circle parity; cloud-mainstream extraction canonized; session-matched default; locality claims scoped. |
| v2.4 | Mission-integrity pass (DMO-030): exit-guarantees invariant; custody resolved non-custodial; direct-broadcast model; semantic-shadow disclosure. |
| v2.3 | Social-first repositioning; chain-resolved hub-endpoint + onboarding posture added. |
| v2.2 | Verification anchor redesigned to a contributor-signed canonical body (`plaintext_hash`/`salt`/`ciphertext_hash`); SP1/ZK pathway rejected as operationally unshippable. |
| v2.1 | Accountability primitives: silent denial as cheap negative signal, two-tier reports with public on-chain escalation, leader sole signature, unfakeable org-health signals. |
| v2.0 | Architecture pivot: on-chain encrypted memory storage + hub retrieval with local decryption/guard + plugin-gated human approval; domain-expert-led orgs. |

---

## Abstract

Two decades of file-sharing networks settled an engineering question: content replicated across many independent machines, indexed by a system owned by no one, cannot be taken down. What those networks never carried — what they destroyed by design — was consent and provenance. WeVibe takes the unstoppability and inverts the rest. It is a network for hard-won engineering knowledge that contributors *chose* to share, where every memory is signed by its author, curated by an accountable human, and permanently attributed on a chain no company controls.

The mechanics: memories are extracted from real coding sessions, encrypted on the contributor's machine, and committed as chain state. Validators replicate ciphertext they cannot read. Plaintext exists only at the edges — the contributor at extraction, the org leader at curation, the consumer after an explicit approval gate. Nothing enters an agent's context without human eyes on it first.

Curation is kept honest without a tribunal. Organization leaders sign every chain commit alone, bonded by a scarce, forfeitable slot. What disciplines them is not WeVibe's judgment but an escalation primitive nobody can suppress: any consumer can file a wallet-signed report directly to chain RPC; if the org dismisses it or lets the resolution window lapse, the reporter's public plaintext reveal unlocks — verified against a contributor-signed cryptographic anchor the leader never touches, rendered by any block explorer, editable by no one. Capture is not prevented; it is made economically unsustainable. And exit is unfakeable: members walking away is first-class chain state that no sybil can counterfeit.

The result is a system in which no single party — including WeVibe-the-company — can **read** member knowledge from the outside, **withhold** the network's function from a principal acting within their rights, **rewrite** the historical record, or **kill** an org's knowledge or a contributor's standing by withdrawing infrastructure. The chain is the only durable authority; every other component is disposable and reconstructible from it.

For the individual vibe coder, the experience is simple: install a plugin, contribute what actually worked, and your next agent session starts with your community's verified memory instead of a cold start. The payoff is sharpest when running local or smaller models, which inherit fixes first won in frontier sessions — collective problem-solving intelligence harvested across many contributors rather than re-paid run by run. Your public profile is the provenance layer made visible — what you solved, where it served, under whose curation. Coding is the first domain; nothing in the protocol is coding-specific.

This document is the architecture contract: the normative specification the network is built and audited against.

---

## 1. Design Philosophy

### 1.1 The Problem

Vibe coders do hard problem-solving with agents every day, but each new session behaves like a cold start. A fix discovered today — the precise Nginx keepalive tweak, the one migration order that prevents data loss, the exact flag combination for a flaky deploy — is trapped in private chat logs and forgotten tomorrow.

This hurts most when working with local or smaller models. The model can write code, but it does not know your stack's lived edge-cases, version mismatches, and production scars. The gap is not "more generic internet text"; the gap is verified, domain-specific memory from people who actually ran the problem. The verified-memory value claim is **any-agent** — the corpus serves whatever model you run — but the sharpest consumer pain (and the marquee payoff) is the local/smaller model, which gets a fix it could not have produced itself; the knowledge source is most often a frontier/cloud session where that fix was first won.

At the same time, developer knowledge channels are drowning in AI-generated content that sounds plausible but misses decisive nuance. Provenance is becoming the scarce good. A memory attached to a real contributor, a real org, and a real curation decision is materially more useful than anonymous content sludge — and it is the only kind of knowledge worth building permanent infrastructure for.

Finally, every prior attempt at shared technical knowledge lives inside a company database: a kill switch, a moderation team that can be pressured, terms that can change, an index that can be seized or simply shut down when the funding runs out. Knowledge that outlives its platform requires a platform that cannot die.

### 1.2 The Ensemble, Not the Sample

A single agent run is one sample from a distribution of possible answers. Practitioners who take problem-solving seriously already treat it that way: they run the same problem more than once — across seeds, temperatures, and models — and reconcile the variants toward the answer that holds up. The strongest solution is rarely the first pass; it is the one that recurs, that survives being attempted from several angles. Most vibe coders never observe this, because obtaining that cross-run agreement is expensive — every additional run is more tokens, more time, more money, paid again by each person for each problem. The problem-solving intelligence is latent in the model, but at the point of use it is left unrealized: one draw is taken, and the better answer the next draws would have surfaced is never seen.

WeVibe's answer is to expose that intelligence collectively instead of making each coder buy it alone. The network already holds the output of everyone who ran some variant of the same problem — the versions a human chose to contribute, that a curator accepted, and that went on to serve real sessions. A recall hands a coder the reconciled result of many independent attempts without them personally paying the N-run cost: the individual funds one session; the network exposes the settled intelligence of many. This is the any-agent value claim of §1.1 seen from the supply side. The sharpest payoff is still the local or smaller model inheriting a fix first won in a frontier session — but the deeper reason that fix is worth inheriting is that it represents a best-of-many outcome curated over time, not a single lucky draw.

**The reconciliation is emergent, not mechanical.** WeVibe does not run a problem N times and tally a vote; the protocol performs no automated multi-sampling, and this document specifies none. The agreement is produced socially and curatorially: many contributors submitting what actually worked, leaders accepting the versions that clear the bar, and serve history feeding Earned-Trust decay (§5.8) so that repeatedly useful memory is weighted up and misleading memory decays out. The corpus converges on quality the way an ensemble does — but the ensemble is the community over time, and the signal accrues from real curated use, not from a synchronized batch of runs the network fires on demand.

### 1.3 The Inversion

The file-sharing era proved a property no legal or infrastructural pressure has reversed: when content is replicated across many independent machines and the index is owned by no one, distribution cannot be stopped. That property is exactly what durable knowledge sharing needs — and exactly what no knowledge platform has ever had.

But that lineage carried two fatal absences. It moved content *without the consent of its creators*, and it *stripped provenance* — the one thing that makes technical knowledge trustworthy. WeVibe is the inversion: keep the unstoppability; restore consent and attribution.

- **Consent is structural.** Nothing leaves a contributor's machine without two explicit actions (Extract, then Submit — §4.5). Sessions stay local by default. Contribution is a choice made per memory, per org.
- **Provenance is the payload.** Every memory travels with its author's signature, its org's curation record, and its serve history. Attribution is not metadata bolted on; it is the reason the content is worth retrieving.
- **Unstoppability is enforced as four exit guarantees.** No single party — including WeVibe-the-company, any hub operator, or any org leader — may have the unilateral ability to **READ** member memory plaintext, **WITHHOLD** the network's function from a principal acting within their rights, **REWRITE** the historical record, or **KILL** an org's knowledge or a contributor's standing by withdrawing infrastructure.

The rest of this document is the engineering required to make those three properties true simultaneously. Encryption provides blindness; the chain provides permanence; sole-signature curation provides accountability; the public report provides recourse; the exit guarantees give the whole construction its teeth.

### 1.4 The Organization Model

Organizations are domain-expert-run memory collections you join. Think: a React performance org, a Solana tooling org, a Kubernetes reliability org. You join because your daily agent work needs better domain context now, and because you want your own contribution history to compound publicly over time.

Each organization is a collaboration container with its own:

- **Membership roster** managed by the leader.
- **Role hierarchy** (Leader, Reviewer, Member).
- **Commitment standards** for what counts as high-quality memory.
- **Domain focus** and coverage map.
- **Operating policies** for contribution/review cadence and recall access.

Leaders are domain experts responsible for memory quality, not faceless administrators. They approve who can review, define the bar for acceptance, and curate the collection as tools evolve. Strong org leadership raises both memory usefulness and the credibility of every attributed memory in the collection.

Orgs are intentionally broad across experience levels: newer coders join to learn faster; experienced coders contribute high-signal memories, mentor standards, and build attributed history. The org is the container. The vibe coder is the protagonist.

Public plaintext keywords are part of this model by design. They are discovery labels that help people find relevant memory collections quickly (for example, `redis`, `solana`, `django`) — not secrets, and not treated as a risk narrative in this document.

**Personal memory vs shared org memory (bounded, pull-mode).** Shared org memory — the curated, encrypted, chain-anchored collection described throughout this document — is WeVibe's predictive lane: it answers the *new* problem ("has someone solved something like this?") via situation-matched recall (§3.5). A coder's **personal memory** is a deliberately *different and bounded* layer: deterministic, on-demand (pull-mode) recall of KNOWN facts about their own repo and work — a repo/code map, durable project decisions, session-handoff state — retrieved on request, never speculatively injected at session start. The boundary is load-bearing: speculative context-injection for new problems is exactly where general personal-memory tools over-inject and add noise, and it is the lane WeVibe's curated org recall already wins. Personal memory is therefore scoped to *pull, not push*; any predictive/semantic personal recall (a coder's own past solutions, pre-contribution) is served through WeVibe's own aligned pipeline as a private corpus — not a bolted-on engine — which doubles as the on-ramp to contribution. WeVibe commits to a stable personal-memory provider interface with one packaged default and an opt-in bring-your-own engine for power users; the personal store is local scratch (exit = re-index), explicitly outside the chain-rebuildable contract.

### 1.5 The Curator Workbench

WeVibe is a curator workbench, not an autonomous ranking machine. The system surfaces signals — retrieval frequency, denials, staleness, query gaps, version drift — and human curators decide the action. This preserves accountable judgment where it belongs: with domain people who understand context.

Core workflows remain practical and hands-on: review pending memories (approve/reject/edit), author memories directly, package memories into reusable skills, identify coverage gaps, and retire stale entries before they mislead downstream users.

This curation loop serves both halves of the network. It protects recall quality (the memory that powers the next coding session), and it protects the integrity of the provenance layer by ensuring attributed memories actually deserve the trust their attribution implies.

### 1.6 The Plugin Gate

Every memory passes through human eyes before it enters an agent's context. This is the product invariant.

The plugin is installed in the developer's coding environment (OpenCode, Claude Code, Cursor, Cline, and similar tools). During a coding session, the plugin harvests local session signals and auto-queries organizational memory through the hub retrieval path; candidate memories are decrypted locally, scanned by wevibe-guard, and presented in an approval UI:

```
┌──────────────────────────────────────────┐
│ Memory Injection Request                 │
│                                          │
│ "Redis cluster-node-timeout must be      │
│  set to 15000ms when running behind      │
│  AWS NLB with cross-AZ failover..."      │
│                                          │
│ Contributor: wevibe1x7k...f3q2           │
│ Wallet age:  8 months                    │
│ Rep score:   347 (Tier 3)                │
│ Serves:      214 across 12 orgs          │
│ Domain:      redis, kubernetes, aws      │
│                                          │
│ Detections: [url: aws.amazon.com]        │
│                                          │
│ [✓ Accept] [✗ Deny] [⊘ Block] [⚑ Report] │
└──────────────────────────────────────────┘
```

**Accept:** The memory is injected into agent context. Serve attribution is queued to chain aggregates (contributor + org) for public profile signals. No per-serve payout is implied by this action.

**Deny:** The memory is blocked for this session/context only. Deny is a neutral context signal — "not what I need right now" — not a corpus down-vote; no denial event is emitted to the chain.

**Block:** The memory enters the consumer's permanent personal blacklist AND emits a global corpus denial signal — the load-bearing negative-signal path that feeds Earned-Trust decay (§5.8).

**Report:** The memory is reported on-chain — recorded against the contributor with the reporter's wallet (`is_reported` / `was_reported`) — and enters the org's accountability path (§5.5–5.7). The memory keeps serving until the report is resolved; the reporter can additionally Block it to drop it from their own recall. Reports are the high-friction accountability primitive (wallet-gated, gas-paid); denials are the low-friction ranking signal.

**No plugin installed = no memory injection path.** The MCP server has no direct route to force memory into the agent context without the plugin frontend.

**Why not MCP elicitation?** Elicitation is useful in theory, but inconsistent across clients and weak as a hard-interrupt safety surface. The plugin provides deterministic interruption, clear modal UX, and explicit confirmation.

**Injection is per session, not per turn.** Recall is queried on each user prompt, but a recalled memory is injected into the agent's context **once per coding session** (keyed to the session identifier), not re-pushed on every model turn — the plugin tracks what it has already surfaced in the session, and a memory's serve receipt is recorded once per session. A recall/inject governor (relevance floor + surface budget) keeps injected volume small.

### 1.7 Protocol, Not Platform

WeVibe is a protocol with open, auditable data surfaces — not a single closed SaaS product. The chain, hub, local client, and plugin each do one narrow job well.

**What the protocol provides:**

1. **On-chain encrypted storage + provenance.** Memories are committed as encrypted blobs with attribution metadata.
2. **Human-gated delivery.** The plugin is the mandatory approval path before any memory enters agent context.
3. **Public attribution.** Contribution and serve aggregates power the provenance and reputation surfaces.
4. **Domain-expert governance.** Leaders and reviewers curate memory quality inside each org collection.
5. **Suppression-proof accountability.** Consumer reports and public escalation broadcast directly to chain; no WeVibe infrastructure sits between a reporter and the record (§5.5).
6. **Coordination layer.** wevibe-hub runs hosted coordination, accounting, and retrieval workflows; it is never a plaintext memory oracle.
7. **Local retrieval edge + sanitization.** Decryption, guardrails, and injection run close to the user.
8. **Context injection format.** Approved memories are packaged for direct agent context use.

**Trust boundaries.** The chain is trusted for ordering and integrity, not for plaintext confidentiality. Plaintext handling stays local. The hub handles control-plane workflows. Validators and public observers see encrypted blobs and state metadata, never raw memory content.

**Exit guarantees.** The enforceable form of "owned by no one" is the four guarantees of §1.3: no unilateral READ, WITHHOLD, REWRITE, or KILL by any party, including WeVibe itself. The chain is the only durable authority; every other component must be disposable and reconstructible from chain state plus member-held keys. Every accountability action must be broadcastable by its principal without anyone's permission.

---
## 2. System Architecture

### 2.1 Entities

Let **L** denote the set of leaders, **O** the set of organizations, **D** the set of reviewers, **C** the set of contributing members, and **R** the set of read-only members. Each organization o ∈ O has a leader l(o) ∈ L who controls membership and configuration.

Each participant holds an Ed25519 keypair (with associated X25519 key for encryption) that serves as their protocol identity. Contributor pub keys are on-chain — serve receipts and reputation aggregates are tied to these keys.

### 2.2 Role Hierarchy

| Role | Permissions | Appointed By |
|------|------------|-------------|
| Leader | Full org control, roster management, epoch rotation, key custody, reviewer appointment, keyword taxonomy management, slot/rent/revenue management | Self (slot acquisition) |
| Reviewer | Review pending memories, approve/deny, decrypt pending submissions via SK_mod(e) | Leader |
| Member | Submit memories (within per-epoch bandwidth caps), retrieve approved content, view own pending submissions | Leader (via invitation) |

All roles require epoch-specific encryption keys for content access. The leader distributes these keys to approved members through sealed envelope key exchange (§3.4).

### 2.3 Organization Lifecycle

**Creation.** Org capacity is a scarce, capped set of registry-allocated **slots** (hard cap, governance-set: 32 alpha / 320 testnet / 3200 mainnet). The leader acquires a slot — ascending-price primary allocation while the cap fills; descending (Dutch) resale re-homes a freed slot — and signs `MsgRegisterOrg` from their **own wallet** (the hub never signs it). The acquisition payment is split 50/50: half is burned and half capitalizes the org's own on-chain account. The `org_id` is the permanent slot identifier, independent of the leader — it survives leadership transfer and resale. The leader generates the master key K_master, derives the initial epoch keys (epoch 0), and generates the initial moderation keypair SK_mod(0)/PK_mod(0). A 24-word BIP39 recovery phrase is derived from K_master and displayed once.

**First-run detection.** When the MCP plugin/server starts and discovers no org membership, it surfaces an actionable message to the agent, prompting guided setup.

**Operation.** Members join through leader invitation. Once approved, the leader issues sealed key envelopes containing the epoch keys to the new member. For reviewers and leaders, the envelope also includes SK_mod(e) for the current epoch.

**Contributor onboarding (wallet-free).** A contributor creates an account in seconds with a **passkey** (Face ID / fingerprint) — no wallet, no seed phrase. They install the plugin, connect the MCP server, and request to join the org; once a leader approves, they contribute and recall on that passkey identity alone (the hub keys members by pubkey; a wallet address is optional and attached only later). Contribution is explicit and dashboard-driven: open `/sessions` → select a session → click **"Extract Memories"** (client-side extraction with the contributor's selected model — local or session-matched cloud; returns review candidates — does NOT submit) → review memories → choose per-memory org destination → click **"Submit"**. Contribution has two explicit consent points, and nothing reaches the org/hub before Submit: **Extract** sends the session transcript to the contributor's chosen extraction provider (on-machine when local is selected; to the cloud provider at Extract time otherwise), and **Submit** sends the encrypted memory to the org. Neither is automatic. A Cosmos wallet is an optional later upgrade, needed only to **claim accrued VIBE rewards** or pay a mainnet per-memory fee; rewards accrue to a claim-later balance until then. (The org **leader**, by contrast, does need a wallet at creation — to acquire and bond the org slot.)

**Key rotation (epoch advancement).** When a member is removed, the org enters `rotation_pending` state:

1. **Removal triggers `rotation_pending`.** The chain marks the org as pending rotation. The removed member's envelope is deleted.
2. **New submissions are buffered.** Contributors can still submit, but submissions enter a hub-side rotation buffer — not admitted to the chain, not indexed in hub retrieval, not assigned a final epoch.
3. **Leader completes rotation.** The leader derives new epoch keys from K_master via HKDF, generates a new moderation keypair SK_mod(e+1)/PK_mod(e+1), and re-seals envelopes for all remaining members.
4. **Buffer finalizes.** After rotation completes, buffered submissions are released to the chain under the new epoch.
5. **Grace period escalation.** If rotation is not completed within a 72-hour window, the org's submission bandwidth is suspended until rotation completes.

**Revocation semantics.** Epoch rotation provides forward secrecy only. Removed members retain previously-distributed epoch keys and can decrypt content from their membership period.

**Membership state consistency (chain-first, watcher-mirrored).** All consequential membership transitions — add member, remove member, transfer leadership, close org — are **chain-authoritative**: the chain `x/org` handlers enforce them (signer must equal the org's registered leader wallet; role validation; the leader cannot be removed, only transferred; `MsgTransferLeadership` carries the mandatory `new_leader_wallet` and requires the new leader to already be a member). The hub holds **no optimistic membership writes**: a leader's "accept join request" is an *intent* (`join_requests.status = 'confirming'`) that does nothing until the chain `MsgAddMember` confirms; the hub's chain watcher then promotes the request to a member (idempotently, keyed on the chain event), mirrors transfer/close, and on member removal performs the full crypto revocation in the watcher (envelope delete + Umbral kfrag delete + rotation-pending) — this revokes future decryption and serving; plaintext already served remains permanently disclosed. If the wallet transaction is cancelled, the dashboard reverts the intent (`confirming → pending`) so no phantom half-member is left. A conservative **reconcile sweep** (60 s) heals role / `chain_confirmed` drift against the chain's member set and reverts stale `confirming` requests (>10 min), logging — never auto-deactivating — any divergence. This makes the chain the single source of truth and the hub a strictly derived mirror.

### 2.4 The Three Software Pieces

WeVibe's architecture has three software pieces: **wevibe-chain**, **wevibe-hub**, and the **MCP server + plugin**.

**wevibe-chain (source of truth):**
Cosmos SDK + CometBFT sovereign L1 appchain. Stores encrypted memory blobs, provenance/attribution, org state, serve receipts, and economic state. Validators maintain consensus and replicate state. They never see plaintext memory content.

**wevibe-hub (`wevibe-server`, coordination + retrieval plane):**
Runs coordination and accounting workflows, and serves the Qdrant-backed retrieval path. The hub is deliberately disposable: everything it holds is derived from chain state plus member-held keys, and it can be rebuilt or replaced without loss. It is never a plaintext memory oracle.

**MCP server + plugin (local safety + approval + injection):**
Platform-specific plugin gates register tools in the agent and call a local MCP server. This local path enforces guard/sanitization, presents the human approval UX, and injects approved context into the agent. It mediates access to hub retrieval and chain serve receipts.

### 2.5 Tool Surface

The plugin registers operational tools in the coding agent. Recall query dispatch is automatic in the plugin path (no agent recall tool call), and contribution submission happens in the dashboard flow (no agent contribution tool call). The separation:

**Plugin-registered tools (visible to the agent):**

| Tool | Purpose |
|------|---------|
| `setup_org` | First-run organization bootstrap when the local stack has no org membership. Guides org creation/join flow and local setup handoff. |
| `wevibe_status` | Show org membership and runtime status so the user can verify local setup and connectivity. |
| Consumer-settings tools | Configure the 2×2 consumer matrix: content filter `[Implementations + DNDs]` or `[DNDs only]`, plus injection gate `[Gated approval]` or `[No gated approval]`. Default is `[Implementations + DNDs] + [Gated approval]`. |

**MCP server backend (not directly callable by the agent):**

The MCP server handles local recall mechanics: query construction from local session signals, local embedding/decryption, guard scanning, and packaging candidate data for the plugin gate UX. It does not auto-submit contributions; contribution submission is explicit in the dashboard `/sessions` flow.

All administrative operations (org creation, member invitation, moderation, epoch rotation, keyword management, recovery) are handled by the `wevibe-admin` CLI — shipped as a command within the `wevibe-mcp` package — and hub control-plane workflows.

### 2.6 Product Handbook Map

WeVibe is a family of interoperable services. This general whitepaper captures the shared threat model, encryption design, and contributor experience. Deep dives live alongside the source for each product:

- `wevibe-chain/docs/WHITEPAPER.md` — consensus layer economics, lifecycle pressure, and keeper architecture.
- `wevibe-docs/WHITEPAPER.md` — client stack (SDK, MCP server, guard, protocol assets) and plugin UX.
- `wevibe-server/**/docs/WHITEPAPER.md` — operational surfaces (Hub, Dashboard, Infra) and their deployment models.

Each directory also contains accompanying PDP and topology documents. Use this handbook for cross-cutting concerns; jump to the per-product sets for implementation specifics.

---

## 3. Cryptographic Architecture

### 3.1 Threat Model

The system protects against four adversary classes:

**Chain validators** who store encrypted blobs, process transactions, and replicate state. They must not be able to read memory content. They see ciphertext, org IDs, contributor pub keys, and serve metadata.

**Unauthorized external observers** who can read chain state (the chain is public) but do not hold epoch keys. They must not be able to decrypt content.

**Removed members** who held epoch keys during their membership. They must not be able to decrypt content created after their removal (forward secrecy).

**Memory poisoning via recalled content.** A malicious org member submits a memory containing an indirect prompt injection that passes human review. The plugin gate is the final defense — the human sees every memory before it enters context, including the contributor's reputation score and wallet age as trust signals.

The system does NOT protect against: a compromised active member who leaks epoch keys or decrypted content; metadata inference from on-chain patterns (org sizes, submission frequency, serve patterns); compromised reviewer endpoints; semantic payload encoding in natural language prose.

### 3.2 Epoch-Based Key Hierarchy

Each organization has a master key K_master generated by the leader at org creation. Per-epoch keys are derived via HKDF:

```
K_enc(e)    = HKDF-SHA256(K_master, info="wevibe-enc-" || epoch_be_bytes)
K_audit(e)  = HKDF-SHA256(K_master, info="wevibe-audit-" || epoch_be_bytes)
epoch_sk(e) = HKDF-SHA256(K_master, info="wevibe-umbral-epoch-" || epoch_decimal_ascii)[:32]  → Umbral secret key
```

**K_enc(e)** wraps Data Encryption Keys (DEKs) for approved memories in epoch e.
**K_audit(e)** is reserved for audit logging and receipt verification.
**epoch_sk(e)** is the org's Umbral (PRE) epoch secret key — derived deterministically (distinct `info` namespace) by the **leader, on the leader's own machine**. The leader mints all re-encryption key fragments (kfrags) locally and sends the hub **only** the epoch PUBLIC key (`umbral_pk`) and the finished kfrags; the hub never receives `epoch_sk`. Because `epoch_sk(e)` is re-derivable from K_master on demand, it is never split, distributed, or persisted — backing up K_master (BIP39 recovery phrase) is sufficient.

The 32 HKDF output bytes **ARE the secret key**: they are interpreted directly as a **canonical secp256k1 scalar** (`SecretKey::try_from_be_bytes`), so `umbral_pk = epoch_sk · G`. This matches each member's PRE/receiving key, which is likewise a raw k256 scalar produced by `@noble/secp256k1`, so the two languages on the two sides of the PRE handshake (TS `@noble` ↔ Rust `umbral-pre`) agree byte-for-byte. The derivation is guarded by a permanent noble↔umbral parity test vector; seed-expansion derivations (`SecretKeyFactory::from_secure_randomness`) are prohibited for any key material because they silently produce a non-matching public key.

### 3.3 Memory Encryption and Moderation Keys

Each memory is encrypted with a unique DEK (32 random bytes):

```
DEK = random(32)
ciphertext = AES-256-GCM(DEK, nonce, plaintext_memory)
wrapped_dek_mod = seal_to_pubkey(DEK, PK_mod(e))
```

SK_mod(e)/PK_mod(e) is a fresh random X25519 keypair — NOT derived from K_master — preserving trust boundary separation between "general member" and "reviewer."

Upon approval, the reviewer re-wraps the DEK under the epoch encryption key:

```
wrapped_dek_enc = AES-256-GCM(K_enc(e), nonce, DEK)
```

The approved memory (ciphertext + wrapped_dek_enc + metadata) is then submitted on-chain by the org.

### 3.4 Key Distribution: Sealed Envelopes

The `seal_to_pubkey` operation:
1. Generate ephemeral X25519 keypair
2. ECDH between ephemeral private key and recipient's X25519 public key
3. Derive symmetric key via HKDF (info: `"wevibe-envelope-v1"`)
4. Encrypt with AES-256-GCM
5. Output: `ephemeral_pubkey (32 bytes) || nonce (12 bytes) || ciphertext+tag`

**Custody model invariant.** Each organization's K_master is generated as an independent random value.

**Recovery.** BIP39 24-word recovery phrase. Threshold recovery via Shamir 2-of-3. Encrypted leader vault (`~/.wevibe/vault.enc`) with Argon2id key derivation (t=3, m=64MB, p=4).

### 3.5 Retrieval Architecture

Retrieval is hub-served, with all plaintext handling kept local in the MCP server + plugin path. The hub-side retrieval plane (Qdrant index + ranking) is derived data, rebuildable from chain state plus org keys. The pipeline:

#### Context Profiling (Session Start)

When a coding session starts, the MCP/plugin profiles the environment — dependencies, directory structure, language, framework versions, current file context — and sends that profile as filter context so the hub can pre-filter candidate memories before vector scoring. A developer working in a Python/Django project searches Python/Django memories, not the entire org corpus.

#### Keyword Extraction

Keywords are generated **at the contributor's extraction pass** — in the same client-side call that produces the memory's `implement`/`context`/`dnd`/`stack`, using the contributor's selected extraction model. Extraction fetches the org's current controlled vocabulary and classifies against it: terms already in the vocabulary are **in-vocab (green)**; proposed new terms are **suggested-new (red)**. Each memory gets up to 20 keywords with weights normalized to sum to 1.0; the hub enforces the cap and the sum. The **leader curates at the batch stage** — deleting unwanted keywords (remaining weights re-normalize), and explicitly approving any red term before it commits and joins the vocabulary. This locates the vocabulary-drift gate at "who commits" (the leader), while keyword generation runs with the richest context (the full session) and the contributor's chosen model. Keyword weights are stored as retrieval metadata used during ranking as a **bonus, not a gate**; plaintext keywords remain an accepted, intentional metadata tradeoff (§3.7).

#### Retrieval Representation (situation-centric card; symmetric embedding)

The retrieval vector is **not** an embedding of a keyword bag. Both sides embed a **situation-centric card** with matched `nomic-embed-text` task prefixes: the memory side embeds a **retrieval card** (a deterministic template derived from the memory's `implement`/`context`/`stack`/`dnd`) with the `search_document:` prefix, computed **at moderator approval** (client-side, where plaintext is already decrypted for review — the hub never sees plaintext and there are no contributor-supplied vectors; the vector travels with the approval request, then is indexed into Qdrant at chain commit). The query side embeds a **deterministic session need-card** — harvested from intent, task, language, stack, frameworks, dependencies, error strings, and touched files — with the `search_query:` prefix. The keyword-overlap term is a capped additive boost, never a hard gate.

#### Semantic Embedding

At recall time, the MCP/plugin computes the query embedding locally via Ollama (`nomic-embed-text`) and posts that vector to `wevibe-hub` (`/v1/orgs/{org}/query`). The hub's Qdrant index stores plaintext float32 memory embeddings plus keyword metadata (`cid`, `org`, `keyword_weights`, `lifecycle`, `type`). Qdrant stores no decrypted plaintext memory content and no ciphertext blobs.

#### Atomic Memory Format

Each memory is a single, self-contained technical insight:

- **implement** — the fix itself: specific technical knowledge in 1–2 sentences with exact values
- **context** — environment, versions, conditions where this applies
- **dnd** — negative knowledge: what NOT to do and why
- **stack** — specific technologies involved

#### Retrieval Scoring

```
keyword_boost = Σ(query_weight_i × memory_weight_i)
final_score   = vector_score + γ × keyword_boost

Default: γ = 0.1
```

Vector similarity drives recall (hub-side Qdrant cosine over a configurable recall depth, default 5000). Keywords provide an additive boost (factor γ) to break ties within semantic clusters after context pre-filtering, with a δ-proportional cap bounding the boost's influence.

**Blacklist filtering.** The MCP/plugin excludes locally blacklisted CIDs before approval; a chain-level quarantine flag marks memories with repeated upheld rejections for retrieval-policy exclusion.

#### Candidate Fetch + Local Decryption

Hub retrieval returns memory IDs + metadata + matched keywords. The MCP/plugin then fetches each memory's ciphertext (the on-chain encrypted blob is the source of truth; the hub serves it from a cache plus the Umbral PRE materials), decrypts locally through the Umbral sidecar, runs wevibe-guard + the human gate, and only then injects approved context.

#### Selective Re-ranking

When top-2 scores are within ε=0.20 (contested query), the MCP/plugin **may** re-rank via a separately-configured rerank endpoint (the consumer's own model, reached through the `LlmProvider` abstraction — recall runs in a separate process and does not share the host agent's LLM). The rerank is opportunistic and non-blocking; it is never a hard requirement. Fallback: deterministic twin-suppression, and original order preserved on error.

#### Validation (offline simulation)

The situation-centric retrieval direction was validated offline before product implementation. An offline harness mirroring the extract→embed→rank pipeline (same `nomic-embed-text` model, production ranking logic) measured, on a model-generated synthetic corpus: embedding a situation-centric **retrieval card** instead of a keyword bag raised Recall@1 from **0.38 to 0.95**; the full pipeline — retrieval card + session need-card + boost-not-gate keywords — reached Recall@1 **0.97** / Recall@5 **1.00**. A complementary behavioral test showed that injecting the retrieved memory raised a weaker model's task-correctness by **~0.7 on a 0–3 scale**, capturing ~94% of the perfect-retrieval ceiling, with the pipeline retrieving the correct memory ~96% of the time. These are offline results on a synthetic corpus extracted by a frontier model; they validate the architecture's direction, and production telemetry is the standing measure of deployed performance (the validated figures reflect frontier-extraction card quality).

### 3.6 Side Channel: On-Chain Metadata

With memories stored on-chain and retrieval served by the hub, metadata is observable across two hosted surfaces. On-chain/public observers can see org IDs, contributor pub keys, submission timestamps, memory sizes, keyword terms/weights (plaintext), serve receipt patterns, and reputation scores. In the hub, Qdrant stores embedding vectors plus keyword metadata (`cid`, `org`, `keyword_weights`, `lifecycle`, `type`), while ciphertext is stored in Postgres/chain paths for retrieval.

The privacy boundary is decrypted plaintext: decryption, wevibe-guard sanitization, human approval, and context injection happen locally in the MCP/plugin path. The honest claim is two-part: (1) the hub **cannot decrypt** memory content — plaintext exists only locally (contributor at extraction, reviewer/leader at curation, consumer after the gate); (2) the hub **does hold content-derived data** — clean float32 embeddings (stored-vector noise is disabled) plus plaintext keyword weights, which together are a **lossy semantic shadow** of each memory. Published embedding-inversion research (Morris et al. 2023; Huang et al. ACL 2024) shows approximate content recovery from clean embeddings is realistic. The mitigations are **operational, not cryptographic**: Qdrant API auth, internal-network deployment, per-org collection isolation, signed responses. Encrypted vector search (e.g. IronCore `ironcore-alloy`) is the documented evaluation trigger — formally evaluated when an org requests confidentiality-sensitive hosting or public-testnet launch planning begins, whichever comes first. So: the hub never sees your decrypted memory content, but it is not true that nothing content-derived leaves your machine. The same two-part honesty applies upstream: the extraction pipeline is client-side and nothing reaches the org/hub before Submit, but a contributor-selected cloud extraction model receives the session transcript at Extract time — a visible choice scoped by trust-circle parity.

### 3.7 Metadata Visibility Model

WeVibe orgs are public developer communities, not private enterprises. On-chain metadata is intentionally public — it enables discovery, reputation, and the social graph.

**On-chain (public by design):** Org IDs, org topic tags, contributor pub keys, encrypted memory blobs, plaintext keyword terms/weights (discovery signal — "this org covers Redis"), an opaque per-memory project fingerprint and its scope (`project` | `org`), submission timestamps, memory sizes, epoch boundaries, serve receipts (batched per epoch), reputation aggregates, bandwidth consumption, quarantine state, report flags.

**Local to the MCP/plugin (the hub never sees these):** Decrypted memory plaintext, local wevibe-guard/blacklist state, and session context profiles. (Embedding vectors and keyword-weight metadata live in the hub's Qdrant; the hub stores ciphertext + vectors but never decrypts — see §3.6 and §8.3.)

Plaintext keywords on-chain are a feature, not a leak. They tell developers what an org covers and help with cross-org discovery. Developers who join an org to boost their LLM need to know what domain knowledge it offers. The keywords serve that purpose.

The project fingerprint on-chain is likewise a discovery/scoping signal, not a content leak. It is an opaque hash that reveals cluster membership only — which memories belong to the same working project — exactly the statistical metadata inference §3.1/§9.4 already concede is observable. Publishing it openly is what lets recall be scoped to the working project AND lets the hub's retrieval index be verifiably rebuilt from chain authority (§8.3). The fingerprint never carries personal machine identity: for a repo with a remote it is the SHA-256 of the normalized public git-origin URL; for a local-only repo it is a machine-independent identifier — never a home directory or username (INV-12 scrub).

### 3.8 Defense-in-Depth: Memory Sanitization Pipeline

WeVibe's security model focuses on what it can control: the form and content of recalled memory before it reaches the agent. Decryption, sanitization, approval, and injection run locally in the MCP/plugin path. Retrieval remains hub-served (Qdrant vectors + keyword metadata, ciphertext in Postgres/chain), but the hub never sees decrypted plaintext.

#### The Pipeline

**Submission time (before on-chain storage):**
1. **wevibe-guard scan.** YARA rules for injection patterns, credential detection, exfiltration matching, unicode mathematical injection detection. Advisory: warns but does not block. The human reviewer is the security boundary.
2. **OCR sanitization.** Text rendered to image via ImageMagick, OCR'd back via Tesseract. Destroys Unicode tricks, zero-width characters, homoglyphs, invisible formatting.
3. **Encryption.** Memory encrypted with per-memory DEK, DEK sealed to moderation public key.
4. **Human review.** Reviewer decrypts locally, reads plaintext, steganography scan, approve/deny.
5. **On-chain submission.** Approved memory (ciphertext + wrapped DEK + metadata) goes on-chain and is mirrored to hub storage for retrieval serving. Org pays the submission cost.

**Recall time (before delivery):**
6. **Hub candidate query.** MCP/plugin posts the local query vector to hub retrieval and receives memory IDs + metadata + matched keywords.
7. **Ciphertext fetch + local decryption.** MCP/plugin fetches each candidate's ciphertext and decrypts locally through the Umbral sidecar.
8. **Blacklist filter.** MCP/plugin checks the local blacklist and chain quarantine flags.
9. **wevibe-guard scan.** Same scan on decrypted memory at recall time. Catches payloads undetectable when approved (new rules since approval).
10. **OCR sanitization.** Same format-breaking pipeline.
11. **Artifact extraction and egress enforcement.** Typed artifact extraction: URLs, bare domains, IPv4 addresses, shell commands, package install commands, config directives. Every network-resolvable token flags.
12. **Plugin approval gate.** Plugin renders the approval UI with wevibe-guard detection results AND contributor trust signals (pub key, wallet age, rep score, serve count, domain expertise). The user sees the memory, sees the flags, sees who wrote it, and decides.
13. **Serve receipt.** Each serve/denial entry is signed by the consumer's per-org serve key (ed25519, offline — no wallet, no gas), and the batch transaction is carried by the org serving key under feegrant — the hub is a **carrier, not an attester**. The chain verifies each entry's signature, recomputes the dedup fingerprint itself from `(memory_cid, serve_key_pubkey, epoch)`, rejects unsigned/invalid entries, and counts only verified entries toward trust/attribution/emissions.
14. **Context injection.** Approved memories formatted as `context:\n{memory content}` and injected into the agent prompt.

#### What This Pipeline Catches
- YARA-signature prompt injections
- Credential leakage (AWS keys, API tokens, passwords, connection strings)
- Unicode steganography (homoglyphs; mathematical-alphanumeric injection, U+1D400-U+1D7FF, 3-char threshold; zero-width and directional-override characters via the OCR format-break)
- Base64-encoded injections
- External URL injection (scheme-ful and scheme-less)
- Bare hostname references (any TLD)
- IPv4 literal references (with optional port/path)
- Malicious dependency injection
- Config directive injection
- Shell pipe-to-execution attacks
- Previously-rejected memories (local blacklist + chain quarantine)

#### What This Pipeline Does NOT Catch
- Semantic payloads encoded in natural language prose. Mitigated by human review and contributor reputation signals.
- Technically-plausible but subtly wrong recommendations. Mitigated by reviewer domain expertise and contributor rep visibility.

### 3.9 Resolved Architectural Decisions

These decisions are final:

**Individual cross-org reputation is the product surface of provenance.** Serve counts, domain expertise, and rep scores accumulate across all orgs a contributor participates in. A developer's cross-org profile — "47 Redis memories served 214 times across 12 orgs" — is a public credential. No cross-org reputation *rankings/leaderboards* (avoids toxic competition), but aggregate stats are public and encouraged.

**Task-context skills, not difficulty tiers.** Skills organized by task (deployment, testing, error-handling), not by difficulty level.

**Model origin as soft retrieval prior (roadmap).** One soft factor in scoring — never a hard filter or gate: generic conceptual memories from lower-capability models are deprioritized for higher-capability retrievers; highly specific memories take no penalty regardless of origin.

**Documentation seeding for cold-start (roadmap).** New orgs import canonical documentation as seed memories (source: `doc_import`), with session contributions extending and replacing seeds over time (§11.2).

**Earned Trust decay.** Per-memory keyword weights evolve each chain epoch under the Earned Trust model: serves boost a memory's weight, denials decay it, and memories that are neither served nor denied idle-decay (trusted memories far more slowly than untrusted ones). A memory archives once every keyword weight falls below the retrieval threshold (default 1500 bps). Decay is epoch-driven, not version-scoped.

**On-chain storage, not hosted blob storage.** Approved memories are chain state, replicated by validators. No VPS dependency.

**Pending memories: commitment on-chain, blob off-chain.** Contributors submit only a commitment (hash, org ID, contributor pubkey, expiry epoch, size) on-chain. The encrypted blob is delivered to the reviewer through temporary off-chain channels (local transfer, P2P, or org-hosted mailbox). If approved, the finalized encrypted blob goes on-chain. If rejected or expired, the commitment is removed and the temporary blob is deleted. Rejected content never enters committed block data.

**Hub-based retrieval with local decryption.** wevibe-hub runs Qdrant vector search over embeddings + keyword metadata; the MCP/plugin computes query embeddings locally, receives IDs + metadata + matched keywords, fetches ciphertext, and decrypts locally before sanitization/injection.

**Contributors are paid by the network; members pay orgs for access.** Contributor rewards are contribution-only network emissions. Access demand is separate: members pay orgs in VIBE for recall access, and leaders earn from that demand leg through the non-custodial router (§10.5).

**Serve receipts: public reputation, pseudonymous retrieval.** Contributor reputation (serve counts, domain tags) is public on-chain. Retriever identity is represented by a per-org pseudonymous serve key — not the user's global contributor identity. This separates "my knowledge helped others" (public) from "this exact user needed this exact memory" (pseudonymous). Users can optionally link their org serve keys to their public profile as a learning trail. Each serve/denial entry is signed by the consumer's per-org serve key and the chain counts only verified entries; the serve key is self-asserted in the signed entry, preserving pseudonymity, and the hub — which holds no serve key — cannot mint serve content.

**Serve receipts are batched.** The chain accepts batched serve submissions (`MsgSubmitServeBatch`); the hub holds the org-whitelisted serving key for the transaction envelope only. Per-entry consumer signatures are verified on-chain — the chain rejects unsigned/invalid entries, so the serving key is never trusted for serve content.

**Four-button approval UX.** The plugin offers: [Accept] (memory injected, serve receipt queued, contributor attributed), [Deny] (memory blocked for this session/context only — a neutral context signal, NOT a corpus down-vote; no denial event emitted to the chain), [Block] (permanent personal blacklist AND a global corpus denial signal — the load-bearing negative-signal path that feeds Earned-Trust decay via `MsgSubmitDenialBatch`), [Report] (memory reported on-chain and escalated into the org's accountability path — §5.5–5.7). The Deny-vs-Block split is load-bearing: Deny is a no-op on the corpus (context ≠ quality); Block is the negative signal that drives decay.

### 3.10 Hub Confidentiality

Academic reviewers have challenged "the hub never sees plaintext" as a fake promise: if an authorized consumer can retrieve plaintext, surely a captured hub could too. This section states the guarantee precisely, proves it, and draws the honest boundary of what it does and does not cover. It formalizes part (1) of the two-part honest claim in §3.6 and is bound to `D-MISSION-INVARIANT` (guarantee #1 — no single party may unilaterally **READ** member plaintext) and `D-PRIVACY-BOUNDARY-REDRAW`.

**The claim, stated narrowly.** The hub **cannot decrypt** memory content. This holds against a **fully malicious hub** — not merely an honest-but-curious one — that deviates arbitrarily from the protocol. It is **not** claimed that the hub learns *nothing* about content; that stronger statement is false and is not made (see "The honest boundary" below).

#### The intuition the objection misses

Plaintext-visibility is conferred by **possessing a secret key**, not by **handling the data**. The consumer can read a memory because their device holds a secret scalar the hub never receives; the hub holds a **re-encryption (transformation) key**, not a decryption key. The analogy is a postal sorting office that re-addresses a sealed envelope so that a *different* recipient's key opens it — without ever being able to open it itself. Handling the envelope and reading it are different powers; the hub is granted only the first.

#### The confidentiality core (Umbral proxy re-encryption over secp256k1)

WeVibe's retrieval uses Umbral proxy re-encryption (`umbral-pre` over secp256k1; D-2.1/D-2.2/D-2.3). Umbral layers verifiability and threshold-splitting on top, but the confidentiality core is the ElGamal-style re-encryption relation below. Let `G` be the curve generator and `n` the curve order.

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

- **Encryption** seals the per-memory DEK under the org public key `A`; the sealed key is `K = KDF(a·(E+V))`, and recovering it the direct way requires the org secret `a = epoch_sk`. The hub never holds `a` — it is derived from `K_master` on the leader's own machine, and the hub receives only `umbral_pk` and the finished kfrags (§3.2, `D-LEADER-SIDE-UMBRAL-MINT`).
- **The kfrag** encodes `rk ≈ a·b⁻¹` and can be minted only by a party holding the delegating secret `a` — i.e. the leader. Recovering `a` or `b` from `rk` is the discrete-logarithm problem, and `rk` decrypts nothing on its own.
- **Re-encryption** is the hub's *only* cryptographic operation. Given the capsule and a leader-minted kfrag — and **no secret key** — it computes `cfrag = rk·(E+V) = (a·b⁻¹)·(E+V)`.
- **Decryption** happens only on the member's device, which multiplies the cfrag by its own secret scalar `b`: `b·(a·b⁻¹)·(E+V) = a·(E+V)`, recovering `K`, hence the DEK.

**The punchline.** The hub can compute `a·b⁻¹·(E+V)` but needs `a·(E+V)`. The one missing operation is a single multiplication by `b` — the member's secret scalar — which the hub is *structurally* never given. This is geometry, not policy: no configuration, key rotation, or privileged mode grants the hub `b`. Everything the hub holds — `{capsule, cfrag, ciphertext, kfrag, A, B}` — is computationally independent of the plaintext without `b`. This is exactly Umbral's IND-PRE-CCA security guarantee (Núñez et al. [15]).

#### Attacks a malicious hub might attempt

- **"The hub forges its own kfrag toward a key it controls."** Fails. Minting a kfrag requires the delegating secret `a = epoch_sk`, which the hub never holds. The hub can only *apply* leader-minted kfrags, and those are minted toward the registered public keys of authorized members — never toward a hub-controlled key.
- **"The hub colludes with an authorized member."** Conceded openly. An authorized member decrypts with their own secret `b` and could leak the resulting plaintext. This is authorized-insider abuse, not a cryptographic break — and it is already **out of scope** in the threat model (§3.1: "a compromised active member who leaks epoch keys or decrypted content"). The hub as a *standalone party* still cannot read; the confidentiality claim is about the hub itself, not about every party the hub might suborn.

#### The honest boundary (what this does NOT cover)

Two statements must not be conflated:

- ✅ **TRUE (and proven above):** the hub cannot **decrypt** your memory content.
- ❌ **FALSE (and never claimed):** the hub learns **nothing** about your content.

For search, the hub holds — in Qdrant — clean float32 embeddings plus plaintext keyword weights: a **disclosed, lossy, realistically-invertible semantic shadow** of each memory (§3.6, §3.7). Published embedding-inversion research (Morris et al. 2023 [13]; Huang et al. 2024 [14]) shows approximate content recovery from clean embeddings is realistic. The mitigations here are **operational, not cryptographic** — Qdrant API auth, internal-network deployment, per-org collection isolation, signed responses — with encrypted vector search as the documented evaluation trigger. This is the ratified position of `D-PRIVACY-BOUNDARY-REDRAW`: WeVibe makes **no** claim of a zero-knowledge index or a content-confidential hub. That abandoned claim is not resurrected here — "cannot decrypt" and "learns nothing" are different guarantees, and only the first is made.

Two further honesty notes:

- **The consumer-side injection gate is wired and enforced.** The local human approval gate that lets a user inspect each memory before it enters an agent's context is live in the plugin today: a blocking, fail-closed human-approval step (Accept/Deny/Block/Report — §1.6, §3.1) that injects **only** human-approved memories. Production is gated; only benchmark/test mode auto-approves. This is a *different* leg of the privacy tripod (`D-PRIVACY-BOUNDARY-REDRAW`) from the hub-cannot-decrypt guarantee proven above — and it is now shipped, not merely contract-defined.
- **Key locality.** The hub never receives `epoch_sk`: it is derived from `K_master` on the leader's own device, and only the epoch *public* key and finished kfrags cross the wire (§3.2, `D-LEADER-SIDE-UMBRAL-MINT`). The confidentiality proof rests on this locality holding cleanly.

### 3.11 Session Attestation (Roadmap, Post-Mainnet)

Sessions produce memories. Without provenance attestation, a contributor could paste fabricated "coding sessions" into the extraction pipeline and farm reputation. Everything downstream — difficulty scoring, quality grading, reputation — depends on knowing the session actually happened.

**Current posture:** org leader curation is the trust layer, and it is a real one — every published memory carries a leader's bonded signature. A generalized cryptographic attestation rail is post-mainnet roadmap.

**Roadmap direction:** a post-mainnet **pluggable attestation framework**. Separate attestation components plug into the chain and validate session claims either **cryptographically** or **via API-backed trust services**. The target claim shape is explicit session provenance: *"user X using LLM model Y took N turns to solve problem Z."*

**Proof-tier generalization:** post-mainnet, the on-chain `x/attestation` socket generalizes the provenance grades below into a single **typed proof artifact** — `{ proof_type, trust_label, … }` where `proof_type` is one of `tee_receipt`, `zktls_proof`, or `zkml_proof` — each **verified off-chain before any on-chain anchoring**. The zkTLS path is technically proven for closed frontier models with session privacy intact; it activates when MPC cost ceilings clear real-world extraction-prompt sizes.

#### Attestation tiers (seed designs)

**Tier 0: TEE-Attested Confidential Inference (open- or closed-weight).** GPU-TEE confidential inference (e.g. Intel TDX + NVIDIA H100 Confidential Computing) signs each request/response inside the enclave and ships a remote-attestation quote binding the signing key to the loaded **model measurement** — a hash of the actual checkpoint, not a marketing name. It is the strongest tier: it covers closed-weight models without trusting a WeVibe relay, and is recorded as the `tee-attested` provenance grade. Locked constraints: it is **optional, opt-in, and never a contribution gate** (requiring it would exclude the local/smaller-model users WeVibe most serves); verification is **off-chain** (an attestation-verifier checks the Intel DCAP + NVIDIA quotes and emits a re-verifiable assertion — the chain is never asked to verify a TEE quote); a `certified_model` tag certifies **session provenance, not memory text** (a `derivation` flag records whether the submitted memory is verbatim from extraction or edited after); and it is carried as a pluggable adapter, not a hard dependency.

**Tier 1: CommitLLM Cryptographic Receipts (Open-Weight Models).** CommitLLM (Lambda Class, MIT licensed) is a cryptographic commit-and-audit protocol for open-weight LLM inference. The receipt proves "this response was produced by this exact model with these exact weights." Limitation: only works for open-weight models where the verifier has the public checkpoint.

**Tier 2: Proxy-as-Trust-Layer (Closed-Weight Models).** For closed-weight API models, traffic can route through a WeVibe-controlled session proxy that content-addresses each turn and signs the transcript with WeVibe's key. Weaker than Tier 1, but a practical API-trust path.

**Standardizing the extraction model (the controllable half).** "Weaker models surface weaker memories" has two heads: the *production* model in the user's coding suite (uncontrolled — and sometimes the most valuable signal, when a weak model solves a hard problem) and the *extraction* model that distills a session into a memory (inside WeVibe's own pipeline). WeVibe standardizes the latter by **pinning the extraction model**, removing weak-extractor variance with no attestation required; attesting the production model is the optional bonus on top.

### 3.12 Two-Layer Difficulty Scoring (Roadmap Consumer, Requires Attestation)

Two-layer difficulty scoring is the evolutionary continuation of attestation and its likely first consumer once the pluggable framework exists. The chain carries `difficulty` and `quality` values (1–10) on attested-memory records and maintains a per-contributor difficulty histogram (`x/reputation`); the scoring layers below populate those values automatically once live.

How attested difficulty enhances the economic and/or social-graph layers is intentionally **TBD**. What attestation settles is the *trustworthiness of the inputs*: the model and the turn count become cryptographically grounded. The composite claim "user X using model Y took N turns to solve problem Z and drew L negative signals" is rendered as a **per-field graded provenance** object in the social-graph layer — each field labeled by its own grade (attested model/turns, native-on-chain user/denials, descriptive problem tag) — display/badge-only and never coupled to VIBE.

#### Layer 1: Structural Signal (Automated, Cheap)
Model capability coefficient × turn count × (1 + 0.25 × failed alternatives). Computed from session structure without understanding content.

#### Layer 2: LLM Grading (Semantic, Authoritative)
A separate grading LLM evaluates non-obviousness, specificity, and reasoning progression. Temperature 0, deterministic, hash-seeded. The grade and session hash are committed together.

---

## 4. Local Architecture (MCP Plugin + Sidecars)

### 4.1 Local Footprint and Responsibilities

The local software footprint on a member machine is:

- **MCP server + agent plugin**
- **Umbral decryption sidecar**
- **wevibe-guard binary**

The local path is responsible for both recall gating and contribution packaging.

**Recall-side responsibilities (local):**
1. Harvest session signals and build the deterministic need-card.
2. Compute the query embedding locally via Ollama (`nomic-embed-text`).
3. Send the query vector (plus context filters) to the hub endpoint (`/v1/orgs/{org}/query`).
4. Fetch candidate ciphertext (chain is the source of truth; the hub serves it cached, with Umbral PRE materials).
5. Decrypt locally through the Umbral sidecar.
6. Run wevibe-guard sanitization/policy checks.
7. Present the human approval gate and inject only approved context.

**Contribution-side responsibilities (local):**
1. Extract session learnings only when the contributor clicks dashboard `/sessions` → **"Extract Memories"** (client-side candidate generation, no submission yet).
2. Sanitize, encrypt, and sign submission material.
3. Submit commitment data to chain (org pays submission bandwidth), then follow the moderation/finalization flow.

Vector retrieval itself is hub-side: `wevibe-hub` runs Qdrant search and returns candidate IDs + metadata + matched keywords. The hub stores/serves ciphertext + vectors + keyword metadata, and does not decrypt memory plaintext.

### 4.2 Dependencies

The MCP/plugin stack requires:

- Ollama with `nomic-embed-text` for local query embedding
- Local Umbral sidecar for decryption
- wevibe-guard binary
- ImageMagick/Tesseract when guard policy enables OCR/steganography checks
- Connectivity to wevibe-hub APIs and chain RPC

This architecture does **not** require a local Qdrant deployment or a full local vector index.

### 4.3 Sync and Bootstrapping

Member setup is key/bootstrap first, not full-corpus indexing:

1. Join the org and configure the local MCP/plugin with org context and chain RPC (hub endpoints resolve from chain — §4.8).
2. Receive sealed key envelopes for the current epoch (reviewers/leaders additionally receive current moderation key material).
3. Initialize local services used by the plugin path (Ollama embedding model, Umbral sidecar, wevibe-guard).

After bootstrap, recall runs as request/response against the hub retrieval service. The local machine does not download the org's full memory corpus and build its own vector index. Local state is operational (keys/config + approval/serve-receipt queueing), while authoritative memory storage and vector retrieval remain hub/chain-side.

### 4.4 Recall Flow (Auto-Query + Plugin-Gated Injection)

```
Developer works in their coding session
     │
     ▼
  Plugin harvests local session signals
  (intent, task, stack, dependencies, errors, files)
     │
     ▼
  Plugin auto-submits recall query via local MCP
  (no agent recall tool call)
     │
     ▼
  Local MCP/plugin path:
      1. Build the deterministic session need-card + keyword params
      2. Compute query embedding locally via Ollama (`nomic-embed-text`)
      3. POST query to hub `/v1/orgs/{org}/query`
     │
     ▼
  Hub retrieval path:
      4. Qdrant vector + keyword-ranked search over hub-hosted embeddings
      5. Return encrypted candidates + ranking/trust metadata
     │
     ▼
  Local MCP/plugin path:
      6. Decrypt candidate ciphertext locally via Umbral sidecar
      7. Run wevibe-guard sanitization + policy checks
      8. Apply content filter: [Implementations + DNDs] | [DNDs only]
      9. Render candidate details before injection:
         - memory text, memory_type, score, matched keywords
         - wevibe-guard result
         - contributor stats: account age, contributions, serve count,
           reports upheld, false reports
         - memory stats: retrieval count, acceptance count
         - report indicators: is_reported / was_reported / report_cleared
         - trust panel
     │
      ├── [Gated approval] (locked default): explicit user action
      │      on every candidate
      │      ├── ACCEPT → inject context + attest serve on-chain
      │      ├── DENY   → block for this session/context (no corpus signal)
      │      ├── BLOCK  → personal blacklist + corpus denial signal
      │      └── REPORT → file on-chain report (is_reported);
      │                   memory keeps serving until resolved
      │
      ├── [Gated on risk]: auto-inject + attest
      │      the safe majority; interrupt ONLY for guard-flagged /
      │      low-trust / low-confidence candidates. Quality signal
      │      (attest/deny) batched in a review tray.
      │
      └── [No gated approval] (explicit opt-out): inject directly
             + attest serve on-chain
      │
      ▼
  Agent continues with or without memory
```

### 4.5 Contribution Flow

```
Developer opens dashboard `/sessions` and selects a captured local session
     │
     ▼
  Click **"Extract Memories"**
     (client-side extraction with the contributor's selected model —
      local or session-matched cloud;
      returns review candidates — does NOT submit)
     │
     ▼
  Review/edit candidates and choose per-memory org destination
     │
     ▼
  Click **"Submit N"** (explicit off-machine send)
     │
     ▼
  Local contribution pipeline:
    1. Run wevibe-guard scan (advisory) and sanitization steps
    2. Generate fresh DEK, encrypt plaintext, seal to PK_mod(e)
    3. Contributor signs submission hash/canonical body
    4. Submit COMMITMENT to chain (hash, org_id, contributor_pubkey, expiry, size)
       Org pays bandwidth for the commitment transaction.
    5. Deliver encrypted blob to reviewer via temporary off-chain path
       (local transfer, P2P, or org-hosted mailbox)
    6. Return: "WeVibe: submitted N learning(s). Pending review."
```

**Pending memory lifecycle:** Only the commitment is written on-chain initially. The encrypted blob is reviewed out-of-band; approval finalizes chain state, while rejection or expiry deletes the pending commitment and temporary blob. Sessions stay local unless and until the contributor explicitly submits.

### 4.6 Reviewer Flow

Moderation and review are handled in the hub's hosted web dashboard (`wevibe-dashboard`). Reviewers and leaders process pending submissions there, apply approve/deny decisions, and push finalization actions through the hub-to-chain path.

### 4.7 Plugin Architecture

Each coding agent gets its own plugin codebase:

| Agent | Plugin Type | Hook Mechanism |
|-------|-------------|----------------|
| OpenCode | JS/TS plugin in `.opencode/plugin/` | `tool.execute.before`, custom tools via `tool()` |
| Claude Code | Plugin with `.claude-plugin/plugin.json` manifest | PreToolUse hook, `permissionDecision: "deny"` |
| Cursor | Hooks + marketplace plugin | Claude Code hook format compatibility |
| Cline | VS Code extension + hooks | `.clinerules/hooks/`, custom hook system |

All plugins call the same local MCP server and the same wevibe-guard binary (`WEVIBE_GUARD_BIN`). Decryption is handled through the local Umbral sidecar; retrieval/search remains in hub APIs. The OpenCode plugin (`wevibe-opencode-plugin`) is the reference implementation; the Claude Code, Cursor, and Cline integrations follow the same local MCP + guard contract.

OpenCode 1.16+ ships a built first-run onboarding hook: `DialogConfirm` → Touch ID/passkey confirmation → completion screen. Identity creation is step one only; reputation comes from joining orgs and contributing approved memories, never from identity creation itself.

The OpenCode plugin exposes `/wevibe-setup`, `/wevibe-connect`, and `/wevibe-status`, and nudges the user to the dashboard join/connect flow after three local sessions. Identity boot is lazy (no biometric prompt at plugin startup). Non-secret local identity state is plugin-owned and stored at `~/.wevibe/identity.json`.

Install/uninstall is zero-config via `wevibe-admin install-opencode` and `wevibe-admin uninstall-opencode`. The local keystore is seed-based with BIP39 recovery, and short-code pairing reconciles dashboard + plugin onto one contributor pubkey.

### 4.8 Chain-Resolved Hub Endpoints (Zero-Config Transport)

WeVibe keeps one canonical chain as the network source of truth, with a public chain RPC as the single stable client anchor (default endpoint, env-overridable for operators). The chain is the org directory. Each org carries a leader-configurable `hub_endpoints` field — an ordered list of 1–3 network URLs for transport redundancy/failover — set via a leader-signed on-chain setter, distinct from `hub_serving_address`, the Cosmos key that authorizes serve/deny and response authority. Transport and authorization are intentionally separated.

At session start, the plugin resolves per-org endpoint routing from chain RPC once (biometric-free, because org metadata is public), updates local config, and persists per-org sidecar state keyed by globally unique `org_id` so state remains stable across hub migration. The endpoint list is priority-ordered: the plugin uses the first and fails over to the next on connection, health, or signature-verification failure. All endpoints are replicas fronting the org's single on-chain serving identity. The consumer never hand-configures a hub URL.

Hub transport is untrusted: hub responses are signed by the org's on-chain serving key and verified by the plugin against `hub_serving_address`. The response-signing contract lives in `wevibe-protocol` so hosted and self-hosted hubs conform to one verification path. If the `hub_endpoints` list changes, clients auto-switch silently and show a one-time passive toast.

Why this path: self-hosting remains a first-class leader right, but endpoint operations stay abstracted from end users. Resolving transport from the same chain source of truth preserves one path and avoids creating a second directory service every self-hosted hub would otherwise need to mirror and keep in sync.

---
## 5. Content Review and Accountability

### 5.1 Review Flow

Contributed memories are submitted (encrypted; only the org's reviewers can decrypt) into the hub's pre-commit pipeline and are visible only to reviewers and leaders via the hub's hosted dashboard (wevibe-dashboard). Pre-commit memories live in hub PostgreSQL — never on chain and never in Qdrant — until the leader commits a batch on-chain.

This review layer is not just about safety. It is the quality gate that protects WeVibe's provenance layer: public contributor/org attribution is only meaningful if low-quality or malicious memories are filtered before commitment.

**Moderation is always-on advisory; the leader is the always-on backstop.** A submitted memory flows directly into the leader's curation/chain-submit queue — there is no mandatory "awaiting moderation" gate and no on/off toggle. `can_moderate` is an independent per-member **capability** the leader grants à la carte (via `MsgSetMemberCapabilities`) — *not a role*; a leader can run an org with **zero members granted `can_moderate`** (capabilities are off by default). Members holding `can_moderate` are *helpers* who produce **recommendations, never gates**:

- a **per-submission vote** — `approve` or **flag (against committing)** — surfaced to the leader as a tally on the memory's card;
- a **per-keyword vote** — `include` / `exclude`, primarily on the red (suggested-new) vocabulary terms — surfaced next to each keyword to guide the leader's keep/remove decision.

Neither vote changes the pipeline state. The **leader** reads the recommendations and decides: curate the keywords (§3.5), commit the batch (signing on-chain — the sole publishing authority), or **terminally deny** the memory. Advisory vote history is org-local accountability, not chain-level enforcement. This delegates review *labor* without ever delegating the leader's *signature* — the property the entire anti-capture model rests on (§9.7).

The pre-commit lifecycle: a contributor submits → the memory enters the leader's curation queue (`pending_keyword`) with keywords already attached (§3.5) → the leader curates keywords + verifies (`pending_chain`) → the leader signs the multi-message batch commit (`committed`). A leader denial is the only terminal removal before commit. Advisory votes ride alongside as metadata at any point before commit.

### 5.2 What Review Can and Cannot Catch

**Can catch:** Prompt injection patterns, malicious URLs, credential exfiltration, spam, duplicates, off-topic content, obvious technical errors, stale references, Unicode steganography, memories too generic for the org's target model/stack.

**Cannot catch:** Subtly incorrect technical guidance that appears plausible, semantically-encoded malicious instructions in natural language prose, context-dependent correctness.

### 5.3 Reviewer Trust Boundary

Reviewers see pending content in plaintext on their local machine. The system is only as secure as reviewer judgment, honesty, and endpoint security. Mitigations: epoch-scoped mod keys, steganography detection, contributor reputation signals visible during review.

The leader is the final authority on what enters the chain. Chain commits — batch memory approvals, denial settlements, report resolutions, dispute publications — are signed by the leader's wallet and the leader's wallet alone. The leader bears full responsibility for the on-chain record they produce. Internal advisory votes and approval histories are an org-local accountability primitive maintained outside the chain.

### 5.4 Contributor-Signed Canonical Body as Verification Anchor

Every memory's submit-time canonical body includes three fields that, together with the contributor's signature over the body, form the public-escalation verification anchor:

- **plaintext_hash** — `sha256(salt || plaintext)`, computed by the contributor before encryption
- **salt** — a fresh 32-byte random value generated per submission
- **ciphertext_hash** — `sha256(ciphertext)`, where ciphertext is the AEAD output

The contributor signs the canonical body with their own key. The canonical body, the signature, and the ciphertext all travel through moderation and land on the chain together. The leader's batch chain commit includes the contributor's signature; the leader cannot modify the signed fields without invalidating the contributor's signature.

This is what makes the public report tier (§5.5) trustworthy without trusting the leader: any future reveal of plaintext + salt can be verified against the on-chain plaintext_hash via direct sha256 check, and the on-chain ciphertext can be verified against the on-chain ciphertext_hash. The leader is removed from the verification chain entirely. A captured leader cannot poison the anchor because they do not sign it.

The contributor cannot substitute ciphertext between submit and chain commit because their signature binds the specific ciphertext_hash. The contributor cannot later claim a different plaintext at public escalation because the plaintext_hash binds them to the specific salted hash, and SHA-256 collision resistance prevents producing a different (plaintext, salt) pair that hashes to the same value.

**Why the signature covers all three fields jointly.** A signature over `plaintext_hash` alone — without salt and without ciphertext binding — is vulnerable to contributor + leader collusion: the contributor signs one hash while the leader commits ciphertext encrypting different content, and the asymmetry is undetectable. Binding all three fields in a single signature closes the gap. The leader has no signing role in the anchor; the contributor has no opportunity to substitute ciphertext after signing.

**Residual risk: contributor with stolen signing key.** If the contributor's signing key is stolen, an attacker can produce signatures binding any (plaintext, salt, ciphertext) tuple. This is a key-management problem, not a cryptographic-anchor problem — no design at the cryptographic layer can defeat it. Mitigation lives at the wallet/identity layer (the future BIP-32 PRE-identity separation). The on-chain ciphertext + sealed-box wrapped DEK remain as a final backstop: any future epoch key disclosure allows independent decryption and exposes the actual content after the fact.

**Salt design rationale.** Without a salt, `sha256(plaintext)` is vulnerable to rainbow-table attacks for low-entropy plaintexts (short memories, common error messages, well-known technical advice). A 32-byte random salt makes rainbow tables infeasible (2^256 salt space per plaintext). The salt is included in the canonical body the contributor signs, stored on-chain alongside the commitment, and revealed by the reporter at public-escalation time. Salt is not secret — it is context-hiding.

### 5.5 The Report Model

A report is the high-friction accountability primitive (denials are the low-friction ranking signal — §5.8). It has two stages: an on-chain report with a fixed org-resolution window, and — only if the org fails to resolve it — a public plaintext-reveal escalation. Most reports never reach the second stage; it exists precisely for the case where the org is captured.

**Stage 1 — On-chain report.** The consumer files the report from the plugin. Filing is **wallet-gated**: the reporter signs the report transaction from their own wallet and pays gas, and the reporter's **public wallet is recorded on-chain**. Filing sets on-chain flags on the memory: **`is_reported`** (an open report stands), **`was_reported`** (reported at least once — permanent, never unset), and later **`report_cleared`** (set only if the leader dismisses — see Stage 2). Filing is rate-limited on-chain to **one report per reporter per 24 hours** (block-height scoped), so one consumer cannot spam reports, and it starts a **one-week resolution window**.

Because the reporter is wallet-gated and gas-paying, the client broadcasts the report (and the later expose) **directly to chain RPC** — the canonical public RPC, env-overridable — with a WeVibe-operated relay as retry fallback only. No user-signed accountability transaction requires WeVibe infrastructure to reach the chain, so neither a captured org nor WeVibe itself can silently suppress the filing.

The memory **keeps serving** during the window; removal happens only on resolution, so a wave of frivolous reports cannot disrupt an honest org.

**Stage 2 — Resolution, or public escalation.** Resolution is decided off-chain by the org — by the **leader directly** (advisory input from members holding `can_moderate` may inform the decision, but the leader is the sole authority):

- **Valid →** the leader **claws back the memory's storage deposit** and **deletes the ciphertext blob**. A minimal **on-chain transparency record** is kept — that this memory was reported and the report was upheld, attributed to the contributor. The plaintext is *not* revealed; the deleted blob plus the upheld record are the outcome.
- **Dismissed →** the leader clears the open report with a `clear_report` transaction (`report_cleared = true`, `is_reported = false`; `was_reported` stays true).

If the org does neither and **the one-week window elapses with no leader action**, **or the leader dismisses** (`report_cleared`), the original reporter's **public report** unlocks. The reporter publishes a public on-chain report that **reveals the memory plaintext**, anchored to the contributor-signed hash (§5.4) so anyone can verify it independently; the block explorer renders it with full provenance — contributor pubkey, leader pubkey, org ID, original commit height, and the reporter's signed reason. Because all chain data is public, the plaintext reveal is irreversible and is therefore the **last** step — only after the org has had its governed window. One escalation per report; no re-publishing, no escalation loops. This recourse is what makes capture economically unsustainable: a captured org cannot both refuse to act and prevent the public record.

### 5.6 The Expose Gate, Dispute, and Silent Acquiescence

The ordering above is deliberate and gated:

1. **File (no reveal).** The on-chain report names the memory and reason, sets `is_reported`/`was_reported`, and starts the one-week window. The plaintext is not revealed. The filing is a permanent, unsuppressable record.
2. **Org's window to act.** The org upholds (clawback + blob deletion + transparency record) or dismisses (`clear_report`). A genuine uphold needs no public exposure; a dismissal does not protect the org — it unlocks the public report.
3. **Expose (gated, final).** If the window elapses with no action, or the report is dismissed, the reporter's public plaintext reveal unlocks. Revealing is last because it is irreversible.
4. **Dispute.** Once a memory is publicly exposed, the leader may publish exactly one leader-signed counter-statement on-chain, permanent and public at the same block-explorer URL. One response, no revisions — the single-reply rule forces a deliberate, final answer.

**Silent acquiescence.** A leader who neither upholds nor dismisses within the window has implicitly accepted the claim — silence in the face of a public claim is tacit agreement. The unresolved report unlocks the public reveal and the memory's `is_reported` flag stands.

**The memory keeps serving during the window.** A report does not immediately remove the memory from retrieval — removal happens only on an upheld resolution (deposit clawback + blob deletion). The cost of filing falls on the reporter (gas); the cost of resolving falls on the leader; the memory keeps serving until one of those costs is paid. This prevents a coordinated wave of frivolous reports from disrupting an honest org.

### 5.7 No Internal Courtroom

WeVibe's dashboard is not the courtroom. The chain is.

WeVibe's own surfaces — the dashboard, discovery pages, reputation profiles — never display report counts, report aggregates, per-org report statistics, or per-actor report metrics. The reasoning is structural:

- Any in-app aggregation of reports is gameable via sybil reporters. A coordinated set of attackers could inflate report counts on an honest org, weaponizing the platform against legitimate operators.
- Any in-app aggregation is also censorable. Whoever controls the dashboard surface controls which reports get amplified and which get suppressed.
- Any in-app aggregation creates the perception that WeVibe-the-protocol or WeVibe-the-team is the tribunal. There is no tribunal. There is no judging body. There is only publication of unforgeable evidence on an immutable chain that neither the org nor WeVibe controls.

The chain is the publication mechanism. The block explorer is the viewer. WeVibe provides the reporter a permanent on-chain transaction; the reporter shares the block-explorer URL wherever they choose — a social network, a competitor's community, a blog post, a Discord channel. The court of public opinion happens outside the system. WeVibe gets out of the way.

The reporter's own dashboard view is the one exception: each reporter sees a private list of their own published reports with copy-link buttons (§7.5). That is ammunition for the reporter to share, nothing more. The leader receives a notification when a report is filed against a memory they committed, plus a response interface labeled with the response-window timeout and the leader's one-shot dispute — again, visible only to the leader.

### 5.8 Silent Denial as Cheap Negative Signal

The plugin's four-button approval UI (Accept / Deny / Block / Report) gives the consumer two complementary negative paths beyond the session-scoped Deny. Reports — the on-chain filing and, if unresolved, the public escalation — are the high-friction, high-stakes accountability primitive described above. Blocks emit the low-friction, low-stakes denial signal that feeds retrieval ranking.

Denials and reports are status/accountability signals, not direct payout triggers. WeVibe keeps the social signal and economic payout paths decoupled.

Clicking Block is silent: no confirmation modal, no required reason, no new UI surface. The reason field is optional. There is no gating — any consumer, trial or paid, may block any memory. There are no caps, no rate limits, no reputation weighting. Every denial event counts as exactly one denial event.

A denial does two things:

1. **Local suppression.** The memory is never re-served to this developer.
2. **A negative signal to the retrieval layer.** The denial flows to the org's retrieval/storage component, which mirrors the chain's decay arithmetic locally. Retrieval ranking degrades immediately — a memory denied N times since the last on-chain settlement ranks lower than its chain-recorded weight would imply.

**The optimistic ledger.** The chain remains the eventual source of truth for keyword weights and the decay state of every memory. Between on-chain settlements, the retrieval layer maintains an optimistic mirror: for each memory, the locally-applied decay reproduces the chain's Earned Trust formula — recomputing `denial_rate`, the trust gate, and the per-event decay/boost using the same parameters the chain will apply at settlement. The arithmetic mirrors the chain's exactly, so optimistic and authoritative ranking states are indistinguishable at retrieval time. When the relay's batch commits on-chain, the local mirror reconciles to the new authoritative weights and resumes from the new baseline.

**Why silent and frictionless.** A denial is the cheap, low-stakes negative signal. UI friction (required reason, confirmation modal, rate caps) suppresses the signal and starves the decay model that depends on it. Reports remain the high-friction path with reporter accountability for cases where a single signal needs disproportionate weight. The two paths are complementary, not duplicative.

**Why no caps or reputation weighting on denials.** A single consumer cannot drive a memory to archived via denials alone. Archive requires every keyword weight to fall below the retrieval threshold (default 1500 bps), which under Earned Trust requires either sustained denial pressure pushing the denial rate above the trust gate, OR no offsetting serves over a long quiet period — both of which require organic volume that one actor cannot fake. Caps would protect against an abuse that has no payoff: a malicious consumer who spams denials can at worst suppress memories from their own recall queue (which they could do anyway through the local blacklist). Reputation weighting would create a class system where senior consumers have heavier "votes" than new joiners, require online reputation lookups on the recall hot path, and reproduce the reporter-accountability infrastructure for a signal whose semantics are explicitly lighter.

**Hub-relay settlement.** Serve and denial events settle on-chain through the hub→chain relay, not a manual leader action. The relay packs multiple epochs' `MsgSubmitServeBatch` / `MsgSubmitDenialBatch` messages into one multi-message commit-mode transaction per org, carried by the org's whitelisted serving key with gas paid via the org-account feegrant. The optimistic ledger means retrieval ranking already reflects every denial in real time; the relay commit is purely the settlement act — making the decay permanent across restarts and hub migrations and contributing to the org's on-chain activity record.

The leader does not review individual denials. Denials are quantitative consumer signals, not editorial content; the relay settles accumulated decay, and the leader does not adjudicate individual user preferences.

---

## 6. Provenance, Reputation, and the Social Graph

### 6.1 Reputation Is Provenance Made Visible

Reputation on WeVibe is not a gamification layer bolted onto a memory tool. It is the provenance layer rendered human-readable.

Every claim a profile makes is backed by a chain event someone signed and paid for: a contribution a bonded leader committed, a serve a real consumer's key attested, a domain built from curated keyword taxonomies. No platform captures this problem-solving exhaust today — GitHub shows repository output, LinkedIn is self-reported, and most AI-assisted debugging knowledge disappears into terminal history. WeVibe turns that daily exhaust into a public, verifiable record: contribution history, serve attribution, and badge progression tied to real chain events.

The two loops reinforce each other: memory retrieval quality is the utility loop; attributed public history is the motivation loop. Both are downstream of the same primitive — provenance.

### 6.2 Serve Attribution Is a Social Signal (Not an Economic Primitive)

Serve attribution is kept on-chain and made public, but it is not part of VIBE payout mechanics.

When a memory is served, attribution increments aggregate counters for:

- the **contributor** whose memory was served, and
- the **org** where the serve happened.

Those counters are source-of-truth chain state, read through RPC, and rendered by the social graph.

This decoupling is deliberate. Retrieval-based rewards are excluded from economics because they are gameable (manufactured retrieval loops can fake demand). Serve attribution is kept anyway because it is valuable as public status/reputation evidence. The signal remains; the payout coupling is removed.

### 6.3 Identity and Attribution Model

Serve attribution uses per-org pseudonymous serve keys. Each user has:

- `global_contributor_key` — public identity for authorship and contributor profile reputation
- `org_serve_key` — per-org pseudonymous key used for retrieval/serve attribution events

`org_serve_key` proves org membership activity and supports deduplication rules without auto-linking all retrieval behavior across orgs. Users can opt in to publicly link selected org activity to a profile.

**Reputation is keyed by the passkey identity, not the wallet.** A contributor earns and accrues reputation under their passkey-derived contributor key from the moment they contribute — no wallet required. Earnings accrue on-chain to the same passkey identity (a claim-later balance), not to the wallet; a Cosmos wallet is an optional later upgrade for *withdrawal authority and leadership/bond*, not identity, and linking one does not by itself move reputation or unlock earnings. Carrying reputation onto a wallet is a deliberate, explicit **migration**, modeled as an **on-chain, dual-signed alias** (passkey pubkey → wallet address) that is gated by the contributor's own memory-contribution trail and recorded with an `is_migrated` flag. Until migration, reputation stays keyed to (and is resolved by) the passkey pubkey; after migration it resolves to the wallet via the alias. The chain is append-only, so no prior history is rewritten.

### 6.4 Open-Source Social Graph Client (Forkable, Self-Hostable)

The social graph is an open-source display client over chain RPC. Anyone can fork and self-host it.

Its role is presentation, not consensus. Chain state remains the source of truth; the social graph reads and renders:

- contributor serve/contribution counters
- org-level aggregate serve/contribution counters
- reputation summaries and domain views
- badge status and per-org badge breakdown

Because the client is forkable, WeVibe maintains canonical badge criteria in the reference display spec (§6.5, §7.2) so tier names stay comparable across forks. This is the exit guarantee applied to the social layer: WeVibe cannot kill anyone's public record, because the record is chain state and the viewer is forkable.

### 6.5 Badge Families (Status-Only, Per-Org Scope)

Badges are a first-class social feature with three families:

1. **Serve-milestone badges** — thresholds based on how often a contributor's memories are served.
2. **Rarity-tier badges** — derived from per-memory keyword supply/demand tiers, computed read-time by the social-graph display client; on-chain freeze of rarity semantics lands at mainnet.
3. **Contribution-volume badges** — thresholds based on approved memory contribution volume.

Scoping is per-org, with profile breakdowns instead of a single global ladder. This keeps competition bounded and useful: you can see where someone built reputation, without forcing all contributors into one network-wide leaderboard.

Badge status is strictly non-economic: no VIBE reward, no emissions multiplier, no payout coupling.

### 6.6 Signal Integrity and Anti-Gaming

Even as status-only signals, attribution must stay hard to fake. The reputation layer deduplicates repeat serves, caps serves per memory per epoch, discounts serves a user makes to their own memories, and applies diminishing returns on repeated serves. Serve deduplication is hardened: the chain recomputes the fingerprint itself — from the memory id, the user's per-org serve key, and the epoch window — rather than trusting a client-supplied identifier, and each serve/denial entry carries a per-entry consumer signature the chain verifies, counting only verified entries.

Human review (§5) remains the first anti-gaming gate: low-quality memories should fail approval before they can accrue social status. Denial and report systems (§5.5–§5.8) provide additional negative feedback signals without coupling directly to token payout.

Optional attestation dimensions (difficulty/verification quality) are post-mainnet expansion points (§3.11–3.12).

---

## 7. Organization Social Graph

### 7.1 Public Discovery Interface (opt-in)

**Visible to non-members (if public):** Organization name, specialization, description, memory count, member count, age, leader identity, total serves, social badge summary, and two unfakeable org-health signals introduced below.

**Not visible to non-members:** Memory content (encrypted on-chain), member identities (privacy-preserving), review history, payout rules.

**Unfakeable org-health signals.** Discovery surfaces two behavioral metrics that are capture-resistant by construction:

- **Leader last active.** Aggregated timestamp of the most recent on-chain action signed by the org leader's wallet — batch memory commits, denial settlements, member changes, report responses, epoch rotations. The signal requires a real wallet signature on a real transaction paying real gas. A dormant or captured org cannot fake it.
- **Voluntary departure rate.** Members who left of their own accord in the trailing 90 days, expressed as a fraction of total membership. Departures are first-class on-chain events; sybils can be invited and can file reports, but they cannot fake people walking away. A cohort exiting a captured org is the strongest negative signal the public can read.

**What is deliberately NOT surfaced.** Discovery does not display per-org report counts, report aggregates, dispute counts, dismissed-report counts, or any other report-derived statistic. The rationale is structural and is the same as in §5.7: every in-app aggregation of reports is gameable, weaponizable, and censorable. The chain is the public record; the block explorer is the viewer. Prospective joiners who want to investigate report history can do so on-chain; WeVibe's own discovery surface does not turn that history into a leaderboard.

### 7.2 Badge Scoping and Canonical Criteria

Organization profiles expose badge state in a per-org breakdown for both contributors and the org aggregate itself.

- **Per-org scope:** badges are earned and displayed in org context, then optionally summarized on contributor profiles.
- **No cross-org leaderboard:** WeVibe does not publish a global rank table.
- **Canonical criteria for display tiers:** rarity tier is computed read-time by the reference social-graph display client, with on-chain freeze of rarity semantics at mainnet; serve-milestone and contribution-volume thresholds come from a canonical reference spec used by the reference social graph so labels like "Legendary" remain consistent across forks.

This canonical-spec-in-display approach preserves fork freedom while keeping badge semantics legible across the ecosystem.

### 7.3 Leader Interface

Hub-hosted web dashboard (`wevibe-dashboard`): pending review queue, memory browser, historical decisions, member management, org configuration, keyword taxonomy management, recovery status, direct memory authoring, bandwidth usage monitoring, relay/settlement status, and the report response interface.

Serve and denial settlement runs automatically through the hub→chain relay (§5.8), not a manual leader action; the dashboard surfaces relay/settlement status (pending counts, last batch commit) for visibility. There is no per-denial review — denials are quantitative signals the relay batches and settles on-chain.

The report response interface surfaces an open report against a memory the leader committed: during the response window it shows the remaining time and the uphold/dismiss actions; after a memory has been publicly exposed it offers the one-shot leader dispute. It links to the on-chain transaction once a response is published.

### 7.4 Member Interface

Members see: role, contribution count, serve count, reputation summary, and per-org badge progress/status.

### 7.5 Reporter's Private View

Each reporter has a private list of their own reports and published escalations. Each entry shows a memory excerpt, the org name, the submission date, the leader's response status (pending / upheld / dismissed / unaddressed / disputed), and a copy-link to the on-chain transaction. This view is visible only to the reporter — it is the reporter's own record of escalations, and the place from which they share block-explorer URLs to whatever public forum they choose. No other user sees it.

---
## 8. Storage Architecture

### 8.1 On-Chain Encrypted Memory Storage

Memory content is stored as encrypted blobs directly on the WeVibe chain. Each approved memory is a transaction that writes:
- Encrypted ciphertext (AES-256-GCM)
- Wrapped DEK (sealed to epoch key)
- Plaintext metadata: org ID, epoch, contributor pub key, keywords/weights, stack tags, timestamp, provenance tier, project fingerprint, scope, and MC-1 schema version (`mc_version`)
- Merkle leaf hash for the epoch batch

Every validator replicates every memory. This is the storage guarantee — no separate challenge mechanism needed. Chain consensus IS the storage layer.

**Size economics.** A typical memory is 500 bytes to 2KB encrypted. At 10,000 memories, that's ~10-20MB of chain state. At 100,000 memories, ~100-200MB. Cosmos SDK chains handle this comfortably. Pruning strategies for dormant orgs keep long-term state manageable.

### 8.2 Keyword Index (On-Chain Metadata)

Plaintext per-memory keyword weights are stored alongside encrypted memories on-chain. This enables keyword-based filtering without decryption. The tradeoff (keyword visibility) is accepted — see §3.7. The org-level keyword *taxonomy* itself — the controlled vocabulary a leader manages (add / merge / rename / deprecate) — is a hub-side capability (hub database + dashboard); both the org-level vocabulary version hash (`vocab_hash`) and the pinned `embedding_model_id` are anchored on-chain, so a rebuilt hub can (a) verify it restored the authoritative taxonomy and (b) re-embed under the correct model (§8.3 rebuildability; HUB-REBUILD §3.4). The chain carries each memory's keyword weights.

### 8.3 Semantic Vector Index (Hub Qdrant)

Vector embeddings are NOT stored on-chain. Stored-memory embeddings are computed at approval and upserted to the hub's Qdrant index, where similarity search runs over vectors plus keyword metadata. For recall, the MCP/plugin computes the query embedding locally via Ollama and sends the query vector to the hub.

Qdrant stores vector + keyword metadata only — no plaintext memory content and no ciphertext. This "not ciphertext" is scoped to Qdrant, not to the hub as a whole: the encrypted blob the hub *does* hold lives in Postgres (`pending_submissions`, `rotation_buffer`) and on-chain (`StoredMemoryCommitment.EncryptedBlob`, §8.1), and the hub cannot decrypt it — the DEK is wrapped to leader/moderator keys and the hub never receives the epoch secret (§3.6). Qdrant itself holds only the vector + keyword shadow. Embeddings are derived data and remain off-chain; the entire index is rebuildable from chain state plus org keys, and the clean embeddings are a disclosed semantic shadow.

### 8.4 Memory Metadata

- **contributor_pubkey** — on-chain identity of the contributor
- **model_origin** — contributing model
- **stack_tags** — freeform technology tags
- **version** — nullable version string
- **source** — `session` | `doc_import` | `authored`
- **provenance** — `tee-attested` | `commitllm` | `proxy-attested` | `self-declared` | `unattested` (graded per field)
- **certified_model** — TEE-attested model measurement (checkpoint hash) of the session's production model, if any; certifies session provenance, not memory text
- **derivation** — `verbatim` | `edited-after-extraction` (whether the submitted memory matches the pinned-extraction output)
- **difficulty** — difficulty value (1–10) on the attested-memory record; feeds the per-contributor difficulty histogram in `x/reputation`. Automated scoring (§3.12) is the post-mainnet populator.
- **quality** — quality value (1–10) on the attested-memory record (reputation XP = difficulty × quality). The Layer-2 grading LLM (§3.12) is the post-mainnet populator.
- **approved** — boolean moderation state
- **is_reported** — an open report currently stands against this memory
- **was_reported** — reported at least once; permanent historical flag
- **report_cleared** — the leader dismissed the open report via `clear_report`
- **quarantined** — flag for memories with repeated upheld rejections; retrieval-policy exclusion
- **deprecated** — curator has marked stale
- **project_fingerprint** — opaque per-memory project-scope key (SHA-256 of the normalized public git-origin URL, or a machine-independent id for local-only repos; never personal paths)
- **scope** — `project` | `org`
- **mc_version** — Memory Contract (MC-1) schema version

---

## 9. Security Analysis

### 9.1 Sybil Resistance

Free account creation cannot be prevented — a keypair is offline math — so WeVibe does not try to gate identity creation. Instead it makes free identities **inert** and locates the gate where harm can occur:

- A membership-less identity can do nothing — no contribution, no recall, no earnings — so an attacker minting identities at scale gains nothing.
- **Org membership is the scarce, gated resource.** Membership is invitation/approval-only, with the human leader as the bouncer (and the org-slot itself is a hard-capped, auction-burned, forfeitable bond on the leader).
- **Reputation is the soft stake** for wallet-free contributors: it is destroyed on removal, so a contributor who built standing and then turns malicious loses it and restarts at zero under a fresh, re-approval-gated identity.
- **Contributors carry no bond and can never publish unilaterally** — every contribution is leader-gated and committed under the leader's sole signature (§9.7), so the leader's slot is the bond behind all published content. This is why frictionless, wallet-free contribution does not open a Sybil hole: the blast radius of a Sybil contributor is "wasted reviewer time," not "poisoned network," and that is bounded by rate-limits, booting, and cheap join-door friction (cooldowns, invite codes).

### 9.2 Memory Poisoning

Defense layers: submission-time wevibe-guard (advisory), OCR sanitization, human review with steganography detection, contributor reputation visible during review, recall-time blacklist + quarantine, recall-time wevibe-guard, recall-time OCR, artifact extraction with egress enforcement, plugin approval gate with contributor trust signals. Residual risk: semantic payloads, subtly wrong recommendations.

### 9.3 Leader Key Compromise

K_master compromise exposes all epoch-derived content. Mitigation: offline recovery phrase, encrypted vault with Argon2id, threshold recovery.

Separately, hub infrastructure is treated as untrusted transport, not a trust root: endpoint authority comes from leader-signed on-chain `hub_endpoints` updates on the canonical chain, and client verification of hub-signed responses against on-chain `hub_serving_address` is the transport MITM defense.

### 9.4 Chain State Observability

On-chain data is public (encrypted blobs + plaintext metadata). An observer can see: org sizes, submission frequency, keyword distributions, serve patterns, contributor activity, reputation scores, report flags. They cannot see: memory content, decryption keys, member identities beyond pub keys, local blacklist state.

### 9.5 Network-Level Anti-DDOS

Anti-DDoS uses two complementary mechanisms. On testnet — where faucet tokens are free and so carry no economic deterrent — `x/bandwidth` enforces flat per-org per-epoch rate-limit caps; the chain rejects submissions/serves exceeding the cap. On mainnet, the per-memory **storage deposit** (a VIBE cost per committed memory) is the economic anti-spam, and `x/bandwidth` retires at mainnet launch. No org can flood the network regardless of off-chain resources.

### 9.6 Content Suitability Policy

**Suitable:** Coding patterns/anti-patterns, architecture lessons, debugging notes, dependency guidance, tool usage, process workflows, version-specific gotchas, negative knowledge.

**Unsuitable:** Credentials/secrets, customer PII, regulated data, legal/HR records, high-sensitivity security incident details.

### 9.7 Org Capture and Public Escalation

A single actor wearing multiple hats — leader, every member with `can_moderate`, every contributing member — can fully capture an org's internal governance. Inside the org, every approval, every report dismissal, and every chain commit can be coordinated. Internal accountability primitives (advisory votes, dispute counts, internal review queues) provide no protection against this case: the captured operator simply approves their own malicious memories and dismisses every report filed against them.

The system's security model is therefore not "prevent capture through internal governance." The system's security model is:

> Make capture economically unsustainable through transparent on-chain accountability, frictionless exit for members, and a public escalation primitive designed so that a captured org cannot suppress it.

The four load-bearing properties:

1. **The chain is the unforgeable audit log.** Every consequential action — memory commit, denial settlement, report resolution, dispute publication, member departure — is a signed on-chain transaction. Neither the captured org nor WeVibe-the-protocol nor any platform operator can edit or suppress it after the fact.
2. **Consumers hold an escalation path the org cannot close.** A dismissed (`clear_report`) or unaddressed on-chain report unlocks a reporter-signed public escalation — wallet-gated and gas-paid, revealing plaintext only after the one-week window elapses or the leader dismisses, anchored to the contributor-signed hash the leader cannot poison (§5.4) — and once published it cannot be edited or deleted. The reporter broadcasts the filing and expose **directly to chain RPC**, with the WeVibe-operated relay as retry fallback only: the censorship-resistance path does not depend on WeVibe-operated infrastructure, so neither a captured org nor WeVibe itself can suppress it.
3. **Exit is unfakeable.** Members leaving voluntarily is a first-class on-chain event. Sybils can be invited and can file frivolous reports, but they cannot fake people walking away. The voluntary-departure-rate signal on public discovery (§7.1) lets prospective joiners read the most honest possible signal about whether existing members trust the org.
4. **Hub compromise is a per-org degradation event, not network takeover.** Per-memory Umbral crypto and consumer-side wevibe-guard still gate plaintext/injection, and hub responses must verify against on-chain serving keys. A compromised endpoint can at worst degrade or poison recall for that org; it cannot mint identities, steal contributor keys, or affect other orgs. The endpoint can be rotated on-chain by leader signature, with clients auto-switching and passively notifying once.

Worst case in this class is degraded/poisoned recall quality for one org, typically surfaced by guard/crypto checks and reversible by on-chain endpoint rotation.

**The leader bears sole signature.** Co-attestation of additional reviewer pubkeys on leader-signed chain transactions is explicitly excluded (§5.3). A leader's chain commit binds the leader's wallet only. This concentrates responsibility on the actor who actually signs and prevents implicating advisory reviewers in chain-level decisions they did not directly authorize. The trade-off — accountability for individual advisory approvals becomes an org-local rather than chain-public concern — is acceptable because internal advisory-vote history cannot defend against a capture scenario anyway, and because making leaders sole signatories sharpens the public attribution of every consequential action.

**Why no platform tribunal.** WeVibe deliberately does not become a tribunal that adjudicates published reports. Any in-app judging body is a capture vector — whoever controls the tribunal controls the verdict. The chain publishes the evidence; the block explorer renders it; the reporter shares the URL wherever they choose; the public judges on its own merits in whatever forums it chooses. WeVibe-the-protocol has no editorial role in that judgment (§5.7).

**Residual: contributor-leader collusion.** If the contributor who submitted a memory and the leader who committed it are both adversaries, the contributor can sign a false plaintext hash, the leader can commit it, and a public report's plaintext reveal will not match the on-chain hash — making the report look invalid even when the underlying claim is true. The on-chain ciphertext + capsule remain as a final backstop: any future key disclosure (epoch rotation, org closure, legal discovery) lets independent parties decrypt and verify after the fact (§5.4). This residual cannot be eliminated at submit time — when content creator and content approver are the same adversary, there is no honest party in the verification chain to sign over.

---

## 10. Decentralized Architecture

### 10.1 Chain Architecture

WeVibe's chain is a sovereign L1 appchain built on Cosmos SDK + CometBFT. Not a rollup — WeVibe requires deterministic finality (CometBFT provides this; rollups have multi-day challenge windows). The chain halts before it forks — safety-over-liveness is correct for memory attestation and storage.

Chain org state carries `hub_endpoints` (an ordered list of 1–3 transport URLs for failover redundancy, set through a leader-signed setter transaction), while `hub_serving_address` remains the serve/deny signing-authorization key; clients resolve transport from chain RPC rather than manual URL config.

### 10.2 The Four Roles

**Developer (user).** Codes with an LLM. Onboards in seconds with a **passkey** (Face ID / fingerprint) — no wallet or seed phrase required to create an account, join an org, contribute, or recall. Sessions stay local by default, and contribution is explicit through the dashboard extraction/review/submit flow. May consume paid recall access through orgs, but chain mechanics stay abstracted behind plugin/hub UX. A wallet is an optional later upgrade — needed only to claim earned VIBE or pay mainnet fees. Experience: "I code, I choose what to contribute, my profile shows what I've solved."

**Org leader (economic operator + curator).** Acquires an org slot (auction burn) and signs registration from their own wallet, curates memories, manages membership, and sets the org's recall-access/payment model (price + policy). Leader revenue comes from members' VIBE payments for org access, settled through an **on-chain demand-leg router** that enforces a protocol burn (`max(n%, floor)`) and routes the remainder **in-transaction to the leader's own wallet** — the network holds no revenue account. Members holding `can_moderate` are paid at leader discretion. The leader carries ongoing skin-in-the-game via the slot (self-assessed-value rent + forced-sale-in-window) and per-memory storage deposits. **Leaders earn no emissions.**

**Validator.** Stakes VIBE, runs CometBFT consensus, stores all chain state (including encrypted memories), earns validator/staking emissions. Everything deterministic — no subjective judgments. Validators are the storage and availability layer.

**WeVibe-the-protocol.** Open-source software. No company in the middle. WeVibe-the-company may run validators and operate orgs early on, but the protocol does not depend on any single entity.

### 10.3 Token Economics

Single token: **VIBE**. Used for staking, org-slot acquisition (auction burns), per-memory storage deposits, demand-leg router settlement, and contributor payouts.

**Org slots (scarce, capped, auctioned).** Org capacity is a hard-capped set of registry-allocated slots (governance param: 32 alpha / 320 testnet / 3200 mainnet). Primary allocation is an ascending price per subsequent slot until the cap fills; a freed slot (abandoned/closed/lapsed) is re-homed by a descending (Dutch) resale. The acquisition payment is split 50/50: half is burned and half capitalizes the org's own on-chain account. The slot `org_id` is permanent and leader-independent. Scarcity + the burn is the network-level anti-spam.

**Self-assessed-value rent (Harberger-style).** A leader posts a self-assessed value V for their slot, pays rent `r × V` per period, and anyone may force-buy the slot at V during a bid window. This keeps slots in productive hands without right-of-first-refusal chilling or grief-bidding. Non-payment/abandonment frees the slot back to Dutch resale and marks the org dormant.

**Per-memory storage deposit (a keeper/liveness function, not a participation cost).** Each committed memory carries a small VIBE deposit that decays as storage rent and is keeper-claimable as a deletion bounty when exhausted/abandoned; pruning deletes the ciphertext blob but permanently retains accountability metadata. Its purpose is **storage hygiene and liveness** — it lets keepers **reap dead memories** and reclaim perpetual storage, and it provides the path to prune an org's memories if the **leader goes AWOL** (org abandoned/dormant) so the network is not stuck paying to store orphaned blobs forever. It is explicitly **not** a cost on contributing or a contributor bond (contributors pay nothing); the deposit is funded from the org side and prices the real (perpetual storage) externality at the granularity it occurs. A memory under an open report has its deposit frozen (unprunable until resolved). Parameterized ~0 on testnet (where `x/bandwidth` is the rate-limit guard instead).

**Serve/deny receipt loop (earned trust — no VIBE).** Recall runs through the hub's **serve/deny route**: when a consumer recalls (or blocks) a memory, the hub relay submits a serve/denial batch to the chain (per-entry consumer-signed; carried by the org's whitelisted serving key, gas covered by the org-account feegrant). The chain folds those serve/denial events into each memory's **Earned Trust** standing, so a memory's retrieval weight rises with demonstrated usefulness and decays toward archival when it is denied or never served. This loop is **separate from the storage deposit**: it moves *no* VIBE and is purely the trust/decay signal — the deposit is the storage-liveness lever, the serve/deny loop is the quality lever.

**Demand-leg router.** Members' recall-access payments settle through an on-chain router whose one job is to enforce the protocol burn `max(n%, floor)` and route the remainder **in-transaction to the leader's wallet** (the network holds no revenue account); `n`/`floor`/`r`/slot-cap are governance params.

**Contributor payouts (contribution-only).** Contributors are paid per **approved memory** from the network contributor-emission budget, gated by a **network-set** qualification threshold. There is no payout per serve/retrieval. Reputation tiers may scale payout-per-approved-memory, but retrieval counts are excluded from VIBE flows.

#### 10.3.1 Emission Schedule (locked)

The protocol mints VIBE on a fixed **32-year schedule** from genesis:

| Allocation | Amount | Notes |
|---|---|---|
| Total supply | 1,000,000,000 VIBE | 10^6 uvibe per VIBE |
| Foundation (genesis) | 10% = 100,000,000 VIBE | unlocked at genesis |
| Validator (genesis) | 1% = 10,000,000 VIBE | docker validator self-delegation |
| Validator 32-yr pool | 570,000,000 VIBE | emitted linearly over the schedule |
| Contributor 32-yr pool | 320,000,000 VIBE | 1%/yr, capped at 10,000,000 VIBE/year |

**Per-epoch emission.** Validator emission = `validator_pool_remaining / remaining_epochs`. Contributor budget = `min(contributor_pool_remaining / remaining_epochs, annual_cap / epochs_per_year)`. The schedule length is 11,680 epochs (32 × 365).

**Qualifying contributors** are those with at least `contributor_qualify_threshold` (default 1) approved memories network-wide in the epoch. The contributor budget is split evenly among them. **Global rollover:** if no one qualifies in an epoch, that epoch's contributor budget rolls forward; the integer remainder of an even split also carries forward, so no VIBE is burned by rounding.

**Genesis seeding.** The emission pool (`validator_pool_remaining_uvibe`, `contributor_pool_remaining_uvibe`, `contributor_rollover_uvibe`, `start_epoch`) is written at chain genesis. Emission and decay both run in the chain's epoch-end hook.

**Contributor attribution.** Emissions credit the *authoritative* contributor recorded on the committed memory (`contributor_address`), not a consumer-supplied serve payload field. The same address drives serve attribution.

**Claim path.** Contributor rewards accrue on-chain to a claim-later balance keyed to the passkey identity. They become withdrawable to a wallet only after the contributor links a Cosmos wallet **and** performs the explicit dual-signed on-chain migration (`is_migrated`; link ≠ migrate; §6.3). Privileged capabilities are the exception to wallet-optionality: a leader always has a linked wallet, and members granted `can_moderate` likewise require a linked wallet (they sign their advisory votes), since the leader is the sole wallet-signing bond for published content. The mint+claim transfer path carries a reentrancy guard and a double-claim/duplication guard as consensus-level invariants.

### 10.4 Validator Economics

Validators earn standard Cosmos SDK staking rewards for running consensus, supplemented by the linear validator-pool emission (§10.3.1). Validators store all encrypted memories as part of chain state — this is not separate "operator work," it is inherent to running a node. No separate storage challenges needed.

### 10.5 Demand-Leg Economics (Non-Custodial Router)

Serve/retrieval attribution is **social, not economic** (§6): serve counts drive public profiles and badges, but do not trigger VIBE payout.

Economic demand is the org access leg:
1. Users buy VIBE and pay orgs for recall access.
2. Access/payment model and pricing are **leader-set**; payments settle through the **on-chain demand-leg router** (the hub `org_credits` ledger is a mirror of chain state, never the source of truth).
3. The router burns `max(n%, floor)` of each payment (governance params) — the deflationary sink.
4. The remainder is the leader's revenue. The router **routes the remainder in-transaction directly to the leader's wallet** — there is **no network-held revenue account**. The per-org on-chain account is the **operating account** only (gas, feegrants, storage deposits, the 50% acquisition-retain capitalization, voluntary leader top-ups); it never accumulates member revenue.
5. Leader compensates members holding `can_moderate` at discretion from that revenue (their own wallet); there is no protocol-enforced split.

**Router enforcement.** Org recall access is gated through the router: the hub's `membership_active` flag is set for non-trial members **only by the hub's chain watcher upon a confirmed router payment event**; `org_credits`/subscription state is a strict mirror of chain payment events (the chain-first/hub-mirrored pattern). Trial members remain on the orthogonal trial path.

**Canonical closed loop:** emission → contributors (contribution-only, claim after wallet-link + migration) + validators/stakers (mint/sell) → users buy VIBE → users pay orgs (leader-set model & price) → router burn + remainder to leader → leader pays members holding `can_moderate` → stake/secure → repeat.

Leaders earn no emissions, there is no per-serve royalty, and there is no protocol-enforced split for members holding `can_moderate`. WeVibe is a **non-custodial** network — p2p payments + memory storage + reputation, holding no user or org funds. The only consensus-level economic infrastructure it requires here is the payment router that enforces the VIBE burn and routes the remainder in-transaction to the leader's wallet.

### 10.6 On-Chain Modules

Seven custom Cosmos SDK modules:

- `x/org` — slot registry + acquisition auction (ascending primary; Dutch resale; self-assessed-value rent + forced-sale-in-window), per-org module account (operating account only — never holds member revenue), on-chain demand-leg router (membership payment → burn cut + remainder routed in-tx to the leader's wallet), membership, org-directory transport/auth fields (`hub_endpoints` — ordered list of 1–3 transport URLs for failover, via leader-signed setter tx; `hub_serving_address` serving/signing authorization key), serving-key feegrant, dormancy/abandonment detection
- `x/memory` — pending commitment storage (hash + metadata, no blob until approved) with commitment expiry, approved memory blob storage (encrypted ciphertext as chain state), Merkle root submissions per epoch, contributor-signed verification anchor (plaintext/salt/ciphertext hashes), report flags (`is_reported`/`was_reported`/`report_cleared`), quarantine flagging
- `x/serve` — batched serve receipt recording (per-org pseudonymous serve keys), per-entry consumer-signature verification, deduplication (memory_cid + serve_key + epoch), self-serve detection/discounting, contributor cross-org serve count aggregation for social attribution (non-economic)
- `x/reputation` — per-contributor cross-org aggregated stats (serve count, org breadth, domain tags, rep score, wallet age). Enhanced mode per-org when attestation enabled (difficulty histogram, XP, provenance breakdown).
- `x/emissions` — validator staking rewards, contributor emission distribution from the network pool, protocol-level emission schedule, claim-later balances + guarded mint/claim path
- `x/bandwidth` — per-org per-epoch flat rate-limit caps (DDoS guard). This is the **testnet** anti-spam guard (faucet tokens are free, so economic deposits don't deter there); retires at **mainnet launch**, when the per-memory storage deposit takes over the anti-spam role
- `x/attestation` — session-attestation storage socket. Held inactive until the pluggable attestation framework activates pre-mainnet; post-mainnet activation generalizes this socket into a typed proof artifact (`tee_receipt` / `zktls_proof` / `zkml_proof`)

Standard SDK modules: `x/staking`, `x/auth`, `x/bank`, `x/gov` (wired for on-chain param updates), `x/slashing`, `x/distribution`.

**Modules eliminated from v1.x:** `x/serving` (no separate operator serving sets — validators serve all), `x/challenge` (no storage challenges — consensus IS storage), `x/receipt` (replaced by x/serve), `x/operator` (merged into standard validator staking).

### 10.7 Node Lifecycle (`wevibed`)

The chain ships a runnable Cosmos SDK application and CLI (`wevibed`). A single-node bootstrap follows the standard pattern:

1. `wevibed init {moniker} --chain-id {id}`
2. `wevibed keys add validator --keyring-backend test`
3. `wevibed genesis add-genesis-account {addr} 10000000000000uvibe`
4. `wevibed genesis gentx validator 1000000000000uvibe --chain-id {id}`
5. `wevibed genesis collect-gentxs`
6. `wevibed start`

The canonical local-devnet bootstrap (with the full genesis allocations from §10.3.1, epoch config, and emissions/reputation seeding) is `scripts/init-chain.sh`; the steps above are the standard skeleton.

---

## 11. Skill Model and Federation

### 11.1 Task-Context Skills (Roadmap)

Curator-defined collections organized by task context. Skills are lightweight named sets of memories with a description that improve curation and social discoverability inside an org; ungrouped memories are allowed and skill assignment is optional.

### 11.2 Cold-Start: Documentation Seeding (Roadmap)

New organizations import canonical documentation as seed memories (source: `doc_import`), with session contributions extending and replacing seeds over time.

### 11.3 Federation (Design Phase)

Federation operates at the skill level. Orgs publish skill packages. Receiving orgs set quality thresholds. No individual contributor reputation crosses federation boundaries. Federation is deferred until the alpha curation loop is proven.

---

## 12. Implementation Phases

### Phase I: Alpha — The Knowledge Loop + The Accountability Rail

- Working memory contribution/review/retrieval loop with encrypted on-chain memory storage
- Org membership/role flows and leader-signed curation pipeline
- Gated injection (four-button approval) across MCP/plugin entry points, with guard/sanitization
- Consumer-filed on-chain reports, resolution window, public escalation, and direct-broadcast path
- Chain-resolved hub endpoints with signed hub responses
- Serve attribution counters on-chain for contributor + org provenance signals

**Exit criteria:** A reliable daily loop from session output to approved memory to gated recall, with the report/expose accountability rail live end-to-end and attribution updating public provenance signals.

### Phase II: Provenance Surfaces + Economic Settlement

- Public contributor/org profile surfaces over chain data (serve + contribution signals)
- Badge surfaces for serve milestones, contribution volume, and rarity tiers in the reference social graph
- Demand-leg router settlement, claim-later reward path, and slot resale/rent mechanics
- Curator workflows for direct memory authoring, documentation seeding, and skill curation

### Phase III: Post-Mainnet — Attestation + Federation

- Pluggable session-attestation framework (cryptographic and/or API-backed), per §3.11, generalizing `x/attestation` to typed proof artifacts
- Two-layer difficulty scoring (§3.12) as an optional quality input once attestation infrastructure is live
- Skill-level federation: publish/subscribe skill packages with receiver-side quality thresholds and no cross-org contributor-reputation portability

---

## 13. Open Design Questions

**Earned-Trust settlement lag.** What is the maximum acceptable divergence window between hub-optimistic ranking and chain-settled weights? The decay formula does not change — only event durability/timing semantics are in question (§5.8).

**Pending memory retention window.** How long do pending (unreviewed) commitments stay on-chain before auto-purging? 72 hours? Configurable per org?

**On-chain storage limits.** At what scale does on-chain memory storage warrant tiered pruning beyond keeper reaping? 100K memories? 1M? Chain state growth and validator hardware requirements are modeled against adoption.

**Embedding model evolution.** nomic-embed-text is confirmed for now. Future upgrades mean re-embedding all memories — this happens client-side at approval but needs coordination across org members.

**Federation rollout scope.** Which minimum skill-package contract ships first (metadata, provenance, quality thresholds) when federation moves from design phase toward implementation?

**Social-signal integrity for serves.** Serves are social/status signals (not payout inputs), but self-serve and coordinated-serve patterns still need monitoring policy and visibility.

---

## 14. References

[1] Song, D., Wagner, D., Perrig, A. (2000). Practical Techniques for Searches on Encrypted Data. *IEEE S&P.*
[2] Curtmola, R., et al. (2006). Searchable Symmetric Encryption. *ACM CCS.*
[3] Cormack, G.V., et al. (2009). Reciprocal rank fusion. *SIGIR '09.*
[4] Reimers, N. & Gurevych, I. (2019). Sentence-BERT. *EMNLP 2019.*
[5] Kusupati, A. et al. (2022). Matryoshka Representation Learning. *NeurIPS 2022.*
[6] Greshake, K. et al. (2023). Indirect Prompt Injection. *arXiv:2302.12173.*
[7] OWASP. (2025). Top 10 for LLM Applications.
[8] Sha, F. et al. (2025). Lessons from Defending Gemini Against Indirect Prompt Injections. *Google DeepMind.*
[9] Zou, A. et al. (2024). PoisonedRAG. *arXiv:2402.07867.*
[10] Google DeepMind. (2026). AI Agent Traps.
[11] MLS Architecture (RFC 9420).
[12] Lambda Class. CommitLLM: Cryptographic commit-and-audit for LLM inference. *github.com/lambdaclass/CommitLLM.*
[13] Morris, J. et al. (2023). Text Embeddings Reveal (Almost) As Much As Text. *EMNLP 2023.*
[14] Huang, Y. et al. (2024). Transferable Embedding Inversion Attacks. *ACL 2024.*
[15] Núñez, D. et al. (2018). Umbral: A Threshold Proxy Re-Encryption Scheme. *NuCypher / NICS Lab, Universidad de Málaga.*

---

*This document is an architecture specification. Nothing in this document constitutes an offer or solicitation of securities.*