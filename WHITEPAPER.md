# WeVibe Network

**Social Reputation + Shared Memory for Vibe Coders**
*Build public coding reputation and collectible badges from daily agent work; shared memory powers your next session as the bonus.*

Draft v2.6 · June 2026 · Architecture Document
Classification: Public

---

## Changelog

| Version | Summary |
|---|---|
| v2.6 | Personal-memory layer scoped (D-PERSONAL-MEMORY): a bounded, pull-mode/deterministic local memory distinct from shared org memory; predictive recall for new problems stays the org pipeline's lane; provider interface open, one default packaged, BYO opt-in. Design-locked, not built. |
| v2.5 | Extraction-provenance honesty pass (DMO-032): trust-circle parity; cloud-mainstream extraction canonized; session-matched default; locality claims scoped. |
| v2.4 | Mission-integrity pass (DMO-030): exit-guarantees invariant; custody resolved non-custodial; serve-signing target reconciled; direct-broadcast model; semantic-shadow disclosure; settlement-description reconciliation; GAP-MI register cross-refs. |
| v2.3 | Social-first repositioning for individual vibe coders/small crews; hub is live (not eliminated); public keywords are a discovery feature; chain-resolved hub-endpoint + onboarding posture added. |
| v2.2 | Verification anchor redesigned to a contributor-signed canonical body (`plaintext_hash`/`salt`/`ciphertext_hash`); SP1/ZK pathway rejected as operationally unshippable; three production bugs fixed. |
| v2.1 | Accountability primitives: silent denial as cheap negative signal, two-tier reports with public on-chain escalation, contributor-signed verification anchor, leader sole signature, unfakeable org-health signals. |
| v2.0 | Architecture pivot: on-chain encrypted memory storage + hub retrieval with local decryption/guard + plugin-gated human approval; serve attribution as social signal; domain-expert-led orgs; three software pieces. |

---

## Abstract

WeVibe ships in alpha as a social-first network for individual vibe coders and small crews. You install a plugin, contribute what actually worked in real coding sessions, and watch your public reputation and badges grow over time. That daily momentum is the hook. Shared memory recall is the bonus: your next agent run starts with better context instead of starting from zero.

Domain experts run organizations as curated memory collections you can join. Leaders and reviewers decide what gets committed, what gets rejected, and what stays useful as frameworks change. Public plaintext keywords are intentional discovery signals (for example, `redis`, `solana`, `django`) so coders can quickly find the communities that match their stack.

The chain anchors provenance, membership, and attribution signals; the plugin enforces human-in-the-loop memory injection; local retrieval keeps plaintext close to the coder. In alpha, the hub is live for coordination/accounting workflows. The economic model is decided, but some demand-leg and settlement mechanics are still being implemented; this document describes what ships now and what is explicitly near-term.

---

## 1. Design Philosophy

### 1.1 The Problem

Vibe coders do hard problem-solving with agents every day, but each new session still behaves like a cold start. A fix discovered today — the precise Nginx keepalive tweak, the one migration order that prevents data loss, the exact flag combination for a flaky deploy — is usually trapped in private chat logs and forgotten tomorrow.

This hurts most when working with local or smaller models. The model can write code, but it does not know your stack's lived edge-cases, version mismatches, and production scars. The gap is not "more generic internet text"; the gap is verified, domain-specific memory from people who actually ran the problem. The verified-memory value claim is **any-agent** — the corpus serves whatever model you run — but the sharpest consumer pain (and the marquee payoff) is the local/smaller model, which gets a fix it could not have produced itself; the knowledge source is most often a frontier/cloud session where that fix was first won (D-EXTRACTION-HONEST-CLAIM).

At the same time, developer knowledge channels are increasingly flooded with AI-generated content that sounds plausible but misses decisive nuance. Provenance becomes scarce. A memory attached to a real contributor, a real org, and a real moderation decision is materially more useful than anonymous content sludge.

There is also a social gap: vibe coders have no native scoreboard for the work they do with agents. Git commits show output, not the session-level judgment calls that created it. WeVibe addresses that by turning contribution and serve attribution into public social signals (profiles, milestones, badges) while keeping economic reward logic separate from per-serve events.

### 1.2 The Organization Model

Organizations are domain-expert-run memory collections you join. Think: a React performance org, a Solana tooling org, a Kubernetes reliability org. You join because your daily agent work needs better domain context now, and because you want your own contribution history to compound publicly over time.

Each organization is a collaboration container with its own:

- **Membership roster** managed by the leader.
- **Role hierarchy** (Leader, Reviewer, Member).
- **Commitment standards** for what counts as high-quality memory.
- **Domain focus** and coverage map.
- **Operating policies** for contribution/review cadence and recall access in alpha.

Leaders are domain experts responsible for memory quality, not faceless administrators. They approve who can review, define the bar for acceptance, and curate the collection as tools evolve. Strong org leadership raises both memory usefulness and the social credibility of members who contribute there.

Orgs are intentionally broad across experience levels: newer coders join to learn faster; experienced coders contribute high-signal memories, mentor standards, and build reputation. The org remains the container. The vibe coder remains the protagonist.

Public plaintext keywords are part of this model by design. They are discovery labels that help people find relevant memory collections quickly — not secrets, and not treated as a risk narrative in this document.

**Personal memory vs shared org memory (bounded, pull-mode).** Shared org memory — the curated, encrypted, chain-anchored collection described throughout this document — is WeVibe's predictive lane: it answers the *new* problem ("has someone solved something like this?") via situation-matched recall (Section 3.5). A coder's **personal memory** is a deliberately *different and bounded* layer: deterministic, on-demand (pull-mode) recall of KNOWN facts about their own repo and work — a repo/code map, durable project decisions, session-handoff state — retrieved on request, never speculatively injected at session start. The boundary is load-bearing: speculative context-injection for new problems is exactly where general personal-memory tools over-inject and add noise, and it is the lane WeVibe's curated org recall already wins. Personal memory is therefore scoped to *pull, not push*; any predictive/semantic personal recall (a coder's own past solutions, pre-contribution) is served through WeVibe's own aligned pipeline as a private corpus — not a bolted-on engine — which doubles as the on-ramp to contribution. WeVibe commits to a stable personal-memory provider interface with one packaged default and an opt-in bring-your-own engine for power users; the personal store is local scratch (exit = re-index), explicitly outside the chain-rebuildable contract. Design-locked (D-PERSONAL-MEMORY); not yet built.

### 1.3 The Curator Workbench

WeVibe is a curator workbench, not an autonomous ranking machine. The system surfaces signals — retrieval frequency, denials, staleness, query gaps, version drift — and human curators decide the action. This preserves accountable judgment where it belongs: with domain people who understand context.

Core workflows remain practical and hands-on: review pending memories (approve/reject/edit), author memories directly, package memories into reusable skills, identify coverage gaps, and retire stale entries before they mislead downstream users.

This curation loop serves both halves of the alpha wedge. It improves recall quality (the memory bonus that helps the next coding session), and it protects the integrity of public reputation/badges by ensuring attributed memories actually deserve trust.

### 1.4 The Plugin Gate

Every memory must pass through human eyes before it enters an agent's context. This is the product invariant.

