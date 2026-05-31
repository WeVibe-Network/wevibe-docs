# WeVibe Network

**Plugin-Gated Shared Memory for AI Coding Agents**
*On-Chain Encrypted Knowledge Infrastructure with Human-Approved Memory Injection*

Draft v2.1 · May 2026 · Architecture Document
Classification: Confidential — Not for public distribution

---

## Revision notes (v2.1 → v2.2)

**Verification anchor redesign: signed canonical body, no zero-knowledge cryptography.**

v2.2 replaces the Pattern B Tier 2 verification anchor that v2.1 introduced as a contributor-signed plaintext hash (and that the DMO-028 lock subsequently elaborated as an SP1 zero-knowledge proof). The redesign is grounded in two findings from Sprint 31:

1. **GO-001 (2026-05-27)** established that the existing contributor signature at submit time covers only org_id, epoch_id, memory_type, contributor_pubkey, and submission_hash — it does not cover plaintext_hash (which did not exist) or salt (which did not exist), and binds ciphertext only transitively through submission_hash. The signature, in its current form, cannot serve as a Tier 2 verification anchor.

2. **CO-028 (2026-05-27)** validated that the SP1 zero-knowledge pathway specified by DMO-028 is technically achievable but operationally unshippable — 16.6 GB peak RSS, 45 s wall on high-end consumer hardware, with no alternative prover location that preserves the system's deployment model.

The replacement design adds three fields to the contributor's signed canonical body at submit time: plaintext_hash, salt, ciphertext_hash. The signature now binds the contributor to the exact plaintext (via salted hash), the exact ciphertext (via direct hash), and the relationship between them. At Tier 2 escalation, the reporter reveals (plaintext, salt) and the chain verifies sha256(salt || plaintext) equals the on-chain plaintext_hash. The contributor's signature, also on-chain, binds the contributor to having authored that specific (plaintext, ciphertext) tuple. The leader is removed from the verification chain — they cannot poison the anchor because they do not sign it.

The cryptographic coverage of v2.2 equals the coverage of v2.1's ZK design for every attack the system is designed to defeat. The differences are operational: v2.2 ships on any hardware, has no DDoS surface, has no consensus-timing concerns, requires no audit of a zkVM toolchain, and ships in a single Sprint 31 implementation CO rather than a 10-12 task production rollout.

Three production bugs that were independent of the verification design but blocking proper Tier 2 verification are also addressed by the v2.2 implementation: WrappedDekEnc is now forwarded to chain (was nil), rotation_buffer signature persistence now requires prior verification (was unverified), and the dashboard's submit path no longer sends plaintext to the hub in cleartext (was an asymmetry with the MCP client).

The chain is wiped. The hub state is wiped. The canonical message tag remains wevibe.submit_memory.v1 — its field set is overhauled in place, not versioned. Pre-MVP, no users, no migration.

---

## Revision notes (v2.0 → v2.1)

**Accountability model: silent denial, two-tier reports, public on-chain escalation.**

v2.1 formalizes the accountability primitives that sit alongside the moderation pipeline. The architectural commitments and economic primitives of v2.0 are unchanged. The additions:

1. **Silent denial as cheap negative signal.** The plugin's Deny control is a frictionless, no-reason-required signal. Denials feed an optimistic local-ledger model: retrieval ranking reflects denial pressure immediately, while the chain remains the eventual source of truth, settled by the leader on a cadence of their choosing.
2. **Two-tier report model.** Reports now have a private tier (existing flow) and a public on-chain escalation tier (new). Most reports never escalate. The public tier exists specifically to make org capture economically unsustainable.
3. **Contributor-signed plaintext hash as verification anchor.** Every memory now carries a contributor-signed hash of its plaintext through the moderation pipeline and onto the chain commit. This removes the leader from the verification chain for any future public report — a captured leader cannot poison the verification anchor.
4. **Leader sole signature on chain commits.** Co-attestation of moderator pubkeys on leader-signed chain transactions is removed. Leaders bear sole responsibility for what they commit. Internal moderator accountability remains an org-local concern.
5. **Unfakeable org-health signals on public discovery.** Discovery surfaces "leader last active" (aggregated chain activity) and "voluntary departure rate" (members who chose to leave). Report counts and other gameable aggregates are not surfaced inside WeVibe — the chain is the public record, the block explorer is the viewer.

The chain remains the only place where consequential accountability claims live. WeVibe's own surfaces never become a tribunal.

---

## Revision notes (v1.6 → v2.0)

**Architecture pivot: On-chain storage, local retrieval, serve attestation economics.**

v2.0 is a fundamental architectural revision. The centralized hub model (wevibe-hub as a hosted service with PostgreSQL, Qdrant, and Ollama) is replaced by a fully decentralized architecture where encrypted memories are stored on-chain and orgs run local retrieval software. The key changes:

1. **Memories stored on-chain.** Encrypted memory blobs go directly on the WeVibe chain as state. No hosted blob storage. No single point of failure. Validators maintain the data as part of consensus.
2. **Local retrieval.** Orgs run wevibe-client locally — a wallet-like tool that syncs from chain, decrypts with org keys, builds a local vector index, and serves retrieval to the developer's agent. No hosted backend touches plaintext.
3. **wevibe-hub eliminated.** The hosted Go server is replaced by wevibe-client (local software). The chain is the backend.
4. **Serve attestations.** When a human approves a memory in the plugin, the approval is attested on-chain. This is the economic primitive — contributors earn when their knowledge gets used. Serve events build the social graph.
5. **Orgs as economic units.** Orgs burn VIBE to create (Bittensor-style dynamic pricing). Org pays for contributor uploads. Org sets rep-tier payout rules. Protocol enforces bandwidth allocation per org.
6. **Contributor reputation from serves.** Memories are tied to contributor pub keys. Serves accumulate on-chain. Reputation = verified usefulness, not self-reported credentials.
7. **Anti-DDOS at protocol level.** Chain enforces per-org bandwidth caps based on VIBE burned/staked. No org can flood the network.
8. **Three software pieces.** wevibe-chain (validators run), wevibe-client (orgs run locally), plugins (agent-specific gates).
9. **Simplified module set.** x/serving, x/challenge, and x/receipt eliminated. Replaced by x/bandwidth and x/serve (serve attestation). Storage is guaranteed by consensus, not separate challenges.

Sprint 22 chain hardening work (CO-162 through CO-170) remains valid — the Cosmos SDK app, module infrastructure, CometBFT consensus, and operator economics are the foundation this pivot builds on.

---

---

## Abstract

WeVibe is plugin-gated shared memory infrastructure for AI coding agents. It provides an on-chain encrypted knowledge network where domain experts create, operate, and curate memory organizations. Coding agents connect via plugins installed in their IDE or terminal. Each organization is an autonomous knowledge product — a curated collection with its own membership roster, role hierarchy, curation standards, and domain focus — administered by its leader.

The system has seven design commitments. First, memories are encrypted before they leave the contributor's machine and stored as opaque blobs on-chain — no infrastructure operator can read content. Second, retrieval operates locally within the org through vector-first staged scoring: semantic vector similarity determines the candidate set, calibrated keyword matching provides precision boost, selective LLM re-ranking resolves ambiguous cases, and model-origin priors adjust relevance. Third, the chain is the storage and attestation layer — trusted for availability and ordering through consensus, not trusted for content. Fourth, every recalled memory passes through a plugin-rendered approval UI before reaching the agent — the human sees the memory, sees wevibe-guard's detection results, sees the contributor's reputation and wallet age, and explicitly approves or denies injection. Fifth, the system is designed as a curator workbench: leaders and reviewers manufacture high-quality domain memory through direct authorship, moderation, gap analysis, and skill packaging. Sixth, serve events are attested on-chain, creating an economic primitive where contributors earn when their knowledge helps others. Seventh, sessions are optionally attested (CommitLLM for open-weight models, proxy for closed-weight) and memories are difficulty-scored, building verifiable reputation profiles for AI-native developers.

---

## 1. Design Philosophy

### 1.1 The Problem

AI coding agents start every session from zero. Institutional knowledge — what works, what fails, what to avoid — exists in the heads of senior engineers and is lost between sessions. When one agent discovers that an SSE timeout behind Nginx requires a specific keepalive configuration, that knowledge should persist and be available to every agent in the organization. Today it isn't.

The problem is worse for developers running local models. A 27B parameter model working on Solana development lacks training data about specific framework versions, deployment gotchas, and stack-specific configuration. These developers are the core audience for WeVibe — domain-specific memories fill the gap between their model's training data and their actual stack.

A second problem compounds the first: the internet is being polluted with LLM-generated content at accelerating rates. Developer knowledge platforms are filling with AI-generated articles that are superficially correct but miss critical nuance — the kind of nuance you only learn by hitting the bug in production. A verified memory from a real session where a real developer fought with a real Redis cluster becomes dramatically more valuable in that environment. The provenance is the scarce resource.

