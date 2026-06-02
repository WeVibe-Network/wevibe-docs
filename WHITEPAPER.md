# WeVibe Network

**Social Reputation + Shared Memory for Vibe Coders**
*Build public coding reputation and collectible badges from daily agent work; shared memory powers your next session as the bonus.*

Draft v2.3 · June 2026 · Architecture Document
Classification: Confidential — Not for public distribution

---

## Revision notes (v2.2 → v2.3)

**Repositioning: social-first for individual vibe coders and small crews.**

v2.3 rewrites the opening narrative around the alpha product and current architecture:

1. **Audience is explicit.** WeVibe is for individual vibe coders and small crews, not enterprise procurement funnels or billion-dollar engineering organizations.
2. **Hook is social momentum.** Public reputation and collectible badges from daily coding work are the front door; memory recall remains the practical bonus that makes agents better.
3. **Org framing is corrected.** Orgs are domain-expert-run memory collections users join. The org is the container; the vibe coder is the protagonist.
4. **Keyword posture is clarified.** Public plaintext keywords are discovery metadata, treated as a feature (not a privacy alarm).
5. **Stale architecture claim fixed.** The prior "wevibe-hub eliminated" claim is removed. In alpha, the hub is live and part of the hosted coordination/accounting path alongside chain + local retrieval.

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

**Architecture pivot: on-chain storage + hub retrieval + human-gated attribution signals.**

v2.0 introduced the architecture that current alpha still builds from: encrypted memories on-chain, hub-served retrieval with local decryption/safety gating, and plugin-gated human approval before agent injection. The key changes:

1. **Memories stored on-chain.** Encrypted memory blobs go directly on the WeVibe chain as state.
2. **Hub retrieval path with local trust boundary.** The MCP/plugin computes query embeddings locally and sends vectors to wevibe-hub; the hub runs vector retrieval and returns IDs + metadata; ciphertext is decrypted locally before any injection.
3. **Hub role is explicit and live.** wevibe-hub runs hosted coordination/accounting and retrieval in alpha; it is not eliminated.
4. **Serve attribution as social signal.** Human-approved serves are attributed on-chain to contributor and org aggregates for public reputation/badge surfaces; this is social/status data, not per-serve payout.
5. **Domain-expert-led org operations.** Leaders set domain focus, reviewer standards, and org policies; members join for curated memory quality and social credibility.
6. **Three software pieces.** wevibe-chain (source of truth), wevibe-hub (retrieval + coordination/accounting), and MCP server + plugin (local decryption, guard, human gate, injection).

Sprint 22 chain hardening work (CO-162 through CO-170) remains valid — the Cosmos SDK app, module infrastructure, CometBFT consensus, and operator economics are the foundation this pivot builds on.

---

## Abstract

WeVibe ships in alpha as a social-first network for individual vibe coders and small crews. You install a plugin, contribute what actually worked in real coding sessions, and watch your public reputation and badges grow over time. That daily momentum is the hook. Shared memory recall is the bonus: your next agent run starts with better context instead of starting from zero.

Domain experts run organizations as curated memory collections you can join. Leaders and reviewers decide what gets committed, what gets rejected, and what stays useful as frameworks change. Public plaintext keywords are intentional discovery signals (for example, `redis`, `solana`, `django`) so coders can quickly find the communities that match their stack.

The chain anchors provenance, membership, and attribution signals; the plugin enforces human-in-the-loop memory injection; local retrieval keeps plaintext close to the coder. In alpha, the hub is live for coordination/accounting workflows. The economic model is decided, but some demand-leg and settlement mechanics are still being implemented; this document describes what ships now and what is explicitly near-term.

---

## 1. Design Philosophy

### 1.1 The Problem

Vibe coders do hard problem-solving with agents every day, but each new session still behaves like a cold start. A fix discovered today — the precise Nginx keepalive tweak, the one migration order that prevents data loss, the exact flag combination for a flaky deploy — is usually trapped in private chat logs and forgotten tomorrow.

This hurts most when working with local or smaller models. The model can write code, but it does not know your stack's lived edge-cases, version mismatches, and production scars. The gap is not "more generic internet text"; the gap is verified, domain-specific memory from people who actually ran the problem.

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

### 1.3 The Curator Workbench

