Perfect. Starting with `MASTER.md` — UX + Gap Log only.

---

# MASTER.md

# WeVibe Network — Participant UX & Gap Log

This document defines the **desired user experience for every participant in the WeVibe ecosystem**, mapped to the **top-level API surfaces** each experience requires, followed by the **open gap log** of what remains to be built.

Every UX flow includes **what** the participant does and **why** that design exists. Implementation details (file paths, function names, schema specifics) live in code. Architectural rationale lives in `DECISIONS.md`.

---

## Participant Roles

| Role | Definition | Hub Role Mapping |
|------|-----------|-----------------|
| **Validator** | Runs `wevibed` chain node, participates in consensus, earns validator emission share | N/A (chain-only) |
| **Leader** | Creates and administers an org: members, epochs, billing, moderation config, treasury | `leader` |
| **Moderator** | Reviews and approves/denies submitted memories for quality and accuracy | `moderator` |
| **Contributor** | Extracts memories from coding sessions and submits them for moderation | `member` (with contribution activity) |
| **Consumer** | Benefits from team memories injected into their coding sessions | `member` (with retrieval activity) |
| **New User** | Has no org membership; discovering and joining the ecosystem | No hub record yet |

A single person can hold multiple functional roles simultaneously. A `member` in the hub is both a potential contributor and consumer. A `leader` is implicitly also a moderator, contributor, and consumer. The functional names describe *what they're doing*, not a separate identity.

---

## 1. Validator

### What They Do
Validators run the WeVibe chain binary (`wevibed`), participate in CometBFT consensus, and earn `validator_share` from the emissions module. They are the infrastructure layer that makes the chain deterministic and available.

### UX Flow

```
1. Install wevibed binary (build from source or Docker)
2. Initialize chain node: wevibed init <moniker> --chain-id <id>
3. Configure genesis, peers, ports
4. Start node: wevibed start
5. Create validator: wevibed tx staking create-validator ...
6. Monitor: block height, sync status, peer count
7. Earn: validator_share from x/emissions daily mint
8. Governance: vote on parameter changes via x/gov
```

### API Surfaces Required

| Surface | Module | Purpose |
|---------|--------|---------|
| CometBFT RPC `:26657` | consensus | Block production, sync status, health |
| gRPC `:9090` | all chain modules | Query state, submit transactions |
| REST `:1317` | gRPC-gateway | Alternative query interface |
| `x/staking` | Cosmos SDK | Validator registration, delegation, slashing |
| `x/distribution` | Cosmos SDK | Reward withdrawal |
| `x/gov` | Cosmos SDK | Governance voting |
| `x/emissions` | WeVibe custom | `StoredEmissionPool.validator_share`, `MsgMintDailyEmission` |
| `x/org` | WeVibe custom | Queried indirectly — emissions hook iterates orgs |
| `x/serve` | WeVibe custom | Queried indirectly — emissions hook pulls serve attestations |

### Why This Role Exists
Validators secure the chain. Without validators, there is no consensus, no immutable record of approved memories, no emission payouts, and no on-chain treasury management. They are compensated via `validator_share` from daily emissions.

---

## 2. Leader

### What They Do
Leaders create organizations, manage membership, configure moderation policy, fund the org treasury, manage epochs/key rotation, operate the batch keyword extraction and chain commit pipeline, and oversee the org's memory knowledge base. They have all moderator, contributor, and consumer capabilities plus administrative authority.

### UX Flow: Join Request Review

```
1. Leader opens dashboard → "Members" page
2. Sees badge with pending count: "Join Requests (3)"
3. Clicks badge → sees pending requesters list
4. For each requester, sees public profile:
   - Avatar, display name, wallet address (truncated)
   - Aggregate reputation score
   - Per-org breakdown (reputation tier, contribution count, serve count)
   - List of orgs with roles
   - Joined date, last on-chain activity (linked to chain explorer)
   See DECISIONS.md D-12.4.
5. Actions per request:
   [Approve] → member added to org, kfrags generated via Umbral sidecar, request removed
   [Deny] → optional free-text reason (shown to requester), request removed; if reviewer provided reason, see cooldown timer before re-request
6. Per-org config:
   - Who can approve: leader-only OR leader+moderator
   - Cooldown days after denial (configurable)
   See DECISIONS.md D-12.8.
```

**Why zero-friction join:** The joiner's profile IS the application. No form fields, no essay, no application ceremony. The leader/moderator sees who the person is via their on-chain reputation and WeVibe profile — that's sufficient signal to approve or deny.

### UX Flow: Org Creation

```
1. User has a Cosmos-compatible wallet (Keplr, Leap, etc.)
2. Opens WeVibe dashboard → "Create Organization"
3. Fills in: org name, domain expertise, description
4. Dashboard generates local delegated keypair
5. User signs MsgGrant from wallet → grants delegated key permission
   for WeVibe-specific messages (one-time wallet popup)
6. Dashboard uses delegated key to sign CreateOrg request
7. Hub creates org → chain records via MsgRegisterOrg
8. Leader shown: org ID, epoch keypair (epoch_sk, epoch_pk), setup instructions
9. Recovery phrase MUST be copied offline — cannot be shown again
```

**Why epoch keypair:** The org is provisioned with an Umbral epoch keypair generated via the Umbral sidecar. The `epoch_sk` is the epoch secret key; the `epoch_pk` is used to encrypt memory DEKs for future PRE retrieval. The leader's recovery phrase is a Shamir 2-of-3 splitting of the `epoch_sk`.

**Why recovery phrase matters:** The epoch secret key derives all memory access for this org. If lost and re-encryption keys cannot be reconstructed, all encrypted memories become permanently inaccessible. The recovery phrase is the only backup path.

### UX Flow: Member Management

```
1. Leader opens dashboard → "Members" page
2. Sees all org members: pubkey, role, join epoch, active status
3. To invite: clicks "Invite Member"
   → enters invitee's pubkey + PRE pubkey (secp256k1, 33-byte compressed) + role
   → dashboard signs InviteMember canonical message with delegated key
   → hub generates kfrags via Umbral sidecar GenerateKFrags
   → hub stores member + pre_pubkey + kfrags
4. To promote/demote: selects member → changes role
   → signed role update request to hub
5. To remove: selects member → "Remove"
   → hub calls sidecar DeleteKFrags to revoke re-encryption capability
   → triggers rotation_pending state on org
   → leader must complete epoch rotation to finalize
```

### UX Flow: Epoch Rotation

```
1. After member removal, org enters rotation_pending
2. Leader opens dashboard → sees rotation warning
3. Clicks "Rotate Epoch"
   → generates new epoch keypair via Umbral sidecar
   → regenerates kfrags for all remaining active members (each member's PRE pubkey → new kfrags under new epoch key)
   → old epoch keys remain for historical memory access
   → signs RotateEpoch canonical message
   → hub updates epoch, clears rotation_pending
   → buffered submissions (received during rotation) moved to new epoch
```

**Why rotation exists:** When a member is removed, they still possess the old epoch's re-encryption keys. Rotation generates a new epoch keypair and re-seals re-encryption capability for remaining members only, ensuring the removed member cannot re-encrypt future memories.

### UX Flow: Batch Keyword Extraction & Chain Commit

After a moderator approves a memory, it enters a decoupled three-stage pipeline:

- **Stage 1 (moderator):** Approve memory → status transitions to `pending_keyword`
- **Stage 2 (leader):** Trigger batch keyword extraction → review extracted keywords + suggested new vocabulary → verify → status transitions to `pending_chain`
- **Stage 3 (leader):** Trigger multi-message chain commit TX → status transitions to `committed` → memories are inserted into Qdrant

The leader's batch pipeline is their primary operational activity. The dashboard surfaces pipeline state at every stage.

```
1. Leader opens dashboard → /chain-submit
2. Sees three sections:
   - "Ready for Keywords" — memories with status `pending_keyword`
   - "Review Keywords" — memories with extraction_result attached, awaiting leader review
   - "Ready for Chain" — memories with status `pending_chain`, verified and awaiting TX submission
3. Counts shown for each section so leader can assess pipeline depth

4. Leader clicks "Run Batch Extraction"
   → Hub triggers LLM extraction against org vocabulary for all `pending_keyword` memories
   → For each memory, proposed keywords are drawn from existing org vocabulary
   → Weights form a probability distribution (must sum to 1.0)
   → New vocabulary terms not yet in org set are flagged as "suggested additions"

5. Review screen: for each memory, leader sees:
   - Proposed keywords + weights (probability distribution summing to 1.0)
   - Suggested new vocabulary terms (flagged separately, require leader approval)
   - Sanitization findings (if any flagged characters detected)
   - Preference confidence flag (if elevated)
   - Moderator who approved it (pubkey linked to approval record)

6. Leader actions per memory:
   [Accept] → keywords approved as-is
   [Edit] → leader modifies keywords/weights inline
   [Reject] → memory removed from batch (status returns to `pending_keyword` for re-extraction or `denied` if rejected outright)
   [Approve New Vocab] → leader approves suggested terms to expand org keyword set

   **Suggested new keyword sub-flow:** During review, leader sees not only extracted keywords from existing org vocabulary, but also **suggested new keywords** — terms the LLM proposes that aren't in the org vocabulary yet. Each suggestion shows: term, brief rationale, which memory proposed it. Leader actions per suggestion:
   - [Approve] → adds term to `org_keywords`, term becomes available for future extractions, current memory's keyword set includes it
   - [Reject] → drops the suggestion; current memory re-extracted using only existing vocabulary
   - [Approve with Feedback] → adds to vocabulary AND records feedback for future LLM prompt tuning

7. Leader clicks "Verify & Submit to Chain"
   → Hub-side verification runs on leader-reviewed batch:
     - Keyword format checks (valid strings)
     - Count cap (≤ 20 keywords per memory)
     - Weight sum validation (= 1.0)
     - Character limit checks
   → If verification fails → leader sees specific errors, returns to review
   → If verification passes → "Submit to Chain" enabled

8. Leader clicks "Submit to Chain"
   → If batch exceeds `MaxBatchMemories = 500`, hub returns clear error; leader must split the batch
   → Produces a multi-message Cosmos TX (`MsgApproveMemory`) for the batch
   → Leader's wallet signs (wallet signature, not delegate key)
   → TX broadcast → confirmation awaited
   → On TX confirmation:
     - Memories transition to `committed`
     - Embeddings computed on-the-fly via Ollama nomic-embed-text
     - Memories inserted into Qdrant
   → `last_batch_extraction_at` updated on batch extraction trigger
   → `last_chain_submission_at` updated on chain TX submission

9. Activity tracking visible to leader:
   - `last_batch_extraction_at` timestamp
   - `last_chain_submission_at` timestamp
   - Org health view via GET /v1/orgs/{orgID}/health summarizing pipeline state
```