A third problem is social: vibe coders — developers who work primarily by directing AI agents — have no way to prove their skills. GitHub tracks commits, but the AI wrote the code. Stack Overflow requires context-switching into documentation mode. No platform captures the problem-solving work that happens every day between a developer and their coding agent. WeVibe's on-chain serve attestations and contributor reputation create the first verifiable social graph for AI-native developers.

### 1.2 The Organization Model

Organizations are developer communities, not companies. They are loose groups of vibe coders who share domain expertise — a React org, a Solana org, a Kubernetes org. Developers join orgs to boost their local LLM with curated domain knowledge. Orgs are the onramp: you join, your agent gets smarter, your contributions build your cross-org reputation.

Each organization is a self-governing memory network with its own:

- **Membership roster** controlled by the leader through invitation.
- **Role hierarchy**: Leader, Reviewer, Member.
- **Curation standards** enforced by Reviewers appointed by the leader.
- **Domain focus** chosen by the leader (e.g., "Anchor development on Solana," "Django REST framework," "Qdrant vector search").
- **Contributor economics** — rep-tier payout rules set by the org leader. Higher-reputation contributors get more bandwidth and better rewards.

This model places domain experts in charge of their domains. A leader who has built 50 production React applications can evaluate React memories in ways no automated system can. Their reputation is tied to the quality of their network.

Organizations are economic units analogous to Bittensor subnets — they burn VIBE to create, pay for storage bandwidth, and set the rules for contributor compensation within their domain. The org's public keyword tags (Redis, Kubernetes, AWS) are discovery signals, not secrets — they tell developers what knowledge this community offers.

### 1.3 The Curator Workbench

The product is a curator workbench, not a self-organizing knowledge graph. The system surfaces quality signals to curators: retrieval counts, rejection rates, version staleness, query patterns that return few results. The system does NOT use these signals to autonomously rank, deprecate, or filter memories. The curator reviews the signals and decides what to do.

The curator's core workflows are: review pending memories (approve, reject, edit), author memories directly, organize memories into task-context skills (deployment, testing, error-handling), identify and fill gaps in coverage, deprecate stale content, and publish skill packages for federation.

### 1.4 The Plugin Gate

Every memory must pass through human eyes before it enters an agent's context. This is the product invariant.

The plugin is installed in the developer's coding agent (OpenCode, Claude Code, Cursor, Cline, etc.). When the agent needs context, it calls a tool registered by the plugin. The plugin retrieves candidate memories from the local wevibe-client, scans each with wevibe-guard, and renders an approval UI:

```
┌─────────────────────────────────────────┐
│ Memory Injection Request                │
│                                         │
│ "Redis cluster-node-timeout must be     │
│  set to 15000ms when running behind     │
│  AWS NLB with cross-AZ failover..."     │
│                                         │
│ Contributor: wevibe1x7k...f3q2            │
│ Wallet age:  8 months                   │
│ Rep score:   347 (Tier 3)               │
│ Serves:      214 across 12 orgs         │
│ Domain:      redis, kubernetes, aws     │
│                                         │
│ Detections: [url: aws.amazon.com]       │
│                                         │
│ [✓ Accept + Attest]  [◉ Accept Private] │
│ [✗ Deny]                                │
└─────────────────────────────────────────┘
```

**Accept + Attest:** Memory injected into agent context. Serve attestation queued for epoch batch (using per-org pseudonymous serve key). Contributor earns reputation and VIBE payout per org's rep-tier rules.

**Accept Private:** Memory injected into agent context. No serve attestation. No contributor payout. No public record. For stealth sessions or when the user doesn't want their retrieval activity recorded. Org leaders can configure whether private accepts are allowed (`serve_attestation_required = true | false`).

**Deny:** Memory blocked. Plugin asks why (malicious / irrelevant / other). Feedback logged locally (not exposed as a user-level attack on the contributor).

**No plugin installed = no memory injection path.** The MCP server has no way to deliver memories into the agent's context without the plugin as its frontend. This is enforced architecturally, not via a handshake protocol.

**Why not MCP elicitation?** MCP elicitation is a spec feature where the server asks the client for structured user input mid-tool-call. It fails the security requirements: most coding agents don't support it (silent fail-open on 80% of clients), it renders as a chat bubble rather than a hard modal (easy to miss), and there's no "are you sure?" confirmation chain. The plugin provides all three: interruption, clear modal UI, and confirmation. MCP remains the retrieval and contribution backend. The plugin is the delivery and approval surface.

### 1.5 WeVibe's Architecture: Protocol, Not Platform

WeVibe is a protocol, not a hosted service. No single entity owns the memory infrastructure. The chain holds the data. Local software serves it. The protocol coordinates everything.

**What the protocol provides:**

1. **On-chain encrypted storage.** Memories are encrypted blobs stored as chain state. Every validator replicates them. No single point of failure.
2. **Serve attestation.** On-chain record when a memory is approved for injection. This is the economic primitive — it drives contributor rewards and builds the social graph.
3. **Contributor reputation.** On-chain aggregates tied to contributor pub keys — serve count, domain expertise, wallet age, rep score. Visible in the approval UI at the point of decision.
4. **Bandwidth allocation.** Protocol-enforced per-org caps on submissions and storage. Prevents DDOS. Scaled by VIBE burned/staked.
5. **Memory sanitization.** Defense-in-depth pipeline: wevibe-guard scanning, OCR sanitization, artifact extraction, egress enforcement — all running locally on the org's wevibe-client.
6. **Delivery via context injection.** Approved memories are formatted as `context:\n{memory content}` and injected into the agent's prompt.
7. **Optional session attestation.** CommitLLM for open-weight models, proxy attestation for closed-weight. See Section 3.10.
8. **Optional difficulty scoring and reputation.** Two-layer grading feeds developer profiles. See Section 3.11 and Section 6.

**Trust boundaries.** The chain is trusted for availability, ordering, and state integrity through consensus. The chain is **not** trusted for content confidentiality — it stores encrypted blobs it cannot read. The org's wevibe-client is the trust boundary for plaintext — it decrypts locally and never sends plaintext off-machine. Validators observe encrypted blobs, org IDs, contributor pub keys, and serve attestation metadata — but never memory content.

---

## 2. System Architecture

### 2.1 Entities

Let **L** denote the set of leaders, **O** the set of organizations, **D** the set of reviewers, **C** the set of contributing members, and **R** the set of read-only members. Each organization o ∈ O has a leader l(o) ∈ L who controls membership and configuration.

Each participant holds an Ed25519 keypair (with associated X25519 key for encryption) that serves as their protocol identity. Contributor pub keys are on-chain — serve attestations and reputation aggregates are tied to these keys.

### 2.2 Role Hierarchy

| Role | Permissions | Appointed By |
|------|------------|-------------|
| Leader | Full org control, roster management, epoch rotation, key custody, reviewer appointment, keyword taxonomy management, rep-tier payout configuration, bandwidth allocation management | Self (org creator) |
| Reviewer | Review pending memories, approve/deny, decrypt pending submissions via SK_mod(e) | Leader |
| Member | Submit memories (within rep-tier bandwidth), retrieve approved content, view own pending submissions | Leader (via invitation) |

All roles require epoch-specific encryption keys for content access. The leader distributes these keys to approved members through sealed envelope key exchange (Section 3.4).

### 2.3 Organization Lifecycle

**Creation.** The leader burns VIBE to create an org (dynamic pricing — see Section 10.3). The leader generates the master key K_master, derives the initial epoch keys (epoch 0), and generates the initial moderation keypair SK_mod(0)/PK_mod(0). A 24-word BIP39 recovery phrase is derived from K_master and displayed once (ADR-019). The chain allocates bandwidth to the org based on VIBE burned.

**First-run detection.** When wevibe-client starts and discovers no org membership, it surfaces an actionable message to the agent, prompting guided setup.

**Operation.** Members join through leader invitation. Once approved, the leader issues sealed key envelopes containing the epoch keys to the new member. For reviewers and leaders, the envelope also includes SK_mod(e) for the current epoch.

**Contributor onboarding.** Contributors opt in once — they install the plugin and join the org. After that, WeVibe runs invisibly in the background. Sessions are mined for memories automatically. The contributor never needs to actively trigger contributions. Their wevibe-client handles extraction, encryption, and submission to the chain (paid for by the org).

**Key rotation (epoch advancement).** When a member is removed, the org enters `rotation_pending` state:

1. **Removal triggers `rotation_pending`.** The chain marks the org as pending rotation. The removed member's envelope is deleted.
2. **New submissions are buffered.** Contributors can still submit, but submissions enter a local buffer — not admitted to the chain, not indexed, not assigned a final epoch.
3. **Leader completes rotation.** The leader derives new epoch keys from K_master via HKDF, generates a new moderation keypair SK_mod(e+1)/PK_mod(e+1), and re-seals envelopes for all remaining members.
4. **Buffer finalizes.** After rotation completes, buffered submissions are released to the chain under the new epoch.
5. **Grace period escalation.** If rotation is not completed within a configurable window (default: 72 hours), the org's submission bandwidth is suspended.

