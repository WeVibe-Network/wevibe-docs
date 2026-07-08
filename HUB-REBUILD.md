# HUB-REBUILD — Hub Retrieval Plane Rebuild Specification

> **Title:** HUB-REBUILD — authoritative rebuild contract for hub retrieval state  
> **Role:** Operator and leader specification for reconstructing Qdrant vectors/payload and retrieval-coupled hub caches from chain authority plus org keys  
> **Status:** **[SPEC — the rebuild-hub program (MI-3.4) is DEFERRED; this is the contract it implements against]**  
> **Grounded in:** WHITEPAPER §3.2/§3.5/§3.10/§8.1-§8.4, DECISIONS `D-HUB-REBUILDABLE`/`D-RECALL-ALIGNMENT`/`D-MISSION-INVARIANT`/`D-13.9`, `RECALL-MC1-DIRECTIVE.md` (CO-B/CO-C, INV-3/8/10/12), `08-07-26-0433-mc1-chain-alignment-chart.md`, `08-07-26-0510-mc1-chain-caveats-and-storage.md`.

---

## 1. Purpose & Scope

Role: define what rebuild covers, what it excludes, and how completion is judged.

### 1.1 Derived/disposable retrieval plane

- **[CURRENT REALITY]** The hub retrieval plane is derived state. Chain commitments plus org keys are authority. (WHITEPAPER §3.5, §8.3, `D-MISSION-INVARIANT`)
- **[TARGET — post-CO-B/CO-C]** Any authorized key-holder can reconstruct a replacement hub retrieval plane from chain authority without incumbent-hub cooperation. (`D-HUB-REBUILDABLE`)

### 1.2 Success criterion: top-k parity, not byte identity

- **Normative acceptance:** parity is measured as top-k overlap against a fixed query set at an agreed threshold. (`D-HUB-REBUILDABLE`: acceptance)
- **Reason:** vector write path injects Gaussian noise at upsert (`injectGaussianNoise`), so raw vector bytes are not a stable equality contract. (`retrieval.go:23-36`, `retrieval.go:346`)
- **Operator consequence:** pin model + schema; verify retrieval equivalence by ranking overlap and delivery integrity, not float-for-float identity. (`D-RECALL-ALIGNMENT`, `watcher_memory.go:121-127`, `watcher_memory.go:125`)

### 1.3 Rebuild triggers

- Disaster recovery or hostile/unavailable incumbent hub. (`D-HUB-REBUILDABLE`)
- Qdrant corruption or payload/index drift.
- Embedding-model upgrade requiring full re-embed. (WHITEPAPER §13)
- Clean-machine portability drill and parity proof. (`D-HUB-REBUILDABLE` acceptance)
- Audit verification of chain-to-serve fidelity. (`D-MISSION-INVARIANT`, WHITEPAPER §8.3)

### 1.4 Rebuild side effect: epoch rotation + endpoint re-point

- **[TARGET — post-CO-B/CO-C]** Rebuild implies epoch rotation on the replacement hub, fresh kfrags minted there, then leader re-points chain hub endpoint authority and clients fail over. (`D-HUB-REBUILDABLE` item 4)

---

## 2. Authoritative-vs-Derived Model (provenance backbone)

Role: map every retrieval payload key to its authority class.

### 2.1 Current Qdrant payload map

Current payload shape is emitted by `payloadMap` in `UpsertPoint` with optional embedding keys and vector dimension. (`retrieval.go:326-344`)