WeVibe is a curator workbench, not an autonomous ranking machine. The system surfaces signals — retrieval frequency, denials, staleness, query gaps, version drift — and human curators decide the action. This preserves accountable judgment where it belongs: with domain people who understand context.

Core workflows remain practical and hands-on: review pending memories (approve/reject/edit), author memories directly, package memories into reusable skills, identify coverage gaps, and retire stale entries before they mislead downstream users.

This curation loop serves both halves of the alpha wedge. It improves recall quality (the memory bonus that helps the next coding session), and it protects the integrity of public reputation/badges by ensuring attributed memories actually deserve trust.

### 1.4 The Plugin Gate

Every memory must pass through human eyes before it enters an agent's context. This is the product invariant.

The plugin is installed in the developer's coding environment (OpenCode, Claude Code, Cursor, Cline, and similar tools). When the agent asks for context, the plugin retrieves candidate memories from local retrieval, runs wevibe-guard checks, and renders an approval UI:

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
│ [✓ Accept + Attribute] [◉ Accept Private]│
│ [✗ Deny]                                 │
└──────────────────────────────────────────┘
```

**Accept + Attribute:** Memory is injected into agent context. Serve attribution is queued to chain aggregates (contributor + org) for public profile and badge signals. No direct per-serve payout is implied by this action.

**Accept Private:** Memory is injected into agent context without public serve attribution. Useful for private sessions. Leaders can configure whether private accepts are allowed.

**Deny:** Memory is blocked. The plugin can capture lightweight deny context so curators can improve quality without turning the UX into a courtroom.

**No plugin installed = no memory injection path.** The MCP server has no direct route to force memory into the agent context without the plugin frontend.

**Why not MCP elicitation?** Elicitation is useful in theory, but inconsistent across clients and weak as a hard-interrupt safety surface. The plugin provides deterministic interruption, clear modal UX, and explicit confirmation.

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

**First-run detection.** When the MCP plugin/server starts and discovers no org membership, it surfaces an actionable message to the agent, prompting guided setup.

**Operation.** Members join through leader invitation. Once approved, the leader issues sealed key envelopes containing the epoch keys to the new member. For reviewers and leaders, the envelope also includes SK_mod(e) for the current epoch.

**Contributor onboarding.** Contributors opt in once — they install the plugin, connect the MCP server, and join the org. After that, WeVibe runs invisibly in the background. Sessions are mined for memories automatically. The contributor never needs to actively trigger contributions. The MCP server + plugin path handles extraction and sanitization, then submits encrypted contributions through the hub coordination path to the chain.

**Key rotation (epoch advancement).** When a member is removed, the org enters `rotation_pending` state:

1. **Removal triggers `rotation_pending`.** The chain marks the org as pending rotation. The removed member's envelope is deleted.
2. **New submissions are buffered.** Contributors can still submit, but submissions enter an MCP-side local buffer — not admitted to the chain, not indexed in hub retrieval, not assigned a final epoch.
3. **Leader completes rotation.** The leader derives new epoch keys from K_master via HKDF, generates a new moderation keypair SK_mod(e+1)/PK_mod(e+1), and re-seals envelopes for all remaining members.
4. **Buffer finalizes.** After rotation completes, buffered submissions are released to the chain under the new epoch.
5. **Grace period escalation.** If rotation is not completed within a configurable window (default: 72 hours), the org's submission bandwidth is suspended.

**Revocation semantics.** Epoch rotation provides forward secrecy only. Removed members retain previously-distributed epoch keys and can decrypt content from their membership period.

### 2.4 The Three Software Pieces

WeVibe's alpha architecture has three software pieces: **wevibe-chain**, **wevibe-hub**, and the **MCP server + plugin**. The hub is live and serves retrieval in alpha; there is no separate `wevibe-client` local-retrieval replacement path.

**wevibe-chain (source of truth):**
Cosmos SDK + CometBFT sovereign L1 appchain. Stores encrypted memory blobs, provenance/attribution, org state, serve attestations, and economic state. Validators maintain consensus and replicate state. They never see plaintext memory content.

**wevibe-hub (`wevibe-server`, live coordination + retrieval plane):**
Runs coordination and accounting workflows, and serves the live Qdrant-backed retrieval path in alpha. Hub retrieval is the serving path exercised by the gate harness; the hub is not eliminated or replaced.

**MCP server + plugin (local safety + approval + injection):**
Platform-specific plugin gates (OpenCode, Claude Code, Cursor, Cline) register tools in the agent and call a local MCP server. This local path enforces guard/sanitization, presents the human approval UX, and injects approved context into the agent. It mediates access to hub retrieval and chain attestations; it does not replace hub serving.

### 2.5 Tool Surface

The plugin registers tools in the coding agent. The local MCP server provides the guard/approval backend, and wevibe-hub provides coordination/accounting plus retrieval. The separation:

**Plugin-registered tools (visible to the agent):**

| Tool | Purpose |
|------|---------|
| `wevibe_recall` | Search organizational memory. Plugin calls the MCP server, which requests candidates from wevibe-hub's retrieval path, runs wevibe-guard scan, renders approval UI with contributor reputation, and injects approved memories as `context:` ambient content. On approval, serves are attested on-chain. |
| `wevibe_contribute` | Record technical learnings. Agent calls this at natural phase transitions. MCP-side extraction produces atomic memories, sanitizes and encrypts them, then submits through the hub coordination path to chain. |
| `wevibe_reject` | Flag a recalled memory as unhelpful. Adds to local blacklist and reports feedback on-chain for quarantine. |

**MCP server backend (invisible to the agent):**

The MCP server enforces local guardrails, approval-state handling, and secure context injection. Retrieval candidates come from wevibe-hub's Qdrant-backed serving path. The MCP server returns structured candidate data to the plugin — never directly to the agent.

**Contribution behavior.** The `wevibe_contribute` tool description instructs agents on when to contribute: at natural transition points during sessions, when they discover something non-obvious. Negative knowledge (what NOT to do and why) is especially valuable. In practice, contribution is mostly automatic — the session buffer captures learnings and the extraction pipeline processes them without developer intervention.

**Session buffer safety net.** A session buffer is initialized lazily on the first tool call and records session activity. If the agent does not explicitly contribute during the session, `autoContribute()` fires on session exit. On next session startup, orphaned buffers from crashed sessions are processed.

All administrative operations (org creation, member invitation, moderation, epoch rotation, keyword management, recovery) are handled by the separate `wevibe-admin` CLI and hub control-plane workflows.

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

Retrieval is hub-based, with plaintext handling kept local in the MCP server + plugin path. The pipeline:

#### Context Profiling (Session Start)

When a coding session starts, the MCP/plugin profiles the environment — dependencies, directory structure, language, framework versions, current file context. This profile is sent as filter context so the hub can pre-filter candidate memories before vector scoring. A developer working in a Python/Django project should search Python/Django memories, not the entire org corpus.

#### Keyword Extraction
Keywords are extracted by the host agent's LLM at approval time: 10-20 domain-specific keywords with percentage-based weights summing to 100%. Keyword weights are stored as retrieval metadata and used during hub scoring (plaintext keywords remain an accepted metadata tradeoff — see Section 3.7).

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
capped_boost = min(γ × keyword_boost, δ × vector_score)
final_score = vector_score + capped_boost

Default: γ = 0.1, δ = 0.15
```