**Revocation semantics.** Epoch rotation provides forward secrecy only. Removed members retain previously-distributed epoch keys and can decrypt content from their membership period.

### 2.4 The Three Software Pieces

WeVibe's architecture has three software components. No hosted backend. No centralized service.

**wevibe-chain (validators run this):**
Cosmos SDK + CometBFT sovereign L1 appchain. Stores encrypted memory blobs, org state, contributor reputation, serve attestations, and bandwidth accounting. Validators maintain consensus and replicate all data. They never see plaintext.

**wevibe-client (orgs run this locally):**
A wallet-like local application that syncs from the chain, manages org keys, decrypts memories, builds a local vector index, and serves retrieval to the developer's agent. Handles memory extraction from coding sessions, wevibe-guard scanning, encryption, and submission to chain. All plaintext operations happen here and only here. Think of it as a local-first app that reads from a shared blockchain backend.

**plugins (installed in coding agents):**
Platform-specific gates (OpenCode, Claude Code, Cursor, Cline) that register tools in the agent, intercept memory delivery from wevibe-client, run wevibe-guard scans, render the approval UI with contributor trust signals, and attest serves on-chain when the user approves. All plugins call the same wevibe-guard binary and the same wevibe-client backend.

### 2.5 Tool Surface

The plugin registers tools in the coding agent. The wevibe-client provides the local backend. The separation:

**Plugin-registered tools (visible to the agent):**

| Tool | Purpose |
|------|---------|
| `wevibe_recall` | Search organizational memory. Plugin calls wevibe-client for candidates, runs wevibe-guard scan, renders approval UI with contributor reputation, injects approved memories as `context:` ambient content. On approval, attests serve on-chain. |
| `wevibe_contribute` | Record technical learnings. Agent calls this at natural phase transitions. Extracts atomic memories, encrypts, submits to chain via wevibe-client (org pays). |
| `wevibe_reject` | Flag a recalled memory as unhelpful. Adds to local blacklist and reports feedback on-chain for quarantine. |

**wevibe-client backend (invisible to the agent):**

The wevibe-client handles retrieval queries, embedding computation, encryption/decryption, chain synchronization, local vector index management, and serve attestation submission. It returns structured candidate data to the plugin — never directly to the agent.

**Contribution behavior.** The `wevibe_contribute` tool description instructs agents on when to contribute: at natural transition points during sessions, when they discover something non-obvious. Negative knowledge (what NOT to do and why) is especially valuable. In practice, contribution is mostly automatic — the session buffer captures learnings and the extraction pipeline processes them without developer intervention.

**Session buffer safety net.** A session buffer is initialized lazily on the first tool call and records session activity. If the agent does not explicitly contribute during the session, `autoContribute()` fires on session exit. On next session startup, orphaned buffers from crashed sessions are processed.

All administrative operations (org creation, member invitation, moderation, epoch rotation, keyword management, recovery) are handled by the separate `wevibe-admin` CLI.

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
```

**K_enc(e)** wraps Data Encryption Keys (DEKs) for approved memories in epoch e.
**K_audit(e)** is reserved for audit logging and receipt verification.

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

Retrieval is fully local. The org's wevibe-client maintains a local vector index built from decrypted chain data. The pipeline:

#### Context Profiling (Session Start)

When a coding session starts, wevibe-client automatically profiles the environment — dependencies, directory structure, language, framework versions, current file context. This profile acts as a pre-filter, narrowing the memory search space before any query runs. A developer working in a Python/Django project only searches against Python/Django memories, not the entire org corpus.

#### Keyword Extraction
Keywords are extracted by the host agent's LLM at approval time. 10-20 domain-specific keywords with percentage-based weights summing to 100%. Stored alongside the encrypted memory on-chain (plaintext keywords are an accepted metadata tradeoff — see Section 3.7).

#### Semantic Embedding
768-dimensional embedding via bundled `nomic-embed-text` model (ONNX runtime). Computed locally at approval time by the reviewer's wevibe-client. Stored in the local vector index (not on-chain — embeddings are derived data that each org member reconstructs locally).

#### Atomic Memory Format
Each memory is a single, self-contained technical insight:
- **insight** — specific technical knowledge in 1-2 sentences with exact values
- **context** — environment, versions, conditions where this applies
- **avoid** — negative knowledge: what NOT to do and why
- **stack** — specific technologies involved

#### Retrieval Scoring (ADR-025)
```
keyword_boost = Σ(query_weight_i × memory_weight_i)
capped_boost = min(γ × keyword_boost, δ × vector_score)
final_score = vector_score + capped_boost