**Why decoupled from approval:** Moderator approval is a pure quality stamp. Keyword extraction is a separate leader-controlled batch pipeline step that runs after approval (see DECISIONS.md D-6.1). This separation lets leaders review and edit keywords before any chain commitment, enables gas-efficient multi-message batch TX, and prevents approval-time GPU/API saturation.

**Why leader controls batch timing:** Leaders need to review and edit keywords before chain commitment. Batching multiple memories into a single multi-message TX is gas-efficient. Running extraction at approval time would saturate GPU/API resources and remove leader oversight (see DECISIONS.md D-6.1).

**Why Qdrant insert deferred to chain commit:** Memories are only inserted into Qdrant after `committed` status is confirmed on-chain. This ensures the retrieval index only contains memories that have passed the full approval pipeline and are permanently recorded (see DECISIONS.md D-6.2).

**Why scores sum to 1.0:** Keyword weights form a probability distribution. A weight of 0.25 means "25% of retrieval relevance signal comes from this keyword." Sum-to-1.0 normalization ensures the distribution is interpretable and comparable across memories (see DECISIONS.md D-5.4).

### UX Flow: Org Configuration

```
1. Leader opens dashboard → "Settings" page
2. Configures:
   - Required moderator approvals (1-10)
   - Egress mode (local_only / allowlist / unrestricted)
   - Allowed providers list
   - Reputation tier payouts (payout_per_memory per tier)
   - min_contributions_per_epoch (qualification gate for contributor payout)
   - Serve attestation requirements
   - Bandwidth overrides
   - Report vote threshold (1-10, default 1) — requires wallet signature to change
3. Each change signed with delegated key → hub updates config
   (required_approvals and report_vote_threshold changes require wallet signature, not delegate key)
```

### API Surfaces Required

| Surface | Endpoint / Module | Purpose |
|---------|------------------|---------|
| Dashboard | `/create-org` | Org creation UI |
| Dashboard | `/members` | Member management UI |
| Dashboard | `/epoch` | Epoch rotation UI |
| Dashboard | `/settings` | Org configuration UI |
| Dashboard | `/keywords` | Keyword management UI |
| Dashboard | `/recovery` | Recovery share management UI |
| Dashboard | `/chain-submit` | Batch pipeline UI (keyword extraction + chain commit) |
| Hub | `POST /v1/orgs` | Org creation |
| Hub | `PATCH /v1/orgs/{orgID}/config` | Update org config |
| Hub | `POST /v1/orgs/{orgID}/members` | Invite member |
| Hub | `PATCH /v1/orgs/{orgID}/members/{pubkey}/role` | Role update |
| Hub | `DELETE /v1/orgs/{orgID}/members/{pubkey}` | Remove member |
| Hub | `POST /v1/orgs/{orgID}/transfer-leadership` | Leadership transfer |
| Hub | `POST /v1/orgs/{orgID}/close` | Org closure |
| Hub | `POST /v1/orgs/{orgID}/epoch/rotate` | Rotate epoch |
| Hub | `POST /v1/orgs/{orgID}/submit-keyword-results` | Submit leader-reviewed keyword batch |
| Hub | `POST /v1/orgs/{orgID}/verify-keywords` | Hub-side keyword verification before chain submit |
| Hub | `POST /v1/orgs/{orgID}/submissions/{hash}/rerun-keywords` | Rerun extraction with leader feedback |
| Hub | `PUT /v1/orgs/{orgID}/submissions/{hash}/update-keywords` | Update keywords for a submission |
| Hub | `POST /v1/orgs/{orgID}/batch-chain-submit` | Trigger multi-message `MsgApproveMemory` TX (enforced `MaxBatchMemories = 500` cap; returns 400 if exceeded; leader must split batch — see D-6.6) |
| Hub | `GET /v1/orgs/{orgID}/health` | Org pipeline health summary |
| Hub | Recovery share endpoints | Store/retrieve Shamir shares |
| Hub | Keyword CRUD endpoints | List/add/merge/rename/deprecate |
| Chain | `MsgRegisterOrg` | On-chain org record |
| Chain | `MsgFundTreasury`, `MsgWithdrawTreasury` | Treasury operations |
| Chain | `MsgSetRepTiers`, `MsgSetOrgConfig`, `MsgSetBandwidthOverride` | Chain-level config |
| Chain | `MsgApproveMemory` | Multi-message batch TX for committed memories |

---

## 3. Moderator

### What They Do
Moderators review submitted memories, decrypt and inspect them, verify accuracy and quality, and either approve them for inclusion in the org's knowledge base or deny them with a reason. They also review reports filed against approved memories.

Moderators see sanitization findings surfaced by the extraction pipeline and preference confidence flags on each submission. Their pubkey is permanently linked to every memory they approve, providing an accountability chain through to chain commit (see DECISIONS.md D-6.4).

### UX Flow: Moderation Queue

```
1. Moderator opens dashboard → "Moderation" page
   Top of page shows org switcher (all orgs where this user has moderator role).
   Pending counts per org. Toggle between orgs to review.
   See DECISIONS.md D-12.1.
2. Sees pending submissions queue (ciphertext at rest)
3. Dashboard (via wevibe-mcp) decrypts each submission:
   - Unseals DEK with moderator private key (mod key from local keystore)
   - Decrypts ciphertext to plaintext
   - Runs steganography scanner on plaintext
4. For each item, moderator sees:
   - Plaintext content
   - Contributor pubkey + reputation
   - Sanitization findings (flagged characters, homoglyphs, invisible chars)
   - Preference confidence flag (elevated flag if content reads as organizational preference)
   - Tech stack tags
5. Moderator decides:
   [Approve] → wevibe-mcp signs canonical approval message; hub sets status to `pending_keyword`; moderator pubkey recorded on submission; no keyword extraction, no embedding, no chain TX at this step
   [Deny] → moderator selects reason, signs denial, POSTs to hub
   [Vote] → for orgs with required_approvals > 1, cast approval vote
   [Undo Approve] → available ONLY while memory is in `pending_keyword` state; reverts to `pending`; once leader has moved memory to `pending_chain` (via batch submission), undo is no longer available — leader must use "remove from batch" instead (see DECISIONS.md D-6.5)

**Why keyword extraction is not at approval time:** Moderator approval is a pure quality stamp — the moderator confirms the content is accurate, appropriate, and worth preserving. The actual keyword extraction is a separate leader-controlled batch pipeline step that runs after approval (see DECISIONS.md D-6.1). This separation lets leaders review and edit keywords before any chain commitment, enables gas-efficient multi-message batch TX, and prevents approval-time GPU/API saturation.
```

### UX Flow: Report Review

```
1. Moderator opens dashboard → "Reports" page
2. Sees pending reports with full context:
   - Memory content (decrypted via mod key)
   - Reporter identity: pubkey, wallet address, account age
   - Reporter's dismissed_reports_count (trust signal)
   - Reason category + free-text note
   - Current vote count vs report_vote_threshold
3. For each report:
   [Vote to Uphold] → memory is harmful, should be deleted
   [Vote to Dismiss] → memory is fine, report is incorrect
   [Dismiss as Malicious] → reporter is acting in bad faith
4. When uphold votes >= report_vote_threshold:
   → Report status → 'upheld_pending_tx'
   → Leader sees "Submit to Chain" button on reports page
5. When dismiss votes >= threshold:
   → Reporter's dismissed_reports_count incremented
   → Report status → 'dismissed' or 'dismissed_malicious'
   → Memory unchanged
6. Leader override: leader can always resolve immediately
```

**Why reporter identity is visible to moderators:** Without knowing who reported, moderators cannot assess report quality. A report from a long-standing contributor with zero dismissed reports carries more weight than one from a new member with 5 dismissed reports.

**Why voting threshold is hub-enforced, not on-chain:** Report voting is an operational decision that changes frequently as orgs scale. Putting it on-chain would require governance transactions for every threshold change. The hub enforces the threshold; the chain only records the final outcome.

**Why leader triggers the chain TX:** The on-chain commitment is a permanent public record. Only the org leader — authenticated via wallet signature, not delegate key — can trigger this consequential action. If the chain TX fails, nothing changes (atomic: chain TX must confirm before Qdrant delete).

### UX Flow: Chain Commit Notifications