| Payload field | Source | Status |
|---|---|---|
| `cid` | chain `content_hash` hex, `StoredMemoryCommitment` field 2 (`state.proto:49-51`) | CURRENT |
| `org_id` | chain `org_id`, field 1 (`state.proto:49`) | CURRENT |
| `epoch_id` | chain `epoch`, field 6 (`state.proto:54`); hub loads via `pending_submissions.epoch_id` (`watcher_memory.go:43-48`, `watcher_memory.go:107`) | CURRENT |
| `memory_type` | chain `memory_type`, field 13 (`state.proto:61`; `watcher_memory.go:111`) | CURRENT |
| `keyword_weights` | re-derived projection from chain keywords + extracted classified weights (`watcher_memory.go:79-85`), then serialized at upsert (`retrieval.go:317-324`, `retrieval.go:331`) | CURRENT |
| `lifecycle_state` | derived projection: initially `ACTIVE` at index (`watcher_memory.go:110`), then synchronized to chain state (`sync.go:98-105`, `sync.go:144-152`) | CURRENT |
| `content_flags` | present on `IndexEntry` but not populated by watcher entry construction (`protocol/types.go:360-367`, `watcher_memory.go:104-112`); upsert writes empty array (`retrieval.go:308-311`, `retrieval.go:330`) | CURRENT (empty) |
| `embedding_model_id` | embedding step metadata, default fallback `nomic-embed-text:v1.5` (`watcher_memory.go:119-127`) | CURRENT (hub DB, not chain) |
| `embedding_schema_version` | constant fallback `retrieval-card-v1` (`watcher_memory.go:123-126`) | CURRENT |
| `vector_dim` | embedding output length; runtime constant 768 (`retrieval.go:42`, `watcher_memory.go:117`) | CURRENT |
| vector body | re-derived from decrypted plaintext via retrieval-card embedding path, then noise-injected on upsert (`retrieval-card.ts:44-63`, `embed-card.ts:21-24`, `embedding.ts:24-32`, `retrieval.go:346`) | CURRENT (re-derived) |
| qdrant point id | deterministic `stableID = FNV-1a(cid)` (`retrieval.go:1448-1455`) | CURRENT |

### 2.2 Target-added payload (CO-C)

| Payload field | Source class | Status |
|---|---|---|
| `language[]` | hub payload projection, content-adjacent, not chain-anchored | **[TARGET — post-CO-C]** |
| `superseded_by` | derived projection from `StoredMemoryRelationship` (`state.proto:74-82`) under INV-3 | **[TARGET — post-CO-C]** |

### 2.3 Current-vs-target collection topology

- **[CURRENT REALITY]** Per-org collections: `org_<orgID>_memories`. (`retrieval.go:38-40`, `retrieval.go:251-305`)
- **[TARGET — post-CO-C]** Pre-ingest payload indexes for retrieval-boost fields (`language`, `superseded_by`) over the per-org collection; recall is relevance-gated over the org pool (INV-6 + keyword boost + relevance floor/τ), with no project-scope query filter. (`RECALL-MC1-DIRECTIVE.md:89-93`)
- **[CURRENT REALITY]** `language`/`superseded_by` do not exist in hub retrieval payload code path. (`retrieval.go:326-344`, `08-07-26-0433-mc1-chain-alignment-chart.md:42-44`)

---

## 3. The On-Chain Field Set Required for Rebuild (the MC-1 chain-leg)

Role: define the mandatory chain anchors for faithful replay.

### 3.1 Per-memory fields already anchored on chain

- `StoredMemoryCommitment` already carries memory identity, ciphertext material, wrapped DEK, keyword weights, lifecycle, epoch, memory type, and integrity fingerprints including `plaintext_hash`/`ciphertext_hash`/`wrapped_dek_hash`/`salt`/`contributor_sig`/`contributor_address`. (`state.proto:48-72`)

### 3.2 Current API gap on chain read path

- **[CURRENT REALITY]** `GetMemory` and `GetMemoriesBatch` gRPC handlers omit six cryptographic fields from response projection: `plaintext_hash`, `salt`, `ciphertext_hash`, `wrapped_dek_hash`, `contributor_sig`, `contributor_address`. (`grpc_query.go:31-48`, `grpc_query.go:166-183`, `state.proto:63-67`, `state.proto:71`)
- **Operator implication:** a rebuild requiring full integrity fingerprints must either read the keeper store directly or widen gRPC responses first.

### 3.3 Per-memory anchors required but absent today

The ratified balance is: publish knowledge verifiably on-chain; protect people-related material only. Recall is relevance-gated over the org pool (INV-6 prompt_digest + keyword boost + relevance floor/τ) — there is no on-chain project-scope field and no server-side project filter. (`08-07-26-0510-mc1-chain-caveats-and-storage.md:37-46`)

- **Required per-memory anchors (missing):**
  - `mc_version`

### 3.4 Per-org / per-epoch anchors required but absent today