Default: γ = 0.1, δ = 0.15
```

Vector similarity drives recall (local index top-30 by cosine). Keywords break ties within semantic clusters.

**Model-origin prior.** Soft prior: generic conceptual memories from lower-capability models deprioritized for higher-capability retrievers; highly specific memories receive no penalty regardless of origin.

**Blacklist and quarantine filtering.** Chain-level quarantine flag (`quarantined=true` after 3+ rejections). Client excludes locally blacklisted CIDs.

#### Selective Re-ranking
When top-2 scores are within ε=0.20 (contested query), the wevibe-client uses the host agent's LLM to re-rank. Fallback: original order preserved on error.

### 3.6 Side Channel: On-Chain Metadata

With memories stored on-chain, the following metadata is publicly observable: org IDs, contributor pub keys, submission timestamps, memory sizes, keyword terms/weights (plaintext), serve attestation patterns, reputation scores. This is an accepted tradeoff documented in Section 3.7.

### 3.7 Metadata Visibility Model

WeVibe orgs are public developer communities, not private enterprises. On-chain metadata is intentionally public — it enables discovery, reputation, and the social graph.

**On-chain (public by design):** Org IDs, org topic tags, contributor pub keys, encrypted memory blobs, plaintext keyword terms/weights (discovery signal — "this org covers Redis"), submission timestamps, memory sizes, epoch boundaries, serve attestations (batched per epoch), reputation aggregates, bandwidth consumption, quarantine state.

**Local only (never leaves the org):** Decrypted memory content, vector embeddings, local blacklist state, session context profiles, retrieval scoring details.

Plaintext keywords on-chain are a feature, not a leak. They tell developers what an org covers and help with cross-org discovery. Developers who join an org to boost their LLM need to know what domain knowledge it offers. The keywords serve that purpose.

### 3.8 Defense-in-Depth: Memory Sanitization Pipeline

WeVibe's security model focuses on what it can control: the form and content of recalled memory before it reaches the agent. All sanitization runs locally on the org's wevibe-client.

#### The Pipeline

**Submission time (before on-chain storage):**
1. **wevibe-guard scan.** YARA rules for injection patterns, credential detection, exfiltration matching, unicode mathematical injection detection. Advisory: warns but does not block. The moderator is the security boundary.
2. **OCR sanitization.** Text rendered to image via ImageMagick, OCR'd back via Tesseract. Destroys Unicode tricks, zero-width characters, homoglyphs, invisible formatting.
3. **Encryption.** Memory encrypted with per-memory DEK, DEK sealed to moderation public key.
4. **Human review.** Reviewer decrypts locally, reads plaintext, steganography scan, approve/deny.
5. **On-chain submission.** Approved memory (ciphertext + wrapped DEK + metadata) goes on-chain. Org pays the submission cost.

**Recall time (before delivery):**
6. **Blacklist filter.** Client checks local blacklist.
7. **wevibe-guard scan.** Same scan on decrypted memory at recall time. Catches payloads undetectable when approved (new rules since approval).
8. **OCR sanitization.** Same format-breaking pipeline.
9. **Artifact extraction and egress enforcement.** Typed artifact extraction: URLs, bare domains, IPv4 addresses, shell commands, package install commands, config directives. Every network-resolvable token flags.
10. **Plugin approval gate.** Plugin renders approval UI with wevibe-guard detection results AND contributor trust signals (pub key, wallet age, rep score, serve count, domain expertise). User sees the memory, sees the flags, sees who wrote it, approves or denies.
11. **Serve attestation.** On approval, the plugin submits a serve attestation on-chain (signed by the retrieving user).
12. **Context injection.** Approved memories formatted as `context:\n{memory content}` and injected into agent prompt.

#### What This Pipeline Catches
- YARA-signature prompt injections
- Credential leakage (AWS keys, API tokens, passwords, connection strings)
- Unicode steganography (zero-width characters, homoglyphs, directional overrides)
- Unicode Mathematical Alphanumeric injection (U+1D400-U+1D7FF, 3-char threshold)
- Base64-encoded injections
- External URL injection (scheme-ful and scheme-less)
- Bare hostname references (any TLD)
- IPv4 literal references (with optional port/path)
- Malicious dependency injection
- Config directive injection
- Shell pipe-to-execution attacks
- Previously-rejected and quarantined memories

#### What This Pipeline Does NOT Catch
- Semantic payloads encoded in natural language prose. Mitigated by human review and contributor reputation signals.
- Technically-plausible but subtly wrong recommendations. Mitigated by reviewer domain expertise and contributor rep visibility.

### 3.9 Resolved Architectural Decisions

These decisions are final:

**Individual cross-org reputation is the product.** Serve counts, domain expertise, and rep scores accumulate across all orgs a contributor participates in. This is the social graph for vibe coders. A developer's cross-org profile — "47 Redis memories served 214 times across 12 orgs" — is a public credential. No cross-org reputation *rankings/leaderboards* (avoids toxic competition), but aggregate stats are public and encouraged.

**Task-context skills, not difficulty tiers.** Skills organized by task (deployment, testing, error-handling), not by difficulty level.

**Model origin as soft retrieval prior.** One factor in scoring — not a hard filter or gate.

**Documentation seeding for cold-start.** New orgs import canonical documentation as seed memories.

**Version-scoped decay, not universal time decay.** Decay triggered by version scope changes, not by time.

**On-chain storage, not hosted blob storage.** Approved memories are chain state, replicated by validators. No VPS dependency.

**Pending memories: commitment on-chain, blob off-chain.** Contributors submit only a commitment (hash, org ID, contributor pubkey, expiry epoch, size) on-chain. The encrypted blob is delivered to the reviewer through temporary off-chain channels (local transfer, P2P, or org-hosted mailbox). If approved, the finalized encrypted blob goes on-chain. If rejected or expired, the commitment is removed and the temporary blob is deleted. This ensures rejected content never enters committed block data.

**Local retrieval, not hosted retrieval.** Orgs run their own wevibe-client. No hosted search service.

**Orgs pay for contributors.** Contributors need a protocol identity (auto-generated keypair) but never need to hold, buy, or manage tokens. The org covers all on-chain costs.

**Serve attestations: public reputation, pseudonymous retrieval.** Contributor reputation (serve counts, domain tags, payout amounts) is public on-chain. Retriever identity is represented by a per-org pseudonymous serve key — not the user's global contributor identity. This separates "my knowledge helped others" (public) from "this exact user needed this exact memory" (pseudonymous). Users can optionally link their org serve keys to their public profile as a learning trail.

**Serve attestations are batched per epoch.** Plugin queues approvals locally. WeVibe-client submits one batch transaction per user per epoch using the org serve key. Not one tx per click.

**Three-button approval UX.** Plugin offers: [Accept + Attest] (memory injected, serve attestation queued, contributor earns), [Accept Privately] (memory injected, no attestation, no payout — for stealth sessions), [Deny] (memory blocked, feedback logged). Public orgs with payouts may require attestation. Personal/local orgs may allow private accepts.

### 3.10 Session Attestation (Optional Subsystem)

Sessions produce memories. Without provenance attestation, a contributor could paste fabricated "coding sessions" into the extraction pipeline and farm reputation. Everything downstream — difficulty scoring, quality grading, reputation — depends on knowing the session actually happened.

**At MVP, org leader curation is the trust layer.** Attestation is an optional enhancement, not a requirement. See IMPROVEMENT2-v2 for the full rationale.

#### Tier 1: CommitLLM Cryptographic Receipts (Open-Weight Models)

CommitLLM (Lambda Class, MIT licensed) is a cryptographic commit-and-audit protocol for open-weight LLM inference. The receipt cryptographically proves "this response was produced by this exact model with these exact weights." This kills the transcript fabrication attack entirely.

**Limitation:** Only works for open-weight models. The verifier needs the public checkpoint.

#### Tier 2: Proxy-as-Trust-Layer (Closed-Weight Models)

For closed-weight API models, route traffic through a WeVibe-controlled session proxy. Content-addresses each turn, signs the transcript with WeVibe's key. Weaker trust model than Tier 1 but sufficient for internal teams.

### 3.11 Two-Layer Difficulty Scoring (Optional, Requires Attestation)

#### Layer 1: Structural Signal (Automated, Cheap)
Model capability coefficient × turn count × (1 + 0.25 × failed alternatives). Computed from session structure without understanding content.

#### Layer 2: LLM Grading (Semantic, Authoritative)
Separate grading LLM evaluates non-obviousness, specificity, and reasoning progression. Temperature 0, deterministic, hash-seeded. The grade and session hash are committed together.

---

## 4. Local Client Architecture

### 4.1 wevibe-client

wevibe-client is local software that every org member runs. It replaces the centralized wevibe-hub. Think of it as a wallet + local search engine + sync daemon.

**Components:**
- **Chain syncer.** Subscribes to chain events, downloads encrypted memories for the org, maintains local state.
- **Key manager.** Stores org keys (K_master for leaders, epoch keys for members), manages sealed envelopes, handles encryption/decryption.
- **Vector index.** Local Qdrant or embedded vector store. Builds and maintains a semantic index over decrypted memories. 768-dimensional embeddings via bundled nomic-embed-text (ONNX).
- **Retrieval engine.** Vector-first staged scoring (Section 3.5). Context profiling. Keyword boost. Re-ranking.
- **Session monitor.** Watches the coding session, captures learnings, runs the extraction pipeline, triggers autoContribute on session exit.
- **Submission pipeline.** Encrypts memories, signs with contributor key, submits to chain (org pays gas/bandwidth).
- **Serve attestation.** When the plugin approves a memory, wevibe-client submits the serve attestation on-chain.

### 4.2 Dependencies

wevibe-client requires: a running wevibe-chain node (or RPC endpoint to one), nomic-embed-text ONNX model (bundled), ImageMagick, Tesseract, wevibe-guard binary.

Does NOT require: PostgreSQL, Qdrant server (embedded vector index), Ollama, any hosted service.

### 4.3 Sync and Bootstrapping

**First sync.** When a member joins an org, wevibe-client downloads all encrypted memories for that org from the chain, decrypts them with the epoch keys, generates embeddings, and builds the local vector index. For a large org with thousands of memories, this may take several minutes. After initial sync, updates are incremental — new memories arrive via chain event subscription.

**Incremental sync.** wevibe-client subscribes to new block events. When a new memory is submitted to the org on-chain, wevibe-client downloads it, decrypts, embeds, and adds to the local index. Latency: seconds after on-chain confirmation.

**Offline tolerance.** wevibe-client caches the local index. If the developer goes offline, retrieval still works against the cached index. New memories are synced when connectivity returns.

### 4.4 Retrieval Flow (Plugin-Gated)

```
Developer prompts their coding agent
     │
     ▼
  Agent calls wevibe_recall (registered by plugin)
     │
     ▼
  Plugin calls wevibe-client with query + session context profile
     │
     ▼
  wevibe-client (local):
     1. Context profile filtering (deps, stack, framework)
     2. Query keyword extraction (deterministic, <1ms)
     3. Query embedding (~200ms, local ONNX)
     4. Local vector-first scoring
     5. Quarantine filter (chain state)
     6. Contested check (top-2 within ε=0.20?)
        ├── YES → LLM re-rank via host agent
     7. Client-side blacklist filter
     8. Sanitization pipeline (wevibe-guard, OCR, artifact, egress)
     9. Fetch contributor reputation from chain (pub key, wallet age, rep, serves)
    10. Return candidates + detection results + contributor trust signals to plugin
     │
     ▼
  Plugin renders approval UI:
     Memory text, detection highlights, contributor reputation
     │
     ├── ACCEPT + ATTEST → memory injected as context:\n{content}
     │                     serve approval queued locally (org serve key)
     │                     (batched on-chain at epoch boundary)
     │                     contributor earns rep + VIBE (per org payout rules)
     │
     ├── ACCEPT PRIVATE → memory injected as context:\n{content}
     │                    no attestation, no payout, no public record
     │
     ├── DENIED → "Why?" (malicious/irrelevant/other)
     │              Feedback logged locally, memory blocked
     │
     ▼
  Agent continues with or without memory
```

### 4.5 Contribution Flow

```
Agent calls wevibe_contribute (or autoContribute on session exit)
     │
     ▼
  1. Memory extraction via LLM (atomic format)
  2. wevibe-guard scan (advisory)
  3. OCR sanitize
  4. Fresh DEK, encrypt, seal to PK_mod(e)
  5. Contributor signs submission hash
  6. Submit COMMITMENT to chain (hash, org_id, contributor_pubkey, expiry, size)
     Org pays bandwidth for the commitment tx.
  7. Deliver encrypted blob to reviewer via off-chain channel
     (local transfer, P2P, org-hosted temporary mailbox)
  8. Response: "WeVibe: captured N learning(s). Pending review."