The plugin is installed in the developer's coding environment (OpenCode, Claude Code, Cursor, Cline, and similar tools). During a coding session, the plugin harvests local session signals and auto-queries organizational memory through the hub retrieval path; candidate memories are decrypted locally, scanned by wevibe-guard, and shown in an approval UI:

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
│ [✓ Accept] [✗ Deny] [⚑ Report]           │
└──────────────────────────────────────────┘
```

**Accept:** Memory is injected into agent context. Serve attribution is queued to chain aggregates (contributor + org) for public profile and badge signals. No direct per-serve payout is implied by this action.

**Deny:** Memory is blocked. The plugin can capture lightweight deny context so curators can improve quality without turning the UX into a courtroom.

**Report:** The memory is reported on-chain — recorded against the contributor with the reporter's wallet (`is_reported` / `was_reported`) — and enters the org's accountability path (§5.5–5.8). The memory keeps serving until the report is resolved; the reporter can additionally Deny it to drop it from their own recall. Reports are the high-friction accountability primitive (wallet-gated, gas-paid); denials are the low-friction ranking signal.

**No plugin installed = no memory injection path.** The MCP server has no direct route to force memory into the agent context without the plugin frontend.

**Why not MCP elicitation?** Elicitation is useful in theory, but inconsistent across clients and weak as a hard-interrupt safety surface. The plugin provides deterministic interruption, clear modal UX, and explicit confirmation.

> **Alpha-status honesty.** [Gated approval] is the product contract and the locked default (D-11.5). The injection gate (`gateMemories`) is defined but not yet wired into the recall path, so memory injection is currently ungated — a known defect, BLOCKING-FOR-EXTERNAL-USERS, tracked as GAP-MI-1. The invariant above describes the contract; this note flags the code.

**Injection is per session, not per turn.** Recall is queried on each user prompt, but a recalled memory is injected into the agent's context **once per coding session** (keyed to the session identifier), not re-pushed on every model turn — the plugin tracks what it has already surfaced in the session and a memory's serve receipt is recorded once per session. A recall/inject governor (relevance floor + surface budget) keeps the injected volume small; an internal, default-off measurement mode can widen those bounds for offline testing/dev without changing the gated, per-session production path.

### 1.5 WeVibe's Architecture: Protocol, Not Platform

WeVibe is a protocol with open, auditable data surfaces — not a single closed SaaS product. In alpha, the chain, hub, local client, and plugin each do one narrow job well.

**What the protocol provides:**

1. **On-chain encrypted storage + provenance.** Memories are committed as encrypted blobs with attribution metadata.
2. **Human-gated delivery.** The plugin is the mandatory approval path before any memory enters agent context.
3. **Public social attribution.** Contribution and serve aggregates power reputation and badge surfaces for vibe coders.
4. **Domain-expert governance.** Leaders and reviewers curate memory quality inside each org collection.
5. **Alpha coordination layer.** wevibe-hub is live for hosted coordination/accounting workflows; it is not positioned as a plaintext memory oracle.
6. **Local retrieval + sanitization.** Decryption, vector retrieval, and guardrails run close to the user.
7. **Context injection format.** Approved memories are packaged for direct agent context use.
8. **Roadmapped attestations.** Session attestation and difficulty scoring remain explicit roadmap items, not implied as universally live today.

**Trust boundaries.** The chain is trusted for ordering and integrity, not for plaintext confidentiality. Plaintext handling stays local by default. The hub handles control-plane workflows in alpha. Validators and public observers see encrypted/state metadata, not raw memory content.

**Exit guarantees.** The enforceable form of "owned by no one" is four guarantees. No single party — including WeVibe-the-company, any hub operator, or any org leader — may have the unilateral ability to READ member memory plaintext, WITHHOLD the network's function from a principal acting within their rights, REWRITE the historical record, or KILL an org's knowledge or a contributor's standing by withdrawing infrastructure. The chain is the only durable authority; every other component must be disposable and reconstructible (D-HUB-REBUILDABLE). And every accountability action must be broadcastable by its principal without anyone's permission (D-REPORT-DIRECT-BROADCAST). See D-MISSION-INVARIANT and the Hub Authority Ledger (MASTER.md).

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

All roles require epoch-specific encryption keys for content access. The leader distributes these keys to approved members through sealed envelope key exchange (Section 3.4).

### 2.3 Organization Lifecycle

**Creation.** Org capacity is a scarce, capped set of registry-allocated **slots** (hard cap, governance-set: 32 alpha / 320 testnet / 3200 mainnet). The leader acquires a slot — by ascending-price primary while the cap fills (descending (Dutch) resale of a freed slot is designed, not yet built — see §10.6) — and signs `MsgRegisterOrg` from their **own wallet** (the hub never signs it); the acquisition payment is split 50/50 — half is burned and half capitalizes the org's own on-chain account (DECISIONS.md `D-ECON-STORAGE-MARKET` amendment 9). The `org_id` is the permanent slot identifier, independent of the leader (it survives leadership transfer/resale). The leader generates the master key K_master, derives the initial epoch keys (epoch 0), and generates the initial moderation keypair SK_mod(0)/PK_mod(0). A 24-word BIP39 recovery phrase is derived from K_master and displayed once (ADR-019). See DECISIONS.md `D-ECON-STORAGE-MARKET` (decided; build in progress).

**First-run detection.** When the MCP plugin/server starts and discovers no org membership, it surfaces an actionable message to the agent, prompting guided setup.

**Operation.** Members join through leader invitation. Once approved, the leader issues sealed key envelopes containing the epoch keys to the new member. For reviewers and leaders, the envelope also includes SK_mod(e) for the current epoch.

**Contributor onboarding (wallet-free).** A contributor creates an account in seconds with a **passkey** (Face ID / fingerprint) — no wallet, no seed phrase (DECISIONS.md `D-IDENTITY-PROGRESSIVE-CUSTODY`). They install the plugin, connect the MCP server, and request to join the org; once a leader approves, they contribute and recall on that passkey identity alone (the hub keys members by pubkey; a wallet address is optional and attached only later). Contribution is explicit and dashboard-driven: open `/sessions` → select a session → click **"Extract Memories"** (client-side extraction with the contributor's selected model — local or session-matched cloud, D-EXTRACTION-HONEST-CLAIM; returns review candidates — does NOT submit) → review memories → choose per-memory org destination (D-12.2) → click **"Submit"**. Contribution has two explicit consent points and nothing reaches the org/hub before Submit: **Extract** sends the session transcript to the contributor's chosen extraction provider (on-machine when local is selected; to the cloud provider at Extract time otherwise), and **Submit** sends the encrypted memory to the org. Neither is automatic. A Cosmos wallet is an optional later upgrade, needed only to **claim accrued VIBE rewards** or pay a mainnet per-memory fee; rewards accrue to a claim-later balance until then. (The org **leader**, by contrast, does need a wallet at creation — to acquire and bond the org slot.)

**Key rotation (epoch advancement).** When a member is removed, the org enters `rotation_pending` state:

1. **Removal triggers `rotation_pending`.** The chain marks the org as pending rotation. The removed member's envelope is deleted.
2. **New submissions are buffered.** Contributors can still submit, but submissions enter a hub-side rotation buffer (the hub `rotation_buffer` store) — not admitted to the chain, not indexed in hub retrieval, not assigned a final epoch.
3. **Leader completes rotation.** The leader derives new epoch keys from K_master via HKDF, generates a new moderation keypair SK_mod(e+1)/PK_mod(e+1), and re-seals envelopes for all remaining members.
4. **Buffer finalizes.** After rotation completes, buffered submissions are released to the chain under the new epoch.
5. **Grace period escalation.** If rotation is not completed within a fixed 72-hour window, the org's submission bandwidth is suspended. (The 72h window is currently a hardcoded constant, not yet a configurable parameter.)

**Revocation semantics.** Epoch rotation provides forward secrecy only. Removed members retain previously-distributed epoch keys and can decrypt content from their membership period.

**Membership state consistency (chain-first, watcher-mirrored).** All consequential membership transitions — add member, remove member, transfer leadership, close org — are **chain-authoritative**: the chain `x/org` handlers enforce them (signer must equal the org's registered leader wallet; role validation; the leader cannot be removed, only transferred; `MsgTransferLeadership` carries the mandatory `new_leader_wallet` and requires the new leader to already be a member). The hub holds **no optimistic membership writes**: a leader's "accept join request" is an *intent* (`join_requests.status = 'confirming'`) that does nothing until the chain `MsgAddMember` confirms; the hub's chain watcher then promotes the request to a member (idempotently, keyed on the chain event), mirrors transfer/close, and on member removal performs the full crypto revocation in the watcher (envelope delete + Umbral kfrag delete + rotation-pending). If the wallet transaction is cancelled, the dashboard reverts the intent (`confirming → pending`) so no phantom half-member is left. A conservative **reconcile sweep** (60 s) heals role / `chain_confirmed` drift against the chain's member set and reverts stale `confirming` requests (>10 min), logging — never auto-deactivating — any divergence. This makes the chain the single source of truth and the hub a strictly derived mirror, eliminating the prior class of "halfway" membership states. *(Known gap — L2 crypto-provisioning: a member's X25519 key is not yet carried on-chain, so join-approved/promoted members are provisioned key envelopes through a separate off-chain registration step; tracked for a future order (GAP-MI-4; design locked in `D-HUB-REBUILDABLE` requirement 1).)*

### 2.4 The Three Software Pieces

WeVibe's alpha architecture has three software pieces: **wevibe-chain**, **wevibe-hub**, and the **MCP server + plugin**. The hub is live and serves retrieval in alpha; there is no separate `wevibe-client` local-retrieval replacement path.

**wevibe-chain (source of truth):**
Cosmos SDK + CometBFT sovereign L1 appchain. Stores encrypted memory blobs, provenance/attribution, org state, serve receipts, and economic state. Validators maintain consensus and replicate state. They never see plaintext memory content.

**wevibe-hub (`wevibe-server`, live coordination + retrieval plane):**
Runs coordination and accounting workflows, and serves the live Qdrant-backed retrieval path in alpha. Hub retrieval is the serving path exercised by the gate harness; the hub is not eliminated or replaced. Authority & exit status: see the Hub Authority Ledger (MASTER.md) and `D-MISSION-INVARIANT`.

**MCP server + plugin (local safety + approval + injection):**
Platform-specific plugin gates register tools in the agent and call a local MCP server (the OpenCode plugin ships today; Claude Code, Cursor, and Cline are planned). This local path enforces guard/sanitization, presents the human approval UX, and injects approved context into the agent. It mediates access to hub retrieval and chain serve receipts; it does not replace hub serving.

### 2.5 Tool Surface

The plugin registers operational tools in the coding agent. Recall query dispatch is automatic in the plugin path (no agent recall tool call), and contribution submission happens in the dashboard flow (no agent contribution tool call). The separation:

**Plugin-registered tools (visible to the agent):**

| Tool | Purpose |
|------|---------|
| `setup_org` | First-run organization bootstrap when the local stack has no org membership. Guides org creation/join flow and local setup handoff. |
| `wevibe_status` | Show org membership and runtime status so the user can verify local setup and connectivity. |
| Consumer-settings tools | Configure the 2×2 consumer matrix: content filter `[Implementations + DNDs]` or `[DNDs only]`, plus injection gate `[Gated approval]` or `[No gated approval]`. Default is `[Implementations + DNDs] + [Gated approval]`. |

The 2×2 settings are the locked consumer model (D-11.5); the current client still exposes the legacy single **risk-appetite** control (`lowest`/`neutral`), and the rename to the explicit content/gate toggles is pending.

**MCP server backend (not directly callable by the agent):**

The MCP server handles local recall mechanics: query construction from local session signals, local embedding/decryption, guard scanning, and packaging candidate data for the plugin gate UX. It does not auto-submit contributions; contribution submission is explicit in the dashboard `/sessions` flow.

All administrative operations (org creation, member invitation, moderation, epoch rotation, keyword management, recovery) are handled by the `wevibe-admin` CLI — shipped as a command within the `wevibe-mcp` package, not a separate program — and hub control-plane workflows.

### 2.6 Product Handbook Map

WeVibe has grown into a family of interoperable services. This general whitepaper captures the shared threat model, encryption design, and contributor experience. Deep dives now live alongside the source for each product:

- `wevibe-chain/docs/WHITEPAPER.md` — consensus layer economics, lifecycle pressure, and keeper architecture.
- `wevibe-docs/WHITEPAPER.md` — client stack (SDK, MCP server, guard, protocol assets) and plugin UX.
- `wevibe-server/**/docs/WHITEPAPER.md` — operational surfaces (Hub, Dashboard, Infra) and their deployment models.

Each directory also contains accompanying PDP and topology documents. Use this handbook for cross-cutting concerns; jump to the per-product sets when you need implementation specifics.

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
**epoch_sk(e)** is the org's Umbral (PRE) epoch secret key — derived deterministically (distinct `info` namespace) by the **leader, on the leader's own machine**. The leader mints all re-encryption key fragments (kfrags) locally and sends the hub **only** the epoch PUBLIC key (`umbral_pk`) and the finished kfrags; the hub never receives `epoch_sk`. Because `epoch_sk(e)` is re-derivable from K_master on demand, it is never split, distributed, or persisted — backing up K_master (BIP39 recovery phrase) is sufficient (D-LEADER-SIDE-UMBRAL-MINT).

The 32 HKDF output bytes **ARE the secret key**: they are interpreted directly as a **canonical secp256k1 scalar** (`SecretKey::try_from_be_bytes`), so `umbral_pk = epoch_sk · G`. This matches each member's PRE/receiving key, which is likewise a raw k256 scalar produced by `@noble/secp256k1` (D-1.3), so the two languages on the two sides of the PRE handshake (TS `@noble` ↔ Rust `umbral-pre`) agree byte-for-byte. A prior implementation mis-treated the HKDF bytes as *seed* material (`SecretKeyFactory::from_secure_randomness`), which derives a **different** scalar and therefore a non-matching public key — silently breaking recall decryption (`Internal validation failed` at open) for the project's entire life. That workaround (CO-216-F4) is **retired**; the canonical-scalar derivation shipped in `wevibe-umbral 2fb7f96` (superseding the earlier `d15d878` attempt) and is guarded by a permanent noble↔umbral parity test vector (`SecretKeyFactory::from_secure_randomness` is no longer used for any key material).

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

**Custody model invariant (ADR-023).** Each organization's K_master is generated as an independent random value.

**Recovery.** BIP39 24-word recovery phrase (ADR-019). Threshold recovery via Shamir 2-of-3 (ADR-024). Encrypted leader vault (`~/.wevibe/vault.enc`) with Argon2id key derivation (t=3, m=64MB, p=4).

### 3.5 Retrieval Architecture

Retrieval is hub-based, with plaintext handling kept local in the MCP server + plugin path. The hub-side retrieval plane (Qdrant index + ranking) is a hub authority — Authority & exit status: see the Hub Authority Ledger (MASTER.md) and `D-MISSION-INVARIANT` (exit: rebuild from chain + keys, `D-HUB-REBUILDABLE`/GAP-MI-3). The pipeline:

#### Context Profiling (Session Start) — *designed, partially built*

The intended model: when a coding session starts, the MCP/plugin profiles the environment — dependencies, directory structure, language, framework versions, current file context — and sends that profile as filter context so the hub can pre-filter candidate memories before vector scoring (a developer working in a Python/Django project should search Python/Django memories, not the entire org corpus). **Today** the plugin collects only a handful of dependency names and folds them into the query text; no structured profile is sent, and the hub runs a plain semantic search over the org corpus, filtered only by org, embedding model, and lifecycle state. The structured profile plus stack pre-filter is a designed extension (DECISIONS.md `GAP-RECALL-HARVEST`), not yet built.

#### Keyword Extraction
Keywords are generated **at the contributor's extraction pass** (DECISIONS.md `D-KEYWORD-AT-EXTRACTION`) — in the same local call that produces the memory's `implement`/`context`/`dnd`/`stack`, using the contributor's selected extraction model (not a fixed startup model). Extraction fetches the org's current controlled vocabulary and classifies against it: terms already in the vocabulary are **in-vocab (green)**; proposed new terms are **suggested-new (red)**. Each memory gets up to 20 keywords with weights normalized to sum to 1.0 (`D-5.4a`); the hub enforces the cap and the sum. The **leader curates at the batch stage** — deleting unwanted keywords (remaining weights re-normalize), and explicitly approving any red term before it commits and joins the vocabulary. This relocates the vocabulary-drift gate from "who generates" to "who commits" (the leader), and keyword generation runs with the richest context (the full session) and the contributor's chosen model. Keyword weights are stored as retrieval metadata used during ranking as a **bonus, not a gate** (`D-RECALL-ALIGNMENT`; plaintext keywords remain an accepted metadata tradeoff — see Section 3.7).

#### Retrieval Representation (situation-centric card; symmetric embedding)
The retrieval vector is **not** an embedding of a keyword bag. Both sides embed a **situation-centric card** with matched `nomic-embed-text` task prefixes (`D-RECALL-ALIGNMENT`): the memory side embeds a **retrieval card** (derived from the memory's `implement`/`context`/`stack`/`dnd`, plus an `anticipated_need` line) with the `search_document:` prefix, computed **at moderator approval** (client-side, where plaintext is already decrypted for review — the hub never sees plaintext and there are no contributor-supplied vectors; the vector travels with the approval request, then is indexed into Qdrant at chain commit per D-6.2); the query side embeds a **deterministic session need-card** (harvested from intent/task/language/stack/frameworks/dependencies/error-strings/files) with the `search_query:` prefix. The keyword-overlap term is a capped additive boost, not a hard gate. This representation is what the offline validation below measured. *(Note: this derived card — `D-RECALL-ALIGNMENT` schema Q-B, locked 2026-06-10 — IS exactly the representation validated at Recall@1 0.95; the extraction object and contributor UI do not change for recall. Dedicated structured situation fields are an optional, sim-gated future refinement that must out-perform this derived card before shipping. Phase 2 is confirm + wire, not enrich.)*

#### Semantic Embedding
At recall time, the MCP/plugin computes the query embedding locally via Ollama (`nomic-embed-text`) and posts that vector to `wevibe-hub` (`/v1/orgs/{org}/query`). The hub's Qdrant index stores plaintext float32 memory embeddings plus keyword metadata (`cid`, `org`, `keyword_weights`, `lifecycle`, `type`). Qdrant stores no decrypted plaintext memory content and no ciphertext blobs.

#### Atomic Memory Format
Each memory is a single, self-contained technical insight:
- **insight** — specific technical knowledge in 1-2 sentences with exact values
- **context** — environment, versions, conditions where this applies
- **avoid** — negative knowledge: what NOT to do and why
- **stack** — specific technologies involved

#### Retrieval Scoring (ADR-025)
```
keyword_boost = Σ(query_weight_i × memory_weight_i)
final_score   = vector_score + γ × keyword_boost