- **Required org/epoch anchors (missing):**
  - `vocab_hash` (authoritative taxonomy verifier for rebuild)
  - `embedding_model_id` (authoritative re-embed model pin)
- **Canonical vocab hash contract (MI-3.3):**

```text
vocab_hash = sha256( join('\n', sort_asc(non_deprecated_org_keywords)) ).hex_lower
```

- Rationale for these anchors is explicit in whitepaper rebuild claims and decision lock. (WHITEPAPER §8.2, §8.3, `D-HUB-REBUILDABLE` item 2-3)

### 3.5 Honest current state

- **[CURRENT REALITY]** MC-1 chain-leg is not wired end-to-end for these fields. The write envelope is dropped at `submitMemory`; only partial fields survive to hub/chain. (`contribution.ts:17-151`, `08-07-26-0433-mc1-chain-alignment-chart.md:5-6`, `08-07-26-0433-mc1-chain-alignment-chart.md:26-33`)
- **Therefore:** faithful rebuild-from-chain is impossible today (the per-org rebuild anchors below are still missing).
- **Blocking dependency:** CO-B (chain-leg anchors) + CO-C (collection/index/filter contract) must land first. (`RECALL-MC1-DIRECTIVE.md:84-93`)
- **Documentation honesty:** whitepaper §8.2/§8.3 currently overstates as-built rebuildability for missing anchors. (`08-07-26-0510-mc1-chain-caveats-and-storage.md:16`)

---

## 4. Inputs / Prerequisites

Role: define strict operator prerequisites before any rebuild run.

- Chain RPC + gRPC endpoint reachable for memory queries and relationship reads.
  - Note: approved-memory enumeration gap remains (see §5.a).
- Leader-controlled `K_master` available after vault unlock.
  - Vault unlock path: `unlockVault` + CLI `wevibe-admin vault-unlock`. (`vault.ts:114-135`, `admin.ts:694-698`, `admin.ts:809`, `admin.ts:853`)
  - Touch ID / biometric local gate exists in custody flows via native biometric addon surface (`biometric.ts:25-29`); keychain passphrase path remains available for vault unlock.
  - Key retrieval path: `loadKeyEnvelope(orgId, 'master')`. (`key-store.ts:212-218`, `admin.ts:734`)
- Local key-derivation capability for epoch encryption key.

```text
K_master --derive_epoch_keys(epoch)--> K_enc(e)
```

(`crypto.rs:273-293`)

- Ollama reachable with pinned embedding model `nomic-embed-text:v1.5` and retrieval-card schema contract.
  - Canon pin and no-`latest` rule: (`D-RECALL-ALIGNMENT`)
  - Local config source: `~/.config/wevibe/dashboard.json`. (`embedding-config.ts:26-28`, `embedding-config.ts:105-108`)
  - Prefix contract: `search_document:` for doc embeddings. (`config.ts:98`, `embedding.ts:29-32`)
- Authoritative vocabulary snapshot available and verifiable against `vocab_hash` anchor (target).
- Qdrant reachable and writable.
- Umbral sidecar at `127.0.0.1:4460` only if rebuild flow also re-stamps PRE capsules for delivery paths; not required for chain-leg decrypt/re-embed itself. (`TOPOLOGY.md:46`, WHITEPAPER §3.2, §3.10)

---

## 5. Step-by-Step Rebuild Procedure

Role: define the exact execution contract for rebuild.

### 5.a Enumerate org memories from chain

- **[CURRENT GAP]** No list-all approved-memory RPC exists.
  - Query API exposes `GetMemory`, `GetMemoriesBatch`, `GetMemoryCount`, `ListRelationships`, `GetValidity`; no pagination/list endpoint for full approved corpus. (`query.proto:11-36`)
  - `GetMemoriesBatch` hard-caps at 50 hashes per request. (`grpc_query.go:151-153`)

Available options:

| Option | Status | Tradeoff |
|---|---|---|
| Add paginated `ListMemories` query RPC | **[TARGET — recommended]** | Correct read-path contract; requires chain API decision |
| `ExportGenesis` and read approved memories | **[CURRENT REALITY]** | Works now; heavy-weight export path (`keeper.go:683-716`) |
| Direct store iteration over `approved/<orgID>/` prefix | **[CURRENT REALITY]** | Works now; bypasses public query interface (`keeper.go:55-60`, `keeper.go:703-716`) |