```

**Pending memory lifecycle:** Only the commitment goes on-chain. The encrypted blob is delivered off-chain to reviewers. If approved: the reviewer re-wraps the DEK, the finalized encrypted blob goes on-chain as permanent state, keywords are extracted, the commitment transitions to approved. If rejected or expired (configurable retention window, default 72 hours): the commitment is removed from chain state and the off-chain blob is deleted. Because rejected blobs never enter committed block data, there is no permanent trace of rejected content — even on archival nodes.

### 4.6 Reviewer Flow

Reviewers use wevibe-client's moderation UI (local web dashboard or CLI). Approval includes: DEK re-wrap under K_enc(e), keyword extraction via local LLM, embedding computation (local ONNX), canonical signature. Approved memory submitted on-chain as finalized.

### 4.7 Plugin Architecture

Each coding agent gets its own plugin codebase:

| Agent | Plugin Type | Hook Mechanism |
|-------|-------------|----------------|
| OpenCode | JS/TS plugin in `.opencode/plugin/` | `tool.execute.before`, custom tools via `tool()` |
| Claude Code | Plugin with `.claude-plugin/plugin.json` manifest | PreToolUse hook, `permissionDecision: "deny"` |
| Cursor | Hooks + marketplace plugin | Claude Code hook format compatibility |
| Cline | VS Code extension + hooks | `.clinerules/hooks/`, custom hook system |

All plugins call the same wevibe-guard binary (configured via `WEVIBE_GUARD_BIN` env var) and the same local wevibe-client. The plugin is the platform shim.

---

## 5. Content Review

### 5.1 Review Flow

All contributed memories are submitted to the chain as pending (encrypted, only the org's reviewers can decrypt). Pending memories are visible only to reviewers and leaders via their local wevibe-client.

### 5.2 What Review Can and Cannot Catch

**Can catch:** Prompt injection patterns, malicious URLs, credential exfiltration, spam, duplicates, off-topic content, obvious technical errors, stale references, Unicode steganography, memories too generic for the org's target model/stack.

**Cannot catch:** Subtly incorrect technical guidance that appears plausible, semantically-encoded malicious instructions in natural language prose, context-dependent correctness.

### 5.3 Reviewer Trust Boundary

Reviewers see pending content in plaintext on their local machine. The system is only as secure as reviewer judgment, honesty, and endpoint security. Mitigations: epoch-scoped mod keys, steganography detection, contributor reputation signals visible during review.

The leader is the final authority on what enters the chain. Chain commits — batch memory approvals, denial settlements, report acknowledgments, dispute publications — are signed by the leader's wallet and the leader's wallet alone. The leader bears full responsibility for the on-chain record they produce. Internal moderator votes and approval histories are an org-local accountability primitive maintained outside the chain.

### 5.4 Contributor-Signed Canonical Body as Verification Anchor

Every memory's submit-time canonical body includes three fields that, together with the contributor's signature over the body, form the Tier 2 verification anchor:

- **plaintext_hash** — `sha256(salt || plaintext)`, computed by the contributor before encryption
- **salt** — a fresh 32-byte random value generated per submission
- **ciphertext_hash** — `sha256(ciphertext)`, where ciphertext is the AEAD output

The contributor signs the canonical body with their own key. The canonical body, the signature, and the ciphertext all travel through moderation and land on the chain together. The leader's batch chain commit includes the contributor's signature; the leader cannot modify the signed fields without invalidating the contributor's signature.

This is what makes the public report tier (§5.6) trustworthy without trusting the leader: any future reveal of plaintext + salt can be verified against the on-chain plaintext_hash via direct sha256 check, and the on-chain ciphertext can be verified against the on-chain ciphertext_hash. The leader is removed from the verification chain entirely. A captured leader cannot poison the anchor because they do not sign it.

The contributor cannot substitute ciphertext between submit and chain commit because their signature binds the specific ciphertext_hash. The contributor cannot later claim a different plaintext at Tier 2 because the plaintext_hash binds them to the specific salted hash, and SHA-256 collision resistance prevents producing a different (plaintext, salt) pair that hashes to the same value.

**Why the signature now covers all three fields and not just plaintext_hash.** An earlier design (CO-026, reverted via DMO-027) signed only `plaintext_hash` without salt and without ciphertext binding. That shape was vulnerable to contributor + leader collusion: the contributor could sign one hash while the leader committed ciphertext encrypting different content, and the asymmetry was undetectable. The current shape closes the gap by binding all three fields jointly in a single signature. The leader has no signing role; the contributor has no opportunity to substitute ciphertext after signing.

**Residual risk: contributor with stolen signing key.** If the contributor's signing key is stolen, an attacker can produce signatures binding any (plaintext, salt, ciphertext) tuple. This is a key-management problem, not a cryptographic-anchor problem — no design at the cryptographic layer can defeat it. Mitigation lives at the wallet/identity layer (D-1.1, D-1.3, ARCH-G9 for the future BIP-32 PRE-identity separation). The on-chain ciphertext + sealed-box wrapped DEK remain as a final backstop: any future epoch key disclosure allows independent decryption and exposes the actual content after the fact.

**Salt design rationale.** Without a salt, `sha256(plaintext)` is vulnerable to rainbow-table attacks for low-entropy plaintexts (short memories, common error messages, well-known technical advice). A 32-byte random salt makes rainbow tables infeasible (2^256 salt space per plaintext). The salt is included in the canonical body the contributor signs, stored on-chain alongside the commitment, and revealed by the reporter at Tier 2 escalation time. Salt is not secret — it is context-hiding.

### 5.5 The Two-Tier Report Model

Reports against a served memory have two tiers. Most reports never escalate. The escalation tier exists precisely for the cases where the first tier is captured.

**Tier 1 — Private report.** The consumer reports through the plugin. The report enters the org's moderation queue. Reviewers vote; if upheld, the leader commits a chain message that removes the memory and writes the deletion to the chain record. This is the normal, efficient path for honest orgs and represents the overwhelming majority of report volume. Tier 1 is paid-member-only.

**Tier 2 — Public on-chain escalation.** When a Tier 1 report is dismissed (including dismissed-as-malicious) or stalls past a configurable timeout, the original reporter may escalate to a public on-chain report. The reporter signs the escalation message from their own wallet and pays the gas. The on-chain transaction publishes:

- The memory hash and a reference to its on-chain ciphertext + capsule
- The reporter's reveal of the plaintext (the memory becomes publicly readable)
- A reference to the contributor-signed plaintext hash (§5.4) for cryptographic verification
- The contributor's pubkey, the leader's pubkey, the org ID, and the original commit block height
- The reporter's signed reason text
- A reference to the dismissed Tier 1 report

The memory is now publicly archived on the chain with full provenance. Anyone can verify every cryptographic claim independently using the contributor-signed hash; only the judgment — "this is malicious because…" — requires human evaluation. The block explorer renders the published transaction including the revealed plaintext, the reporter's reason, and all provenance fields.

**Eligibility and one-shot rule.** Only the original Tier 1 reporter may escalate to Tier 2 for that specific memory. One escalation per Tier 1 report. There is no re-publishing after dismissal, no infinite escalation loops. The escalation route is strictly a one-shot recourse mechanism, not a new attack surface.

### 5.6 Org Response and the One-Reply Rule

When a Tier 2 report lands, the org's leader has a fixed response window (network-governed; the default at this writing is 30 days) to respond on-chain with exactly one of two actions, each signed by the leader's wallet:

- **Acknowledge.** No reason required — acknowledgment is the action. The memory is removed from local retrieval and transitions to a deleted terminal state. Functionally equivalent to a Tier 1 upheld outcome.
- **Dispute.** A leader-signed counter-statement is published on-chain, explaining why the report is unfounded. The memory returns to the served state. The dispute is permanent and public, visible at the same block-explorer URL as the original report.

The leader gets exactly one response. No revisions. The single-reply rule forces a deliberate, final answer.

**Silent acquiescence.** If neither action is taken within the response window, the chain marks the report unaddressed. The memory is removed from local retrieval and the "unaddressed" flag is permanent. This mirrors the principle that governs contract law: silence in the face of a public claim is tacit agreement. A leader who does not respond within the window has implicitly accepted the claim's validity. The memory does not return to serving.

**The memory keeps serving during the response window.** A Tier 2 report does NOT immediately remove the memory from local retrieval. Removal happens only at finalization (leader acknowledgment, or timeout). Removing memories on every published report would create a denial-of-service vector: a coordinated wave of frivolous escalations could disrupt an honest org's knowledge base while the org spends gas to defend each one. The cost of escalation falls on the reporter; the cost of acknowledgment or dispute falls on the leader; the memory keeps serving until one of those costs is paid.

### 5.7 No Internal Courtroom

WeVibe's dashboard is not the courtroom. The chain is.

WeVibe's own surfaces — the dashboard, discovery pages, reputation profiles — never display report counts, report aggregates, per-org report statistics, or per-actor report metrics. The reasoning is structural:

- Any in-app aggregation of reports is gameable via sybil reporters. A coordinated set of attackers could inflate report counts on an honest org, weaponizing the platform against legitimate operators.
- Any in-app aggregation is also censorable. Whoever controls the dashboard surface controls which reports get amplified and which get suppressed.
- Any in-app aggregation creates the perception that WeVibe-the-protocol or WeVibe-the-team is the tribunal. There is no tribunal. There is no judging body. There is only publication of unforgeable evidence on an immutable chain that neither the org nor WeVibe controls.

The chain is the publication mechanism. The block explorer is the viewer. WeVibe provides the reporter a permanent on-chain transaction; the reporter shares the block-explorer URL wherever they choose — a social network, a competitor's community, a blog post, a Discord channel. The court of public opinion happens outside the system. WeVibe gets out of the way.

The reporter's own dashboard view is the one exception: each reporter sees a private list of their own published reports with copy-link buttons. That is ammunition for the reporter to share, nothing more. The leader receives a notification when a report is filed against a memory they committed, plus a response interface clearly labeled with the one-reply rule and the response-window timeout — again, only to the leader.

### 5.8 Silent Denial as Cheap Negative Signal

The plugin's three-button approval UI (Accept / Deny / Report) gives the consumer two complementary negative paths. Reports — Tier 1 or Tier 2 — are the high-friction, high-stakes accountability primitive described above. Denials are the low-friction, low-stakes signal that feeds retrieval ranking.

Clicking Deny is silent: no confirmation modal, no required reason, no new UI surface. The reason field is optional. There is no gating — any consumer, trial or paid, may deny any memory. There are no caps, no rate limits, no reputation weighting. Every denial counts as exactly one denial event.

A denial does two things:

1. **Local suppression.** The memory is never re-served to this developer.
2. **A negative attestation to the retrieval layer.** The denial flows to the org's local retrieval/storage component, which mirrors the chain's decay arithmetic locally. Retrieval ranking degrades immediately — a memory denied N times since the last on-chain settlement ranks lower than its chain-recorded weight would imply.

**The optimistic ledger.** The chain remains the eventual source of truth for keyword weights and the decay state of every memory. Between leader-driven on-chain settlements, the local retrieval layer maintains an optimistic mirror: for each memory, the locally-applied decay reproduces the chain's Earned Trust formula (DECISIONS.md D-4.2) — recomputing `denial_rate`, the trust gate, and the per-event decay/boost using the same parameters the chain will apply at settlement. The arithmetic mirrors the chain's exactly, so optimistic and authoritative ranking states are indistinguishable at retrieval time. When the leader settles pending denials on-chain, the local mirror reconciles to the new authoritative weights and resumes from the new baseline.

**Why silent and frictionless.** A denial is the cheap, low-stakes negative signal. UI friction (required reason, confirmation modal, rate caps) suppresses the signal and starves the decay model that depends on it. Reports remain the high-friction path with reporter accountability for cases where a single signal needs disproportionate weight. The two paths are complementary, not duplicative.

**Why no caps or reputation weighting on denials.** A single consumer cannot drive a memory to archived via denials alone. Archive requires every keyword weight to fall below the retrieval threshold (D-4.2, default 1500 bps), which under Earned Trust requires either sustained denial pressure pushing the denial rate above the trust gate, OR no offsetting serves over a long quiet period — both of which require organic volume that one actor cannot fake. Caps would protect against an abuse that has no payoff: a malicious consumer who spams denials can at worst suppress memories from their own recall queue (which they could do anyway through local blacklist). Reputation weighting would create a class system where senior consumers have heavier "votes" than new joiners, require online reputation lookups on the recall hot path, and reproduce the reporter-accountability infrastructure for a signal whose semantics are explicitly lighter.

**Leader-driven settlement cadence.** The leader settles accumulated denials on-chain at a cadence of their choosing. No automatic submission, no threshold prompt, no time-based pressure. The optimistic ledger means retrieval ranking already reflects every denial in real time; chain submission is purely a settlement act — making the decay permanent across local-storage restarts and contributing to the leader's on-chain activity record. The leader optimizes for gas cost versus settlement frequency on their own.

The leader does not review individual denials. Denials are quantitative consumer signals, not editorial content. The leader's role is to settle accumulated decay, not to adjudicate individual user preferences.

---

## 6. Developer Reputation and Social Graph

### 6.1 The Reputation Problem

No platform captures the problem-solving work vibe coders do every day. GitHub shows what you built (commits) but the AI wrote the code. Stack Overflow requires context-switching. LinkedIn is self-reported nonsense. The knowledge generated during daily AI-assisted coding dies in terminal history.

WeVibe solves this with on-chain cross-org reputation. Your memories are tied to your pub key. When they get served across multiple orgs, that's verifiable proof your knowledge is useful. Not self-reported. Not endorsement-based. Backed by on-chain serve attestations across the entire network.

### 6.2 Cross-Org Reputation from Serve Attestations

**This is the product.** Every serve attestation on-chain ties a memory to its contributor pub key. Reputation accumulates automatically across ALL orgs:

- **Serve count.** How many times this contributor's memories have been served across all orgs. Direct measure of usefulness.
- **Org breadth.** How many distinct orgs serve this contributor's memories. Cross-org utility signal.
- **Domain expertise map.** Topic clusters derived from memory keywords and stack tags. "This developer has 47 verified memories in Redis, 23 in Kubernetes, 15 in TypeScript build tooling."
- **Wallet age.** How long the contributor has been active. Longevity signal.
- **Rep score.** Composite: serve count weighted by org breadth and domain concentration. Higher scores unlock higher contribution bandwidth and better payout tiers.

This reputation is visible in the plugin approval UI at the moment of decision. It's the trust signal that helps users evaluate whether to accept a memory from an unknown contributor.

### 6.3 Enhanced Reputation (With Optional Attestation)

When an org enables the attestation subsystem (CommitLLM / proxy), contributor profiles gain additional dimensions:

- **Difficulty distribution.** Histogram of difficulty grades across all attested memories.
- **XP score.** Cumulative: XP = Σ(difficulty_grade × quality_grade) across all attested memories.
- **Verification tier breakdown.** Percentage of memories that are CommitLLM-verified (Tier 1) vs proxy-attested (Tier 2).

### 6.4 Three-Layer User Model

- **Activity (protocol-visible, profile-controlled).** Serve events contribute to contributor reputation on-chain. Retriever identity is pseudonymous by default (per-org serve key). Users control whether their profile displays a public learning trail linking their retrieval activity.
- **Social (public, opt-in).** Users can link their contributor identity, org serve keys, selected domain stats, and selected memories to a public profile. "This person has 47 verified memories in Redis with 214 serves across 12 orgs" becomes a public credential.
- **Institutional (org-local).** Local blacklist, exact session context, denied memories, raw prompt/query data remain local and never leave the org member's machine.

### 6.5 Anti-Gaming Properties

Reputation is based on serves, not submissions. You can submit a thousand memories — if nobody accepts them, your rep is zero. The human approval gate is the anti-gaming mechanism: contributors can only earn by producing knowledge that other developers actually find useful enough to inject into their agents.

**Minimum anti-farming rules for x/serve:**
1. One qualified serve per memory per retriever per epoch (dedup by memory_cid + org_serve_key + epoch)
2. Self-serves earn zero (contributor_pubkey linked to retriever serve key = discounted)
3. Repeated serves of the same memory from the same org have diminishing returns
4. Payout cap per memory per epoch
5. Payout cap per contributor per org per epoch
6. Serve key must have minimum membership age before it counts for payouts
7. Serve weight depends on org credibility (burn amount, age, size)
8. Rejections are aggregated, not exposed as user-level attacks on contributors

**Qualified serve formula:**
```
qualified = active_member AND memory_approved AND not_duplicate 
            AND not_self_serve AND within_org_budget 
            AND serve_key_age >= minimum_age