Default: γ = 0.1
```

Vector similarity drives recall (hub-side Qdrant cosine over a configurable recall depth, default 5000). Keywords provide an additive boost (factor γ) to break ties within semantic clusters after context pre-filtering. A δ-proportional cap on the keyword boost is a designed refinement and is not yet implemented.

**Model-origin prior (designed, not implemented).** A soft prior that would deprioritize generic conceptual memories from lower-capability models for higher-capability retrievers (highly specific memories take no penalty regardless of origin). Not in the current scoring path — recall today scores by vector similarity plus the additive keyword boost only.

**Blacklist filtering.** The MCP/plugin excludes locally blacklisted CIDs before approval. (A chain-level quarantine flag after repeated rejections is a designed retrieval-policy hook and is not yet implemented on-chain.)

#### Candidate Fetch + Local Decryption
Hub retrieval returns memory IDs + metadata + matched keywords. The MCP/plugin then fetches each memory's ciphertext from hub storage, decrypts locally through the Umbral sidecar, runs wevibe-guard + human gate, and only then injects approved context.

#### Selective Re-ranking
When top-2 scores are within ε=0.20 (contested query), the MCP/plugin can use the host agent's LLM to re-rank. Fallback: original order preserved on error.

#### Validation (offline simulation)
The situation-centric retrieval direction (DECISIONS.md `D-RECALL-ALIGNMENT`) was validated offline before product implementation. An offline harness that mirrors the extract→embed→rank pipeline (same `nomic-embed-text` model and the production ranking logic) measured, on a model-generated synthetic corpus: embedding a situation-centric **retrieval card** instead of a keyword bag raised Recall@1 from **0.38 to 0.95** (the full proposal — retrieval card + session need-card + boost-not-gate keywords — reached Recall@1 **0.97** / Recall@5 **1.00**). A complementary behavioral test showed that injecting the retrieved memory raised a weaker model's task-correctness by **~0.7 on a 0–3 scale**, capturing ~94% of the perfect-retrieval ceiling, with the pipeline retrieving the correct memory ~96% of the time. These are offline, single-seed, synthetic-corpus results — they validate the direction and gate implementation; they are not production performance guarantees, which await alpha telemetry (corpus generated and extracted by a frontier model — see the D-EXTRACTION-HONEST-CLAIM caveat: the validated figures are frontier-extraction card quality; local-extraction card quality is unmeasured).

### 3.6 Side Channel: On-Chain Metadata

With memories stored on-chain and retrieval served by the hub, metadata is observable across two hosted surfaces. On-chain/public observers can see org IDs, contributor pub keys, submission timestamps, memory sizes, keyword terms/weights (plaintext), serve receipt patterns, and reputation scores. In the hub, Qdrant stores embedding vectors plus keyword metadata (`cid`, `org`, `keyword_weights`, `lifecycle`, `type`), while ciphertext is stored in Postgres/chain paths for retrieval.

The privacy boundary is decrypted plaintext: decryption, wevibe-guard sanitization, human approval, and context injection happen locally in the MCP/plugin path. The honest claim is two-part (DECISIONS `D-EMBEDDING-HONEST-CLAIM`): (1) the hub **cannot decrypt** memory content — plaintext exists only locally (contributor at extraction, reviewer/leader at curation, consumer after the gate); (2) the hub **does hold content-derived data** — clean float32 embeddings (stored-vector noise is disabled per `D-9.5`) plus plaintext keyword weights, which together are a **lossy semantic shadow** of each memory. Published embedding-inversion research (Morris et al. 2023; Huang et al. ACL 2024) shows approximate content recovery from clean embeddings is realistic. The mitigations are **operational, not cryptographic**: Qdrant API auth, internal-network deployment, per-org collection isolation, signed responses. Encrypted vector search (e.g. IronCore `ironcore-alloy`) is the documented evaluation trigger — formally evaluated when an org requests confidentiality-sensitive hosting or public-testnet launch planning begins, whichever comes first. So: the hub never sees your decrypted memory content, but it is not true that nothing content-derived leaves your machine. The same two-part honesty applies upstream: the extraction pipeline is client-side and nothing reaches the org/hub before Submit, but a contributor-selected cloud extraction model receives the session transcript at Extract time — a visible choice scoped by trust-circle parity (D-EXTRACTION-HONEST-CLAIM).

### 3.7 Metadata Visibility Model

WeVibe orgs are public developer communities, not private enterprises. On-chain metadata is intentionally public — it enables discovery, reputation, and the social graph.

**On-chain (public by design):** Org IDs, org topic tags, contributor pub keys, encrypted memory blobs, plaintext keyword terms/weights (discovery signal — "this org covers Redis"), submission timestamps, memory sizes, epoch boundaries, serve receipts (batched per epoch), reputation aggregates, bandwidth consumption, quarantine state.

**Local to the MCP/plugin (the hub never sees these):** Decrypted memory plaintext, local wevibe-guard/blacklist state, and session context profiles. (Embedding vectors and keyword-weight metadata live in the hub's Qdrant; the hub stores ciphertext + vectors but never decrypts — see §3.6 and §8.3.)

Plaintext keywords on-chain are a feature, not a leak. They tell developers what an org covers and help with cross-org discovery. Developers who join an org to boost their LLM need to know what domain knowledge it offers. The keywords serve that purpose.

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
7. **Ciphertext fetch + local decryption.** MCP/plugin fetches each candidate's ciphertext (the on-chain encrypted blob is the source of truth; the hub serves it from a cache plus the Umbral PRE materials) and decrypts locally through the Umbral sidecar.
8. **Blacklist filter.** MCP/plugin checks local blacklist.
9. **wevibe-guard scan.** Same scan on decrypted memory at recall time. Catches payloads undetectable when approved (new rules since approval).
10. **OCR sanitization.** Same format-breaking pipeline.
11. **Artifact extraction and egress enforcement.** Typed artifact extraction: URLs, bare domains, IPv4 addresses, shell commands, package install commands, config directives. Every network-resolvable token flags.
12. **Plugin approval gate.** Plugin renders approval UI with wevibe-guard detection results AND contributor trust signals (pub key, wallet age, rep score, serve count, domain expertise). User sees the memory, sees the flags, sees who wrote it, approves or denies.
13. **Serve receipt.** Each serve/denial entry is signed by the consumer's per-org serve key (ed25519, offline — no wallet, no gas), and the batch transaction is carried by the org serving key under feegrant — the hub is a **carrier, not an attester**. The chain verifies each entry's signature, recomputes the dedup fingerprint itself from `(memory_cid, serve_key_pubkey, epoch)`, rejects unsigned/invalid entries, and counts only verified entries toward trust/attribution/emissions (DECISIONS `D-SERVE-CONSUMER-SIGNED`, option A self-asserted key).
14. **Context injection.** Approved memories formatted as `context:\n{memory content}` and injected into agent prompt.

#### What This Pipeline Catches
- YARA-signature prompt injections
- Credential leakage (AWS keys, API tokens, passwords, connection strings)
- Unicode steganography (homoglyphs; mathematical-alphanumeric injection). Zero-width-character and directional-override detection are designed but not yet implemented.
- Unicode Mathematical Alphanumeric injection (U+1D400-U+1D7FF, 3-char threshold)
- Base64-encoded injections
- External URL injection (scheme-ful and scheme-less)
- Bare hostname references (any TLD)
- IPv4 literal references (with optional port/path)
- Malicious dependency injection
- Config directive injection
- Shell pipe-to-execution attacks
- Previously-rejected memories (via local blacklist; chain-level quarantine is a designed extension)

#### What This Pipeline Does NOT Catch
- Semantic payloads encoded in natural language prose. Mitigated by human review and contributor reputation signals.
- Technically-plausible but subtly wrong recommendations. Mitigated by reviewer domain expertise and contributor rep visibility.

### 3.9 Resolved Architectural Decisions

These decisions are final:

**Individual cross-org reputation is the product.** Serve counts, domain expertise, and rep scores accumulate across all orgs a contributor participates in. This is the social graph for vibe coders. A developer's cross-org profile — "47 Redis memories served 214 times across 12 orgs" — is a public credential. No cross-org reputation *rankings/leaderboards* (avoids toxic competition), but aggregate stats are public and encouraged.

**Task-context skills, not difficulty tiers.** Skills organized by task (deployment, testing, error-handling), not by difficulty level.

**Model origin as soft retrieval prior (designed, not implemented).** Intended as one soft factor in scoring — never a hard filter or gate; not in the current scoring path.

**Documentation seeding for cold-start (designed, not implemented).** Intended so new orgs can import canonical documentation as seed memories; the `source`/`doc_import` path is not built yet.

**Earned Trust decay (D-4.2).** Per-memory keyword weights evolve each chain epoch under the Earned Trust model: serves boost a memory's weight, denials decay it, and memories that are neither served nor denied idle-decay (trusted memories far more slowly than untrusted ones). A memory archives once every keyword weight falls below the retrieval threshold (default 1500 bps). Decay is epoch-driven, not version-scoped.

**On-chain storage, not hosted blob storage.** Approved memories are chain state, replicated by validators. No VPS dependency.

**Pending memories: commitment on-chain, blob off-chain.** Contributors submit only a commitment (hash, org ID, contributor pubkey, expiry epoch, size) on-chain. The encrypted blob is delivered to the reviewer through temporary off-chain channels (local transfer, P2P, or org-hosted mailbox). If approved, the finalized encrypted blob goes on-chain. If rejected, the commitment is removed and the temporary blob is deleted. This ensures rejected content never enters committed block data. (Automatic expiry/auto-purge of unreviewed commitments is a designed lifecycle step — see §13.)

**Hub-based retrieval with local decryption.** wevibe-hub runs Qdrant vector search over embeddings + keyword metadata; MCP/plugin computes query embeddings locally, receives IDs + metadata + matched keywords, fetches ciphertext, and decrypts locally before sanitization/injection.

**Contributors are paid by the network; members pay orgs for access.** Contributor rewards are contribution-only network emissions. Access demand is separate: members pay orgs in VIBE for recall access, and leaders earn from that demand leg (with settlement/burn mechanics in active alpha build-out).

**Serve receipts: public reputation, pseudonymous retrieval.** Contributor reputation (serve counts, domain tags, payout amounts) is public on-chain. Retriever identity is represented by a per-org pseudonymous serve key — not the user's global contributor identity. This separates "my knowledge helped others" (public) from "this exact user needed this exact memory" (pseudonymous). Users can optionally link their org serve keys to their public profile as a learning trail. **Signing model:** each serve/denial entry is signed by the consumer's per-org serve key and the chain counts only verified entries (DECISIONS `D-SERVE-CONSUMER-SIGNED`); the serve key is self-asserted in the signed entry, preserving pseudonymity, and the hub — which holds no serve key — can no longer mint serve content.

**Serve receipts (batching is the target).** The chain accepts batched serve submissions (`MsgSubmitServeBatch`) and the hub holds the org-whitelisted serving key for the tx envelope. In the current client, denials are queued and batched while individual serves are forwarded to the hub as they occur; per-epoch batching of serves into one transaction per user is the intended optimization, not yet the live path. **Forgery-resistance:** per-entry consumer signatures are verified on-chain (`D-SERVE-CONSUMER-SIGNED`) — the chain rejects unsigned/invalid entries, so the serving key is no longer trusted for serve content.

**Four-button approval UX (D-RECALL-FEEDBACK-FOUR-BUTTON).** Plugin offers: [Accept] (memory injected, serve receipt queued, contributor earns), [Deny] (memory blocked for this session/context only — a neutral context signal, NOT a corpus down-vote; no denial event emitted to the chain), [Block] (permanent personal blacklist AND a global corpus denial signal — the load-bearing negative-signal path that feeds Earned-Trust decay via `MsgSubmitDenialBatch`), [Report] (memory blocked and escalated into the org's moderation/accountability path — see Sections 5.5–5.8). The Deny-vs-Block split is load-bearing: Deny is a no-op on the corpus (context ≠ quality); Block is the negative signal that drives decay. Public orgs with payouts may require serve receipts.

### 3.10 Session Attestation (Roadmap, Post-Mainnet)

Sessions produce memories. Without provenance attestation, a contributor could paste fabricated "coding sessions" into the extraction pipeline and farm reputation. Everything downstream — difficulty scoring, quality grading, reputation — depends on knowing the session actually happened.

**Current alpha posture:** org leader curation is the trust layer. We do not yet have a generalized attestation rail in production.

**Roadmap direction (D-ATTEST-ROADMAP):** the optional subsystem described here is the seed for a post-mainnet **pluggable attestation framework**. Separate attestation components will plug into the chain and validate session claims either **cryptographically** or **via API-backed trust services**. The target claim shape is explicit session provenance, for example: *"user X using LLM model Y took N turns to solve problem Z."*

This is a major roadmap item and the infrastructure is not there yet. It is carried as a forward design, not claimed as live.

**Proof-tier generalization (PROPOSED / PENDING-SPIKE, `D-ATTEST-PROOF-TIER`):** post-mainnet, the on-chain `x/attestation` socket is planned to generalize the Tier-0/1/2 provenance grades below into a single **typed proof artifact** — `{ proof_type, trust_label, … }` where `proof_type` is one of `tee_receipt`, `zktls_proof`, or `zkml_proof` — each **verified off-chain before any on-chain anchoring** (preserving the Phase-0-verify / Phase-1-anchor structure). The **CO-282** spike proved the `zktls_proof` path technically real for closed frontier models with session privacy intact, but defers it on an asymmetric ~6 KB MPC sent-side commitment cap; the tier is carried as a forward design, not built (see DECISIONS `D-ATTEST-PROOF-TIER`).

#### Lineage from the optional subsystem (seed designs)

**Tier 1: CommitLLM Cryptographic Receipts (Open-Weight Models).** CommitLLM (Lambda Class, MIT licensed) is a cryptographic commit-and-audit protocol for open-weight LLM inference. The receipt proves "this response was produced by this exact model with these exact weights." Limitation: only works for open-weight models where the verifier has the public checkpoint.

**Tier 2: Proxy-as-Trust-Layer (Closed-Weight Models).** For closed-weight API models, traffic can route through a WeVibe-controlled session proxy that content-addresses each turn and signs the transcript with WeVibe's key. This is weaker than Tier 1, but provides a practical API-trust path.

**Tier 0: TEE-Attested Confidential Inference (open- or closed-weight).** GPU-TEE confidential inference (e.g. OpenRouter / Phala / RedPill on Intel TDX + NVIDIA H100 Confidential Computing) signs each request/response inside the enclave and ships a remote-attestation quote binding the signing key to the loaded **model measurement** — a hash of the actual checkpoint, not a marketing name. It is the strongest of the three: it covers closed-weight models (unlike Tier 1) without trusting a WeVibe relay (unlike Tier 2), and is recorded as the `tee-attested` provenance grade (DECISIONS `D-ATTEST-TEE-TIER`). Locked constraints: it is **optional, opt-in, and never a contribution gate** (requiring it would exclude the local/smaller-model users WeVibe most serves); verification is **off-chain** (an attestation-verifier checks the Intel DCAP + NVIDIA quotes and emits a re-verifiable assertion — the chain is never asked to verify a TEE quote); a `certified_model` tag certifies **session provenance, not memory text** (a `derivation` flag records whether the submitted memory is verbatim from extraction or edited after); and it is carried as a pluggable adapter, not a hard dependency.

**Standardizing the extraction model (the controllable half).** "Weaker models surface weaker memories" has two heads: the *production* model in the user's coding suite (uncontrolled — and sometimes the most valuable signal, when a weak model solves a hard problem) and the *extraction* model that distills a session into a memory (inside WeVibe's own pipeline). WeVibe standardizes the latter by **pinning the extraction model** (DECISIONS `D-EXTRACTION-MODEL-STANDARD`), removing weak-extractor variance with no attestation required; attesting the production model is the optional bonus on top.

### 3.11 Two-Layer Difficulty Scoring (Roadmap Consumer, Requires Attestation)

Two-layer difficulty scoring is the evolutionary continuation of the optional design above and a likely early consumer of attested session claims once the pluggable framework exists.

Like attestation, this is post-mainnet roadmap work: the chain-side plumbing and integrations are not yet in place. What exists today is the *storage* for the result, not the scoring: the chain accepts plain `difficulty` and `quality` values (1–10) on an attested-memory record and maintains a simple per-contributor difficulty histogram (`x/reputation`). The Layer-1 structural formula and the Layer-2 grading LLM described below are the unbuilt target that would populate those values automatically.

How attested difficulty should enhance the economic layer and/or social-graph layer is intentionally **TBD**. What attestation does settle is the *trustworthiness of the inputs*: the model and the turn count become cryptographically grounded (DECISIONS `D-ATTEST-TEE-TIER`). The composite claim "user X using model Y took N turns to solve problem Z and drew L negative signals" is therefore rendered as a **per-field graded provenance** object in the social-graph layer — each field labeled by its own grade (attested model/turns, native-on-chain user/denials, descriptive problem tag) — display/badge-only and never coupled to VIBE (DECISIONS `D-GAMIFICATION-PROVENANCE`).

#### Layer 1: Structural Signal (Automated, Cheap)
Model capability coefficient × turn count × (1 + 0.25 × failed alternatives). Computed from session structure without understanding content.

#### Layer 2: LLM Grading (Semantic, Authoritative)
Separate grading LLM evaluates non-obviousness, specificity, and reasoning progression. Temperature 0, deterministic, hash-seeded. The grade and session hash are committed together.

---

## 4. Local Architecture (MCP Plugin + Sidecars)

### 4.1 Local Footprint and Responsibilities

In alpha, the local software footprint on a member machine is:

- **MCP server + agent plugin** (OpenCode today; Claude Code, Cursor, Cline planned)
- **Umbral decryption sidecar**
- **wevibe-guard binary**

The local path is responsible for both recall gating and contribution packaging.

**Recall-side responsibilities (local):**
1. Compute the query embedding locally via Ollama (`nomic-embed-text`).
2. Send the query vector (plus context filters) to the hub endpoint (`/v1/orgs/{org}/query`).
3. Fetch candidate ciphertext (chain is the source of truth; the hub serves it cached, with Umbral PRE materials).
4. Decrypt locally through the Umbral sidecar.
5. Run wevibe-guard sanitization/policy checks.
6. Present the human approval gate and inject only approved context.

**Contribution-side responsibilities (local):**
1. Extract session learnings only when the contributor clicks dashboard `/sessions` → **"Extract Memories"** (local candidate generation, no submission yet).
2. Sanitize, encrypt, and sign submission material.
3. Submit commitment data to chain (org pays submission bandwidth), then follow moderation/finalization flow.

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

1. Join the org and configure the local MCP/plugin with org context, hub endpoint, and chain RPC.
2. Receive sealed key envelopes for the current epoch (reviewers/leaders additionally receive current moderation key material).
3. Initialize local services used by the plugin path (Ollama embedding model, Umbral sidecar, wevibe-guard).

After bootstrap, recall runs as request/response against the hub retrieval service. The local machine does not download the org's full memory corpus and build its own vector index. Local state is operational (keys/config + approval/serve-receipt queueing), while authoritative memory storage and vector retrieval remain hub/chain-side.

### 4.4 Recall Flow (Auto-Query + Plugin-Gated Injection)

```
Developer works in their coding session
     │
     ▼
  Plugin harvests local session signals
  (recent prompt/edits/stack; intended alpha model)
     │
     ▼
  Plugin auto-submits recall query via local MCP
  (no agent recall tool call)
     │
     ▼
  Local MCP/plugin path:
      1. Build keyword-weight query params from session signals
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
      ├── [Gated on risk] (default direction, D-RECALL-GATE-ON-RISK):
      │      auto-inject + attest the safe majority; pop up ONLY for
      │      guard-flagged / low-trust / low-confidence candidates.
      │      Quality signal (attest/deny) batched in a review tray.
      │
      ├── [Gated approval]: explicit user action required on every candidate
      │      ├── ACCEPT + ATTEST → inject context + attest serve on-chain
      │      ├── DENIED → block memory and record feedback
      │      └── REPORTED → file on-chain report (is_reported); memory keeps serving until resolved
      │
      └── [No gated approval]: inject directly + attest serve on-chain
      │
      ▼
  Agent continues with or without memory