Recommended target: add paginated `ListMemories` RPC and make it the canonical rebuild reader.

State constraints:

- `MEMORY_STATE_COMMITTED = 4`; committed records are persisted under `approved/` keys. (`state.proto:16`, `keeper.go:385`, `keeper.go:401`)
- Pending states persist under `pending/` and are excluded from rebuild. (`keeper.go:47-53`)

### 5.b Create collection + payload indexes before ingest

- **[CURRENT REALITY]** Hub creates per-org collections with cosine vectors, size 768, unnamed vector; no payload-index creation in current path. (`retrieval.go:38-40`, `retrieval.go:251-305`, `retrieval.go:42`)
- **[TARGET — post-CO-C]** Build into target collection topology with payload indexes created before any point ingest:
  - `org_id` (tenant key)
  - `language`
  - `superseded_by` placeholder
  (`RECALL-MC1-DIRECTIVE.md:89-93`)
- **Why pre-ingest is mandatory:** filterable HNSW links are built with index awareness at ingest time; retrofitting payload indexes after ingest forces another rebuild cycle. (`RECALL-MC1-DIRECTIVE.md:91-92`)
- **Filter/query contract:** disable unindexed-field filtering; recall is relevance-gated over the org pool — the query is bounded by `org_id` (tenant) plus the relevance floor/τ, with no project-scope filter. (`RECALL-MC1-DIRECTIVE.md:90-93`)
- **[CURRENT REALITY]** Hub query filters on `org_id` and optional `embedding_model_id` only — which is the intended contract; no project-scope filter is required. (`retrieval.go:391-399`, `08-07-26-0433-mc1-chain-alignment-chart.md:43`)

### 5.c Per memory: verify -> decrypt -> re-embed -> payload -> batch upsert

Normative pipeline:

```text
for each committed memory:
  verify ciphertext_hash == sha256(encrypted_blob)
  K_enc(e) = derive_epoch_keys(K_master, epoch).enc_key
  DEK = decrypt_symmetric(wrapped_dek_enc, K_enc(e))
  plaintext = decrypt_symmetric(encrypted_blob, DEK)
  verify plaintext_hash == sha256(salt || plaintext)
  card = buildRetrievalCard(parseMemoryText(plaintext), stack)
  vector = computeLocalEmbedding(card, role=document, prefix=true, pinned model)
  payload = chain-anchored fields + derived projections
  point_id = stableID(cid)
batch upsert
```

Evidence anchors:

- Epoch key derivation contract: (`crypto.rs:273-293`, WHITEPAPER §3.2)
- Symmetric decrypt primitive exists in SDK core: (`crypto.rs:247-258`)
- Plaintext hash composition uses `salt || plaintext`: (`contribution.ts:76-80`, `state.proto:63-66`)
- Retrieval-card construction: (`retrieval-card.ts:44-63`)
- Embedding call contract (`role=document`, `prefix=true`): (`embed-card.ts:21-24`, `embedding.ts:24-32`, `config.ts:98`)
- Point id determinism: (`retrieval.go:1448-1455`)

Critical distinction:

- **Chain-leg rebuild path** must use on-chain `wrapped_dek_enc` + epoch enc-key derivation.
- **Live moderation path** uses `wrapped_dek_mod` + moderator private key and is Postgres-bound, not chain-authoritative for chain-only rebuild. (`moderation.ts:180-194`, `state.proto:60`)

### 5.d Rebuild `superseded_by` as derived projection

- **[TARGET — post-CO-C]** For each memory, derive `superseded_by` from approved relationships where:
  - memory is `source_cid`
  - `approved = true`
  - `relation_type` in `{SUPERSEDES, REPLACES, DEPRECATES}`
  - projection value is `target_cid`
  (`state.proto:22-28`, `state.proto:74-82`)
- INV-3 constraint: never mutate on-chain lifecycle state for supersession; this field is disposable retrieval projection only. (`RECALL-MC1-DIRECTIVE.md:26`, `D-SUPERSESSION-DEMOTE`)