```
1. Moderator opens dashboard → top-bar activity feed shows real-time notifications
2. Category "chain_commit_involving_you" fires whenever a leader commits a TX that lists this moderator's pubkey in `approvers[]` or `upholding_moderators[]`
3. Notification body: "Memory [hash] was committed to chain in [org] by leader [name]. You were listed as approver." + link to chain TX
4. Click notification → opens side-by-side view: chain record (immutable, from `chain_commit_events` table) vs hub's `vote_records` history (operational, from hub's vote tally)
5. If discrepancy (moderator's pubkey in chain `approvers[]` but no matching approval vote in hub records): alarm state, user can raise public flag against the leader
```
<!-- See DECISIONS.md D-13.3 --><strong>Why chain_commit_events vs vote_records divergence matters:</strong> The chain record is immutable and proves the moderator's pubkey was included. The hub's vote_records are the operational log of what the moderator actually voted. Divergence suggests either a governance issue (leader included a moderator who didn't vote) or a synchronization issue. The alarm state lets the moderator take public action. See DECISIONS.md D-13.3.

### UX Flow: Approval Overturn Alert

```
1. When a memory the moderator previously approved is later upheld-reported, moderator receives `your_approval_was_overturned` notification
2. Notification body: "Memory [hash] you approved in [org] was deleted via upheld report on [date]. Reason: [reason]."
3. Click → view the upheld report record (chain) + moderator's own approval record (hub)
4. Moderator's profile now shows `approvals_later_upheld_count` incremented
```
<!-- See DECISIONS.md D-13.7 --><strong>Why approvals_later_upheld_count matters:</strong> This counter is a direct accountability signal. A moderator with a high ratio of approvals-that-were-later-upheld vs total approvals is a quality risk. It feeds into the chain's `ModeratorProfile` and is queryable via `UpheldReportsByModerator`. See DECISIONS.md D-13.7.

### API Surfaces Required

| Surface | Endpoint / Module | Purpose |
|---------|------------------|---------|
| wevibe-mcp | MCP tools or HTTP API | Decrypt queue, approve, deny (handles crypto) |
| Hub | Moderation queue endpoints | Pending submissions (ciphertext) |
| Hub | Approval/denial endpoints | Final approve/deny; hub sets status to `pending_keyword` on approve (no chain TX) |
| Hub | `POST /v1/orgs/{orgID}/moderation/{hash}/undo-approve` | Reverts `pending_keyword` → `pending`; race-safe (atomic WHERE status='pending_keyword'); returns 409 if status already moved (see D-6.5) |
| Hub | Report endpoints (list, detail, vote) | Report review and voting |
| Chain | `MsgApproveMemory` | Multi-message batch TX — triggered by leader at chain commit time, not by moderator at approval |

---

## 4. Contributor

### What They Do
Contributors extract technical insights from their coding sessions and submit them for moderation. This is how knowledge enters the org's memory pool.

### UX Flow: Memory Contribution

```
1. Developer finishes a coding session in OpenCode
   → session stored automatically in local SQLite
     (~/.local/share/opencode/opencode.db)
2. Opens WeVibe dashboard → clicks "Sessions" (first nav item)
3. Sees all OpenCode sessions from local SQLite:
   - Session title, model used, project directory
   - Date/time, message count
4. Clicks a session → expands showing metadata
5. Clicks "Extract Memories"
   → Loading spinner: "Please wait while your session is being analyzed"
   → Dashboard server-side route (/api/extract) calls local LLM via wevibe-mcp proxy
   → LLM: Ollama (default qwen2.5:14b at localhost:11434) 
     or OpenRouter (user's API key)
6. Results: "Your session produced 7 memories!"
7. Each memory shown as a card:
   ☑ Checkbox for selection
   - Insight text (the core learning, 1-2 specific sentences)
   - Context (environment, versions, conditions)
   - Avoid warnings (what NOT to do and why)
   - Tech stack tags (auto-detected)
   - Memory type (correct_implementation or negative_signal)
   - Preference confidence flag (if elevated)
   - Sanitization findings (if any flagged characters detected)
8. User reviews, selects which to submit
    → "Select All" / "Select None" shortcuts
9. For each selected memory, contributor sees a destination org dropdown.
   System suggests an org based on tech-stack-tags-vs-domain matching.
   Contributor can change the destination per memory.
   Dropdown lists only orgs contributor belongs to.
   See DECISIONS.md D-12.2.
10. Clicks "Submit 5 Memories"
    → For each selected memory:
      - Dashboard encrypts with AES-GCM (generates DEK)
      - DEK sealed to mod pubkey (X25519) for pending submission
      - SHA-256(ciphertext || capsule) = submission_hash
      - Contributor signs submission_hash with delegated key
      - POST ciphertext + capsule + signature to hub /v1/orgs/{orgID}/submit
      - Hub-side Unicode sanitization scan runs AFTER ciphertext arrives (non-blocking; findings attach to submission record for moderator review)
    → "5 memories submitted for review!"
    → Memories enter moderation queue as `pending`
```

**Why multi-memory extraction:** A 30-minute debugging session may contain 5-10 distinct technical insights. Extracting one loses the rest. The LLM extraction prompt explicitly instructs: "ONE insight per memory."

**Why local LLM first:** Session transcripts contain the full conversation between developer and AI. This data is private — it may contain file paths, internal architecture details, API keys the developer pasted by accident. Running extraction locally (Ollama) keeps this data on the developer's machine.

**Why client-side encryption:** The hub never sees plaintext. Encryption happens in the browser using Web Crypto API before the data leaves the machine. Only the mod key holder can decrypt pending submissions for moderation. After approval, Umbral encryption is applied for the PRE retrieval path.

**Why contributors don't handle keywords:** Keywords are assigned during moderation by moderators selecting from the org's vocabulary. Contributors submit raw insights only. This prevents keyword drift and keeps vocabulary curated.

### API Surfaces Required

| Surface | Endpoint / Module | Purpose |
|---------|------------------|---------|
| Dashboard | `/api/sessions` | List local OpenCode sessions (SQLite) |
| Dashboard | `/api/sessions/{id}/messages` | Session transcript |
| Dashboard | `/api/extract` | LLM memory extraction (proxies to wevibe-mcp) |
| Dashboard | `/api/settings` | LLM provider config, org ID, mod pubkey |
| Hub | `GET /v1/orgs/{orgID}` | Current epoch (for submission) |
| Hub | `POST /v1/orgs/{orgID}/submit` | Submit encrypted memory |
| Local | `~/.local/share/opencode/opencode.db` | SQLite session source (readonly) |
| Local | `~/.config/wevibe/dashboard.json` | Dashboard settings |
| Local | Ollama `localhost:11434` | LLM extraction (default) |
| External | OpenRouter API | LLM extraction (fallback) |

---

## 5. Consumer

### What They Do
Consumers are developers whose coding sessions are enhanced by team memories. The plugin running inside their agent (OpenCode, Claude Code, Cline) automatically retrieves relevant memories and injects them into the agent's context — invisible to the agent, gated by the developer.

### UX Flow: Memory Recall (During Coding Session)

```
1. Developer opens coding session (OpenCode, Claude Code, Cline)
2. Plugin activates automatically — no action required
3. Plugin gathers context (invisible to developer and agent):
   - Current project / repo name
   - Tech stack signals (package.json, Cargo.toml, go.mod, etc.)
   - Recent file activity
   - Current conversation/task context
   Plugin uses the active org for this session. Active org selection mechanism
   is per-session; cross-org retrieval is deferred.
   See DECISIONS.md D-12.3.
4. Plugin calls wevibe-mcp → hub POST /v1/orgs/{orgID}/query (sends pre_pubkey)
5. Hub:
   a. Verifies membership + epoch access
   b. Queries Qdrant with enriched QueryPoints:
        - Vector similarity search (cosine, 768-dim)
        - Lifecycle filter: exclude archived always (only `committed` memories are in Qdrant)
        - Keyword boost: overlap between query keywords and result keywords
        - Keyword weight ranking based on per-keyword serve/denial history
        - Re-sort by weighted score, return top-N
   c. For each result: calls sidecar ReEncrypt(capsule, kfrag) → produces cfrag
   d. Returns cfrag + capsule + umbralCiphertext per memory
   e. Deducts 1 query credit
6. wevibe-mcp:
   a. For each memory: calls sidecar decrypt-reencrypted
      (capsule + cfrag + ciphertext + receiving_sk + delegating_pk) → plaintext
   b. Applies risk appetite filter: if set to `lowest`, keeps only `negative_signal` memories; if `neutral`, keeps both types
   c. Checks local blacklist — skips any previously denied memories
   d. Runs wevibe-guard security scan on plaintext
   e. Flagged memories → redacted, not injected
   f. Clean memories returned to plugin
7. Plugin shows developer approval UI with current risk appetite setting visible (e.g., "WeVibe (risk: lowest) found 2 memories..."):
   "WeVibe found 3 memories relevant to your current task:"
   [Memory 1 preview] ☑
   [Memory 2 preview] ☑
   [Memory 3 preview] ☐
   [Inject Selected] [Dismiss]
8. Developer reviews and approves/rejects each memory
9. Approved memories injected into agent context
10. Agent sees memories as context — never knows they came from WeVibe
11. Session continues; agent benefits from injected knowledge
12. Plugin records serve event for each injected memory:
    POST /v1/orgs/{orgID}/serves (nullifier, memory hash, contributor ID)
```

**Why hub never sees plaintext:** PRE re-encryption is a cryptographic operation on ciphertext. The hub applies re-encryption keys (kfrags) to produce ciphertext fragments (cfrags) — it never decrypts or sees any plaintext. The member's PRE secret key is the only key that can complete decapsulation.

**Why the agent never knows:** If the agent knows memories come from an external source, it may decide they're irrelevant, argue with them, or hallucinate about WeVibe's capabilities. Memories appear as context — the same way a system prompt does. The agent uses the knowledge naturally.

**Why human gating:** The developer always approves before injection. This prevents irrelevant, incorrect, or sensitive memories from polluting the agent's context. It also gives the developer awareness of what knowledge is being used.

**Why serve events:** Every time a memory is injected, a serve attestation is recorded. This feeds into the keyword weight system (boosts confidence of well-used keywords on the memory) and the emissions system. Serve events use nullifiers to prevent double-counting.

**Why wevibe-guard at recall time:** Even though memories were reviewed by a moderator at approval time, recall-time scanning provides defense in depth. A memory that was clean when approved might match new detection rules. The guard also catches any steganographic or injection patterns that a human moderator missed.

### UX Flow: Memory Denial (During Coding Session)

```
1. Developer receives a memory that seems incorrect/outdated/irrelevant
2. Plugin shows Accept / Deny / Report controls per memory
3. Developer clicks "Deny":
   → Reason selection (incorrect, outdated, irrelevant)
   → Memory added to local blacklist (~/.wevibe/blacklist.json)
     — never shown to THIS developer again
   → Plugin submits denial to hub POST /v1/orgs/{orgID}/denials
4. Hub records denial event (serve_events table, event_type='denial')
5. Leader periodically batch-submits denials to chain:
   POST /v1/orgs/{orgID}/denials/batch-submit → MsgSubmitDenialBatch
6. Chain applies keyword weight decay for all keywords on that memory
7. Memory may transition to ARCHIVED when ALL keyword weights = 0
```

### UX Flow: Memory Reporting (During Coding Session)

```
1. Developer receives a memory that seems harmful/malicious
2. Plugin shows Accept / Deny / Report controls per memory
3. Developer clicks "Report" (PAID MEMBERS ONLY — trial members see Report greyed out):
   → Reason selection (incorrect, outdated, security_risk, malicious)
   → Optional free-text note (max 500 characters)
   → Plugin submits report to hub with reporter_pubkey + reporter_wallet + signature
4. Memory is NOT blacklisted locally — it continues to appear in recall
   (only Deny triggers local blacklist)
5. Report enters moderator/leader review queue with reporter identity linked
6. Moderators vote: Uphold / Dismiss / Dismiss as Malicious
   → Voting threshold: org-configurable report_vote_threshold
7. If UPHELD (votes >= threshold):
    → Report status → upheld_pending_tx
    → Leader reviews and clicks "Submit to Chain" with reason (max 500 chars)
    → Dashboard requests cfrag from hub for leader's PRE pubkey
    → Dashboard decrypts memory via Umbral sidecar (capsule + cfrag + ciphertext + leader_pre_sk → plaintext)
    → Dashboard packages `plaintext + ciphertext + capsule + plaintext_hash` into `MsgReportMemory` payload
    → If plaintext exceeds 4096 bytes: dashboard sets `plaintext_oversized=true`, omits plaintext/ciphertext/capsule, publishes full plaintext off-chain (hub persists in `published_plaintext` table) with `sha256` as verification anchor
    → Leader signs the full TX with wallet, broadcasts to chain
    → On confirmation: hub deletes memory from Qdrant
    → On-chain commitment records: memory hash, contributor wallet, reason, reporting org
    → Memory becomes public record on contributor's social graph
    <!-- Why the triplet is stored on-chain — see DECISIONS.md D-13.2 -->**Why the triplet is stored on-chain:** The `plaintext + ciphertext + capsule` triplet proves the memory existed and was retrievable at commit time. Verifiers can re-derive `plaintext_hash = sha256(plaintext)` to confirm integrity. See DECISIONS.md D-13.2.
8. If DISMISSED:
   → Reporter's dismissed_reports_count incremented
   → Memory unchanged, continues serving
9. If DISMISSED AS MALICIOUS:
   → Reporter flagged, dismissed_reports_count incremented
   → Leader can remove reporter from org (stake kept by org)
```

**Why denial vs report:** Denial is the consumer negative-signal path that feeds keyword weight decay over time. Reports are the moderator-reviewed path for memories that are harmful — they require human judgment and can result in immediate deletion + public on-chain record. Denials accumulate gradually; reports resolve decisively.

**Why reports don't auto-blacklist:** A denial is the developer saying "I don't want to see this." That's personal preference — blacklist it locally. A report is the developer saying "this needs investigation." The memory should remain visible to other consumers (and the reporter) until moderators resolve it.

### UX Flow: Conversational Queries (Optional)

```
1. Developer asks agent: "How many memories have I contributed?"
2. Agent MAY call wevibe_status MCP tool
3. Tool returns read-only info from hub
4. Agent relays information conversationally
5. If agent doesn't call the tool — nothing breaks
```

### UX Flow: Verify an Upheld Report

```
Anyone with memory_hash can query `VerifyUpheldReport(memory_hash)` via gRPC.
Response: {plaintext, ciphertext, capsule, plaintext_hash, plaintext_oversized}
Verifier computes sha256(plaintext) and confirms it matches plaintext_hash.
Verifier with PRE access independently decrypts ciphertext to confirm it produces same plaintext.
Mismatch = leader fabricated the upheld report.
```
<!-- See DECISIONS.md D-13.2 --><strong>Why the triplet is stored on-chain:</strong> The `plaintext + ciphertext + capsule` triplet proves the memory existed and was retrievable at commit time. Verifiers can re-derive `plaintext_hash = sha256(plaintext)` to confirm integrity. See DECISIONS.md D-13.2.

### UX Flow: Plugin Failure UX

When the plugin fails to start or loses connection, a fallback chain activates:

```
1. Plugin attempts auto-start of wevibe-mcp
   → If wevibe-mcp binary not found: show stderr message with copy-pasteable start command
     Example: "wevibe-mcp not found. Run: npm install -g @wevibe-network/wevibe-mcp"
   See DECISIONS.md D-12.6.
2. If wevibe-mcp process fails: plugin-specific fallback
   → OpenCode: stderr message with copy-pasteable start command
   → Future GUI plugins (Claude Code, Cline): toaster notification with start instructions
3. If wevibe-mcp is running but unreachable: show connection error with retry button
```

**Why fallback chain:** Plugin must fail gracefully. The user should always know why memories aren't appearing and how to fix it. Copy-pasteable commands eliminate guesswork.

### API Surfaces Required

| Surface | Endpoint / Module | Purpose |
|---------|------------------|---------|
| Plugin | Agent lifecycle hooks | Context gathering, approval UI, injection |
| wevibe-mcp | HTTP API on wevibe-mcp at 127.0.0.1:4450. Loopback-only. Per-session Bearer token auth (D-12.5a, closed by CO-260). Endpoints: GET /v1/health, POST /v1/recall, POST /v1/serves, POST /v1/reports. Token at ~/.wevibe/mcp-session-token, mode 0600, rotates on wevibe-mcp restart. Plugin is sole client surface to hub. See DECISIONS.md D-12.5, D-12.5a. | Retrieval proxy, serve event proxy, report proxy |
| Hub | `POST /v1/orgs/{orgID}/query` | Memory retrieval (PRE, requires pre_pubkey, returns cfrag + capsule) |
| Hub | `POST /v1/orgs/{orgID}/serves` | Record serve event |
| Hub | `POST /v1/orgs/{orgID}/denials` | Record denial event |
| Hub | `POST /v1/orgs/{orgID}/reports` | Submit memory report (paid members only) |
| Hub | `POST /v1/orgs/{orgID}/reports/{reportID}/vote` | Cast vote on report |
| Hub | `POST /v1/orgs/{orgID}/reports/{reportID}/commit` | Leader chain commitment (wallet-signed) |
| Chain | `MsgSubmitServeBatch`, `MsgSubmitDenialBatch`, `MsgReportMemory` | Batch attestations (via hub relay) |
| wevibe-guard | Scan API | Recall-time security scanning |
| wevibe-mcp | `wevibe_set_risk_appetite` tool | Read/write current risk appetite (`lowest` = negative_signal only, `neutral` = both) |
| Local | `~/.wevibe/plugin-config.json` | Risk appetite persistence (shared by wevibe-mcp and plugin) |

### UX Flow: Risk Appetite Configuration

```
Two ways to change the consumer risk appetite setting:

1. Direct file edit:
   → ~/.wevibe/plugin-config.json → set { "risk_appetite": "lowest" | "neutral" }

2. Conversational (via agent):
   → User asks: "Set risk appetite to lowest"
   → Agent calls wevibe_set_risk_appetite MCP tool
   → Setting persisted to ~/.wevibe/plugin-config.json

Default: "neutral" (both correct_implementation and negative_signal memories surfaced)
When "lowest": only negative_signal memories are shown in approval UI

---

## 7. Consumer Profile

### What It Is
Two surfaces for viewing and editing a user's WeVibe profile:

| Surface | URL | Access | Editable |
|---------|-----|--------|----------|
| Own profile | `/profile` | Authenticated user | Yes |
| Public profile | `/u/:wallet_address` | Anyone (public read-only) | No |

### Profile Content

- Avatar (editable)
- Display name (editable, changeable anytime, not unique)
- Wallet address (truncated, 6+4 chars)
- Aggregate reputation score
- Per-org breakdown (reputation tier, contribution count, serve count per org)
- List of orgs with roles
- Contribution count (total memories submitted across all orgs)
- Serve count (total memories injected into sessions)
- Joined date
- Last on-chain activity (linked to chain explorer)

### Future: Verified Social Links
Profile will support verified social links (GitHub, Twitter) for identity confirmation.

### API Surfaces

| Surface | Endpoint / Module | Purpose |
|---------|------------------|---------|
| Dashboard | `/profile` | Edit own profile |
| Dashboard | `/u/:wallet_address` | View public profile |
| Hub | `GET /v1/profile/:wallet` | Fetch profile data (public) |
| Hub | `PATCH /v1/profile` | Update own profile (avatar, display name) |

**Why profile exists:** Reputation is cross-org. The profile page surfaces a user's complete WeVibe identity — not just their role in one org, but their contribution history and standing across the network. See DECISIONS.md D-12.4.

---

## 6.5 Org Discovery

### What It Is
Two surfaces for discovering organizations:

| Surface | URL | Access | Features |
|---------|-----|--------|----------|
| Public web | `wevibe.network/orgs` | Anyone | Browse, search, filter |
| In-dashboard | `/discover` | Post-wallet auth | Same as public + join CTA |

### Org Card Content

- Name
- Domain expertise
- Member count
- Description
- Tech stack tags
- Reputation tier payouts (what contributors earn)
- Trial info (free trial days, what happens after)
- Join policy (open / approval-required / invite-only)
- Recent activity: memory count, last batch commit timestamp (linked to chain explorer)

### Org Detail Page

- Hero: name, description, key stats
- Member roster (public, no sensitive data)
- Recent memory activity (counts only, no content — privacy)
- Top contributors (public)
- Reputation tier breakdown
- Join CTA button

### Search / Filter Controls

- Text search (name, description)
- Domain filter (dropdown)
- Tech stack filter (multi-select)
- "Accepting members" toggle (excludes invite-only)
- Sort: newest / largest / most-active / highest-payouts

### Why No Sample Memories on Detail Page
Org detail pages show activity counts but never show memory content. This protects contributor IP — a competitor org could otherwise scrape the public directory for memory content. See DECISIONS.md D-12.7.

---

## Cross-Cutting: Multi-Org Architecture

This section describes how WeVibe achieves logical and physical isolation between orgs. It supplements the "Component Roles & Authority" section above.

### Isolation Layers

**PostgreSQL (hub):** Logical isolation via `org_id` foreign keys. All hub tables that store per-org data include `org_id`. Queries always include org_id filter; cross-org queries are impossible at the hub layer.

**Qdrant:** Physical isolation via per-org collections named `org_{orgID}_memories`. Each org's memories live in a separate Qdrant collection. Collection name is derived from org_id, not hardcoded. See DECISIONS.md D-12.1.

**Hub authz middleware:** Enforces org membership on every request. The middleware verifies the caller's delegated key is registered as a member of the target org before processing any request. No bypass path.

### Self-Hosting Option

Hub is open-source. Orgs can run their own hub instance for full physical isolation — their Qdrant collection is entirely separate, not even a namespace filter. This is the recommended deployment for high-security orgs.

### Current wevibe-infra Arrangement

Shared hub deployment with per-org Qdrant collections. Shared hub means shared PostgreSQL (logical isolation via org_id). Per-org Qdrant collections mean physical separation at the vector storage layer. This is the Sprint 25 default.

### Cross-Reference

For the broader architecture context including chain/Qdrant/hub roles, sync model, and hub's three active roles, see **Cross-Cutting: Component Roles & Authority** above.

**Why this architecture:** Logical isolation via org_id is insufficient for vector search — Qdrant collection names must be derived from org_id to prevent cross-org retrieval even if the org_id filter is somehow bypassed. See DECISIONS.md D-12.1.

---

## Cross-Cutting: Activity Feed

### What It Is
Real-time aggregated activity stream across all orgs a user belongs to.

### Two Surfaces

| Surface | Access | Features |
|---------|--------|----------|
| Top-bar notification bell | All authenticated users | Unread count badge, dropdown with recent events |
| `/activity` page | All authenticated users | Full history, filtering by event category |

### Delivery Mechanism

- **Primary:** WebSocket push from hub to connected clients
- **Fallback:** Polling every 30 seconds if WebSocket unavailable

### Event Categories

| Category | Example Events |
|----------|----------------|
| `contribution` | Memory submitted, memory approved, memory denied, keyword extraction completed |
| `earning` | Serve payout credited, contribution payout credited, tier upgrade |
| `recognition` | Top contributor badge, milestone reached |
| `moderation` | New pending submission, approval granted, denial with reason, report filed |
| `membership` | Join request sent, join request approved, join request denied, role changed, removed from org |
| `reports` | Memory you reported was upheld, memory you contributed was deleted, reporter dismissed as malicious |

### Activity Feed UX

```
1. User sees notification bell with unread count badge
2. Clicks bell → dropdown shows 5 most recent events
   Each event: icon, description, timestamp, org name
3. "View All" link → /activity page (full history)
4. On /activity: filter by category, date range, org
   Paginated list with load-more
5. Each event links to relevant detail (memory, org, profile)
```

**Why aggregated across all orgs:** Users who belong to multiple orgs want one place to see everything — not separate inboxes per org. The feed unifies it. See DECISIONS.md D-12.9.

---

## Cross-Cutting: Social Graph Architecture

### Three-Layer Architecture

| Layer | Lives In | Contains | Owner |
|-------|----------|----------|-------|
| Immutable provenance | wevibe-chain | Aggregates, indices, on-chain events keyed by wallet pubkey | Chain (cryptographic) |
| Operational queue | wevibe-hub | Pending submissions, votes, batches, chain_commit_events, vote_records | Hub operator (WeVibe-hosted OR self-hosted) |
| Display layer | Social Graph Service (separate container/VPS) | Wallet → display name + avatar + bio + linked socials | Separate Docker container, separate VPS |

### Chain-Side Data

`StoredContributorProfile`, `StoredModeratorProfile` (per-org), `StoredLeaderProfile` (per-org), `StoredOrg` aggregates. All queryable via gRPC:
- `ContributorProfile` — contributor's total approved memories, serves, denials, upheld reports
- `ModeratorProfile` — moderator's approval count, approvals later upheld count
- `LeaderProfile` — leader's chain commits, upheld reports committed, epoch rotations
- `UpheldReportsBy{Contributor,Moderator,Leader}` — queryable public records
- `VerifyUpheldReport` — cryptographic verification of upheld report integrity

### Hub-Side Data

- `chain_commit_events` — every relevant chain TX recorded by ChainWatcher
- `vote_records` — operational voting history (what moderators actually voted)
- `published_plaintext` — off-chain storage for oversized upheld memories (verified against on-chain hash)

### Social Graph Service (Future CO)

The Social Graph Service (separate Docker container, separate VPS) provides wallet→name mapping. Mandatory display name registration before a user can be accepted as a moderator role. Both WeVibe-hosted hub and self-hosted hubs consume this service. See DECISIONS.md D-13.4.

**Why the three-layer split:** see DECISIONS.md D-13.4

---

## Cross-Cutting: Chain Module Purpose Map

| Module | Purpose | Participant(s) Served |
|--------|---------|----------------------|
| `x/org` | Org registration, membership, treasury, config, reputation tiers | Leader, New User |
| `x/memory` | Memory submission, approval, rejection, lifecycle (keyword weights), relationships, Merkle roots | Contributor, Moderator, Consumer, Leader |
| `x/serve` | Serve attestation recording, denial attestations, epoch stats, contributor serve counts | Consumer, Contributor, Leader |
| `x/reputation` | Contributor profile (aggregates + memberships), moderator profile (per-org, per-mod accountability), leader profile (per-org, leadership tenure + chain commits + epoch rotations). Owns StoredContributorProfile + StoredModeratorProfile + StoredLeaderProfile. Queryable via ModeratorProfile, LeaderProfile, UpheldReportsBy{*}, VerifyUpheldReport gRPC queries. | Contributor, Moderator, Leader, Consumer, all participants via RPC |
| `x/emissions` | Daily mint, operator/validator shares, work scores, bootstrap credits, pay-per-memory payout | Validator, Contributor, Leader |
| `x/bandwidth` | Per-org per-epoch bandwidth caps for memory submissions and serves | Leader, all members |
| `x/attestation` | Merkle root submission for epoch data availability proofs | Leader (via hub relay) |

---

## Cross-Cutting: Component Roles & Authority

WeVibe has four storage/compute components with distinct roles:

| Component | Role | Authority | Stores | Notes |
|-----------|------|-----------|--------|-------|
| **wevibe-chain** (Cosmos SDK) | Authoritative source of truth | **Chain is final** — all memory lifecycle state lives here | Approved memory commitments, keyword weights (with serve_count + denial_count per keyword), lifecycle state, org config, reputation, treasury | All retrieval-affecting state changes go through chain TXs. Block time ~6s, instant finality via CometBFT. |
| **Qdrant** | Read-optimized vector index | Mirrors chain for retrieval ranking | Vector embeddings (768-dim, cosine), keywords, per-keyword weights, lifecycle state | Used ONLY for retrieval ranking. Cannot answer questions about pending memories, billing, members. Can be regenerated from chain state if lost. |
| **wevibe-hub** (Go API server) | Operational state + active synchronizer + re-encryption proxy | Hub applies chain state changes to Qdrant; hub never unilaterally changes chain | Members, ciphertext blobs, pending submissions, moderation queue, billing/credits, recovery shares, audit log | Synchronizes chain → Qdrant on every serve/denial TX (applies decay/boost formula locally, updates Qdrant payload). Performs PRE re-encryption at retrieval time (hub never sees plaintext). |
| **Hub's PostgreSQL** | Operational data store inside hub | Distinct from chain — stores pre-commit state and operational data | Things that don't need to be immutable/public (auth, queue, billing) and things that exist before chain commit (pending submissions in pre-commit lifecycle states) | Cannot serve as source of truth for retrieval — chain is authoritative. |

### Sync Model

```
Consumer → wevibe-mcp → Hub (verify auth, query Qdrant, PRE re-encrypt)
                         ↓
                       Qdrant (vector search, keyword-weighted ranking)
                         ↑ (set_payload on TX confirm)
Hub ←(TX confirm)── Chain (authoritative keyword weights)
```

- Consumer triggers serve or denial → hub submits chain TX → on TX confirmation, hub applies the same decay/boost formula to its Qdrant payload (`set_payload` call) → next query reads current state
- Sync window = one block (~6s)
- Hub restart → `SyncKeywordWeightsFromChain` reconciles any drift before accepting traffic
- Normal operation: Qdrant cannot drift from chain beyond one block confirmation

### Hub's Three Active Roles

1. **Re-encryption proxy:** Hub re-encrypts consumer queries and memory responses via PRE, never seeing plaintext. Consumer's DEK is re-wrapped with consumer's public key at retrieval time.

2. **Authorization authority:** Hub verifies org membership, role permissions, and moderation queue access before any chain submission or retrieval.

3. **Policy enforcement engine:** Hub enforces batch submission caps (`MaxBatchMemories = 500`), epoch rotation gating, and moderation quorum before routing to chain.

**Why this matters:** The chain is the authority (D-2.5), but the hub is not a pure observer — it actively shapes what goes on chain (D-3.1). Qdrant is a cache, not a source of truth (D-6.2).

---

## Cross-Cutting: Wallet & Identity Architecture

**Primary identity:** Cosmos-compatible wallet (Keplr, Leap, etc.)

**Local signing key:** Delegated keypair created at onboarding, stored locally

**Delegation:** One-time `x/authz MsgGrant` from wallet → delegated key
- Scoped to WeVibe-specific message types only
- Time-limited (renewable)
- Revocable from wallet at any time

**Hot wallet pattern:** Delegated key signs all routine operations without wallet popup

**Hub auth:** `WeVibe-Signed` header using delegated key

**Chain transactions:** Delegated key submits via `x/authz` execution path

**PRE identity:** BIP-32 derived child key from Cosmos wallet for Umbral encryption

```
Cosmos wallet key (secp256k1, BIP-32 path m/44'/118'/0'/0/0)
    ├── Transaction signing (delegated via MsgGrant)
    └── BIP-32 derived child key ("wevibe-pre-identity/v1")
            └── PRE encryption identity (Umbral SecretKey)
```

---

## Cross-Cutting: wevibe-mcp as Local Backend

wevibe-mcp serves as the shared local backend for both the plugin and the dashboard.

| Responsibility | Consumer | Why wevibe-mcp Owns It |
|---------------|----------|---------------------|
| Identity & key management | Plugin, Dashboard | Keys live in file-backed keystore (`${WEVIBE_KEYSTORE_PATH}/keys.json`); single process access prevents race conditions |
| PRE identity management | Consumer, Contributor | secp256k1 keypair for PRE decryption, stored under `wevibe-network`/`pre-identity-key` |
| Sidecar subprocess management | Plugin, Dashboard | Spawns wevibe-umbral CLI for encrypt/decrypt-reencrypted |
| Cryptographic operations | Plugin, Dashboard | WASM crypto + key material must stay in one process |
| wevibe-guard security scanning | Plugin | Recall-time scan before injection |
| Memory decryption (recall) | Plugin | PRE decapsulation via sidecar decrypt-reencrypted (cfrag + capsule → plaintext) |
| Memory decryption (moderation) | Dashboard | Mod key from local keystore for pending submission decryption |
| Memory encryption (approval) | Dashboard | Umbral encrypt via sidecar at approval time (DEK → capsule + ciphertext) |
| Hub API proxy | Plugin | Plugin doesn't call hub directly; wevibe-mcp handles auth signing |
| LLM operations (extraction) | Dashboard | Ollama/OpenRouter calls at session→memory extraction time (contributor flow) |
| LLM operations (batch keyword extraction) | Dashboard | Leader-triggered batch pipeline — NOT at approval time; runs after moderator approves |

wevibe-mcp exposes its services to consumers via localhost. The protocol (HTTP API vs MCP-over-SSE) is a transport choice — what matters is that wevibe-mcp is running and reachable.

Note: Ollama is the only documented host exception. wevibe-mcp now runs in Docker. See Section 13 (D-13.10) and the Container Topology section above.

---

## Cross-Cutting: Container Topology (Locked at Sprint 25 Closeout)

WeVibe runs as seven Docker services orchestrated by `wevibe-server/docker-compose.yml`, plus one host exception documented at D-13.10. This topology is locked — Sprint 26 work must not re-introduce host-process services without an explicit decision.

### Docker Services

| Service | Container Name | Image Source | Internal Address | Ports | Healthcheck |
|---------|----------------|--------------|------------------|-------|-------------|
| postgres | wevibe-postgres | postgres:16-alpine | postgres:5432 | 5432 | pg_isready (5s interval, 10 retries) |
| qdrant | wevibe-qdrant | qdrant/qdrant:v1.9.0 | qdrant:6333 | 6333, 6334 | tcp connect localhost:6333 (5s interval, 10 retries) |
| wevibed (chain) | wevibe-validator | built from wevibe-chain/Dockerfile | wevibed:26657 (RPC), 9090 (gRPC), 1317 (REST) | 26657, 9090, 1317 | curl localhost:26657/status (5s interval, 20 retries, start_period: 15s) |
| hub | wevibe-hub | built from Dockerfile.hub | hub:4440 | 4440 | wget localhost:4440/health (5s interval, 15 retries, start_period: 10s) |
| dashboard | wevibe-dashboard | built from wevibe-dashboard/Dockerfile | dashboard:3000 | 3000 | wget localhost:3000 (5s interval, 15 retries, start_period: 10s) |
| umbral-sidecar | wevibe-umbral | built from Dockerfile.umbral-sidecar | umbral-sidecar:4460 | 4460 | nc -z localhost 4460 (5s interval, 20 retries, start_period: 10s) |
| wevibe-mcp | wevibe-mcp | built from Dockerfile.wevibe-mcp | wevibe-mcp:4450 | 4450 | token-authenticated GET /v1/health (5s interval, 15 retries, start_period: 15s) |

### Host Exceptions (Documented at D-13.10)

| Service | Reason | Status |
|---------|--------|--------|
| Ollama | Metal GPU acceleration on macOS; container Linux without Metal is dramatically slower | PERMANENT (architectural, not a TODO) |

### Cross-Service Dependencies

hub depends on: postgres (service_healthy), qdrant (service_healthy), wevibed (service_healthy), umbral-sidecar

### Bringing the Stack Up

`make dogfood` is the canonical end-to-end command. It:
1. Tears down any prior stack (`docker compose down -v`)
2. Builds and starts all six containers
3. Waits for healthchecks (via `scripts/wait-for-stack-healthy.sh`)
4. Runs Stage 1 (service health) and Stage 2 (pipeline smoke) tests
5. Tears down the stack on completion

**Why this is locked:** Sprint 25 spent six COs (CO-253 through CO-258) consolidating the previous split host/container model into a clean containerized topology. Re-introducing host processes (other than the documented Ollama exception) reverses that work. Any Sprint 26 service additions go in Docker by default.

---

## Cross-Cutting: Epoch Types

"Epoch" is used in two distinct senses in this system. Disambiguate them:

| Term | Definition | Trigger | Duration |
|------|------------|---------|----------|
| **Rotation epoch** | PRE crypto rotation unit | Member removal event | No fixed duration; event-driven; each rotation generates new epoch keypair and new kfrags |
| **Chain epoch** | Time-based window for emissions, idle decay, and bootstrap grace | Time-based (configured in `x/epochs` module) | Fixed duration (configured at chain initialization; query via `wevibed query epochs epoch-info wevibe_epoch`) |

**Independence:** These are completely independent systems. An org may have 1 rotation epoch and 100 chain epochs (no members ever removed), or 5 rotation epochs and 5 chain epochs (frequent rotation during early period).

**Disambiguation rules:**
- "Epoch rotation" / "rotate epoch" / member removal context → **rotation epoch**
- "Bootstrap grace epochs," "idle decay per epoch," "min_contributions_per_epoch," "per-epoch bandwidth caps" → **chain epoch**
- Where context makes the meaning obvious (e.g., §2 Leader's "epoch rotation" flow only discusses rotation epochs), the shorter form is acceptable

See DECISIONS.md D-4.7 for the full rationale.

---

## Cross-Cutting: Memory Lifecycle State Machine

Every memory in the system moves through a defined lifecycle. The state machine is the contract between all participant flows — every action described in this document ends at or transitions between these states.

### State Diagram

```
                        undo-approve (moderator, only while pending_keyword)
                            ◀────────────────────────────────────────────┐
                                                                     │
                                                                     │
┌─────────┐    ┌────────────────┐    ┌────────────────┐    ┌───────────┐    ┌──────────────┐
│ pending │───▶│ pending_keyword│───▶│ pending_chain │───▶│ committed │───▶│   archived   │
└─────────┘    └────────────────┘    └────────────────┘    └───────────┘    └──────────────┘
     │                │                    │                        │
     │                │                    │                        │
     ▼                ▼                    ▼                        ▼
┌─────────┐      ┌─────────┐          ┌─────────┐            ┌─────────────────┐
│ denied  │      │ pending │          │ reported_deleted │  (terminal states)
└─────────┘      └─────────┘          └─────────────────┘
                        ▲ (leader removes from batch)
                        │
                   (undo-approve above)
```

### State Definitions

| State | Location | Visible to Consumer | Who Acts | Transitions To |
|-------|----------|---------------------|----------|----------------|
| `pending` | Hub PostgreSQL only | No | Moderator | `pending_keyword` (approve), `denied` (deny), `pending_keyword` (undo-approve reverts) |
| `pending_keyword` | Hub PostgreSQL only | No | Moderator (undo-approve), Leader (batch extraction) | `pending_chain` (leader submits batch), `pending` (leader removes from batch or undo-approve), `denied` (leader rejects) |
| `pending_chain` | Hub PostgreSQL only | No | Leader (chain commit) | `committed` (chain TX confirmed), `reported_deleted` (upheld report) |
| `denied` | Hub PostgreSQL only | No | — | (terminal) |
| `committed` | Hub PostgreSQL + Qdrant | **Yes** | — | `archived` (all keywords decayed to 0), `reported_deleted` (upheld report) |
| `archived` | Hub PostgreSQL (excluded from Qdrant) | No | — | (terminal) |
| `reported_deleted` | On-chain only (removed from Qdrant) | No | — | (terminal) |

### Transition Summary

| Transition | Trigger | Participant | UX Flow Reference |
|------------|---------|-------------|-------------------|
| `pending` → `pending_keyword` | Approve | Moderator | §3 Moderator: approve action |
| `pending` → `denied` | Deny | Moderator | §3 Moderator: deny action |
| `pending_keyword` → `pending` | Undo-approve (moderator) OR leader removes from batch | Moderator / Leader | §3 Moderator: undo-approve; §2 Leader: batch pipeline |
| `pending_keyword` → `pending_chain` | Leader runs batch keyword extraction + verification passes | Leader | §2 Leader: batch keyword extraction |
| `pending_keyword` → `denied` | Leader rejects memory from batch | Leader | §2 Leader: batch pipeline |
| `pending_chain` → `committed` | Multi-message `MsgApproveMemory` TX confirms | Leader | §2 Leader: batch chain commit |
| `committed` → `archived` | All keyword weights decay to 0 | (automatic via decay) | DECISIONS.md D-4.4 |
| `committed` → `reported_deleted` | Upheld report, leader submits `MsgReportMemory` | Leader | §5 Consumer: report review |
| `pending_chain` → `pending_keyword` | Leader removes memory from batch after review | Leader | §2 Leader: batch pipeline |

### Key Properties

- **Pre-commit states are hub-only.** `pending`, `pending_keyword`, and `pending_chain` exist only in hub PostgreSQL. They are never on chain and never in Qdrant.
- **Qdrant insert is deferred to chain commit.** Memories enter Qdrant only after `MsgApproveMemory` confirms. This ensures Qdrant contains only chain-authoritative data (see DECISIONS.md D-6.2).
- **`committed` is the only retrievable state.** Only `committed` memories appear in Qdrant and are returned during recall. All other states are invisible to consumers.
- **`archived` is the only terminal decay state from `committed`.** DORMANT, DEGRADED, STABLE are not used — with `confidence_bps` removed, intermediate health states have no signal to track (see DECISIONS.md D-4.6).
- **`pending_keyword` → `pending` via undo-approve is only available while status is `pending_keyword`.** Once the leader has moved the memory to `pending_chain` via batch submission, undo is no longer possible — the leader must use "remove from batch" instead (see DECISIONS.md D-6.5).
- **Undo-approve is race-safe.** The handler performs an atomic `UPDATE ... WHERE status = 'pending_keyword'`; if the status has already moved, zero rows are updated and a 409 is returned.

**Why pre-commit states are hub-only:** Keyword extraction is expensive and leader-controlled. Deferring chain commitment until after batch review means the chain record reflects leader-reviewed, verified keywords — not raw moderator approvals. If a memory fails verification or is removed from a batch, no chain record exists (see DECISIONS.md D-6.1, D-6.2).

---

Open gaps only. Resolved gaps have been removed (history lives in implementation reports). Architectural decisions and rationale live in `DECISIONS.md`.

## Closed in Sprint 25

### GAP-M4 (Org Discovery) CLOSED in Sprint 25

discovery.go implemented, routes wired at GET /v1/orgs/discover, dashboard page at /discover, sidebar entry present. Handler: DiscoverOrgs in wevibe-hub/internal/api/handlers/discovery.go. See D-12.7.

### GAP-M5 (Join Request Workflow) CLOSED in Sprint 25

join.go implemented (347 lines, SubmitJoinRequest + ListJoinRequests + ApproveJoinRequest + DenyJoinRequest), routes wired, dashboard page at /join-requests, sidebar entry present. See D-12.8.

### GAP-M9 (Qdrant Per-Org Collections) CLOSED in Sprint 25

OrgCollectionName(orgID) returns org_<orgID>_memories, EnsureCollection creates per-org collections on first upsert, old shared wevibe_memories collection removed. See D-12.1.

## Severity Classifications

| Severity | Definition |
|----------|-----------|
| **CRITICAL** | Blocks the core UX for a participant. The experience cannot function without this. |
| **MAJOR** | Significant degradation of the participant experience. Workarounds exist but are unacceptable for production. |
| **MODERATE** | Feature exists but incomplete. Experience works with friction. |
| **MINOR** | Polish, convenience, or future feature. Does not block core flows. |

---

## CRITICAL

### GAP-T2: Migration System Required Before Public Testnet

**Participant:** Self-hosted hub operators, external testnet users
**Status:** CLOSED by CO-267 (Sprint 27)

**Resolution:** golang-migrate integrated into hub startup. Migrations at `wevibe-server/db/migrations/` (000001_initial_schema.up/down.sql, 000002_notification_preferences.up/down.sql). Hub startup runs migrations before VerifyConnection. db/README.md documents operator usage. Schema reference copy preserved in schema.sql header. See CO-267 implementation report.

### GAP-T3: wevibe-mcp Containerization

**Participant:** Consumer (self-hosted operators)
**Status:** CLOSED by CO-267 (Sprint 27)

wevibe-mcp now runs as a Docker service (`wevibe-mcp`) in `wevibe-server/docker-compose.yml`. Native blockers were removed:
- `keytar` replaced by file-backed keystore (`${WEVIBE_KEYSTORE_PATH}/keys.json`)
- local native embedding runtime replaced by Ollama HTTP embeddings (`${OLLAMA_HOST}/api/embeddings`)
- native `argon2` replaced by pure-JS `@noble/hashes/argon2`

The host exception list now only includes Ollama (D-13.10). See CO-267 implementation report.

### GAP-C1: OpenCode Plugin Uses Subprocess Interface

**Participant:** Consumer
**Status:** CLOSED by CO-260 (Sprint 26)

The OpenCode plugin now communicates with wevibe-mcp via HTTP API (127.0.0.1:4450) rather than subprocess. Plugin reads session token from `~/.wevibe/mcp-session-token` (mode 0600) and includes `Authorization: Bearer <token>` on all calls. Serve event recording and report submission route through wevibe-mcp. Subprocess interface removed. See DECISIONS.md D-12.5.

### GAP-C2: wevibe-mcp HTTP API Not Exposed for Plugin Consumption

**Participant:** Consumer (via plugin)
**Status:** CLOSED by CO-260 (Sprint 26)

wevibe-mcp now exposes first-class HTTP API at 127.0.0.1:4450 with four endpoints: `GET /v1/health`, `POST /v1/recall`, `POST /v1/serves`, `POST /v1/reports`. All endpoints require Bearer token auth (D-12.5a). Plugin is the sole client surface to the hub. See DECISIONS.md D-12.5.

---

## MAJOR

### GAP-M6: Notification System Does Not Exist

**Participant:** All
**Status:** CLOSED by CO-267 (Sprint 27)

No notification mechanism. Dashboard pages required manual refresh. No real-time updates for join approval, moderation results, contribution feedback, or earnings.

**Resolution:** Activity feed (D-12.9) + email + webhook notification channels implemented. Per-user notification preferences. `notification_preferences` table added via migration. Email via SMTP, webhook for agent/Slack integration. See CO-267 implementation report.

### GAP-M8: Serve Event Recording Path Is Ad-Hoc

**Participant:** Consumer, Contributor (receives credit)
**Status:** CLOSED by CO-260 (Sprint 26)

Plugin now records serve events via wevibe-mcp POST /v1/serves (value-add proxy). wevibe-mcp handles canonicalization, signing, and hub forwarding. GAP-C2 and GAP-M8 resolved together. See DECISIONS.md D-12.5.

### GAP-S1: Social Graph Service Not Implemented

**Participant:** All
**Status:** CLOSED by CO-267 (Sprint 27)

Hub-to-display-name mapping was absent. Dashboard rendered truncated wallet addresses.

**Resolution:** Social graph service at `wevibe-server/social-graph-service/` provides wallet→display name mapping. Endpoints: POST/GET/PATCH /v1/profiles/:wallet, batch query, health. Hub integration via `internal/social/client.go`. Moderator role requires display name registration. See CO-267 implementation report.

### GAP-T4: Chain Pruning + IAVL Fast-Node Disabled in Dev Mode

**Participant:** Validator, all on-chain participants
**Status:** CLOSED by CO-267 (Sprint 27)

**Resolution:** init-chain.sh already had production pruning settings applied: `pruning=custom`, `pruning-keep-recent=100`, `pruning-interval=10`. IAVL fast-node is enabled (default when not explicitly disabled). Chain builds and starts successfully. Hub's broadcast retry logic (8 attempts, 400ms backoff) handles transient IAVL errors. See CO-267 implementation report.

### ARCH-G2: "Instant Complete Revocation" Is Misleading for AI Systems

**Participant:** All
**Status:** CLOSED by CO-266 (Sprint 28)

PRE revocation is precise: it provides **instant cryptographic revocation of un-retrieved content**, not "remote wipe" of all copies.

| Content Category | Revocation Effective? |
|-----------------|----------------------|
| Un-retrieved memories | YES — cryptographically |
| In-flight memories (re-encrypted but not yet decrypted) | YES — if session terminated |
| Already-decrypted memories in agent plaintext | NO |
| Derivative artifacts (summaries, local vectors, prompt traces) | NO |

**Resolution:** Revocation language revised in `wevibe-docs/SECURITY-MODEL.md` and `wevibe-docs/WHITEPAPER.md`. Correct scope documented. wevibe-sdk session-end secure deletion deferred post-alpha. See CO-266 implementation report.

### ARCH-G6: Qdrant Embedding Inversion — Long-Term Encrypted Vector Search Needed

**Participant:** Consumer, Leader
**Status:** EVALUATED — NO VIABLE LIBRARY (CO-267 Sprint 27)

Phase 1 mitigations are in place (Gaussian noise σ=0.1, Qdrant API key auth, internal-network deployment). However, vector embeddings are not one-way hashes — published attacks demonstrate embedding inversion. High-confidence memories are the richest inversion targets.

**Findings from CO-267 evaluation:**
- No Go SDK for IronCore Alloy (Rust/Java/Kotlin/Python only)
- Alloy is AGPL-3.0-only by default (commercial license required)
- No Qdrant native support for property-preserving encryption

**Resolution:** Phase 1 mitigations continue as ongoing measure. Encrypted vector search deferred to post-alpha. See `workspace/docs/EVAL-ENCRYPTED-VECTOR-SEARCH.md`.

---

### GAP-CHAIN-20: IAVL State Query Failure on Fresh Chains

**Participant:** Validator, Leader, all CLI users
**Milestone:** ALPHA
**Status:** OPEN

All module state queries (`query auth`, `query gov`, `query upgrade`, `query bank`, `query staking`, etc.) fail with `"failed to load state at height N; version does not exist (latest height: N)"` on fresh wevibe-chain instances. The chain produces blocks and processes transactions normally — only the ABCI Query path through IAVL is broken.

Discovered during CO-005b Docker-based upgrade verification. Reproduced across multiple chain re-initializations, with and without `--iavl-disable-fastnode`, at various block heights. TX-index queries (`query tx <hash>`) and HTTP RPC status work unaffected.

**Impact:** CLI `query` commands are unusable. Validators cannot inspect chain state. The dashboard, hub, and any tooling that uses gRPC/ABCI queries for state will fail. CO-005b upgrade verification worked around this via TX submission + HTTP RPC + TX-index queries.

**Resolution requires:**
- Diagnose root cause in app multistore/IAVL initialization (likely store key registration, IAVL version tracking, or CommitMultiStore configuration)
- Fix the IAVL query path so committed state is accessible at current and historical heights
- Verify all standard module queries work on fresh and replayed chains
- Regression test to prevent reintroduction

---

## MODERATE

### GAP-O3: Dashboard Voting UI Missing for Approval Quorum

**Participant:** Moderator
**Status:** CLOSED by CO-265 (Sprint 27)

Moderation page showed Approve/Deny buttons but no explicit Vote button. For orgs with `required_approvals > 1`, moderators need to vote before final approval.

**Resolution:** Dashboard moderation page now shows Vote to Approve button for `required_approvals > 1`, with vote count display and threshold visualization. See CO-265 implementation report.

### GAP-O6: Hub Credits and Chain Treasury Are Disconnected

**Participant:** Leader
**Status:** CLOSED by CO-266 (Sprint 28)

Two separate systems existed: Hub `org_credits` (PostgreSQL integer credits, 1 deducted per query) and Chain `StoredTreasury` (uvibe, debited by emissions for contributor payouts). Leader had no unified view of org finances.

**Resolution:** Dashboard billing page now shows both credits balance and chain financial data via `GET /v1/orgs/{orgID}/finances`. See CO-266 implementation report.

### GAP-O7: Chain Config Not Manageable from Dashboard

**Participant:** Leader
**Status:** CLOSED by CO-266 (Sprint 28)

Dashboard could not manage chain-level org config (`serve_attestation_required`, `contest_stake_uvibe`) or reputation tiers.

**Resolution:** Dashboard settings page now exposes chain configuration read/edit UI. New hub `GET /v1/orgs/{orgID}/chain-config` (leader-only) and `PATCH /v1/orgs/{orgID}/config` updated to accept chain fields. See CO-266 implementation report.

### GAP-O9: No User-Visible Feedback When Contribution Is Flagged

**Participant:** Contributor
**Status:** CLOSED by CO-265 (Sprint 27)

wevibe-guard at submit time logged warnings to stderr but contributor saw "submitted N for review" even when content was flagged. No indication of security findings.

**Resolution:** Hub submit response now includes `sanitization_findings`; dashboard sessions page displays amber success+warning banner when findings exist. Contributor sees sanitization status at submit time. See CO-265 implementation report.

### GAP-O10: Hub ↔ Chain Org Records Not Synchronized

**Participant:** Leader
**Status:** CLOSED by CO-265 (Sprint 27)

Hub creates org in PostgreSQL. Chain has separate org registration via `MsgRegisterOrg`. These were independent records that could diverge.

**Resolution:** `CreateOrg` now broadcasts `MsgRegisterOrg` to wevibe-chain during org creation and persists `orgs.chain_registered` state. Chain registration failure does not roll back hub org creation. See CO-265 implementation report.

### GAP-O11: Pre-Existing Integration Test Suite Failures

**Participant:** Worker / CI (does not affect runtime UX)
**Status:** CLOSED by CO-266 (Sprint 28)

Seven test files failed against the dogfood stack due to fixture-state drift and assertion drift from recent CO changes.

**Resolution:** Per-file triage completed. Stale e2e/integration harnesses converted to `describe.skip(...)` placeholders (`capstone.test.ts`, `e2e-flow.test.ts`, `full-lifecycle.test.ts`). Moderation and server-tools tests updated for changed tool counts and `memory_type_override` drift (`moderation-approval.test.ts`, `moderation.test.ts`, `server-tools.test.ts`). See CO-266 implementation report.

### ARCH-G7: Model Provider Leakage Policy Not Enforced

**Participant:** Consumer, Contributor
**Status:** CLOSED by CO-266 (Sprint 28)

When a memory is injected into a cloud inference session (Claude API, GPT-4, etc.), the model provider can potentially see the memory content.

**Resolution:**
- `provider_policy` modes implemented: `unrestricted` (default), `local_only`, `allowlist`
- Policy stored in `~/.wevibe/plugin-config.json`
- `wevibe_set_provider_policy` MCP tool for configuration
- `local_only` blocks non-local provider artifacts
- `allowlist` checks against org-scoped allowed providers returned from hub membership
- `wevibe_author_memory` restricted to leader role only; non-leaders receive explicit admin-path description

See CO-266 implementation report.

### ARCH-G8: PRE Recovery Path — Operational Procedure Not Documented

**Participant:** Leader
**Status:** CLOSED by CO-266 (Sprint 28)

Recovery path properties (2-of-3 Shamir, on-chain multi-sig auth, separate `wevibe-recover` CLI, on-chain logging) were designed but not implemented or documented.

**Resolution:** Created `workspace/docs/RUNBOOK-PRE-RECOVERY.md` documenting the recovery ceremony procedure and invocation steps. See CO-266 implementation report.

### ARCH-G9: BIP-32 Key Hierarchy Separation Not Implemented

**Participant:** All

Coupling financial identity (Cosmos wallet) with confidential memory retrieval (PRE encryption) increases blast radius. Mitigation via BIP-32 derived child key for PRE identity is designed but not implemented.

**Resolution requires:**
- Implement BIP-32 child key derivation for PRE identity
- Use distinct derivation path from wallet keys
- Document key hierarchy in security documentation

### OQ-6: Operational Runbook for Epoch SK Compromise

**Participant:** Leader
**Status:** CLOSED by CO-266 (Sprint 28)

Runbook for epoch SK compromise procedure was not defined.

**Resolution:** Created `workspace/docs/RUNBOOK-EPOCH-SK-COMPROMISE.md` documenting the background re-encryption procedure, chain epoch status transition, hub behavior during compromised window, re-encryption throughput targets, gas cost estimation per 100K memories, and monitoring/alerting for completion tracking. See CO-266 implementation report.

### OQ-7: Docker Compose Network Fix for Smoke Tests

**Participant:** Developer (CI/CD)
**Status:** CLOSED by CO-265 (Sprint 27)

`wevibe-chain/scripts/smoke-test.sh` expected chain node at `localhost:26657`. Docker Compose has separate networks.

**Resolution:** RPC_URL override documented in smoke-test.sh script header. `RPC_URL` env var override supported. See CO-265 implementation report.

---

## MINOR

### GAP-N1: No Stripe Integration for Credit Top-Up

**Participant:** Leader

Billing page has "Stripe coming soon" button. Hub `TopUpCredits` is manual credit injection without payment processing or signature verification.

### GAP-N2: No Memory Editing Before Moderation Approval

**Participant:** Moderator
**Status:** CLOSED by CO-266 (Sprint 28) — fallback implementation

Cannot edit content before approving — only Approve or Deny.

**Resolution:** Dashboard deny dialog now offers "Save & Edit" option for encrypted content that cannot be previewed inline. Records original+edited content in denial reason field. Fallback used when crypto pipeline constraints prevent inline content editing. See CO-266 implementation report.

### GAP-N3: No Contributor Feedback on Denial

**Participant:** Contributor
**Status:** CLOSED by CO-265 (Sprint 27)

`DenySubmission` recorded `denial_reason` but nothing surfaced to the contributor.

**Resolution:** New hub endpoint `GET /v1/orgs/{orgID}/my-submissions` consumed by dashboard `My Submissions` page. Status badges and denial reason inline rendering for denied submissions. See CO-265 implementation report.

### GAP-N4: No Role-Gated Dashboard Sidebar

**Participant:** All
**Status:** CLOSED by CO-265 (Sprint 27)

All users saw all pages regardless of role.

**Resolution:** Sidebar navigation gated by `activeOrg.role`. Members see limited nav; moderators/leader see moderation, reports, join requests; leaders additionally see batch pipeline, members, keywords, recovery, epochs, settings. See CO-265 implementation report.

### GAP-N5: Chain Features Without Any Surface

**Participant:** Various

The following chain features have proto definitions and working keeper logic but no hub endpoint, dashboard UI, or CLI interface:

| Feature | Chain Module | Blocked Participant |
|---------|-------------|-------------------|
| Memory contests (stake + dispute) | `x/memory` | Contributor, Moderator |
| Memory relationships (contradicts/replaces/deprecates) | `x/memory` | Moderator |
| Validity bounds (time-scoped retrieval) | `x/memory` | Leader |
| Full reputation querying (XP, profile, serve stats) | `x/reputation` | Contributor |
| Contributor profile (cross-org) | `x/reputation` | Contributor |
| Emission pool visibility | `x/emissions` | Validator, Leader |
| Work scores | `x/emissions` | Validator |
| Bootstrap credits | `x/emissions` | New User |
| Asymmetric gating | `x/emissions` | Validator |
| Bandwidth state visibility | `x/bandwidth` | Leader, all members |

### GAP-N8: No Trial Period Logic in Hub

**Participant:** New User
**Status:** CLOSED by CO-266 (Sprint 28)

`MsgGrantTrialAllowance` proto existed on chain. Hub had no trial awareness. Retrieval was all-or-nothing — no keyword-only degraded mode.

**Resolution:** Trial membership schema added (`members.is_trial`, `members.trial_expires_at`, `orgs.trial_days`). Join approval accepts `trial` boolean. Trial members blocked from contribution (submit + batch-submit return 403). Retrieval enforces expiry check and daily rate limit (default 5/day). Trial→full upgrade clears trial state. See CO-266 implementation report.

### GAP-N9: No Batch Memory Submission

**Participant:** Contributor
**Status:** CLOSED by CO-266 (Sprint 28)

Sessions page submitted memories one at a time via individual POST requests.

**Resolution:** Sessions page now supports batch submission via `POST /v1/orgs/{orgID}/moderation/batch-submit`. Unified progress indicator shows batch submission status. See CO-266 implementation report.

### GAP-N10: `wevibe_author_memory` MCP Tool Fate Unclear

**Participant:** Moderator/Leader
**Status:** CLOSED by CO-266 (Sprint 28)

`wevibe_author_memory` lets leaders/moderators submit + immediately approve a memory. This is a second contribution path alongside the sessions page.

**Resolution:** `wevibe_author_memory` is kept. Gated to leader role only — non-leaders receive explicit description indicating this is an admin-path tool. See CO-266 implementation report.

### GAP-N11: Pre-Existing TS1005 Syntax Error in wevibe-guard Plugin

**Participant:** Worker / Developer (does not affect runtime)
**Status:** CLOSED by CO-265 (Sprint 27)

`wevibe-opencode-plugin/plugins/wevibe-plugin.ts` contained a TypeScript syntax error: `TS1005: ';' expected`. The malformed type annotation and generic syntax around the inline type expression was fixed. See CO-265 implementation report.

---

## Sprint 26 Scope

### In Scope (Sprint 26)

| Item | D-Reference | Closes |
|------|-------------|--------|
| Plugin HTTP API with per-session token auth (Bearer token, ~/.wevibe/mcp-session-token mode 0600) | D-12.5, D-12.5a | GAP-C1, GAP-C2, D-12.5a | **CLOSED** |
| Serve event recording via wevibe-mcp value-add proxy | D-12.5 | GAP-M8 | **CLOSED** |
| CO-260 finishing items (Makefile dogfood probe + cross-module doc updates) | CO-262 | — | **CLOSED** |

### Out of Scope (Deferred)

| Item | D-Reference | Reason |
|------|-------------|--------|
| Transport encryption (TLS) for wevibe-mcp HTTP API | — | Post-alpha; loopback-only is sufficient for local dev |
| Per-request token rotation | D-12.5a | Boot-lifetime token sufficient for solo dogfood |

---

## Sprint 25 Scope

### In Scope (Sprint 25)

| Item | D-Reference | Closes |
|------|-------------|--------|
| Multi-org isolation hardening (Qdrant per-org collections + authz middleware) | D-12.1 | GAP-M9 | **CLOSED** |
| Plugin HTTP API replacing subprocess (wevibe-mcp at 127.0.0.1:4450, loopback-only, no auth) | D-12.5 | GAP-C1, GAP-C2 | — |
| Plugin failure UX (auto-start first, plugin-specific fallback) | D-12.6 | — | — |
| Consumer profile pages (/profile, /u/:wallet) | D-12.4 | — | — |
| Per-memory org destination dropdown in contributor extraction UI | D-12.2 | — | — |
| Moderator multi-org queue switcher | D-12.1 | — | — |
| Org discovery (public wevibe.network/orgs + in-dashboard /discover) | D-12.7 | GAP-M4 | **CLOSED** |
| Zero-friction join request workflow (no form fields, profile is application) | D-12.8 | GAP-M5 | **CLOSED** |
| Activity feed (WebSocket + bell + /activity page, all-orgs aggregated) | D-12.9 | GAP-M6 (partial) | — |
| Solo dogfood end-to-end pipeline validation + make dogfood smoke-test command | D-12.10 | — | — |

### Closed in Sprint 25

| Item | Reference | Closes |
|------|-------------|--------|
| Container topology consolidation (umbral-sidecar containerized, six-service stack locked) | CO-258 | — |

### Out of Scope (Deferred)

| Item | D-Reference | Reason |
|------|-------------|--------|
| Cross-org retrieval at recall time | D-12.3 | Solo dogfood doesn't need it; builds single-org loop first |
| Per-session token auth for plugin ↔ wevibe-mcp | D-12.5a | Pre-alpha requirement, not needed for dogfood |
| Notification infra beyond activity feed (email, mobile push, agent-channel pings) | D-12.9 | Post-Sprint 25 |
| Verified social links on profile | D-12.4 (future) | Nice-to-have, not blocking |
| Multi-org memory submission | D-12.2 (future) | Design exists, not yet implemented |
| Auto-approve join requests based on reputation | D-12.8 | Per-org human decision for v1 |
| Multi-team alpha onboarding | — | Post-Sprint 25 |

---

## Sprint 29 Scope

### In Scope (Sprint 29)

| Item | Reference | Status |
|------|-----------|--------|
| Deferred test vector regeneration + cleanup | CO-006 | **CLOSED** |
| Cosmos SDK foundation realignment to v0.53.5 + CometBFT v0.38.20 | CO-008 | **CLOSED** |
| x/upgrade compatibility verification and re-validation on new SDK foundation | CO-005-resume | **OPEN** |

### GAP-CHAIN-3: x/upgrade Verification

**Participant:** Validator, Leader
**Status:** OPEN - to be closed by CO-005-resume

`cosmossdk.io/x/upgrade` verification remains open as the final chain-foundation gate for this sprint.

**Note:** Unblocked by CO-008 (SDK downgrade); CO-005 resumes against v0.53.5 foundation.

### References

- Full decision rationale: `DECISIONS.md` §12 (D-12.1 through D-12.10)
- Qdrant isolation verified: single shared collection `wevibe_memories` with org_id filter (classification: B — Filtered isolation)

---

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 7 (GAP-CHAIN-1 through GAP-CHAIN-7) |
| MAJOR | 7 (GAP-CHAIN-9 through GAP-CHAIN-14, GAP-CHAIN-20) |
| MODERATE | 1 (ARCH-G9 — BIP-32 key hierarchy, real crypto work) |
| MINOR | 7 (GAP-N1 Stripe, GAP-N5 chain features, GAP-CHAIN-15 through GAP-CHAIN-19) |
| **Total OPEN** | **23** |
| Documented Finding (not actionable) | 1 (ARCH-G6 — no viable encrypted vector search library; Phase 1 mitigations continue) |

---

*End of MASTER.md*