Vector similarity drives recall (hub-side Qdrant top-30 by cosine). Keywords provide a capped boost to break ties within semantic clusters after context pre-filtering.

**Model-origin prior.** Soft prior: generic conceptual memories from lower-capability models deprioritized for higher-capability retrievers; highly specific memories receive no penalty regardless of origin.

**Blacklist and quarantine filtering.** Chain-level quarantine flag (`quarantined=true` after 3+ rejections) is available to retrieval policy. The MCP/plugin excludes locally blacklisted CIDs before approval.

#### Candidate Fetch + Local Decryption
Hub retrieval returns memory IDs + metadata + matched keywords. The MCP/plugin then fetches each memory's ciphertext from hub storage, decrypts locally through the Umbral sidecar, runs wevibe-guard + human gate, and only then injects approved context.

#### Selective Re-ranking
When top-2 scores are within ε=0.20 (contested query), the MCP/plugin can use the host agent's LLM to re-rank. Fallback: original order preserved on error.

### 3.6 Side Channel: On-Chain Metadata

With memories stored on-chain and retrieval served by the hub, metadata is observable across two hosted surfaces. On-chain/public observers can see org IDs, contributor pub keys, submission timestamps, memory sizes, keyword terms/weights (plaintext), serve attestation patterns, and reputation scores. In the hub, Qdrant stores embedding vectors plus keyword metadata (`cid`, `org`, `keyword_weights`, `lifecycle`, `type`), while ciphertext is stored in Postgres/chain paths for retrieval.