### 5.e Restore + verify vocabulary taxonomy against `vocab_hash`

- **[TARGET — post-MI-3.3]** Rebuild restores leader-signed vocabulary snapshot and verifies canonical hash equals on-chain anchor before query service goes live. (WHITEPAPER §8.2, `D-HUB-REBUILDABLE` item 2)

### 5.f Verify top-k parity before cutover

- Run fixed query set against source hub and rebuilt hub.
- Compute overlap at agreed top-k threshold.
- Promote rebuilt collection only on pass.
- This parity gate is mandatory acceptance, not optional smoke. (`D-HUB-REBUILDABLE` acceptance)

---

## 6. Verifiability & Integrity

Role: define the proof package a leader/auditor must produce.

Required fidelity proof bundle:

1. **Per-memory integrity checks**
   - `ciphertext_hash` matches `encrypted_blob` bytes.
   - `plaintext_hash` matches recovered plaintext plus `salt`.
   (`state.proto:63-66`, `contribution.ts:76-80`)

2. **Anchor checks**
   - Rebuild run uses chain-anchored `embedding_model_id` and `vocab_hash` (target state).
   - Local model config must equal anchor.
   (`D-HUB-REBUILDABLE` item 2-3, WHITEPAPER §8.2)

3. **Top-k parity report**
   - Fixed query corpus.
   - Overlap threshold pass/fail.
   - Delivery-integrity gate satisfied before any number is believed: required check is `delivery=YES`, never `CALLED`/`NO`.
   (`D-HUB-REBUILDABLE` acceptance, `RECALL-MC1-DIRECTIVE.md:40`)

4. **Archived-memory handling proof**
   - Rebuild includes archived memories in index corpus for audit/parity continuity.
   - Query path excludes archived from serving via `must_not lifecycle_state=ARCHIVED`.
   (`retrieval.go:401-402`)
   - Chain retains archived commitment under approved store key with archived epoch set; rebuild must preserve that fact, not delete it.
   (`lifecycle.go:209-212`, `lifecycle.go:95`, `keeper.go:702-716`)

Outcome requirement:

- Leader can demonstrate, with evidence, that served recall is exactly the chain-committed semantic shadow under the pinned retrieval contract.

---

## 7. Infra for the (DEFERRED) rebuild PROGRAM — spec, do not build

Role: define requirements for future `rebuild-hub` implementation.

**Status:** **[DEFERRED — GAP-MI-3.4]**

### 7.1 CLI shape

```bash
wevibe-admin rebuild-hub --org <org_id> [--dry-run] [--verify-only] [--resume] [--re-embed-model <id>]
```

### 7.2 Resumability contract (R-32)

- Persist checkpoint + resume marker after each memory or batch.
- `--resume` continues from last completed checkpoint only.
- Never restart from zero unless operator explicitly requests reset.

### 7.3 Live logging contract (R-31 / R-37)

- Log file path: `runs/rebuild-hub/<ISO8601>.log`.
- Emit per-memory or per-batch progress lines.
- Log crypto fingerprints only: first-8-hex SHA256 fingerprints of key identities; never raw keys, DEKs, plaintext, ciphertext.
- Thread a trace id through operations (`X-WeVibe-Trace-Id` semantic contract).
- Emit full errors; never swallow catches.
- No silent operations; no `stdio: ignore` behavior.

### 7.4 Idempotency and cutover contract

- Point identity is deterministic (`stableID(cid)`), so repeat upserts overwrite, not duplicate. (`retrieval.go:1448-1455`)
- Blue-green collection cutover is mandatory:
  1. build into new collection
  2. run parity/integrity gates
  3. atomically swap alias/endpoint
  4. keep previous collection for rollback window

### 7.5 Runtime dependencies

- Chain node endpoints (RPC/gRPC)
- Qdrant
- Ollama with pinned embedding model
- Umbral sidecar only when rebuild run includes capsule restamping for PRE delivery

### 7.6 Throughput contract

- Local embedding step is serial in operator path; optimize via batch upserts and staged I/O.
- This is the pinned deterministic embedder path, not a local LLM production path; it does not violate R-33’s ban on local LLM on production path. (`RECALL-MC1-DIRECTIVE.md:24`, `D-RECALL-ALIGNMENT`)