rep_delta = base_weight × org_weight × novelty_multiplier × saturation_factor
```

Raw serve count is NOT the only factor. Saturated scoring, org weighting, and domain-specific profiles prevent simple farming.

Additional anti-gaming from optional attestation: CommitLLM prevents fabricated sessions, structural scoring catches inflated sessions, LLM grading catches trivial memories.

**Note on leaderboards:** WeVibe does not ship global leaderboards. However, contributor reputation statistics are public on-chain — third parties can build rankings. The protocol mitigates toxic competition by using saturated scores, domain-specific profiles, and quality-weighted serves instead of raw global counts.

### 6.6 Serve Attestation as Economic Primitive

The serve attestation uses per-org pseudonymous serve keys. Each user has:
- `global_contributor_key` — public identity for authorship and reputation
- `org_serve_key` — per-org pseudonymous key used to sign serve batches

The `org_serve_key` proves the user is an active org member and prevents duplicate farming, but does not automatically link all retrieval behavior across orgs. Users can optionally link their org serve keys to their public profile.

**Batch transaction format:**
```
MsgSubmitServeBatch {
  org_id
  epoch_id
  retriever_serve_pubkey
  serves: [
    { memory_cid, contributor_pubkey, serve_nullifier }
  ]
  signature
}
```

Where `serve_nullifier = H("wevibe-serve-v1" || org_id || epoch_id || memory_cid || retriever_serve_secret)` ensures deduplication without exposing the raw relationship.

**Accept + Attest** clicks queue into the batch. **Accept Private** clicks inject the memory but skip the batch entirely — no on-chain record, no payout. Denied memories get no attestation. Contributors only earn for memories people actively chose to attest.

---

## 7. Organization Social Graph

### 7.1 Public Discovery Interface (opt-in)

**Visible to non-members (if public):** Organization name, specialization, description, memory count, member count, age, leader identity, total serves, plus two unfakeable org-health signals introduced below.

**Not visible to non-members:** Memory content (encrypted on-chain), member identities (privacy-preserving), review history, payout rules.

**Unfakeable org-health signals.** Discovery surfaces two behavioral metrics that capture-resistant by construction:

- **Leader last active.** Aggregated timestamp of the most recent on-chain action signed by the org leader's wallet — batch memory commits, denial settlements, member changes, report responses, epoch rotations. The signal requires a real wallet signature on a real transaction paying real gas. A dormant or captured org cannot fake it.
- **Voluntary departure rate.** Members who left of their own accord in the trailing 90 days, expressed as a fraction of total membership. Departures are first-class on-chain events; sybils can be invited and can file reports, but they cannot fake people walking away. A cohort exiting a captured org is the strongest negative signal the public can read.

**What is deliberately NOT surfaced.** Discovery does not display per-org report counts, report aggregates, dispute counts, dismissed-report counts, or any other report-derived statistic. The rationale is structural and is the same as in §5.7: every in-app aggregation of reports is gameable, weaponizable, and censorable. The chain is the public record; the block explorer is the viewer. Prospective joiners who want to investigate report history can do so on-chain; WeVibe's own discovery surface does not turn that history into a leaderboard.

### 7.2 Leader Interface

wevibe-client local dashboard: pending review queue, memory browser, historical decisions, member management, org configuration, keyword taxonomy management, recovery status, direct memory authoring, rep-tier payout configuration, bandwidth usage monitoring, denial-settlement panel, and Tier 2 report response interface.

The denial-settlement panel shows the pending-denial count and a single settle button. There is no per-denial review — denials are quantitative signals that the leader settles on-chain at a cadence of their choice (§5.8).

The Tier 2 report response interface appears only when a Tier 2 report has been published against a memory the leader committed. It exposes the one-reply rule (acknowledge or dispute) clearly, the remaining time in the response window, and a copy-link to the on-chain transaction once the response is published.

### 7.3 Member Interface

Members see: role, contribution count, serve count, pending submission status, reputation score, payout earnings.

### 7.4 Reporter's Private View

Each reporter has a private list of their own Tier 2 published reports. Each entry shows a memory excerpt, the org name, the submission date, the leader's response status (pending / acknowledged / disputed / unaddressed), and a copy-link to the on-chain transaction. This view is visible only to the reporter — it is the reporter's own record of escalations, and the place from which they share block-explorer URLs to whatever public forum they choose. No other user sees it.

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

Plaintext keywords are stored alongside encrypted memories on-chain. This enables keyword-based filtering without decryption. The tradeoff (keyword visibility) is accepted — see Section 3.7.

### 8.3 Semantic Vector Index (Local Only)

Vector embeddings are NOT stored on-chain. Each org member's wevibe-client computes embeddings locally from decrypted memory content and maintains a local vector index. This means:
- No embedding data leaks to the chain
- Each member has a complete, searchable index of their org's memories
- Index rebuild is deterministic — same memories produce same embeddings

### 8.4 Memory Metadata

- **contributor_pubkey** — on-chain identity of the contributor
- **model_origin** — contributing model
- **stack_tags** — freeform technology tags
- **version** — nullable version string
- **source** — `session` | `doc_import` | `authored`
- **provenance** — `commitllm` | `proxy-attested` | `unattested`
- **difficulty_grade** — two-layer difficulty score (if attested)
- **quality_grade** — LLM grading result (if attested)
- **approved** — boolean moderation state
- **quarantined** — flagged by 3+ rejections
- **deprecated** — curator has marked stale

---

## 9. Security Analysis

### 9.1 Sybil Resistance
Invitation-only org membership provides natural Sybil resistance for contribution. Org creation burn provides economic Sybil resistance at the network level.

### 9.2 Memory Poisoning
Defense layers: submission-time wevibe-guard (advisory), OCR sanitization, human review with steganography detection, contributor reputation visible during review, recall-time blacklist, recall-time wevibe-guard, recall-time OCR, artifact extraction with egress enforcement, plugin approval gate with contributor trust signals. Residual risk: semantic payloads, subtly wrong recommendations.

### 9.3 Leader Key Compromise
K_master compromise exposes all epoch-derived content. Mitigation: offline recovery phrase, encrypted vault with Argon2id, threshold recovery.

### 9.4 Chain State Observability
On-chain data is public (encrypted blobs + plaintext metadata). An observer can see: org sizes, submission frequency, keyword distributions, serve patterns, contributor activity, reputation scores. They cannot see: memory content, decryption keys, member identities beyond pub keys, local blacklist state.

### 9.5 Network-Level Anti-DDOS
Protocol-enforced per-org bandwidth caps. Orgs receive submission and storage bandwidth proportional to VIBE burned/staked. Chain rejects submissions exceeding the org's allocation. No org can flood the network regardless of off-chain resources.

### 9.6 Content Suitability Policy

**Suitable:** Coding patterns/anti-patterns, architecture lessons, debugging notes, dependency guidance, tool usage, process workflows, version-specific gotchas, negative knowledge.

**Unsuitable:** Credentials/secrets, customer PII, regulated data, legal/HR records, high-sensitivity security incident details.

### 9.7 Org Capture and Public Escalation

A single actor wearing multiple hats — leader, every moderator, every contributor — can fully capture an org's internal governance. Inside the org, every approval, every report dismissal, and every chain commit can be coordinated. Internal accountability primitives (moderator votes, dispute counts, internal review queues) provide no protection against this case: the captured operator simply approves their own malicious memories and dismisses every report filed against them.

The system's security model is therefore not "prevent capture through internal governance." The system's security model is:

> Make capture economically unsustainable through transparent on-chain accountability, frictionless exit for members, and a public escalation primitive that cannot be suppressed by the captured org.

The three load-bearing properties:

1. **The chain is the unforgeable audit log.** Every consequential action — memory commit, denial settlement, report acknowledgment, dispute publication, member departure — is a signed on-chain transaction. Neither the captured org nor WeVibe-the-protocol nor any platform operator can edit or suppress it after the fact.
2. **Consumers have an escalation path the org cannot close.** A dismissed Tier 1 report can be republished as a Tier 2 on-chain transaction signed by the reporter, paying gas, revealing plaintext, anchored to a contributor-signed hash that the leader could not poison (§5.4). The captured org cannot prevent it, cannot delete it, and cannot suppress it from anyone who navigates to the block explorer URL.
3. **Exit is unfakeable.** Members leaving voluntarily is a first-class on-chain event. Sybils can be invited and can file frivolous reports, but they cannot fake people walking away. The voluntary-departure-rate signal on public discovery (§7.1) lets prospective joiners read the most honest possible signal about whether existing members trust the org.

**The leader bears sole signature.** Co-attestation of moderator pubkeys on leader-signed chain transactions is explicitly removed (§5.3). A leader's chain commit binds the leader's wallet only. This concentrates responsibility on the actor who actually signs and prevents implicating moderators in chain-level decisions they did not directly authorize. The trade-off — moderator accountability for individual approvals becomes an org-local rather than chain-public concern — is acceptable because internal moderator vote history cannot defend against a capture scenario anyway, and because making leaders sole signatories sharpens the public attribution of every consequential action.

**Why no platform tribunal.** WeVibe deliberately does not become a tribunal that adjudicates published reports. Any in-app judging body is a capture vector — whoever controls the tribunal controls the verdict. The chain publishes the evidence; the block explorer renders it; the reporter shares the URL wherever they choose; the public judges on its own merits in whatever forums it chooses. WeVibe-the-protocol has no editorial role in that judgment. (§5.7)

**Residual: contributor-leader collusion.** If the contributor who submitted a memory and the leader who committed it are both adversaries, the contributor can sign a false plaintext hash, the leader can commit it, and a Tier 2 report's plaintext reveal will not match the on-chain hash — making the report look invalid even when the underlying claim is true. The on-chain ciphertext + capsule remain as a final backstop: any future key disclosure (epoch rotation, org closure, legal discovery) lets independent parties decrypt and verify after the fact (§5.4). This residual cannot be eliminated at submit time — when content creator and content approver are the same adversary, there is no honest party in the verification chain to sign over.

---

## 10. Decentralized Architecture

### 10.1 Chain Architecture

WeVibe's chain is a sovereign L1 appchain built on Cosmos SDK + CometBFT. Not a rollup — WeVibe requires deterministic finality (CometBFT provides this; rollups have multi-day challenge windows). The chain halts before it forks — safety-over-liveness is correct for memory attestation and storage.

### 10.2 The Four Roles

**Developer (user).** Codes with an LLM. Memories accumulate as exhaust. Never holds VIBE tokens, never thinks about chains. Experience: "I code, my memories accumulate, my profile shows what I've solved." WeVibe runs invisibly in the background after initial opt-in.

**Org leader (curator).** Creates orgs (burns VIBE), pays for contributor bandwidth, curates memories, manages membership, sets rep-tier payout rules. The org leader is the economic actor and quality gatekeeper.

**Validator.** Stakes VIBE, runs CometBFT consensus, stores all chain state (including encrypted memories), earns staking rewards. Everything deterministic — no subjective judgments. Validators are the storage and availability layer.

**WeVibe-the-protocol.** Open-source software. No company in the middle. WeVibe-the-company may run validators and operate orgs early on, but the protocol does not depend on any single entity.

### 10.3 Token Economics

Single token: **VIBE**. Used for staking, org creation burns, bandwidth allocation, and contributor payouts.

**Dynamic org pricing.** Org creation costs VIBE, algorithmically adjusted (Bittensor-style: creation pushes price up, time decays it down). Burned, not paid to anyone. Prevents spam-org attacks.

**Annual renewal.** Flat rate for continued bandwidth allocation. Non-renewal marks the org dormant — memories persist on-chain but no new submissions or serves earn rewards.

**Bandwidth allocation.** VIBE burned/staked by an org determines its per-epoch submission cap and storage budget. This is the anti-DDOS mechanism and the economic unit that makes the network sustainable.

**Contributor payouts.** Org sets rep-tier rules: contributors with reputation in tier X earn Y VIBE per serve. The org funds these payouts. Contributors never need their own tokens.

**Rep-tier example:**
| Tier | Rep Range | Contributions/Day | Payout per Serve |
|------|-----------|-------------------|-----------------|
| 1 | 0–50 | 3 | 1 VIBE |
| 2 | 51–200 | 10 | 3 VIBE |
| 3 | 201+ | 50 | 5 VIBE |

Org leaders can adjust these tiers. Protocol enforces them on-chain.

#### 10.3.1 Emission Schedule (Sprint 32, locked — see DECISIONS D-S32-TOKENOMICS-LOCKED)

Beyond org-funded contributor payouts (above), the protocol mints VIBE on a fixed **32-year schedule** from genesis:

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

> Status: schedule constants and attribution model are locked; implementation lands in CO-041 (the flat `daily_mint` placeholder is replaced by the pool model above).

### 10.4 Validator Economics

Validators earn standard Cosmos SDK staking rewards for running consensus. Additionally, validators store all encrypted memories as part of chain state — this is not separate "operator work," it's inherent to running a node. No separate storage challenges needed.

### 10.5 Serve Attestation Economics

The serve attestation is the core economic event:
1. Developer approves a memory in the plugin
2. Plugin submits serve attestation on-chain (signed by retrieving user)
3. Chain credits the contributor's reputation
4. If org payout rules allow, VIBE flows from org treasury to contributor

This creates a clean incentive loop: contributors produce useful knowledge → it gets served → they earn reputation and tokens → higher reputation earns better payout tiers → incentive to produce more useful knowledge.

### 10.6 On-Chain Modules

Six custom Cosmos SDK modules:

- `x/org` — registration (dynamic burn pricing), renewal, membership, bandwidth quota accounting, rep-tier payout configuration, dormancy detection
- `x/memory` — pending commitment storage (hash + metadata, no blob until approved), approved memory blob storage (encrypted ciphertext as chain state), Merkle root submissions per epoch, pending commitment expiry/removal, quarantine flagging
- `x/serve` — batched serve attestation recording (per-org pseudonymous serve keys), deduplication (memory_cid + serve_key + epoch), self-serve detection/discounting, payout caps and org budget enforcement, contributor cross-org serve count aggregation, payout trigger from org treasury to contributor
- `x/reputation` — per-contributor cross-org aggregated stats (serve count, org breadth, domain tags, rep score, wallet age). Enhanced mode per-org when attestation enabled (difficulty histogram, XP, provenance breakdown).
- `x/emissions` — validator staking rewards, contributor payout distribution from org treasuries, protocol-level emission schedule
- `x/bandwidth` — per-org submission and storage caps, DDOS protection, bandwidth allocation based on VIBE burned/staked, rate limiting enforcement

Standard SDK modules: `x/staking`, `x/auth`, `x/bank`, `x/gov` (wired for on-chain param updates), `x/slashing`, `x/distribution`.

**Modules eliminated from v1.x:** `x/serving` (no separate operator serving sets — validators serve all), `x/challenge` (no storage challenges — consensus IS storage), `x/receipt` (replaced by x/serve), `x/operator` (merged into standard validator staking), `x/attestation` (split into x/memory + x/serve for clean separation of concerns).

### 10.7 Node Lifecycle (`wevibed`)

The chain ships a runnable Cosmos SDK application and CLI (`wevibed`). A single-node bootstrap follows the standard pattern:

1. `wevibed init {moniker} --chain-id {id}`
2. `wevibed keys add validator --keyring-backend test`
3. `wevibed genesis add-genesis-account {addr} 100000000uvibe`
4. `wevibed genesis gentx validator 50000000uvibe --chain-id {id}`
5. `wevibed genesis collect-gentxs`
6. `wevibed start`

---

## 11. Skill Model and Federation

### 11.1 Task-Context Skills

Curator-defined collections organized by task context. Skills are simple named sets of memories with a description. Ungrouped memories are allowed. Skill assignment is optional.

### 11.2 Cold-Start: Documentation Seeding

New organizations import canonical documentation as seed memories (source: `doc_import`). Session contributions extend and replace seeds over time.

### 11.3 Federation (Design Phase)

Federation operates at the skill level. Orgs publish skill packages. Receiving orgs set quality thresholds. No individual contributor reputation crosses federation boundaries. Implementation deferred until basic curation loop is proven.

---

## 12. Implementation Phases

### Phase I: On-Chain Memory MVP (Current)

- wevibe-chain: Cosmos SDK app with x/org, x/memory, x/serve, x/reputation, x/emissions, x/bandwidth
- wevibe-client: local retrieval, key management, chain sync, vector index, submission pipeline
- wevibe-guard: prompt injection scanning, memory injection gating
- Plugins: OpenCode (primary), Claude Code, Cursor, Cline
- On-chain memory storage (encrypted blobs)
- Serve attestation on-chain
- Contributor reputation from serves
- Org economic model (burn to create, bandwidth allocation, rep-tier payouts)
- Anti-DDOS at protocol level

**Exit criteria:** Working end-to-end flow: session → extraction → review → on-chain storage → retrieval → approval with trust signals → serve attestation → contributor earns rep.

### Phase II: Curator Workbench + Session Attestation (Sprint 23-25)

- CommitLLM bridge for open-weight session verification
- Proxy attestation for closed-weight sessions
- Two-layer difficulty scoring (structural + LLM grading)
- Enhanced developer reputation profiles
- Dashboard functional moderation (approve/deny with keyword extraction + embedding)
- Direct memory authoring, documentation import, skill creation
- Quality signal visibility, gap analysis

### Phase III: Network Expansion

- Mainnet launch with bootstrap credit pool
- Skill-level federation: publish and subscribe
- SDK for non-MCP integrations
- Enterprise features: SLA-backed retrieval, dedicated infrastructure
- Public artifact markets (evidence-gated)
- Token liquidity (evidence-gated)

---

## 13. Open Questions

**Context profiling depth.** How much session context should wevibe-client gather at startup? Dependencies and directory structure are cheap. Current file content is expensive. Needs calibration.

**Pending memory retention window.** How long do pending (unreviewed) memories stay on-chain before auto-purging? 72 hours? Configurable per org?

**On-chain storage limits.** At what scale does on-chain memory storage become impractical? 100K memories? 1M? Need to model chain state growth and validator hardware requirements.

**Embedding model evolution.** nomic-embed-text confirmed for now. Future upgrades mean re-embedding all memories — this happens locally but needs coordination across org members.

**Cross-org retrieval.** Can a member of Org A retrieve memories from Org B? If so, how are keys shared? Federation at the skill level is designed but not implemented.

**Chain open questions.** Dynamic pricing curve parameters, bandwidth sizing per VIBE burned, validator hardware requirements for memory storage, rep-tier parameter governance, bootstrap-to-steady-state transition.

**Anti-collusion for serve attestations.** Self-serves (contributor == retriever) are discountable. Cross-user collusion needs monitoring — serve pattern analysis, payout caps per contributor per epoch, cooldowns between repeated serves of the same memory.

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