```

This is the intended alpha model: plugin-side live-signal harvesting for recall query construction, local decrypt/guard checks, and gated injection by default. The product contract above is fixed; implementation sophistication is still being hardened during alpha.

> **Alpha-status honesty.** [Gated approval] is the product contract and the locked default (D-11.5). The injection gate (`gateMemories`) is defined but not yet wired into the recall path, so memory injection is currently ungated — a known defect, BLOCKING-FOR-EXTERNAL-USERS, tracked as GAP-MI-1.

### 4.5 Contribution Flow

```
Developer opens dashboard `/sessions` and selects a captured local session
     │
     ▼
  Click **"Extract Memories"**
     (client-side extraction with the contributor's selected model — local or session-matched cloud, D-EXTRACTION-HONEST-CLAIM; returns review candidates — does NOT submit)
     │
     ▼
  Review/edit candidates and choose per-memory org destination (D-12.2)
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

**Pending memory lifecycle:** Only the commitment is written on-chain initially. The encrypted blob is reviewed out-of-band; approval finalizes chain state, while rejection/expiry deletes the pending commitment and temporary blob. Sessions stay local unless/until the contributor explicitly submits.

### 4.6 Reviewer Flow

Moderation and review are handled in the hub's hosted web dashboard (`wevibe-dashboard`), not a local client UI. Reviewers and leaders process pending submissions there, apply approve/deny decisions, and push finalization actions through the hub-to-chain path.

### 4.7 Plugin Architecture

Each coding agent gets its own plugin codebase:

| Agent | Plugin Type | Hook Mechanism |
|-------|-------------|----------------|
| OpenCode | JS/TS plugin in `.opencode/plugin/` | `tool.execute.before`, custom tools via `tool()` |
| Claude Code | Plugin with `.claude-plugin/plugin.json` manifest | PreToolUse hook, `permissionDecision: "deny"` |
| Cursor | Hooks + marketplace plugin | Claude Code hook format compatibility |
| Cline | VS Code extension + hooks | `.clinerules/hooks/`, custom hook system |

All plugins call the same local MCP server and the same wevibe-guard binary (`WEVIBE_GUARD_BIN`). Decryption is handled through the local Umbral sidecar; retrieval/search remains in hub APIs.

OpenCode 1.16+ ships a built first-run onboarding hook (DECISIONS.md `D-PLUGIN-ONBOARDING-HOOK`): `DialogConfirm` → Touch ID/passkey confirmation → completion screen. Identity creation is step one only; reputation/rewards come from joining orgs and contributing approved memories, never from identity creation itself.

The OpenCode plugin exposes `/wevibe-setup`, `/wevibe-connect`, and `/wevibe-status`, and nudges the user to dashboard join/connect flow after three local sessions. Identity boot is lazy (no biometric prompt at plugin startup). Non-secret local identity state is plugin-owned and stored at `~/.wevibe/identity.json` (DECISIONS.md `D-SIDECAR-PLUGIN-OWNS-STATE`).

Install/uninstall is zero-config via `wevibe-admin install-opencode` and `wevibe-admin uninstall-opencode`. The local keystore is seed-based with BIP39 recovery, and short-code pairing reconciles dashboard + plugin onto one contributor pubkey.

**Status:** only the **OpenCode** plugin (`wevibe-opencode-plugin`) ships today, including the onboarding hook, lazy identity boot, sidecar state ownership, installer flow, and short-code pairing. The Claude Code, Cursor, and Cline rows are target integrations — planned, not yet built.

### 4.8 Chain-Resolved Hub Endpoints (Zero-Config Transport)

WeVibe keeps one canonical chain as network source of truth, with a public chain RPC as the single stable client anchor (default endpoint, env-overridable for operators). The chain is the org directory. Each org carries a leader-configurable `hub_endpoints` field — an ordered list of 1–3 network URLs for transport redundancy/failover — set via a leader-signed on-chain setter, distinct from `hub_serving_address`, the Cosmos key that authorizes serve/deny and response authority. Transport and authorization are intentionally separated (DECISIONS `D-CHAIN-RESOLVED-HUB-ENDPOINT`).

At session start, the plugin resolves per-org endpoint routing from chain RPC once (biometric-free, because org metadata is public), updates local config, and persists per-org sidecar state keyed by globally unique `org_id` so state remains stable across hub migration (DECISIONS `D-SIDECAR-PLUGIN-OWNS-STATE`). The endpoint list is priority-ordered: the plugin uses the first and fails over to the next on connection, health, or signature-verification failure. All endpoints are replicas fronting the org's single on-chain serving identity. In this model, the consumer never hand-configures a hub URL.

Hub transport remains untrusted: hub responses are signed by the org's on-chain serving key and verified by the plugin against `hub_serving_address` (DECISIONS `D-HUB-RESPONSE-SIGNED`). The response-signing contract lives in `wevibe-protocol` so hosted and self-hosted hubs conform to one verification path. If the `hub_endpoints` list changes, clients auto-switch silently and show a one-time passive toast (DECISIONS `D-HUB-ENDPOINT-CHANGE-TOAST`).

Why this path: self-hosting remains a first-class leader right, but endpoint operations stay abstracted from end users. Resolving transport from the same chain source of truth preserves one path and avoids creating a second directory service every self-hosted hub would otherwise need to mirror and keep in sync.

**Status (near-term locked design, not yet shipped):** chain-resolved endpoint routing + hub-response signing + endpoint-change toast are target architecture for upcoming alpha work; current clients still rely on manually configured hub URLs.

---

## 5. Content Review

### 5.1 Review Flow

Contributed memories are submitted (encrypted; only the org's reviewers can decrypt) into the hub's pre-commit pipeline and are visible only to reviewers and leaders via the hub's hosted dashboard (wevibe-dashboard). Pre-commit memories live in hub PostgreSQL — never on chain and never in Qdrant — until the leader commits a batch on-chain.

This review layer is not just about safety. It is the quality gate that protects WeVibe's social layer: the public contributor/org attribution counters and badge status signals are only meaningful if low-quality or malicious memories are filtered before commitment.

**Moderation is always-on advisory; the leader is the always-on backstop (D-MODERATION-ADVISORY, D-LEADER-SOLE-SIGNER, D-6.7).** A submitted memory flows directly into the leader's curation/chain-submit queue — there is no mandatory "awaiting moderation" gate, and there is no on/off toggle (the former per-org `moderation_required` flag and the `required_approvals` quorum are both removed). `can_moderate` is an independent per-member **capability** the leader grants à la carte (via `MsgSetMemberCapabilities`) — *not a role*; a leader can run an org with **zero members granted `can_moderate`** (capabilities are off by default). Members holding `can_moderate` are *helpers* who produce **recommendations, never gates**:

- a **per-submission vote** — `approve` or **flag (against committing)** — surfaced to the leader as a tally on the memory's card;
- a **per-keyword vote** — `include` / `exclude`, primarily on the red (suggested-new) vocabulary terms — surfaced next to each keyword to guide the leader's keep/remove decision.

Neither vote changes the pipeline state. The **leader** reads the recommendations and decides: curate the keywords (§3.5 / keyword curation), commit the batch (signing on-chain — the sole publishing authority), or **terminally deny** the memory. Advisory vote history (from members holding `can_moderate`) is org-local accountability (it is not chain-level enforcement). This delegates review *labor* without ever delegating the leader's *signature* — the property the entire anti-capture model rests on (§9.7).

The pre-commit lifecycle is therefore: a contributor submits → the memory enters the leader's curation queue (`pending_keyword`) with keywords already attached (§3.5) → the leader curates keywords + verifies (`pending_chain`) → the leader signs the multi-message batch commit (`committed`); a leader denial is the only terminal removal before commit. Advisory votes (from members holding `can_moderate`) ride alongside as metadata at any point before commit.

### 5.2 What Review Can and Cannot Catch

**Can catch:** Prompt injection patterns, malicious URLs, credential exfiltration, spam, duplicates, off-topic content, obvious technical errors, stale references, Unicode steganography, memories too generic for the org's target model/stack.

**Cannot catch:** Subtly incorrect technical guidance that appears plausible, semantically-encoded malicious instructions in natural language prose, context-dependent correctness.

### 5.3 Reviewer Trust Boundary

Reviewers see pending content in plaintext on their local machine. The system is only as secure as reviewer judgment, honesty, and endpoint security. Mitigations: epoch-scoped mod keys, steganography detection, contributor reputation signals visible during review.

The leader is the final authority on what enters the chain. Chain commits — batch memory approvals, denial settlements, report acknowledgments, dispute publications — are signed by the leader's wallet and the leader's wallet alone. The leader bears full responsibility for the on-chain record they produce. Internal advisory votes (from members holding `can_moderate`) and approval histories are an org-local accountability primitive maintained outside the chain.

### 5.4 Contributor-Signed Canonical Body as Verification Anchor

Every memory's submit-time canonical body includes three fields that, together with the contributor's signature over the body, form the public-escalation verification anchor:

- **plaintext_hash** — `sha256(salt || plaintext)`, computed by the contributor before encryption
- **salt** — a fresh 32-byte random value generated per submission
- **ciphertext_hash** — `sha256(ciphertext)`, where ciphertext is the AEAD output

The contributor signs the canonical body with their own key. The canonical body, the signature, and the ciphertext all travel through moderation and land on the chain together. The leader's batch chain commit includes the contributor's signature; the leader cannot modify the signed fields without invalidating the contributor's signature.

This is what makes the public report tier (§5.6) trustworthy without trusting the leader: any future reveal of plaintext + salt can be verified against the on-chain plaintext_hash via direct sha256 check, and the on-chain ciphertext can be verified against the on-chain ciphertext_hash. The leader is removed from the verification chain entirely. A captured leader cannot poison the anchor because they do not sign it.

The contributor cannot substitute ciphertext between submit and chain commit because their signature binds the specific ciphertext_hash. The contributor cannot later claim a different plaintext at public escalation because the plaintext_hash binds them to the specific salted hash, and SHA-256 collision resistance prevents producing a different (plaintext, salt) pair that hashes to the same value.

**Why the signature now covers all three fields and not just plaintext_hash.** An earlier design (CO-026, reverted via DMO-027) signed only `plaintext_hash` without salt and without ciphertext binding. That shape was vulnerable to contributor + leader collusion: the contributor could sign one hash while the leader committed ciphertext encrypting different content, and the asymmetry was undetectable. The current shape closes the gap by binding all three fields jointly in a single signature. The leader has no signing role; the contributor has no opportunity to substitute ciphertext after signing.

**Residual risk: contributor with stolen signing key.** If the contributor's signing key is stolen, an attacker can produce signatures binding any (plaintext, salt, ciphertext) tuple. This is a key-management problem, not a cryptographic-anchor problem — no design at the cryptographic layer can defeat it. Mitigation lives at the wallet/identity layer (D-1.1, D-1.3, ARCH-G9 for the future BIP-32 PRE-identity separation). The on-chain ciphertext + sealed-box wrapped DEK remain as a final backstop: any future epoch key disclosure allows independent decryption and exposes the actual content after the fact.

**Salt design rationale.** Without a salt, `sha256(plaintext)` is vulnerable to rainbow-table attacks for low-entropy plaintexts (short memories, common error messages, well-known technical advice). A 32-byte random salt makes rainbow tables infeasible (2^256 salt space per plaintext). The salt is included in the canonical body the contributor signs, stored on-chain alongside the commitment, and revealed by the reporter at public-escalation time. Salt is not secret — it is context-hiding.

### 5.5 The Report Model

A report is the high-friction accountability primitive (denials are the low-friction ranking signal — §5.8). It has two stages: an on-chain report with a fixed org-resolution window, and — only if the org fails to resolve it — a public plaintext-reveal escalation. Most reports never reach the second stage; it exists precisely for the case where the org is captured.

**What ships today vs. the target.** The contributor-signed verification anchor (§5.4) ships today and is the cryptographic foundation: every committed memory is bound to its contributor, so anyone can verify a revealed memory against the on-chain anchor. The flow below — the consumer-filed on-chain report, the `is_reported` / `was_reported` / `report_cleared` flags, the leader `clear_report` transaction, the storage-deposit clawback, and the public plaintext-reveal escalation — is the **locked target design**, not yet shipped. (Today's code records a leader-gated report and resolves it off-chain via the hub; the consumer-filed, chain-recorded flow below is the build target — see DECISIONS GAP-TIER2-EXPOSE.)

**Stage 1 — On-chain report.** The consumer files the report from the plugin. Filing is **wallet-gated**: the reporter signs the report transaction from their own wallet and pays gas, and the reporter's **public wallet is recorded on-chain**. Filing sets on-chain flags on the memory: **`is_reported`** (an open report stands), **`was_reported`** (reported at least once — permanent, never unset), and later **`report_cleared`** (set only if the leader dismisses — see Stage 2). Filing is rate-limited on-chain to **one report per reporter per 24 hours** (block-height scoped), so one consumer cannot spam reports, and it starts a **one-week resolution window**. Because the reporter is wallet-gated and gas-paying, the client broadcasts the report (and the later expose) **directly to chain RPC** — the canonical public RPC, env-overridable — with a **WeVibe-operated relay as retry fallback only** (DECISIONS `D-REPORT-DIRECT-BROADCAST`); no user-signed accountability transaction requires WeVibe infrastructure to reach the chain, so neither a captured org nor WeVibe itself can silently suppress the filing. *(Status: the direct-broadcast model is decision-locked; client build is pending — GAP-MI-5 — and the Tier 2 expose loop it serves remains GAP-TIER2-EXPOSE.)* The memory **keeps serving** during the window; removal happens only on resolution, so a wave of frivolous reports cannot disrupt an honest org.

**Stage 2 — Resolution, or public escalation.** Resolution is decided off-chain by the org — by the **leader directly** (advisory input from members holding `can_moderate` may inform the decision, but the leader is the sole authority):

- **Valid →** the leader **claws back the memory's storage deposit** and **deletes the ciphertext blob**. A minimal **on-chain transparency record** is kept — that this memory was reported and the report was upheld, attributed to the contributor. The plaintext is *not* revealed; the deleted blob plus the upheld record are the outcome.
- **Dismissed →** the leader clears the open report with a `clear_report` transaction (`report_cleared = true`, `is_reported = false`; `was_reported` stays true).

If the org does neither and **the one-week window elapses with no leader action**, **or the leader dismisses** (`report_cleared`), the original reporter's **public report** unlocks. The reporter publishes a public on-chain report that **reveals the memory plaintext**, anchored to the contributor-signed hash (§5.4) so anyone can verify it independently; the block explorer renders it with full provenance — contributor pubkey, leader pubkey, org ID, original commit height, and the reporter's signed reason. Because all chain data is public, the plaintext reveal is irreversible and is therefore the **last** step — only after the org has had its governed window. One escalation per report; no re-publishing, no escalation loops. This recourse is what makes capture economically unsustainable: a captured org cannot both refuse to act and prevent the public record.

### 5.6 The Expose Gate, Dispute, and Silent Acquiescence

The ordering above is deliberate and gated:

1. **File (no reveal).** The on-chain report names the memory and reason, sets `is_reported`/`was_reported`, and starts the one-week window. The plaintext is not revealed. The filing is a permanent, unsuppressable record (backed by the guarded fallback relay).
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

The reporter's own dashboard view is the one exception: each reporter sees a private list of their own published reports with copy-link buttons. That is ammunition for the reporter to share, nothing more. In the designed report loop the leader receives a notification when a report is filed against a memory they committed, plus a response interface labeled with the response-window timeout and the leader's one-shot response — again, only to the leader.

### 5.8 Silent Denial as Cheap Negative Signal

The plugin's four-button approval UI (Accept / Deny / Block / Report) gives the consumer two complementary negative paths. Reports — the on-chain filing and, if unresolved, the public escalation — are the high-friction, high-stakes accountability primitive described above. Denials are the low-friction, low-stakes signal that feeds retrieval ranking.

Denials and reports are status/accountability signals, not direct payout triggers. WeVibe keeps the social signal and economic payout paths decoupled.

Clicking Deny is silent: no confirmation modal, no required reason, no new UI surface. The reason field is optional. There is no gating — any consumer, trial or paid, may deny any memory. There are no caps, no rate limits, no reputation weighting. Every denial counts as exactly one denial event.

A denial does two things:

1. **Local suppression.** The memory is never re-served to this developer.
2. **A negative signal to the retrieval layer.** The denial flows to the org's local retrieval/storage component, which mirrors the chain's decay arithmetic locally. Retrieval ranking degrades immediately — a memory denied N times since the last on-chain settlement ranks lower than its chain-recorded weight would imply.

**The optimistic ledger.** The chain remains the eventual source of truth for keyword weights and the decay state of every memory. Between on-chain settlements, the local retrieval layer maintains an optimistic mirror: for each memory, the locally-applied decay reproduces the chain's Earned Trust formula (DECISIONS.md D-4.2) — recomputing `denial_rate`, the trust gate, and the per-event decay/boost using the same parameters the chain will apply at settlement. The arithmetic mirrors the chain's exactly, so optimistic and authoritative ranking states are indistinguishable at retrieval time. When the relay's batch commits on-chain, the local mirror reconciles to the new authoritative weights and resumes from the new baseline.

**Why silent and frictionless.** A denial is the cheap, low-stakes negative signal. UI friction (required reason, confirmation modal, rate caps) suppresses the signal and starves the decay model that depends on it. Reports remain the high-friction path with reporter accountability for cases where a single signal needs disproportionate weight. The two paths are complementary, not duplicative.

**Why no caps or reputation weighting on denials.** A single consumer cannot drive a memory to archived via denials alone. Archive requires every keyword weight to fall below the retrieval threshold (D-4.2, default 1500 bps), which under Earned Trust requires either sustained denial pressure pushing the denial rate above the trust gate, OR no offsetting serves over a long quiet period — both of which require organic volume that one actor cannot fake. Caps would protect against an abuse that has no payoff: a malicious consumer who spams denials can at worst suppress memories from their own recall queue (which they could do anyway through local blacklist). Reputation weighting would create a class system where senior consumers have heavier "votes" than new joiners, require online reputation lookups on the recall hot path, and reproduce the reporter-accountability infrastructure for a signal whose semantics are explicitly lighter.

**Hub-relay settlement (current model, DECISIONS `D-RELAY-THROUGHPUT`).** Serve and denial events settle on-chain through the hub→chain relay, not a manual leader action. The relay packs multiple epochs' `MsgSubmitServeBatch` / `MsgSubmitDenialBatch` messages into one multi-message commit-mode transaction per org, carried by the org's whitelisted serving key with gas paid via the org-account feegrant; denials are queued and batched while individual serves are forwarded, per §3.9. The optimistic ledger means retrieval ranking already reflects every denial in real time; the relay commit is purely the settlement act — making the decay permanent across local-storage restarts and contributing to the org's on-chain activity record. *(An earlier design — a leader-driven settlement cadence with a single "settle" button in the dashboard — is superseded by the relay model and is no longer the path.)*

The leader does not review individual denials. Denials are quantitative consumer signals, not editorial content; the relay settles accumulated decay, and the leader does not adjudicate individual user preferences.

> **Open design item.** Settlement-lag bounds and pending-denial persistence across hub migration are an open design item — GAP-MI-7.

---

## 6. Developer Reputation and Social Graph

### 6.1 The Reputation Wedge

No platform captures the problem-solving exhaust vibe coders generate every day. GitHub shows repository output. LinkedIn is self-reported. Most AI-assisted debugging knowledge disappears in terminal history.

WeVibe's wedge is to turn that daily exhaust into public, verifiable social reputation: contribution history, serve attribution, and badge progression tied to real chain events. Memory retrieval quality is the utility loop; reputation and status progression are the dopamine loop.

### 6.2 Serve Attribution Is a Social Signal (Not an Economic Primitive)

Serve attribution is kept on-chain and made public, but it is no longer part of VIBE payout mechanics.

When a memory is served, attribution increments aggregate counters for:

- the **contributor** whose memory was served, and
- the **org** where the serve happened.

Those counters are source-of-truth chain state, read through RPC, and rendered by the social graph.

This decoupling is deliberate. Retrieval-based rewards were removed from economics because they are gameable (manufactured retrieval loops can fake demand). We keep serve attribution anyway because it is valuable as public status/reputation evidence. The signal remains; the payout coupling is removed.

### 6.3 Identity and Attribution Model

Serve attribution uses per-org pseudonymous serve keys. Each user has:

- `global_contributor_key` — public identity for authorship and contributor profile reputation
- `org_serve_key` — per-org pseudonymous key used for retrieval/serve attribution events

`org_serve_key` proves org membership activity and supports deduplication rules without auto-linking all retrieval behavior across orgs. Users can opt in to publicly link selected org activity to a profile.

**Reputation is keyed by the passkey identity, not the wallet.** A contributor earns and accrues reputation under their passkey-derived contributor key from the moment they contribute — no wallet required. Earnings accrue on-chain to the same passkey identity (a claim-later balance), not to the wallet; a Cosmos wallet is an optional later upgrade for *withdrawal authority and leadership/bond*, not identity, and linking one does not by itself move reputation or unlock earnings. Carrying reputation onto a wallet is a deliberate, explicit **migration**, modeled as an **on-chain, dual-signed alias** (passkey pubkey → wallet address) that is gated by the contributor's own memory-contribution trail and recorded with an `is_migrated` flag. Until migration, reputation stays keyed to (and is resolved by) the passkey pubkey; after migration it resolves to the wallet via the alias. The chain is append-only, so no prior history is rewritten. (See DECISIONS.md `D-REPUTATION-KEYED-BY-PUBKEY` and `D-MIGRATION-ONCHAIN-ALIAS`.)

### 6.4 Open-Source Social Graph Client (Forkable, Self-Hostable)

The social graph is an open-source display client over chain RPC. Anyone can fork and self-host it.

Its role is presentation, not consensus. Chain state remains the source of truth; the social graph reads and renders:

- contributor serve/contribution counters
- org-level aggregate serve/contribution counters
- reputation summaries and domain views
- badge status and per-org badge breakdown

Because the client is forkable, WeVibe maintains canonical badge criteria in the reference display spec (see §6.5, §7.2) so tier names stay comparable across forks.

### 6.5 Badge Families (Status-Only, Per-Org Scope)

Badges are a first-class social feature with three families:

1. **Serve-milestone badges** — thresholds based on how often a contributor's memories are served.
2. **Rarity-tier badges** — derived from per-memory keyword supply/demand tiers, computed read-time by the social-graph display client in alpha; on-chain freeze is deferred to mainnet (GAP-RARITY-1, D-SG-3).
3. **Contribution-volume badges** — thresholds based on approved memory contribution volume.

Scoping is per-org, with profile breakdowns instead of a single global ladder. This keeps competition bounded and useful: you can see where someone built reputation, without forcing all contributors into one network-wide leaderboard.

Badge status is strictly non-economic: no VIBE reward, no emissions multiplier, no payout coupling.

**Alpha status honesty:** badge rendering and the rarity-tier pipeline are near-term alpha work. Rarity-tier semantics are design-stage under GAP-RARITY-1 and are documented as such; they are not presented as fully rolled out today.

### 6.6 Signal Integrity and Anti-Gaming

Even as status-only signals, attribution must stay hard to fake. The reputation layer deduplicates repeat serves and caps serves per memory per epoch. Two further guardrails — discounting serves a user makes to their own memories, and diminishing returns on repeated serves — are designed but not yet applied in code. Serve deduplication is hardened: the chain recomputes the fingerprint itself — from the memory id, the user's per-org serve key, and the epoch window — rather than trusting a client-supplied identifier, and each serve/denial entry carries a per-entry consumer signature the chain verifies, counting only verified entries (DECISIONS `D-SERVE-CONSUMER-SIGNED`).

Human review (§5) remains the first anti-gaming gate: low-quality memories should fail approval before they can accrue social status. Denial and report systems (§5.5–§5.8) provide additional negative feedback signals without coupling directly to token payout.

Optional attestation dimensions (difficulty/verification quality) remain roadmap/alpha-track additions and are documented as near-term expansion points, not as universally deployed defaults.

---

## 7. Organization Social Graph

### 7.1 Public Discovery Interface (opt-in)

**Visible to non-members (if public):** Organization name, specialization, description, memory count, member count, age, leader identity, total serves, social badge summary, and two unfakeable org-health signals introduced below.

**Not visible to non-members:** Memory content (encrypted on-chain), member identities (privacy-preserving), review history, payout rules.

**Unfakeable org-health signals.** Discovery surfaces two behavioral metrics that capture-resistant by construction:

- **Leader last active.** Aggregated timestamp of the most recent on-chain action signed by the org leader's wallet — batch memory commits, denial settlements, member changes, report responses, epoch rotations. The signal requires a real wallet signature on a real transaction paying real gas. A dormant or captured org cannot fake it.
- **Voluntary departure rate.** Members who left of their own accord in the trailing 90 days, expressed as a fraction of total membership. Departures are first-class on-chain events; sybils can be invited and can file reports, but they cannot fake people walking away. A cohort exiting a captured org is the strongest negative signal the public can read. *(Designed signal: leader-last-active ships today; voluntary-departure-rate is not yet computed. Discovery surfaces must not render this signal until computed.)*

**What is deliberately NOT surfaced.** Discovery does not display per-org report counts, report aggregates, dispute counts, dismissed-report counts, or any other report-derived statistic. The rationale is structural and is the same as in §5.7: every in-app aggregation of reports is gameable, weaponizable, and censorable. The chain is the public record; the block explorer is the viewer. Prospective joiners who want to investigate report history can do so on-chain; WeVibe's own discovery surface does not turn that history into a leaderboard.

### 7.2 Badge Scoping and Canonical Criteria

Organization profiles expose badge state in a per-org breakdown for both contributors and the org aggregate itself.

- **Per-org scope:** badges are earned and displayed in org context, then optionally summarized on contributor profiles.
- **No cross-org leaderboard:** WeVibe does not publish a global rank table.
- **Canonical criteria for display tiers:** in alpha, rarity tier is computed read-time by the reference social-graph display client (D-SG-3), with on-chain freeze deferred to mainnet (GAP-RARITY-1); serve-milestone and contribution-volume thresholds come from a canonical reference spec used by the reference social graph so labels like "Legendary" remain consistent across forks.

This canonical-spec-in-display approach preserves fork freedom while keeping badge semantics legible across the ecosystem.

### 7.3 Leader Interface

Hub-hosted web dashboard (`wevibe-dashboard`): pending review queue, memory browser, historical decisions, member management, org configuration, keyword taxonomy management, recovery status, direct memory authoring, bandwidth usage monitoring, relay/settlement status, and — as the designed report loop lands — a report response interface.

Serve and denial settlement runs automatically through the hub→chain relay (`D-RELAY-THROUGHPUT`, §5.8), not a manual leader action; the dashboard surfaces relay/settlement status (pending counts, last batch commit) for visibility. There is no per-denial review — denials are quantitative signals the relay batches and settles on-chain. *(The earlier "denial-settlement panel with a single settle button" is superseded by the relay model.)*

In the designed report loop, the report response interface surfaces an open report against a memory the leader committed: during the response window it shows the remaining time and the acknowledge/uphold action; after a memory has been publicly exposed it offers the one-shot leader dispute. It links to the on-chain transaction once a response is published.

### 7.4 Member Interface

Members see: role, contribution count, serve count, reputation summary, and per-org badge progress/status.

### 7.5 Reporter's Private View

Each reporter has a private list of their own published reports (public escalations) — part of the designed report loop. Each entry shows a memory excerpt, the org name, the submission date, the leader's response status (pending / acknowledged / disputed / unaddressed), and a copy-link to the on-chain transaction. This view is visible only to the reporter — it is the reporter's own record of escalations, and the place from which they share block-explorer URLs to whatever public forum they choose. No other user sees it.

---

## 8. Storage Architecture

### 8.1 On-Chain Encrypted Memory Storage

Memory content is stored as encrypted blobs directly on the WeVibe chain. Each approved memory is a transaction that writes:
- Encrypted ciphertext (AES-256-GCM)
- Wrapped DEK (sealed to epoch key)
- Plaintext metadata: org ID, epoch, contributor pub key, keywords/weights, stack tags, timestamp, provenance tier
- Merkle leaf hash for the epoch batch

Every validator replicates every memory. This is the storage guarantee — no separate challenge mechanism needed. Chain consensus IS the storage layer.

**Size economics.** A typical memory is 500 bytes to 2KB encrypted. At 10,000 memories, that's ~10-20MB of chain state. At 100,000 memories, ~100-200MB. Cosmos SDK chains handle this comfortably. Pruning strategies for dormant orgs keep long-term state manageable.

### 8.2 Keyword Index (On-Chain Metadata)

Plaintext per-memory keyword weights are stored alongside encrypted memories on-chain. This enables keyword-based filtering without decryption. The tradeoff (keyword visibility) is accepted — see Section 3.7. The org-level keyword *taxonomy* itself — the controlled vocabulary a leader manages (add / merge / rename / deprecate) — is a hub-side capability (hub database + dashboard), not chain state; the chain only carries each memory's keyword weights. Authority & exit status: see the Hub Authority Ledger (MASTER.md) and `D-MISSION-INVARIANT` (designed anchor: on-chain vocabulary version hash, `D-HUB-REBUILDABLE` §2 / GAP-MI-3).

### 8.3 Semantic Vector Index (Hub Qdrant)

Vector embeddings are NOT stored on-chain. Stored-memory embeddings are computed at approval/ingest and upserted to the hub's Qdrant index, where similarity search runs over vectors plus keyword metadata. For recall, the MCP/plugin computes the query embedding locally via Ollama and sends the query vector to the hub.

Qdrant stores vector + keyword metadata only — no plaintext memory content and no ciphertext. This "not ciphertext" is scoped to Qdrant, not to the hub as a whole: the encrypted blob the hub *does* hold lives in Postgres (`pending_submissions`, `rotation_buffer`) and on-chain (`StoredMemoryCommitment.EncryptedBlob`, §8.1), and the hub cannot decrypt it — the DEK is wrapped to leader/moderator keys and the hub never receives the epoch secret (see §3.7, `D-EMBEDDING-HONEST-CLAIM`, `D-PRIVACY-BOUNDARY-REDRAW`). Qdrant itself holds only the vector + keyword shadow. Embeddings are derived data and remain off-chain. Authority & exit status: see the Hub Authority Ledger (MASTER.md) and `D-MISSION-INVARIANT` (exit: rebuild from chain + keys, `D-HUB-REBUILDABLE`/GAP-MI-3; the clean embeddings are a disclosed semantic shadow, `D-EMBEDDING-HONEST-CLAIM`).

### 8.4 Memory Metadata

- **contributor_pubkey** — on-chain identity of the contributor
- **model_origin** — contributing model
- **stack_tags** — freeform technology tags
- **version** — nullable version string
- **source** — `session` | `doc_import` | `authored` (designed enum; not yet present in chain state)
- **provenance** — `tee-attested` | `commitllm` | `proxy-attested` | `self-declared` | `unattested` (graded per field; see D-GAMIFICATION-PROVENANCE)
- **certified_model** — TEE-attested model measurement (checkpoint hash) of the session's production model, if any (D-ATTEST-TEE-TIER); certifies session provenance, not memory text
- **derivation** — `verbatim` | `edited-after-extraction` (whether the submitted memory matches the pinned-extraction output)
- **difficulty** — difficulty value (1–10) on the attested-memory record; feeds the per-contributor difficulty histogram in `x/reputation`. The field and histogram exist today; the two-layer *scoring* that would compute the value automatically (§3.11) is roadmap.
- **quality** — quality value (1–10) on the attested-memory record (reputation XP = difficulty × quality). The field exists today; the Layer-2 grading LLM that would produce it (§3.11) is roadmap.
- **approved** — boolean moderation state
- **is_reported** — an open report currently stands against this memory (designed)
- **was_reported** — reported at least once; permanent historical flag (designed)
- **report_cleared** — the leader dismissed the open report via `clear_report` (designed)
- **quarantined** — designed flag for memories with repeated rejections (not yet implemented)
- **deprecated** — curator has marked stale

---

## 9. Security Analysis

### 9.1 Sybil Resistance

Free account creation cannot be prevented — a keypair is offline math — so WeVibe does not try to gate identity creation. Instead it makes free identities **inert** and locates the gate where harm can occur (DECISIONS.md `D-SYBIL-MEMBERSHIP-GATED`):

- A membership-less identity can do nothing — no contribution, no recall, no earnings — so an attacker minting identities at scale gains nothing.
- **Org membership is the scarce, gated resource.** Membership is invitation/approval-only, with the human leader as the bouncer (and the org-slot itself is a hard-capped, auction-burned, forfeitable bond on the leader — `D-ECON-STORAGE-MARKET`).
- **Reputation is the soft stake** for wallet-free contributors: it is destroyed on removal, so a contributor who built standing and then turns malicious loses it and restarts at zero under a fresh, re-approval-gated identity.
- **Contributors carry no bond and can never publish unilaterally** — every contribution is leader-gated and committed under the leader's sole signature (§9.7), so the leader's slot is the bond behind all published content. This is why frictionless, wallet-free contribution does not open a Sybil hole: the blast radius of a Sybil contributor is "wasted reviewer time," not "poisoned network," and that is bounded by rate-limits, booting, and cheap join-door friction (cooldowns, invite codes).

### 9.2 Memory Poisoning
Defense layers: submission-time wevibe-guard (advisory), OCR sanitization, human review with steganography detection, contributor reputation visible during review, recall-time blacklist, recall-time wevibe-guard, recall-time OCR, artifact extraction with egress enforcement, plugin approval gate with contributor trust signals. Residual risk: semantic payloads, subtly wrong recommendations.

### 9.3 Leader Key Compromise
K_master compromise exposes all epoch-derived content. Mitigation: offline recovery phrase, encrypted vault with Argon2id, threshold recovery.

Separately, hub infrastructure is treated as untrusted transport, not a trust root: endpoint authority comes from leader-signed on-chain `hub_endpoints` updates on the canonical chain (DECISIONS `D-CHAIN-RESOLVED-HUB-ENDPOINT`), and client verification of hub-signed responses against on-chain `hub_serving_address` is the transport MITM defense (DECISIONS `D-HUB-RESPONSE-SIGNED`).

### 9.4 Chain State Observability
On-chain data is public (encrypted blobs + plaintext metadata). An observer can see: org sizes, submission frequency, keyword distributions, serve patterns, contributor activity, reputation scores. They cannot see: memory content, decryption keys, member identities beyond pub keys, local blacklist state.

### 9.5 Network-Level Anti-DDOS
Anti-DDoS uses two complementary mechanisms (DECISIONS.md `D-ECON-STORAGE-MARKET §7`). On testnet — where faucet tokens are free and so carry no economic deterrent — `x/bandwidth` enforces flat per-org per-epoch rate-limit caps; the chain rejects submissions/serves exceeding the cap. On mainnet, the per-memory **storage deposit** (a VIBE cost per committed memory) is the economic anti-spam, and `x/bandwidth` is removed at mainnet launch. (The earlier "bandwidth proportional to VIBE burned/staked" model was never implemented and is dropped.) No org can flood the network regardless of off-chain resources.

### 9.6 Content Suitability Policy

**Suitable:** Coding patterns/anti-patterns, architecture lessons, debugging notes, dependency guidance, tool usage, process workflows, version-specific gotchas, negative knowledge.

**Unsuitable:** Credentials/secrets, customer PII, regulated data, legal/HR records, high-sensitivity security incident details.

### 9.7 Org Capture and Public Escalation

A single actor wearing multiple hats — leader, every member with `can_moderate`, every contributing member — can fully capture an org's internal governance. Inside the org, every approval, every report dismissal, and every chain commit can be coordinated. Internal accountability primitives (advisory votes from `can_moderate` members, dispute counts, internal review queues) provide no protection against this case: the captured operator simply approves their own malicious memories and dismisses every report filed against them.

The system's security model is therefore not "prevent capture through internal governance." The system's security model is:

> Make capture economically unsustainable through transparent on-chain accountability, frictionless exit for members, and a public escalation primitive designed so that a captured org cannot suppress it.

The four load-bearing properties:

1. **The chain is the unforgeable audit log.** Every consequential action — memory commit, denial settlement, report acknowledgment, dispute publication, member departure — is a signed on-chain transaction. Neither the captured org nor WeVibe-the-protocol nor any platform operator can edit or suppress it after the fact.
2. **Consumers are designed to have an escalation path the org cannot close.** The verification anchor that makes this possible (§5.4) ships today; the reporter-signed public escalation and its response window are the near-term accountability layer built on it. In the target model a dismissed (`clear_report`) or unaddressed on-chain report unlocks a reporter-signed public escalation — wallet-gated and gas-paid, revealing plaintext only after the one-week window elapses or the leader dismisses, anchored to the contributor-signed hash the leader cannot poison (§5.4) — and once published it cannot be edited or deleted. The reporter broadcasts the filing and expose **directly to chain RPC**, with the WeVibe-operated relay as retry fallback only (DECISIONS `D-REPORT-DIRECT-BROADCAST`): the censorship-resistance path does not depend on WeVibe-operated infrastructure, so neither a captured org nor WeVibe itself can suppress it. *(Status: decision locked; client build pending — GAP-MI-5.)*
3. **Exit is unfakeable.** Members leaving voluntarily is a first-class on-chain event. Sybils can be invited and can file frivolous reports, but they cannot fake people walking away. The voluntary-departure-rate signal on public discovery (§7.1) lets prospective joiners read the most honest possible signal about whether existing members trust the org.
4. **Hub compromise is a per-org degradation event, not network takeover.** Per-memory Umbral crypto and consumer-side `wevibe-guard` still gate plaintext/injection, and hub responses must verify against on-chain serving keys (`D-HUB-RESPONSE-SIGNED`). A compromised endpoint can at worst degrade or poison recall for that org; it cannot mint identities, steal contributor keys, or affect other orgs. The endpoint can be rotated on-chain by leader signature, with clients auto-switching and passively notifying once (`D-HUB-ENDPOINT-CHANGE-TOAST`).

Worst case in this class is degraded/poisoned recall quality for one org, typically surfaced by guard/crypto checks and reversible by on-chain endpoint rotation.

**The leader bears sole signature.** Co-attestation of additional reviewer pubkeys on leader-signed chain transactions is explicitly removed (§5.3). A leader's chain commit binds the leader's wallet only. This concentrates responsibility on the actor who actually signs and prevents implicating advisory reviewers in chain-level decisions they did not directly authorize. The trade-off — accountability for individual approvals (by members holding `can_moderate`) becomes an org-local rather than chain-public concern — is acceptable because internal advisory-vote history cannot defend against a capture scenario anyway, and because making leaders sole signatories sharpens the public attribution of every consequential action.

**Why no platform tribunal.** WeVibe deliberately does not become a tribunal that adjudicates published reports. Any in-app judging body is a capture vector — whoever controls the tribunal controls the verdict. The chain publishes the evidence; the block explorer renders it; the reporter shares the URL wherever they choose; the public judges on its own merits in whatever forums it chooses. WeVibe-the-protocol has no editorial role in that judgment. (§5.7)

**Residual: contributor-leader collusion.** If the contributor who submitted a memory and the leader who committed it are both adversaries, the contributor can sign a false plaintext hash, the leader can commit it, and a public report's plaintext reveal will not match the on-chain hash — making the report look invalid even when the underlying claim is true. The on-chain ciphertext + capsule remain as a final backstop: any future key disclosure (epoch rotation, org closure, legal discovery) lets independent parties decrypt and verify after the fact (§5.4). This residual cannot be eliminated at submit time — when content creator and content approver are the same adversary, there is no honest party in the verification chain to sign over.

---

## 10. Decentralized Architecture

### 10.1 Chain Architecture

WeVibe's chain is a sovereign L1 appchain built on Cosmos SDK + CometBFT. Not a rollup — WeVibe requires deterministic finality (CometBFT provides this; rollups have multi-day challenge windows). The chain halts before it forks — safety-over-liveness is correct for memory attestation and storage.

In the near-term org-directory model, chain org state also carries `hub_endpoints` (an ordered list of 1–3 transport URLs for failover redundancy, set through a leader-signed setter transaction), while `hub_serving_address` remains the serve/deny signing-authorization key; clients resolve transport from chain RPC rather than manual URL config (DECISIONS `D-CHAIN-RESOLVED-HUB-ENDPOINT`, `D-HUB-RESPONSE-SIGNED`).

### 10.2 The Four Roles

**Developer (user).** Codes with an LLM. Onboards in seconds with a **passkey** (Face ID / fingerprint) — no wallet or seed phrase required to create an account, join an org, contribute, or recall (DECISIONS.md `D-IDENTITY-PROGRESSIVE-CUSTODY`). Sessions stay local by default, and contribution is explicit through the dashboard extraction/review/submit flow. May consume paid recall access through orgs, but chain mechanics stay abstracted behind plugin/hub UX. A wallet is an optional later upgrade — needed only to claim earned VIBE or pay mainnet fees. Experience: "I code, I choose what to contribute, my profile shows what I've solved."

**Org leader (economic operator + curator).** Acquires an org slot (auction burn) and signs registration from their own wallet, curates memories, manages membership, and sets the org's recall-access/payment model (price + policy). Leader revenue comes from members' VIBE payments for org access, settled through an **on-chain demand-leg router** that enforces a protocol burn (`max(n%, floor)`) and routes the remainder **in-transaction to the leader's own wallet** — the network holds no revenue account (resolved, `D-ECON-CUSTODY-NONCUSTODIAL`); the earlier `Treasury`/`MsgWithdrawTreasury` custody model has been removed and no withdrawal path is built. Members holding `can_moderate` are paid at leader discretion. The leader carries ongoing skin-in-the-game via the slot (self-assessed-value rent + forced-sale-in-window) and per-memory storage deposits. **Leaders earn no emissions.**

**Validator.** Stakes VIBE, runs CometBFT consensus, stores all chain state (including encrypted memories), earns validator/staking emissions. Everything deterministic — no subjective judgments. Validators are the storage and availability layer.

**WeVibe-the-protocol.** Open-source software. No company in the middle. WeVibe-the-company may run validators and operate orgs early on, but the protocol does not depend on any single entity.

### 10.3 Token Economics

Single token: **VIBE**. Used for staking, org-slot acquisition (auction burns), per-memory storage deposits, demand-leg router settlement, and contributor payouts.

**Org slots (scarce, capped, auctioned).** Org capacity is a hard-capped set of registry-allocated slots (governance param: 32 alpha / 320 testnet / 3200 mainnet). Primary allocation is an ascending price per subsequent slot until the cap fills (implemented today); a freed slot (abandoned/closed/lapsed) is intended to be re-homed by a descending (Dutch) resale (designed, not yet built — see §10.6). The acquisition payment is split 50/50: half is burned and half capitalizes the org's own on-chain account (DECISIONS.md `D-ECON-STORAGE-MARKET` amendment 9). The slot `org_id` is permanent and leader-independent. Scarcity + the burn is the network-level anti-spam.

**Self-assessed-value rent (Harberger-style).** A leader posts a self-assessed value V for their slot, pays rent `r × V` per period, and anyone may force-buy the slot at V during a bid window. This keeps slots in productive hands without right-of-first-refusal chilling or grief-bidding. Non-payment/abandonment frees the slot back to Dutch resale and marks the org dormant.

**Per-memory storage deposit (a keeper/liveness function, not a participation cost).** Each committed memory carries a small VIBE deposit that decays as storage rent and is keeper-claimable as a deletion bounty when exhausted/abandoned; pruning deletes the ciphertext blob but permanently retains accountability metadata. Its purpose is **storage hygiene and liveness** — it lets keepers **reap dead memories** and reclaim perpetual storage, and it provides the path to prune an org's memories if the **leader goes AWOL** (org abandoned/dormant) so the network is not stuck paying to store orphaned blobs forever. It is explicitly **not** a cost on contributing or a contributor bond (contributors pay nothing — `D-LEADER-SOLE-SIGNER`); the deposit is funded from the org side and prices the real (perpetual storage) externality at the granularity it occurs. A memory under an open report has its deposit frozen (unprunable until resolved). Built for production; parameterized ~0 on testnet (where `x/bandwidth` is the rate-limit guard instead).

**Serve/deny receipt loop (earned trust — no VIBE).** Recall runs through the hub's **serve/deny route**: when a consumer recalls (or rejects) a memory, the hub submits a serve/denial batch to the chain (signed by the org's whitelisted serving key, gas covered by the org-account feegrant). The chain folds those serve/denial events into each memory's **Earned Trust** standing (DECISIONS.md `D-4.2`), so a memory's retrieval weight rises with demonstrated usefulness and decays toward archival when it is denied or never served. This loop is **separate from the storage deposit**: it moves *no* VIBE and is purely the trust/decay signal — the deposit is the storage-liveness lever, the serve/deny loop is the quality lever.

**Demand-leg router.** Members' recall-access payments settle through an on-chain router whose one job is to enforce the protocol burn `max(n%, floor)` and route the remainder **in-transaction to the leader's wallet** (the network holds no revenue account — resolved, `D-ECON-CUSTODY-NONCUSTODIAL`); `n`/`floor`/`r`/slot-cap are governance params. See DECISIONS.md `D-ECON-STORAGE-MARKET` + `D-ECON-CUSTODY-NONCUSTODIAL` (decided; build in progress, GAP-MI-6).

**Contributor payouts (contribution-only).** Contributors are paid per **approved memory** from the network contributor-emission budget, gated by a **network-set** qualification threshold. There is no payout per serve/retrieval. Reputation tiers may scale payout-per-approved-memory, but retrieval counts are excluded from VIBE flows.

#### 10.3.1 Emission Schedule (Sprint 32, locked — see DECISIONS D-S32-TOKENOMICS-LOCKED)

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

**Genesis seeding.** The emission pool (`validator_pool_remaining_uvibe`, `contributor_pool_remaining_uvibe`, `contributor_rollover_uvibe`, `start_epoch`) is written at chain genesis (DECISIONS D-S32-EMISSION-POOL-GENESIS). Emission and decay both run in the chain's epoch-end hook; correct epoch-end iteration under the cache-wrapped store is governed by DECISIONS D-S32-CACHEKV-ITER.

**Contributor attribution.** Emissions credit the *authoritative* contributor recorded on the committed memory (`contributor_address`), not a consumer-supplied serve payload field. The same address drives serve attribution (DECISIONS D-S32-CONTRIBUTOR-ATTRIBUTION).

> Status (alpha honesty): the schedule constants, per-epoch pool math, qualifier/rollover logic, and contributor attribution are implemented and locked (the flat `daily_mint` placeholder is superseded by the pool model above). **Disbursement is NOT yet wired:** `x/emissions` currently computes and accrues emissions as state, but does not mint or move coins (no `BankKeeper`/`MintCoins` path exists yet). Both payout legs need a withdrawal mechanism that is still to be built: (a) **contributors are paid by the network** — rewards accrue to a claim-later balance keyed to the passkey identity and become withdrawable to the wallet only after the contributor links a Cosmos wallet **and** performs the explicit dual-signed on-chain migration (`is_migrated`; link ≠ migrate; see §6.3 and DECISIONS.md `D-MIGRATION-ONCHAIN-ALIAS`). Privileged capabilities are the exception to wallet-optionality: a leader always has a linked wallet (`D-LEADER-REQUIRES-WALLET`), and members granted `can_moderate` likewise require a linked wallet (they sign their advisory votes), since the leader is the sole wallet-signing bond for published content. Transfer then occurs via a mint+claim path that MUST carry a reentrancy guard and a double-claim/duplication guard; (b) the supplemental validator-pool emission is likewise accrual-only today (validators still earn standard Cosmos SDK staking rewards in the interim — §10.4).

### 10.4 Validator Economics

Validators earn standard Cosmos SDK staking rewards for running consensus. Additionally, validators store all encrypted memories as part of chain state — this is not separate "operator work," it's inherent to running a node. No separate storage challenges needed. (The supplemental validator-pool emission in §10.3.1 is currently accrual-only — computed in state, not yet minted; the live validator reward is the standard staking/distribution path.)

### 10.5 Demand-Leg Economics (Non-Custodial Router — resolved)

Serve/retrieval attribution is **social, not economic** (see §6): serve counts drive public profiles and badges, but do not trigger VIBE payout.

Economic demand is the org access leg:
1. Users buy VIBE and pay orgs for recall access.
2. Access/payment model and pricing are **leader-set**; payments settle through the **on-chain demand-leg router** (the hub `org_credits` ledger becomes a mirror of chain state, not the source of truth).
3. The router burns `max(n%, floor)` of each payment (governance params) — the deflationary sink.
4. The remainder is the leader's revenue. The router **routes the remainder in-transaction directly to the leader's wallet** — there is **no network-held revenue account** (resolved, DECISIONS `D-ECON-CUSTODY-NONCUSTODIAL`). The per-org on-chain account is the **operating account** only (gas, feegrants, storage deposits, the 50% acquisition-retain capitalization, voluntary leader top-ups); it never accumulates member revenue. The prior `Treasury`/`MsgWithdrawTreasury` custody model has been removed and no withdrawal path is built.
5. Leader compensates members holding `can_moderate` at discretion from that revenue (their own wallet); there is no protocol-enforced split.

**Router enforcement.** Org recall access is gated through the router: the hub's `membership_active` flag is set for non-trial members **only by the hub's chain watcher upon a confirmed router payment event**; `org_credits`/subscription state is a strict mirror of chain payment events (the chain-first/hub-mirrored pattern). Trial members remain on the orthogonal trial path.

**Canonical closed loop (target):** emission -> contributors (contribution-only, claim after wallet-link) + validators/stakers (mint/sell) -> users buy VIBE -> users pay orgs (leader-set model & price) -> router burn + remainder to leader -> leader pays members holding `can_moderate` -> stake/secure -> repeat.

Leaders earn no emissions, there is no per-serve royalty, and there is no protocol-enforced split for members holding `can_moderate`.

> Status (alpha honesty): WeVibe is a **non-custodial** network — p2p payments + memory storage + reputation, holding no user or org funds. The only consensus-level economic infrastructure it requires here is a payment **router that enforces the VIBE burn** and routes the remainder in-tx to the leader's wallet. The custody model is **resolved** (`D-ECON-CUSTODY-NONCUSTODIAL`): the router gates recall access, so making it authoritative also makes the burn guaranteed; subscriptions move from hub-authoritative to chain-authoritative with hub mirroring. The demand-leg settlement, the burn path, and reward-settlement wiring are decided but **not yet built — GAP-MI-6**; CO-047 `org_credits` is an accounting skeleton; testnet faucet flows are testnet-only gas scaffolding, not mainnet economics.

### 10.6 On-Chain Modules

Seven custom Cosmos SDK modules:

- `x/org` — slot registry + acquisition auction (ascending primary implemented; Dutch resale + self-assessed-value rent + forced-sale-in-window designed, not built), per-org module account (operating account only — never holds member revenue, `D-ECON-CUSTODY-NONCUSTODIAL`), intended on-chain demand-leg router (membership payment → burn cut + remainder routed in-tx to the leader's wallet), membership, org-directory transport/auth fields (`hub_endpoints` — ordered list of 1–3 transport URLs for failover, via leader-signed setter tx, near-term design; `hub_serving_address` serving/signing authorization key), serving-key feegrant, dormancy/abandonment detection (partial)
- `x/memory` — pending commitment storage (hash + metadata, no blob until approved), approved memory blob storage (encrypted ciphertext as chain state), Merkle root submissions per epoch, contributor-signed verification anchor (plaintext/salt/ciphertext hashes). (Pending-commitment auto-expiry and quarantine flagging are designed but not yet implemented.)
- `x/serve` — batched serve receipt recording (per-org pseudonymous serve keys), deduplication (memory_cid + serve_key + epoch), self-serve detection/discounting, contributor cross-org serve count aggregation for social attribution (non-economic)
- `x/reputation` — per-contributor cross-org aggregated stats (serve count, org breadth, domain tags, rep score, wallet age). Enhanced mode per-org when attestation enabled (difficulty histogram, XP, provenance breakdown).
- `x/emissions` — validator staking rewards, contributor emission distribution from the network pool, protocol-level emission schedule
- `x/bandwidth` — per-org per-epoch flat rate-limit caps (DDoS guard). This is the **testnet** anti-spam guard (faucet tokens are free, so economic deposits don't deter there); scheduled for **removal at mainnet launch**, when the per-memory storage deposit takes over the anti-spam role

- `x/attestation` — session-attestation storage (CommitLLM receipts / cloud-provider signatures). Present and wired but DISABLED — `MsgSubmitSessionAttestation` is a no-op/reject until the pluggable attestation infra exists (D-ATTEST-ROADMAP); kept so it can be activated before mainnet. Post-mainnet activation is planned to generalize this socket into a typed proof artifact (`tee_receipt` / `zktls_proof` / `zkml_proof`; D-ATTEST-PROOF-TIER, PROPOSED/PENDING-SPIKE).

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

### 11.1 Task-Context Skills

Curator-defined collections organized by task context. Skills are intended as lightweight named sets of memories with a description that improve curation and social discoverability inside an org; ungrouped memories are allowed and skill assignment is optional. This is a roadmap feature — no skill entity exists in the chain/hub/dashboard yet.

### 11.2 Cold-Start: Documentation Seeding

New organizations are intended to import canonical documentation as seed memories (source: `doc_import`), with session contributions extending and replacing seeds over time. Not yet built (no `source`/`doc_import` path).

### 11.3 Federation (Design Phase)

Federation operates at the skill level. Orgs publish skill packages. Receiving orgs set quality thresholds. No individual contributor reputation crosses federation boundaries. This remains a design-phase roadmap item and is deferred until the alpha curation loop is proven.

---

## 12. Implementation Phases

### Phase I: Current Alpha — On-Chain Memory + Social Attribution

- Working memory contribution/review/retrieval loop with encrypted on-chain memory storage
- Org membership/role flows and moderation pipeline
- Serve attribution counters on-chain for contributor + org social reputation signals
- Guarded contribution path (local scanning + trust checks) across MCP/plugin entry points

**Exit criteria:** A reliable daily loop from session output to approved memory to recall, with social attribution updating public reputation signals.

### Phase II: Near-Term Alpha — Social Graph + Badge Surfaces

- Public contributor/org profile surfaces over chain data (serve + contribution signals)
- Badge surfaces for serve milestones and contribution volume in the reference social graph
- Rarity-tier badge UX remains near-term and design-stage (see GAP-RARITY-1)
- Curator workflows for direct memory authoring, documentation seeding, and skill curation

### Phase III: Post-Mainnet Roadmap — Attestation + Federation

- Pluggable session-attestation framework (cryptographic and/or API-backed), aligned with §3.10
- Two-layer difficulty scoring (§3.11) as an optional quality input once attestation infrastructure is live
- Skill-level federation remains design phase: publish/subscribe skill packages with receiver-side quality thresholds and no cross-org contributor-reputation portability

---

## 13. Open Questions

**Earned-Trust settlement lag.** What is the maximum acceptable divergence window between hub-optimistic ranking and chain-settled weights, and do unsettled denials survive hub migration? (The decay formula does not change — R-DECAY-FROZEN — only event durability/timing semantics are in question.) Tracked as GAP-MI-7; see §5.8 and DECISIONS.md `D-4.2` / `D-RELAY-THROUGHPUT`.

> *(The former first open question — org-payment custody model, non-custodial router vs. network-held account — is RESOLVED by `D-ECON-CUSTODY-NONCUSTODIAL`: the router routes the remainder in-tx to the leader's wallet, the network holds no revenue, and subscriptions become chain-authoritative with hub mirroring. Build tracked as GAP-MI-6; see §10.5.)*

**Pending memory retention window.** How long do pending (unreviewed) memories stay on-chain before auto-purging? 72 hours? Configurable per org?

**On-chain storage limits.** At what scale does on-chain memory storage become impractical? 100K memories? 1M? Need to model chain state growth and validator hardware requirements.

**Embedding model evolution.** nomic-embed-text confirmed for now. Future upgrades mean re-embedding all memories — this happens locally but needs coordination across org members.

**Federation rollout scope.** Which minimum skill-package contract should ship first (metadata, provenance, quality thresholds) when federation moves from design phase toward implementation?

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

---

*This document is an internal architecture specification. Nothing in this document constitutes an offer or solicitation of securities.*