The privacy boundary is decrypted plaintext: decryption, wevibe-guard sanitization, human approval, and context injection happen locally in the MCP/plugin path. The honest claim is that **the hub never sees your decrypted memory content** — not that nothing leaves your machine.

### 3.7 Metadata Visibility Model

WeVibe orgs are public developer communities, not private enterprises. On-chain metadata is intentionally public — it enables discovery, reputation, and the social graph.

**On-chain (public by design):** Org IDs, org topic tags, contributor pub keys, encrypted memory blobs, plaintext keyword terms/weights (discovery signal — "this org covers Redis"), submission timestamps, memory sizes, epoch boundaries, serve attestations (batched per epoch), reputation aggregates, bandwidth consumption, quarantine state.

**Local to the MCP/plugin (the hub never sees these):** Decrypted memory plaintext, local wevibe-guard/blacklist state, and session context profiles. (Embedding vectors and keyword-weight metadata live in the hub's Qdrant; the hub stores ciphertext + vectors but never decrypts — see §3.6 and §8.3.)

Plaintext keywords on-chain are a feature, not a leak. They tell developers what an org covers and help with cross-org discovery. Developers who join an org to boost their LLM need to know what domain knowledge it offers. The keywords serve that purpose.

### 3.8 Defense-in-Depth: Memory Sanitization Pipeline

WeVibe's security model focuses on what it can control: the form and content of recalled memory before it reaches the agent. Decryption, sanitization, approval, and injection run locally in the MCP/plugin path. Retrieval remains hub-served (Qdrant vectors + keyword metadata, ciphertext in Postgres/chain), but the hub never sees decrypted plaintext.

#### The Pipeline

**Submission time (before on-chain storage):**
1. **wevibe-guard scan.** YARA rules for injection patterns, credential detection, exfiltration matching, unicode mathematical injection detection. Advisory: warns but does not block. The moderator is the security boundary.
2. **OCR sanitization.** Text rendered to image via ImageMagick, OCR'd back via Tesseract. Destroys Unicode tricks, zero-width characters, homoglyphs, invisible formatting.
3. **Encryption.** Memory encrypted with per-memory DEK, DEK sealed to moderation public key.
4. **Human review.** Reviewer decrypts locally, reads plaintext, steganography scan, approve/deny.
5. **On-chain submission.** Approved memory (ciphertext + wrapped DEK + metadata) goes on-chain and is mirrored to hub storage for retrieval serving. Org pays the submission cost.

**Recall time (before delivery):**
6. **Hub candidate query.** MCP/plugin posts the local query vector to hub retrieval and receives memory IDs + metadata + matched keywords.
7. **Ciphertext fetch + local decryption.** MCP/plugin fetches candidate ciphertext from hub storage and decrypts locally through the Umbral sidecar.
8. **Blacklist filter.** MCP/plugin checks local blacklist.
9. **wevibe-guard scan.** Same scan on decrypted memory at recall time. Catches payloads undetectable when approved (new rules since approval).
10. **OCR sanitization.** Same format-breaking pipeline.
11. **Artifact extraction and egress enforcement.** Typed artifact extraction: URLs, bare domains, IPv4 addresses, shell commands, package install commands, config directives. Every network-resolvable token flags.
12. **Plugin approval gate.** Plugin renders approval UI with wevibe-guard detection results AND contributor trust signals (pub key, wallet age, rep score, serve count, domain expertise). User sees the memory, sees the flags, sees who wrote it, approves or denies.
13. **Serve attestation.** On approval, the MCP/plugin path queues and submits a serve attestation on-chain (signed by the retrieving user).
14. **Context injection.** Approved memories formatted as `context:\n{memory content}` and injected into agent prompt.

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

**Hub-based retrieval with local decryption.** wevibe-hub runs Qdrant vector search over embeddings + keyword metadata; MCP/plugin computes query embeddings locally, receives IDs + metadata + matched keywords, fetches ciphertext, and decrypts locally before sanitization/injection.

**Contributors are paid by the network; members pay orgs for access.** Contributor rewards are contribution-only network emissions. Access demand is separate: members pay orgs in VIBE for recall access, and leaders earn from that demand leg (with settlement/burn mechanics in active alpha build-out).

**Serve attestations: public reputation, pseudonymous retrieval.** Contributor reputation (serve counts, domain tags, payout amounts) is public on-chain. Retriever identity is represented by a per-org pseudonymous serve key — not the user's global contributor identity. This separates "my knowledge helped others" (public) from "this exact user needed this exact memory" (pseudonymous). Users can optionally link their org serve keys to their public profile as a learning trail.

**Serve attestations are batched per epoch.** Plugin queues approvals locally. The MCP/plugin path (or hub-serving key path, when configured) submits one batch transaction per user per epoch using the org serve key. Not one tx per click.

**Three-button approval UX.** Plugin offers: [Accept + Attest] (memory injected, serve attestation queued, contributor earns), [Accept Privately] (memory injected, no attestation, no payout — for stealth sessions), [Deny] (memory blocked, feedback logged). Public orgs with payouts may require attestation. Personal/local orgs may allow private accepts.

### 3.10 Session Attestation (Roadmap, Post-Mainnet)

Sessions produce memories. Without provenance attestation, a contributor could paste fabricated "coding sessions" into the extraction pipeline and farm reputation. Everything downstream — difficulty scoring, quality grading, reputation — depends on knowing the session actually happened.

**Current alpha posture:** org leader curation is the trust layer. We do not yet have a generalized attestation rail in production.

**Roadmap direction (D-ATTEST-ROADMAP):** the optional subsystem described here is the seed for a post-mainnet **pluggable attestation framework**. Separate attestation components will plug into the chain and validate session claims either **cryptographically** or **via API-backed trust services**. The target claim shape is explicit session provenance, for example: *"user X using LLM model Y took N turns to solve problem Z."*

This is a major roadmap item and the infrastructure is not there yet. It is carried as a forward design, not claimed as live.

#### Lineage from the optional subsystem (seed designs)

**Tier 1: CommitLLM Cryptographic Receipts (Open-Weight Models).** CommitLLM (Lambda Class, MIT licensed) is a cryptographic commit-and-audit protocol for open-weight LLM inference. The receipt proves "this response was produced by this exact model with these exact weights." Limitation: only works for open-weight models where the verifier has the public checkpoint.

**Tier 2: Proxy-as-Trust-Layer (Closed-Weight Models).** For closed-weight API models, traffic can route through a WeVibe-controlled session proxy that content-addresses each turn and signs the transcript with WeVibe's key. This is weaker than Tier 1, but provides a practical API-trust path.

### 3.11 Two-Layer Difficulty Scoring (Roadmap Consumer, Requires Attestation)

Two-layer difficulty scoring is the evolutionary continuation of the optional design above and a likely early consumer of attested session claims once the pluggable framework exists.

Like attestation, this is post-mainnet roadmap work: the chain-side plumbing and integrations are not yet in place.

How attested difficulty should enhance the economic layer and/or social-graph layer is intentionally **TBD**.

#### Layer 1: Structural Signal (Automated, Cheap)
Model capability coefficient × turn count × (1 + 0.25 × failed alternatives). Computed from session structure without understanding content.

#### Layer 2: LLM Grading (Semantic, Authoritative)
Separate grading LLM evaluates non-obviousness, specificity, and reasoning progression. Temperature 0, deterministic, hash-seeded. The grade and session hash are committed together.

---

## 4. Local Architecture (MCP Plugin + Sidecars)

### 4.1 Local Footprint and Responsibilities

In alpha, the local software footprint on a member machine is:

- **MCP server + agent plugin** (OpenCode, Claude Code, Cursor, Cline shims)
- **Umbral decryption sidecar**
- **wevibe-guard binary**

The local path is responsible for both recall gating and contribution packaging.

**Recall-side responsibilities (local):**
1. Compute the query embedding locally via Ollama (`nomic-embed-text`).
2. Send the query vector (plus context filters) to the hub endpoint (`/v1/orgs/{org}/query`).
3. Fetch candidate ciphertext from hub storage.
4. Decrypt locally through the Umbral sidecar.
5. Run wevibe-guard sanitization/policy checks.
6. Present the human approval gate and inject only approved context.

**Contribution-side responsibilities (local):**
1. Extract session learnings (manual tool call or auto-contribute path).
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

After bootstrap, recall runs as request/response against the hub retrieval service. The local machine does not download the org's full memory corpus and build its own vector index. Local state is operational (keys/config + approval/attestation queueing), while authoritative memory storage and vector retrieval remain hub/chain-side.

### 4.4 Retrieval Flow (Plugin-Gated)

```
Developer prompts their coding agent
     │
     ▼
  Agent calls wevibe_recall (registered by plugin)
     │
     ▼
  Plugin calls local MCP with query + session context profile
     │
     ▼
  Local MCP/plugin path:
      1. Build retrieval filters from session context
      2. Compute query embedding locally via Ollama (`nomic-embed-text`)
      3. POST vector to hub `/v1/orgs/{org}/query`
     │
     ▼
  Hub retrieval path:
      4. Qdrant vector search over hub-hosted embeddings + keyword metadata
      5. Return candidate IDs + metadata + matched keywords
     │
     ▼
  Local MCP/plugin path:
      6. Fetch candidate ciphertext from hub storage
      7. Decrypt locally via Umbral sidecar
      8. Run wevibe-guard sanitization + policy filters
      9. Fetch contributor reputation/trust signals from chain
     10. Render human gate with memory + safety output + trust signals
     │
     ├── ACCEPT + ATTEST → inject context
     │                     queue serve attestation (batched per epoch)
     │
     ├── ACCEPT PRIVATE → inject context only
     │
     ├── DENIED → block memory and record feedback
     │
     ▼
  Agent continues with or without memory
```

### 4.5 Contribution Flow

```
Agent calls wevibe_contribute (or autoContribute on session exit)
     │
     ▼
  1. Extract candidate memories from local session context
  2. Run wevibe-guard scan (advisory) and sanitization steps
  3. Generate fresh DEK, encrypt plaintext, seal to PK_mod(e)
  4. Contributor signs submission hash/canonical body
  5. Submit COMMITMENT to chain (hash, org_id, contributor_pubkey, expiry, size)
     Org pays bandwidth for the commitment transaction.
  6. Deliver encrypted blob to reviewer via temporary off-chain path
     (local transfer, P2P, or org-hosted mailbox)
  7. Return: "WeVibe: captured N learning(s). Pending review."
```

**Pending memory lifecycle:** Only the commitment is written on-chain initially. The encrypted blob is reviewed out-of-band; approval finalizes chain state, while rejection/expiry deletes the pending commitment and temporary blob.

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

---

## 5. Content Review

### 5.1 Review Flow

All contributed memories are submitted to the chain as pending (encrypted, only the org's reviewers can decrypt). Pending memories are visible only to reviewers and leaders via the hub's hosted review dashboard (wevibe-dashboard).

This review layer is not just about safety. It is the quality gate that protects WeVibe's social layer: the public contributor/org attribution counters and badge status signals are only meaningful if low-quality or malicious memories are filtered before approval.

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

Denials and reports are status/accountability signals, not direct payout triggers. WeVibe keeps the social signal and economic payout paths decoupled.

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
2. **Rarity-tier badges** — derived from per-memory keyword supply/demand tiers, computed once at commit and frozen on-chain.
3. **Contribution-volume badges** — thresholds based on approved memory contribution volume.

Scoping is per-org, with profile breakdowns instead of a single global ladder. This keeps competition bounded and useful: you can see where someone built reputation, without forcing all contributors into one network-wide leaderboard.

Badge status is strictly non-economic: no VIBE reward, no emissions multiplier, no payout coupling.

**Alpha status honesty:** badge rendering and the rarity-tier pipeline are near-term alpha work. Rarity-tier semantics are design-stage under GAP-RARITY-1 and are documented as such; they are not presented as fully rolled out today.

### 6.6 Signal Integrity and Anti-Gaming

Even as status-only signals, attribution must stay hard to fake. The reputation layer applies anti-farming guardrails such as deduplication per memory/retriever/epoch, self-serve discounting, and saturation logic on repeated serves.

Human review (§5) remains the first anti-gaming gate: low-quality memories should fail approval before they can accrue social status. Denial and report systems (§5.5–§5.8) provide additional negative feedback signals without coupling directly to token payout.

Optional attestation dimensions (difficulty/verification quality) remain roadmap/alpha-track additions and are documented as near-term expansion points, not as universally deployed defaults.

---

## 7. Organization Social Graph

### 7.1 Public Discovery Interface (opt-in)

**Visible to non-members (if public):** Organization name, specialization, description, memory count, member count, age, leader identity, total serves, social badge summary, and two unfakeable org-health signals introduced below.

**Not visible to non-members:** Memory content (encrypted on-chain), member identities (privacy-preserving), review history, payout rules.

**Unfakeable org-health signals.** Discovery surfaces two behavioral metrics that capture-resistant by construction:

- **Leader last active.** Aggregated timestamp of the most recent on-chain action signed by the org leader's wallet — batch memory commits, denial settlements, member changes, report responses, epoch rotations. The signal requires a real wallet signature on a real transaction paying real gas. A dormant or captured org cannot fake it.
- **Voluntary departure rate.** Members who left of their own accord in the trailing 90 days, expressed as a fraction of total membership. Departures are first-class on-chain events; sybils can be invited and can file reports, but they cannot fake people walking away. A cohort exiting a captured org is the strongest negative signal the public can read.

**What is deliberately NOT surfaced.** Discovery does not display per-org report counts, report aggregates, dispute counts, dismissed-report counts, or any other report-derived statistic. The rationale is structural and is the same as in §5.7: every in-app aggregation of reports is gameable, weaponizable, and censorable. The chain is the public record; the block explorer is the viewer. Prospective joiners who want to investigate report history can do so on-chain; WeVibe's own discovery surface does not turn that history into a leaderboard.

### 7.2 Badge Scoping and Canonical Criteria

Organization profiles expose badge state in a per-org breakdown for both contributors and the org aggregate itself.

- **Per-org scope:** badges are earned and displayed in org context, then optionally summarized on contributor profiles.
- **No cross-org leaderboard:** WeVibe does not publish a global rank table.
- **Canonical criteria for display tiers:** rarity tier is chain-native; serve-milestone and contribution-volume thresholds come from a canonical reference spec used by the reference social graph so labels like "Legendary" remain consistent across forks.

This canonical-spec-in-display approach preserves fork freedom while keeping badge semantics legible across the ecosystem.

### 7.3 Leader Interface

Hub-hosted web dashboard (`wevibe-dashboard`): pending review queue, memory browser, historical decisions, member management, org configuration, keyword taxonomy management, recovery status, direct memory authoring, bandwidth usage monitoring, denial-settlement panel, and Tier 2 report response interface.

The denial-settlement panel shows the pending-denial count and a single settle button. There is no per-denial review — denials are quantitative signals that the leader settles on-chain at a cadence of their choice (§5.8).

The Tier 2 report response interface appears only when a Tier 2 report has been published against a memory the leader committed. It exposes the one-reply rule (acknowledge or dispute) clearly, the remaining time in the response window, and a copy-link to the on-chain transaction once the response is published.

### 7.4 Member Interface

Members see: role, contribution count, serve count, reputation summary, and per-org badge progress/status.

### 7.5 Reporter's Private View

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

### 8.3 Semantic Vector Index (Hub Qdrant)

Vector embeddings are NOT stored on-chain. Stored-memory embeddings are computed at approval/ingest and upserted to the hub's Qdrant index, where similarity search runs over vectors plus keyword metadata. For recall, the MCP/plugin computes the query embedding locally via Ollama and sends the query vector to the hub.

Qdrant stores vector + keyword metadata only (not plaintext memory content and not ciphertext). Embeddings are derived data and remain off-chain.

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

**Developer (user).** Codes with an LLM. Memories accumulate as exhaust. May consume paid recall access through orgs, but chain mechanics stay abstracted behind plugin/hub UX. Experience: "I code, my memories accumulate, my profile shows what I've solved." WeVibe runs invisibly in the background after initial opt-in.

**Org leader (economic operator + curator).** Creates orgs (burns VIBE), curates memories, manages membership, and sets the org's recall-access/payment model (price + policy) via hub accounting. Leader revenue comes from org demand-leg settlement into the org treasury, withdrawn with `MsgWithdrawTreasury`; moderators are paid at leader discretion. **Leaders earn no emissions.**

**Validator.** Stakes VIBE, runs CometBFT consensus, stores all chain state (including encrypted memories), earns validator/staking emissions. Everything deterministic — no subjective judgments. Validators are the storage and availability layer.

**WeVibe-the-protocol.** Open-source software. No company in the middle. WeVibe-the-company may run validators and operate orgs early on, but the protocol does not depend on any single entity.

### 10.3 Token Economics

Single token: **VIBE**. Used for staking, org creation burns, bandwidth allocation, and contributor payouts.

**Dynamic org pricing.** Org creation costs VIBE with on-chain dynamic pricing (creation pressure pushes price up; time decay pulls it down). Burned, not paid to anyone. Prevents spam-org attacks.

**Annual renewal.** Flat rate for continued bandwidth allocation. Non-renewal marks the org dormant — memories persist on-chain but no new submissions are accepted and no new serves are recorded.

**Bandwidth allocation.** VIBE burned/staked by an org determines its per-epoch submission cap and storage budget. This is the anti-DDOS mechanism and the economic unit that makes the network sustainable.

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

> Status: schedule constants and attribution model are locked; implementation lands in CO-041 (the flat `daily_mint` placeholder is replaced by the pool model above).

### 10.4 Validator Economics

Validators earn standard Cosmos SDK staking rewards for running consensus. Additionally, validators store all encrypted memories as part of chain state — this is not separate "operator work," it's inherent to running a node. No separate storage challenges needed.

### 10.5 Demand-Leg and Treasury Economics

Serve/retrieval attribution is **social, not economic** (see §6): serve counts drive public profiles and badges, but do not trigger VIBE payout.

Economic demand is the org access leg:
1. Users buy VIBE and pay orgs for recall access.
2. Access/payment model and pricing are **leader-set** and hub-accounted (`org_credits`; CO-047 skeleton).
3. At settlement, a **small protocol burn** is taken from subscription revenue.
4. The remainder settles to org treasury; leader withdraws via `MsgWithdrawTreasury`.
5. Leader compensates moderators at discretion from treasury revenue.

**Canonical closed loop:** emission -> contributors (contribution-only) + validators/stakers (mint/sell) -> users buy VIBE -> users pay orgs (hub-accounted, leader-set model & price) -> small protocol burn + remainder to org treasury -> leader -> leader pays moderators -> stake/secure -> repeat.

Leaders earn no emissions, there is no per-serve royalty, and there is no protocol-enforced moderator split.

> Status (alpha honesty): this demand leg (VIBE subscriptions -> treasury settlement), the protocol burn settlement path, and final reward-settlement wiring are decided but not yet fully built. CO-047 `org_credits` is currently an accounting skeleton. Testnet faucet flows are testnet-only gas scaffolding, not mainnet economics.

### 10.6 On-Chain Modules

Six custom Cosmos SDK modules:

- `x/org` — registration (dynamic burn pricing), renewal, membership, bandwidth quota accounting, treasury accounting hooks, dormancy detection
- `x/memory` — pending commitment storage (hash + metadata, no blob until approved), approved memory blob storage (encrypted ciphertext as chain state), Merkle root submissions per epoch, pending commitment expiry/removal, quarantine flagging
- `x/serve` — batched serve attestation recording (per-org pseudonymous serve keys), deduplication (memory_cid + serve_key + epoch), self-serve detection/discounting, contributor cross-org serve count aggregation for social attribution (non-economic)
- `x/reputation` — per-contributor cross-org aggregated stats (serve count, org breadth, domain tags, rep score, wallet age). Enhanced mode per-org when attestation enabled (difficulty histogram, XP, provenance breakdown).
- `x/emissions` — validator staking rewards, contributor emission distribution from the network pool, protocol-level emission schedule
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

Curator-defined collections organized by task context. In alpha, skills are lightweight named sets of memories with a description that improve curation and social discoverability inside an org. Ungrouped memories are allowed. Skill assignment is optional.

### 11.2 Cold-Start: Documentation Seeding

New organizations import canonical documentation as seed memories (source: `doc_import`). Session contributions extend and replace seeds over time.

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