### 7.7 Failure handling

- Fail closed on any integrity mismatch.
- Persist checkpoint before process exit on interrupt where possible.
- Resume must be deterministic and auditable.

---

## 8. Security & Safety During Rebuild

Role: preserve mission invariants while replaying derived retrieval state.

- All plaintext handling remains local on leader-operated machine.
- Hub must never receive `K_master`, `K_enc(e)`, or `epoch_sk`.
  - Epoch keys derive client-side from `K_master` using HKDF namespace contract. (`crypto.rs:273-293`, WHITEPAPER §3.2)
  - Hub confidentiality claim remains: hub cannot decrypt even if malicious. (WHITEPAPER §3.10)
- Rebuild does not alter confidentiality boundary; it re-derives the same disclosed semantic shadow already held for retrieval service. (WHITEPAPER §3.5, §8.3)
- INV-12 hygiene remains mandatory for any on-chain payload:
  - never publish personal paths/usernames/hostnames
  (`RECALL-MC1-DIRECTIVE.md:44`, `08-07-26-0510-mc1-chain-caveats-and-storage.md:42-43`)
- Mission invariant preserved: no single party gains unilateral plaintext READ power from rebuild mechanics. (`D-MISSION-INVARIANT`)

---

## 9. Edge Cases

Role: define required behavior under non-happy-path conditions.

- **Embedding-model upgrade**
  - Re-embed all committed memories.
  - Update on-chain `embedding_model_id` anchor before declaring rebuild complete.
  - Coordinate org-wide model alignment; mismatched query model id can yield zero hits.
  (WHITEPAPER §13, `D-HUB-REBUILDABLE` item 3)

- **Partial or interrupted rebuild**
  - Resume from checkpoint.
  - No full restart from zero unless explicit operator reset.
  (R-32 contract)

- **Multi-org isolation**
  - **[CURRENT REALITY]** Per-org collection reduces cross-org bleed by topology. (`retrieval.go:38-40`)
  - **[TARGET — CO-C acceptance]** Two-org leak test must pass: seed X and Y, query X, observe zero Y hits.
  (`RECALL-MC1-DIRECTIVE.md:93`)

- **Pending memories excluded**
  - Pending pipeline state is ephemeral and excluded from rebuild corpus.
  - Only committed memories are replayed.
  (`D-HUB-REBUILDABLE` item 5, `keeper.go:47-60`, `keeper.go:355-407`)

---

## 10. Relationship to MC-1 Program & Deferred List

Role: pin this spec to the MC-1 delivery sequence and deferred inventory.

- This specification is the contract MI-3.4 implements; it depends on:
  - **CO-B** chain-leg anchoring of missing MC-1 rebuild-critical fields
  - **CO-C** collection rebuild with pre-ingest retrieval-boost payload indexes (relevance-gated recall over the org pool; no project-scope query filter)
  (`RECALL-MC1-DIRECTIVE.md:84-93`)

- Explicit deferred list:
  - **MI-3.4** rebuild program/tool implementation (this spec only)
  - **MI-3.3** vocab canonical-hash anchor + leader-signed taxonomy snapshot contract
  - **MI-3.5** clean-machine parity runbook + reproducibility drill

- Honest shipped-vs-missing summary:
  - MI-3.1 and MI-3.2 are recorded as shipped in program tracking.
  - Per-memory MC-1 anchor (`mc_version`) and per-org rebuild anchors (`vocab_hash`, `embedding_model_id`) required for chain-faithful rebuild are still not wired end-to-end.
  - Therefore chain-faithful rebuild-from-chain remains blocked until CO-B + CO-C complete.
  (`08-07-26-0433-mc1-chain-alignment-chart.md:5-6`, `08-07-26-0433-mc1-chain-alignment-chart.md:65-77`)

Final statement:

- **[CURRENT REALITY]** Full authoritative rebuild is impossible today.
- **[TARGET — post-CO-B/CO-C]** This spec becomes executable as an operator-grade rebuild contract.
- **[DEFERRED — GAP-MI-3.x]** `rebuild-hub` implementation must conform to this document exactly.
