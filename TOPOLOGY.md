# WeVibe Network — Complete Repository Topology

**Generated:** 2026-05-19  
**Scope:** Full repository — all packages audited

---

## Workspace Tooling — Proto Generation

Protobuf-derived Go code is regenerated via `make proto-gen` from the
`wevibe-meta` Makefile. The target wraps Docker-pinned tooling and
covers every proto tree in the workspace.

**Subtargets:**
- `make proto-gen-chain` — regenerates `wevibe-chain` proto outputs via `ghcr.io/cosmos/proto-builder:0.18.1`
- `make proto-gen-umbral` — regenerates `wevibe-server/wevibe-hub/internal/umbral/umbralpb` via `bufbuild/buf:1.34.0`

**Proto trees covered:**
| Repo | Proto path | Output path | Image |
|------|-----------|-------------|-------|
| wevibe-chain | `proto/wevibe/{org,memory,serve,reputation,emissions,bandwidth,attestation}/v1/*.proto` | `x/*/types/*.pb.go` | `ghcr.io/cosmos/proto-builder:0.18.1` |
| wevibe-umbral | `proto/umbral/v1/sidecar.proto` | (consumed by wevibe-server) | — |
| wevibe-server | `wevibe-hub/internal/umbral/umbralpb/sidecar.proto` | `wevibe-hub/internal/umbral/umbralpb/*.pb.go` | `bufbuild/buf:1.34.0` |

**Adding a new proto tree:** add a new subtarget to `wevibe-meta/Makefile`,
add it to the `proto-gen` umbrella target's dependency list, and pin
any new Docker image in the image-pins block at the top of the Makefile.

**Rule:** never invoke local `protoc`/`buf`/`protoc-gen-*` binaries. All
proto generation runs inside Docker images pinned by exact tag. See
DECISIONS.md D-14.21, R-PROTO-REGEN.

---

## Deployment Topology (Sprint 26)

### Docker Services

| Service | Container | Internal Address | Host Port | Purpose |
|---------|-----------|------------------|-----------|---------|
| postgres | wevibe-postgres | postgres:5432 | 5432 | Hub operational DB |
| qdrant | wevibe-qdrant | qdrant:6333 | 6333, 6334 | Vector storage |
| wevibed | wevibe-validator | wevibed:26657 (RPC), 9090 (gRPC), 1317 (REST) | 26657, 9090, 1317 | Cosmos appchain |
| hub | wevibe-hub | hub:4440 | 4440 | Go API server |
| dashboard | wevibe-dashboard | dashboard:3000 | 3000 | Next.js UI |
| umbral-sidecar | wevibe-umbral | umbral-sidecar:4460 | — | PRE encryption (GPL-3.0 sidecar) |
| wevibe-mcp | wevibe-mcp | wevibe-mcp:4450 | 4450 | MCP + local crypto + HTTP API |

### Host Exceptions

| Service | Reason | Status |
|---------|--------|--------|
| Ollama | Metal GPU on macOS; no Metal in Linux containers | PERMANENT |

### Chain Broadcast (CO-258)

Hub broadcasts via Comet RPC `broadcast_tx_sync` at `tcp://wevibed:26657` (D-13.12). Fees calculated as `ceil(gas × 0.01 uvibe)`. Retry on transient state-load errors.

### Schema Bootstrap

Hub schema at `wevibe-server/db/schema.sql`. Applied on Postgres container init (D-13.10).

---

## Repository Structure

```
./
├── wevibe-guard/          # Rust — YARA-X prompt injection scanner
├── wevibe-mcp/           # TypeScript — MCP client for AI agents
├── wevibe-sdk/           # Rust + Python — crypto primitives & WASM bindings
├── wevibe-umbral/        # Rust — PRE sidecar (GPL-3.0, NEW in CO-216)
├── anchor/               # Rust/Solana — Solana Anchor programs
├── protocol/             # Protocol definitions, OpenAPI spec
└── docs/                 # Documentation
```

## Sprint 23 Highlights (CO-230 through CO-234)

- **Old-format memory wipe**: Qdrant wevibe_memories collection wiped and recreated. Hub PostgreSQL stale records cleared. Schema migrations applied to running database via RunMigrations.
- **Report flow**: Complete report lifecycle implemented. Reports are NOT blacklists — they enter moderation quorum. Reporter identity linked. Trial members cannot report. Upheld reports require leader wallet-signed chain TX (upheld_pending_tx → upheld). Dismissed reports track reporter's dismissed_reports_count.
- **Deny/blacklist separation**: Deny triggers local blacklist + denial attestation (feeds confidence decay). Report triggers moderation queue (no local effect until resolved).
- **Blacklist in recall pipeline**: `is_blacklisted()` wired into retrieve-cli.ts after PRE decrypt, before guard scan.
- **Wallet-signed security config**: `required_approvals` and `report_vote_threshold` changes require leader wallet `signArbitrary` signature, not just Ed25519 delegate key.
- **New hub endpoints**: `POST /v1/orgs/{orgID}/reports/{reportID}/vote`, `POST /v1/orgs/{orgID}/reports/{reportID}/commit`.
- **New hub file**: `internal/verify/wallet_sig.go` — secp256k1 wallet signature verification.

## Sprint 24 Highlights

- wevibe-hub introduces moderation vote endpoints, report APIs, and the `required_approvals` org config surfaced through PostgreSQL migrations.
- wevibe-dashboard adds the Reports queue and settings control for `required_approvals`, aligned with the plugin's Accept / Deny / Report flow.
- wevibe-chain authorizes moderator approvals and validates the fee grant trial allowance path, keeping chain state synchronous with hub quorum policy.
- **Vocabulary-constrained keyword extraction (CO-236):** wevibe-mcp fetches org keyword vocabulary from hub (`GET /v1/orgs/{orgID}/keywords`) before extraction. The LLM classifies memories against the established vocabulary, with new keyword suggestions requiring moderator approval. Classified keywords are auto-approved; suggestions submitted to hub for verification via keyword pipeline (CO-238).

## Sprint 26 Highlights (CO-238)

- **Multi-stage memory lifecycle**: Memory submissions now flow through four distinct states: `pending` → `pending_keyword` → `pending_chain` → `committed`. Moderator approval at `pending` only transitions to `pending_keyword` — keyword extraction, Qdrant indexing, and chain submission are decoupled and handled by separate pipeline stages.
- **Batch keyword extraction**: Dashboard orchestrates keyword extraction via `wevibe_extract_memories` MCP tool. Results submitted to hub for verification (`POST /v1/orgs/{orgID}/submissions/{hash}/keywords`). Hub verifies keywords against vocabulary and transitions status to `pending_chain`.
- **Hub keyword verification endpoints** (new in `internal/api/handlers/keyword_extraction.go`):
  - `POST /v1/orgs/{orgID}/submissions/{submissionHash}/keywords` — submit verified/extracted keywords
  - `POST /v1/orgs/{orgID}/submissions/{submissionHash}/keywords/rerun` — trigger re-extraction via Ollama
  - `PUT /v1/orgs/{orgID}/submissions/{submissionHash}/keywords` — update keyword set
  - `DELETE /v1/orgs/{orgID}/submissions/{submissionHash}` — remove submission from pipeline
  - `GET /v1/orgs/{orgID}/submissions/keywords/pending` — list submissions awaiting keyword verification
  - `GET /v1/orgs/{orgID}/submissions/keywords/pending-chain` — list submissions ready for chain submission
- **Batch chain submission**: Leaders review `pending_chain` submissions in dashboard and trigger `POST /v1/orgs/{orgID}/submissions/batch-chain-submit`. Multi-message Cosmos TX (atomic: all-or-nothing). Embeddings computed on-the-fly via Ollama at submit time. Post-commit: Qdrant upsert, `memory_keywords` population, status → `committed`.
- **Leader activity tracking**: `leader_activity` table tracks `last_batch_extraction_at` and `last_chain_submission_at` per leader.
- **GAP-O8 resolved**: `/api/extract` endpoint (dashboard) now proxies through MCP `wevibe_extract_memories` tool — no direct Ollama/OpenRouter calls.
- **wevibe-mcp changes**: `src/extraction.ts` weight normalization (weights sum to 1.0). `approveSubmissionMessageSimple` replaces complex multi-step approval message. `wevibe_extract_keywords_batch` and `wevibe_extract_memories` MCP tools added.

## Sprint 27 Highlights (CO-265)

- **Role-gated dashboard navigation**: Sidebar now uses org membership role (`leader` / `moderator` / `member`) to hide unauthorized sections and expose a new **My Submissions** page to all roles.
- **Contributor denial visibility**: New hub endpoint `GET /v1/orgs/{orgID}/my-submissions` is consumed by dashboard to display submission status and `denial_reason` back to contributors.
- **Submit-time sanitization feedback**: Hub submit response now includes additive `sanitization_findings`; dashboard sessions page displays an amber success+warning banner when findings exist.
- **Quorum voting UX**: Dashboard moderation supports vote-based approval for `required_approvals > 1` (vote count, threshold, voter list), with direct approve retained for `required_approvals == 1`.
- **Hub chain-sync on org creation**: Hub now broadcasts `MsgRegisterOrg` to wevibe-chain during org creation and persists `orgs.chain_registered` state without rolling back hub org creation on chain failure.
- **Smoke-test RPC topology clarity**: `wevibe-chain/scripts/smoke-test.sh` now documents `RPC_URL` override and the Docker default `localhost:26657` mapping.

## Sprint 28 Highlights (CO-266)

- **Trial membership (GAP-N8):** New `members.is_trial`, `members.trial_expires_at`, `orgs.trial_days` schema fields. Join approval accepts `trial` boolean. Trial members blocked from contribution (`POST /v1/orgs/{orgID}/submit` and batch-submit return 403). Retrieval enforces expiry check and daily rate limit (default 5/day). Trial→full upgrade clears trial state.
- **Provider policy enforcement (GAP-O11, N10):** wevibe-mcp `provider_policy` setting (`unrestricted|local_only|allowlist`) evaluated at recall time. `local_only` blocks non-local providers; `allowlist` checks against org-scoped allowed providers from hub membership. Block returns `reason_code: "provider_not_allowed"`. New MCP tool `wevibe_set_provider_policy` for configuration.
- **Leader-only author memory (GAP-ARCH-G7):** `wevibe_author_memory` MCP tool restricted to leader role. Non-leaders receive explicit admin-path description in tool response.
- **Unified finances UI (GAP-O6):** Dashboard billing page shows credits balance + chain financial data. New hub `GET /v1/orgs/{orgID}/finances` endpoint.
- **Chain config relay (GAP-O7):** New hub `GET /v1/orgs/{orgID}/chain-config` endpoint (leader-only). Dashboard settings page exposes chain config read/edit UI.
- **Moderation edit-before-approval fallback (GAP-N2):** Dashboard deny dialog offers "Save & Edit" option for encrypted content that cannot be previewed inline. Denial records original+edited content in reason field.
- **Batch submit path (GAP-N9):** Sessions page supports batch submission of memories via `POST /v1/orgs/{orgID}/moderation/batch-submit`. Unified progress indicator shows batch submission status.
- **Test triage (Task A):** Stale e2e/integration harnesses converted to `describe.skip(...)`. Moderation and server-tools tests updated for changed tool counts and `memory_type_override` drift.

## Sprint 24 Content Sanitization + Preference Flagging (CO-239)

- **Unicode threat scanner**: New `internal/sanitize/` package in wevibe-hub with `scanner.go` (Unicode category scanner) and `homoglyphs.go` (Cyrillic/Greek look-alike detection). Detects: bidirectional overrides, format characters, control characters, invisible spaces, zero-width joiners, homoglyphs, zalgo combining marks.
- **Submission-time sanitization**: `POST /v1/orgs/{orgID}/submit` calls `sanitize.Scan()` on plaintext before encryption. Findings stored in `pending_submissions.sanitization_findings` JSONB column (not blocking — moderator decides).
- **Preference confidence**: `MemoryCandidate` in `src/extraction.ts` now includes `preference_confidence` field (0.0–1.0). Extraction prompt rates confidence; default 0.0 if missing. wevibe-mcp passes through to dashboard.
- **Dashboard flag visibility**: Moderation page and chain-submit page display sanitization finding badges (critical=red, warning=amber) and `preference_confidence` badges (>0.8=red, >0.5=amber).
- **Sanitization findings shape**: `{ findings: [{ type, category, char, position }], summary: { critical, warning } }`

## Sprint 24 Keyword Weight Decay (CO-240)

- **Confidence field KILLED**: Memory-level `confidence_bps` removed from `StoredMemoryCommitment`. Keyword weights ARE the health metric.
- **`KeywordWeight` on-chain**: `{ keyword, weight (string decimal), serve_count, denial_count }` stored as `repeated KeywordWeight` on `StoredMemoryCommitment` and `StoredPendingCommitment`.
- **Decay parameters**: denial_decay_bps (500), serve_boost_bps (100, cap 5/epoch), idle_decay_bps (50), bootstrap_grace_epochs (14).
- **Serve boost**: `ApplyServeBoost` on serve TX confirmation: all keywords +100 bps, cap 5 serves/epoch.
- **Denial decay**: `ApplyDenialDecay` on denial TX confirmation: all keywords -500 bps.
- **Idle decay**: `ApplyIdleDecay` epoch hook: all keywords -50 bps for memories with `last_active_epoch < current_epoch`.
- **Archive condition**: ALL keyword weights = 0.0 (not confidence == 0).
- **Hub mirroring**: `ApplyServeBoostLocal` / `ApplyDenialDecayLocal` in wevibe-hub applied on confirmed TX (R-ATOMIC). Same formula as chain.
- **Qdrant update**: Keyword weight changes reflected in Qdrant payload metadata.
- **Crash recovery**: `SyncEpochData` resyncs from chain on hub restart.
- **Gas model**: Each memory retrieval = one chain TX (serve or denial). Predictable per-event gas cost.
- **Starter packs DEFERRED**: Org responsibility, not WeVibe platform.

## Sprint 25 Highlights (CO-225)

- **Dual-vector decay model** (deprecated by CO-240): wevibe-chain `x/memory` previously applied idle decay (50 bps) and negative-signal decay (500 bps) at epoch close based on memory-level `confidence_bps`. CO-240 replaces this with per-keyword weight decay.
- **Pay-per-memory**: `x/emissions` `ProcessOrgPayouts` counts approved memories per contributor (not serves). `payout_per_memory` replaces `payout_per_serve`. Qualification via `min_contributions_per_epoch` org config; tier cap via `MaxContributionsPerEpoch`.
- **`MsgSubmitDenialBatch`**: `x/serve` accepts batched denial attestations from org leaders; stores `StoredDenialAttestation` keyed by org/epoch/memory-hash.
- **Hub denial wiring**: wevibe-hub `POST /v1/orgs/{orgID}/denials` records denials; `POST /v1/orgs/{orgID}/denials/batch-submit` relays to chain. `serve_events` table uses `event_type IN ('serve', 'denial')` and `reason` column. `SubmitDenialBatch` added to chain client.

## Sprint 25 Highlights (CO-247)

- **Multi-org UI**: Org switcher in dashboard topbar (`components/layout/org-switcher.tsx`), OrgProvider context wrapping dashboard layout (`lib/org-context.tsx`).
- **Per-memory org destination**: Sessions extraction page now supports submitting different memories to different orgs via per-memory org dropdown (D-12.2).
- **Consumer profile pages**: `/profile` (own profile, client component) and `/u/:wallet` (public profile, server component) displaying on-chain reputation data (D-12.4).
- **Hub profile endpoint**: `GET /v1/profile/{wallet}` public endpoint aggregating wallet/org memberships/chain stats (CO-247).
- **Org context persistence**: Active org ID stored in `localStorage` under `wevibe_active_org_id`.

## Sprint 25 Highlights (CO-245)

- **Workspace Makefile**: New `Makefile` at workspace root with targets: `start`, `stop`, `clean`, `health`, `dogfood`, `dogfood-health`, `dogfood-pipeline`. Primary dogfood command: `make dogfood`.
- **Pipeline health dashboard page**: New `/health` page in wevibe-dashboard showing status of 7 services: PostgreSQL (via Hub), Qdrant, wevibe-chain, wevibe-hub, wevibe-mcp HTTP (port 4450), Ollama, and Dashboard self.
- **Dogfood pipeline smoke test**: New `tests/e2e/dogfood-pipeline.test.ts` exercising the full memory lifecycle: submit → approve with keywords → batch chain submit → recall via wevibe-mcp HTTP `/v1/recall`.
- **wevibe-mcp HTTP URL in test config**: `tests/lib/config.ts` added `wevibeMcpHttpUrl` pointing to `http://127.0.0.1:4450`.

## Sprint 25 Highlights (CO-246)

- **Per-org Qdrant collections**: Multi-org isolation via per-org collection naming `org_{orgID}_memories`. The old shared `wevibe_memories` collection is dead. Collection is derived via `OrgCollectionName(orgID)` function.
- **Lazy collection creation**: Collections created on first upsert, not at startup. `EnsureCollection(ctx, client, orgID)` called with org ID before first memory stored.
- **Authz middleware**: New `RequireOrgMembership` middleware in `internal/auth/middleware.go` enforces org membership on all org-scoped routes. Public routes: health, member orgs, create org, get org, epoch manifest, billing topup.
- **Defense-in-depth**: org_id filter retained on Qdrant queries even though collection name provides org scoping.
- **Context helpers**: `GetMemberPubkey(ctx)` and `GetMemberOrgID(ctx)` in `internal/auth/middleware.go` for handlers to read authenticated member info.

**Chain broadcast rewrite (CO-258):** Hub now uses Comet RPC `broadcast_tx_sync` instead of Cosmos gRPC `BroadcastTx`. Fees: gas × 0.01 uvibe with 2000 uvibe floor. Transient error retry (8 attempts, 400ms backoff). See D-13.12.

## Sprint 25 Highlights (CO-245c — Post-CO-245 Chain Hardening Surfaces)

This section documents the surface areas that CO-245 will implement, locked by CO-245c decisions.

### wevibe-chain / x/reputation

- **New file:** `keeper/moderator_profile.go` — moderator profile CRUD + per-org accountability tracking
- **New file:** `keeper/leader_profile.go` — leader profile CRUD + epoch rotation history
- **New proto types:** `StoredModeratorProfile`, `StoredLeaderProfile`, `EpochRotation`
- **New gRPC queries:** `ModeratorProfile`, `LeaderProfile`, `UpheldReportsBy{Contributor,Moderator,Leader}`, `VerifyUpheldReport`
- **Expanded type:** `StoredContributorProfile` (added aggregates + memberships)

### wevibe-chain / x/memory

- **`MsgApproveMemory` signature:** `approvers []string`, `committing_leader string` (replacing single-string `approver`)
- **`MsgReportMemory` signature:** + `plaintext`, `ciphertext`, `capsule`, `plaintext_hash`, `plaintext_oversized`, `approving_moderators []string`, `upholding_moderators []string`, `reporter_pubkey`
- **`StoredMemoryCommitment`:** + `approvers []string`, `committing_leader_pubkey`, `committed_at_epoch`
- **`StoredMemoryReport`:** overhauled to upheld-report record
- **`MemoryState` enum:** 7-value lifecycle (PENDING → REPORTED_DELETED; DORMANT/DEGRADED/STABLE removed)

### wevibe-chain / x/emissions

- **`epoch_hooks.go ProcessOrgPayouts`:** now injects `reputationKeeper.GetContributorProfile` for tier lookup via `getRepTierForContributor`
- **New dependency:** `ReputationKeeper` interface added to `expected_keepers.go`

### wevibe-chain / x/org

- **New file:** `keeper/aggregates.go` — increment methods for `total_committed_memories`, `total_upheld_reports`, `total_epoch_rotations`, `last_activity_epoch`
- **Expanded type:** `StoredOrg` (added aggregate counters)

### wevibe-chain / app/app.go

- **Keeper construction order:** reputation keeper initialized before memory/serve/org keepers (cross-keeper injection dependency order)

### wevibe-hub / internal/chain

- **New file:** `watcher.go` — `ChainWatcher` subscribes to CometBFT blocks, parses `MsgApproveMemory` + `MsgReportMemory` TXs, restart-safety via `watcher_state` table
- **New table:** `chain_commit_events` (idempotent writes, GIN index on `approving_moderators` array)

### wevibe-hub / internal/postgres/migrations

- **New migration:** `chain_commit_events.sql` — table + GIN index on `approving_moderators` array
- **`watcher_state` table:** for resume tracking
- **`published_plaintext` table:** for oversized upheld memory off-chain storage (verified against on-chain `plaintext_hash`)

### wevibe-hub / internal/notifications

- **New package (CO-248):** `internal/notifications/hub.go` — NotificationHub manages per-pubkey WebSocket client sets
- **WebSocket endpoint:** `GET /v1/notifications/ws` — realtime push of new notifications
- **REST endpoints (CO-248):**
  - `GET /v1/notifications` — list notifications (all-orgs aggregated)
  - `GET /v1/notifications/unread-count` — fast unread count
  - `POST /v1/notifications/mark-read` — mark notifications read
- **NotificationHub wired into ChainWatcher:** `emitModeratorNotifications()` broadcasts via hub after DB insert
- **Notification categories:** `chain_commit_involving_you`, `your_approval_was_overturned`, `report_upheld_committed`

## Sprint 22 Highlights (CO-221)

- Hub approval now stores PRE payloads in PostgreSQL (`umbral_capsule`, `umbral_ciphertext`) and retrieval sources capsule/ciphertext from Postgres instead of chain cross-ref payloads.

- Hub approval now stores PRE payloads in PostgreSQL (`umbral_capsule`, `umbral_ciphertext`) and retrieval sources capsule/ciphertext from Postgres instead of chain cross-ref payloads.
- Epoch manifest now carries `umbral_pk`, persisted at org creation and rotation, and served to clients.
- Members can register PRE public keys through dedicated endpoints (`POST/GET /v1/orgs/{orgID}/members/{pubkey}/pre-key`), and invite flow accepts `pre_pubkey`.
- wevibe-mcp approval flow now calls Umbral sidecar CLI to produce approval-time capsule/ciphertext and signs canonical payload with PRE fields.

## Sprint 22 Highlights (CO-222)

- wevibe-mcp now generates and stores a local secp256k1 PRE identity in the file-backed keystore (`wevibe-network` / `pre-identity-key`) and exposes PRE key accessors from `src/auth.ts`.
- wevibe-mcp startup path now registers member PRE pubkeys with hub (`registerPrePubkey`) after identity/membership initialization.
- All hub query calls from wevibe-mcp now include `pre_pubkey`; retrieval decrypt path is PRE-only (`capsule` + `cfrag` + `umbral_ciphertext` -> sidecar `decrypt-reencrypted` -> DEK -> AES decrypt).
- Invite flow now sends both `epoch_sk` and invitee `pre_pubkey`; create/rotate flows persist returned `epoch_sk` + `epoch_pk` locally for future invites.

## Sprint 22 Highlights (CO-224)

- Qdrant payload parity with chain restored in wevibe-hub: approved-memory payload now carries `keywords`, `confidence_bps`, and `lifecycle_state` alongside existing metadata.
- Query scoring in hub retrieval now combines vector similarity, keyword overlap boost, and confidence weighting; lifecycle tiers enforce `ARCHIVED` hard exclusion and `DORMANT` hidden-by-default behavior.
- Hub now runs epoch metadata synchronization (`internal/chain/sync.go`) that polls chain gRPC and reconciles changed confidence/state into Qdrant each interval.
- Keyword rename/merge handlers restored Qdrant sync through `UpdateMemoryKeywords`; list/scroll paths now read keywords directly from Qdrant payload.

## Sprint 21 Highlights (CO-214)

- **Delegate key infrastructure:** Dashboard now generates secp256k1 delegate keys and authorizes them via Cosmos SDK `x/authz` MsgGrant from the user's Keplr/Leap wallet. The delegate key becomes the signing identity for all chain operations.
- **15 WeVibe message TypeURLs** authorized for delegation: memory (SubmitCommitment, ApproveMemory, RejectMemory, ReportMemory), serve (SubmitServeBatch), org (RegisterOrg, AddMember, RemoveMember, SetOrgConfig, SetRepTiers, FundTreasury, WithdrawTreasury), reputation (IncrementContribution, IncrementServe, RecordBan). 90-day grant expiration.
- **Hub delegate key registration:** New `POST /v1/orgs/{orgID}/members/delegate-key` endpoint maps delegate key addresses to wallet addresses. `ResolveDelegateToWallet` enables auth resolution from delegate address to wallet (for CO-215 migration).
- **wevibe-mcp delegate identity storage:** Delegate mnemonic storage added to file-backed keystore under `wevibe-delegate-{walletAddress}` service name.

---

## Chain Upgrade Procedure

WeVibe chain upgrades follow the standard Cosmos SDK x/upgrade flow. The canonical wiring is in wevibe-chain/app/app.go.

### InitChainer contract (manually-wired chain)

wevibe-chain is manually wired (no depinject/app-wiring). The InitChainer must perform these steps in order:

1. Unmarshal genesis JSON from req.AppStateBytes
2. Call ModuleManager.InitGenesis(ctx, cdc, genesisState)
3. Call UpgradeKeeper.SetModuleVersionMap(ctx, ModuleManager.GetVersionMap()) — persists the module version map so future ApplyUpgrade reads a populated fromVM (D-S29-INITCHAINER-VERSION-MAP)
4. Write genesis init marker to every mounted KV store — prevents IAVL empty-tree restart panic (D-S29-CHAIN-RESTART-FOUNDATION)
5. Return the ResponseInitChain from step 2

Step 3 is not needed for depinject-wired chains (PopulateVersionMap handles it automatically). Step 4 is WeVibe-specific (modules without appmodule.HasGenesis would otherwise have empty IAVL trees).

### Upgrade execution flow

1. Submit governance proposal with upgrade plan (name, height)
2. Pre-binary halts at upgrade height with "UPGRADE NEEDED" (no handler registered for that plan name)
3. Swap binary: new binary has SetUpgradeHandler registered for the plan name
4. Post-binary starts, x/upgrade PreBlocker detects upgrade-info.json
5. ApplyUpgrade calls GetModuleVersionMap (populated from step 3 of InitChainer)
6. RunMigrations runs per-module migrations using the version map as fromVM
7. Block production resumes

### Required wiring (app/app.go)

- SetUpgradeHandler: registers the handler function for each upgrade name (CO-005b)
- ReadUpgradeInfoFromDisk + SetStoreLoader(UpgradeStoreLoader(...)): makes LoadLatestVersion upgrade-aware (CO-005c, D-S29-UPGRADE-STORE-LOADER)
- SetModuleVersionMap in InitChainer: persists version map at genesis (CO-005e, D-S29-INITCHAINER-VERSION-MAP)
- Genesis init marker in InitChainer: prevents empty-tree restart panic (CO-005d, D-S29-CHAIN-RESTART-FOUNDATION)

### Adding a new upgrade

To add a v3 upgrade:
1. Create app/upgrades/v3/upgrades.go with UpgradeName and CreateUpgradeHandler
2. Register SetUpgradeHandler for v3 in NewWeVibeApp (alongside existing v2 handler)
3. If v3 adds/removes module stores, populate StoreUpgrades in the UpgradeStoreLoader block
4. Test with the standard fixture rig: build pre-binary (with v2 handler but no v3), build post-binary (with both), run upgrade at target height

---

## wevibe-server/wevibe-hub — Go API Server

**Module:** `github.com/wevibe-network/wevibe-hub`  
**Go version:** 1.24.0  
**Default port:** 4440

### Entry Point

#### `cmd/wevibe-hub/main.go`
**Role:** HTTP server entry point — loads config, connects DB + Qdrant, registers routes, starts server

**Key behavior:**
- Graceful degradation: starts even if DB or Qdrant unavailable (logs warning)
- Qdrant init: `retrieval.NewQdrantClient(cfg.QdrantAddr, cfg.QdrantAPIKey)` → `handlers.SetQdrantClient()` → `retrieval.EnsureCollection()`
- Epoch sync poller: ticker goroutine calls `chain.SyncEpochData(...)` every 60 seconds (Phase 1 interval)
- Node privkey: `handlers.SetNodePrivkey(cfg.NodePrivkey)`
- CORS: reads `CORS_ALLOWED_ORIGINS` env var (comma-separated), defaults to `http://*`, `https://*`

**Post-fix route table:**
```
GET    /health
GET    /v1/members/{pubkey}/orgs
GET    /v1/profile/{wallet}      # Public profile (CO-247)
POST   /v1/orgs
GET    /v1/orgs/{orgID}
PATCH  /v1/orgs/{orgID}/config
POST   /v1/orgs/{orgID}/epoch/rotate
GET    /v1/orgs/{orgID}/epoch/{epochID}/manifest
POST   /v1/orgs/{orgID}/members
GET    /v1/orgs/{orgID}/members
GET    /v1/orgs/{orgID}/members/{pubkey}
POST   /v1/orgs/{orgID}/members/{pubkey}/pre-key
GET    /v1/orgs/{orgID}/members/{pubkey}/pre-key
DELETE /v1/orgs/{orgID}/members/{pubkey}
POST   /v1/orgs/{orgID}/members/wallet
POST   /v1/orgs/{orgID}/members/delegate-key
GET    /v1/orgs/{orgID}/keys/envelope
POST   /v1/orgs/{orgID}/dashboard/keys
DELETE /v1/orgs/{orgID}/dashboard/keys/{pubkey}
POST   /v1/orgs/{orgID}/recovery/shares
GET    /v1/orgs/{orgID}/recovery/shares
POST   /v1/orgs/{orgID}/submit
GET    /v1/orgs/{orgID}/moderation/queue
POST   /v1/orgs/{orgID}/moderation/{submissionHash}/vote
POST   /v1/orgs/{orgID}/moderation/{submissionHash}/approve
POST   /v1/orgs/{orgID}/moderation/{submissionHash}/deny
POST   /v1/orgs/{orgID}/moderation/batch-submit
POST   /v1/orgs/{orgID}/serves
POST   /v1/orgs/{orgID}/serves/batch-submit
POST   /v1/orgs/{orgID}/denials              # Record denial event (CO-225)
POST   /v1/orgs/{orgID}/denials/batch-submit  # Batch submit denials to chain (CO-225)
POST   /v1/orgs/{orgID}/query
GET    /v1/orgs/{orgID}/memories
GET    /v1/orgs/{orgID}/memories/{cid}
POST   /v1/orgs/{orgID}/reject
GET    /v1/orgs/{orgID}/keywords
POST   /v1/orgs/{orgID}/keywords
PUT    /v1/orgs/{orgID}/keywords/merge
PUT    /v1/orgs/{orgID}/keywords/{keyword}/rename
DELETE /v1/orgs/{orgID}/keywords/{keyword}
POST   /v1/orgs/{orgID}/submissions/{submissionHash}/keywords          # Submit verified keywords (CO-238)
POST   /v1/orgs/{orgID}/submissions/{submissionHash}/keywords/rerun     # Rerun extraction via Ollama (CO-238)
PUT    /v1/orgs/{orgID}/submissions/{submissionHash}/keywords           # Update keyword set (CO-238)
DELETE /v1/orgs/{orgID}/submissions/{submissionHash}                     # Remove submission from pipeline (CO-238)
GET    /v1/orgs/{orgID}/my-submissions                                   # Contributor-only submission status view (CO-265)
GET    /v1/orgs/{orgID}/submissions/keywords/pending                     # List pending keyword verification (CO-238)
GET    /v1/orgs/{orgID}/submissions/keywords/pending-chain               # List ready for chain submit (CO-238)
POST   /v1/orgs/{orgID}/submissions/batch-chain-submit                  # Batch chain submission (CO-238)
GET    /v1/orgs/{orgID}/health
GET    /v1/orgs/{orgID}/finances                                         # Credits + chain financial data (CO-266, GAP-O6)
GET    /v1/orgs/{orgID}/chain-config                                     # Chain config read (CO-266, GAP-O7, leader-only)
POST   /v1/billing/topup
GET    /v1/orgs/{orgID}/credits
POST   /v1/orgs/{orgID}/reports
GET    /v1/orgs/{orgID}/reports
GET    /v1/orgs/{orgID}/reports/{reportID}
PATCH  /v1/orgs/{orgID}/reports/{reportID}
POST   /v1/orgs/{orgID}/reports/{reportID}/vote   # Report voting (CO-231)
POST   /v1/orgs/{orgID}/reports/{reportID}/commit  # Leader chain commitment, wallet-signed (CO-233)
GET    /v1/orgs/discover                                           # List/search public orgs (D-12.7, GAP-M4 CLOSED)
POST   /v1/orgs/{orgID}/join                                       # Submit join request (D-12.8, GAP-M5 CLOSED)
GET    /v1/orgs/{orgID}/join-requests                              # List join requests (leader/moderator)
POST   /v1/orgs/{orgID}/join-requests/{requestID}/approve          # Approve join request
POST   /v1/orgs/{orgID}/join-requests/{requestID}/deny             # Deny join request
POST   /v1/internal/reencrypt          # PRE re-encryption (CO-218, was 501)
POST   /v1/internal/epoch-keypair     # Generate epoch keypair (CO-218, was 501)
POST   /v1/internal/orgs/{orgID}/kfrags # Generate kfrags (CO-218, was 501)
```

---

### Config

#### `internal/config/config.go`
**Exports:** `Config` struct, `Load() Config`
**Fields:**
```go
Port                  int       // env: WEVIBE_HUB_PORT, default: 4440
DatabaseURL           string    // env: DATABASE_URL
QdrantAddr            string    // env: QDRANT_ADDR, default: "localhost:6333"
QdrantAPIKey         string    // env: QDRANT_API_KEY (required, min 32 chars)
OllamaURL             string    // env: OLLAMA_URL, default: "http://localhost:11434"
StripeSecretKey       string    // env: STRIPE_SECRET_KEY
S3Bucket              string    // env: WEVIBE_S3_BUCKET, default: "wevibe-memories"
NodePrivkey           string    // env: HUB_NODE_PRIVKEY
ChainGRPCURL          string    // env: WEVIBE_CHAIN_GRPC_URL
ChainRPCURL           string    // env: WEVIBE_CHAIN_RPC_URL (optional in Phase 1; required for Sprint 23 WebSocket)
ChainID               string    // env: WEVIBE_CHAIN_ID
ChainSubmitterMnemonic string // env: WEVIBE_CHAIN_SUBMITTER_MNEMONIC
ChainEnabled          bool      // env: WEVIBE_CHAIN_ENABLED
```
**Known issues:** None
**CO-219:** Hub panics at startup if `QDRANT_API_KEY` is missing or < 32 chars.

---

### Handler Layer

#### `internal/api/handlers/pool.go`
**Role:** Package-level dependency holders for all handlers
**Exports:**
```go
var pool *pgxpool.Pool
var qdrantClient *retrieval.QdrantClient
var nodePrivkeyHex string
var chainClient *chain.GrpcClient
var umbralService *umbral.Service        // CO-218 — PRE sidecar service
func SetPool(p *pgxpool.Pool)
func SetQdrantClient(c *retrieval.QdrantClient)
func SetNodePrivkey(key string)
func SetChainClient(c *chain.GrpcClient)
func SetUmbralService(s *umbral.Service) // CO-218
```
**Known issues:** None

#### `internal/api/handlers/health.go`
**Exports:** `Health(w, r)` — returns JSON `{"status":"ok","timestamp":...,"version":"0.2.0","db":"connected|disconnected"}`
**Known issues:** None

#### `internal/api/handlers/orgs.go`
**Exports:**
```go
func notImplemented(w, name)     // utility — still used by SessionLookup, GetReceipts
func CreateOrg(w, r)            // POST /v1/orgs — sig verified, leader-only; persists epoch umbral_pk and relays MsgRegisterOrg to chain
func GetOrg(w, r)               // GET /v1/orgs/{orgID}
func RotateEpoch(w, r)          // POST — sig verified, leader-only; persists epoch umbral_pk
func GetEpochManifest(w, r)     // GET — supports epochID="current" via -1; includes umbral_pk
```
**Known issues:** None

#### `internal/api/handlers/members.go`
**Exports:**
```go
func InviteMember(w, r)     // POST — sig verified, leader-only
func GetMember(w, r)        // GET — no auth (follows GET pattern)
func RegisterPreKey(w, r)   // POST /v1/orgs/{orgID}/members/{pubkey}/pre-key (CO-221)
func GetPreKey(w, r)        // GET /v1/orgs/{orgID}/members/{pubkey}/pre-key (CO-221)
func RemoveMember(w, r)     // DELETE — sig verified, leader-only
func GetKeyEnvelope(w, r)   // GET — reads from key_envelopes table, auth required
func ListMembers(w, r)      // GET — no auth
func GetMemberOrgs(w, r)    // GET /v1/members/{pubkey}/orgs — lists all orgs a member belongs to
func RegisterDelegateKey(w, r) // POST /v1/orgs/{orgID}/members/delegate-key (CO-214)
```
**Known issues:** None

#### `internal/api/handlers/profile.go` (CO-247)
**Exports:**
```go
func GetProfile(w, r)    // GET /v1/profile/{wallet} — public profile endpoint
```
**Known issues:** None

#### `internal/api/handlers/moderation.go`
**Exports:**
```go
func SubmitMemory(w, r)        // POST — calls moderation.SubmitToQueue(), handles rotation buffering
func GetPendingQueue(w, r)     // GET — moderator pubkey from auth header
func VoteOnSubmission(w, r)    // POST — quorum vote for required_approvals > 1
func ApproveSubmission(w, r)   // POST — direct approve path for single-approval orgs
func DenySubmission(w, r)      // POST — calls moderation.DenySubmission() with reason
```
**Known issues:** None

#### `internal/api/handlers/billing.go`
**Exports:**
```go
func TopUpCredits(w, r)   // POST /v1/billing/topup — reads {org_id, amount, signed_by}
func GetOrgCredits(w, r)  // GET /v1/orgs/{orgID}/credits — returns balance + transactions
```
**Known issues:**
- `TopUpCredits` does NOT verify signature — anyone can top up any org (low risk: adds credits, doesn't remove them)
- No actual Stripe integration — `TopUpCredits` is manual credit injection, not payment processing
- `GetOrgCredits` does not verify caller is a member of the org — balance visible to anyone who knows orgID

#### `internal/api/handlers/retrieval.go`
**Exports:**
```go
func QueryMemories(w, r)       // POST — keyword + vector query, creates receipt, deducts credit, returns PRE payload from PostgreSQL
func ListMemories(w, r)        // GET — scroll through approved memories with pagination (keywords/confidence/state from Qdrant payload)
func SessionLookup(w, r)       // STUB — returns 501
func RejectMemory(w, r)       // POST — increments rejection count in Qdrant
func GetMemory(w, r)          // GET — retrieves a single memory by CID
func GetReceipts(w, r)         // STUB — returns 501
func extractCIDs(results)      // helper
```
**Known issues:**
- Two stubs remain: `SessionLookup`, `GetReceipts`

**CO-224 retrieval behavior:**
- `QueryMemories` passes `include_dormant` to retrieval layer (default false)
- Query ranking combines vector score, keyword overlap boost, and confidence weighting
- Lifecycle state filter always excludes `ARCHIVED`; excludes `DORMANT` unless explicitly included

#### `internal/api/handlers/dashboard.go`
**Exports:**
```go
func RegisterDashboardKey(w, r)   // POST — leader registers a dashboard pubkey for API access
func RevokeDashboardKey(w, r)    // DELETE — leader revokes a dashboard key
func IsDashboardKey(ctx, orgID, pubkey) bool  // helper — checks if pubkey is active dashboard key
```
**Known issues:** None

#### `internal/api/handlers/keywords.go`
**Exports:**
```go
func ListKeywords(w, r)      // GET — any active member can read org keyword vocabulary
func AddKeyword(w, r)       // POST — leader only
func MergeKeywords(w, r)    // PUT — leader only, merges source into target
func RenameKeyword(w, r)     // PUT — leader only, renames keyword
func DeprecateKeyword(w, r) // DELETE — leader only
```
**Known issues:** None

#### `internal/api/handlers/recovery.go`
**Exports:**
```go
func StoreRecoveryShares(w, r)  // POST — leader stores sealed Shamir shares
func GetRecoveryShare(w, r)    // GET — holder retrieves their share by holder_pubkey
```
**Known issues:** None

---

### Internal Packages

#### `internal/chain/grpc_client.go`
**Role:** gRPC connection to wevibe-chain node, signing key derivation, query client stubs
**Exports:** `New(grpcURL, chainID, mnemonic) (*Client, error)`, `Close()`, `SubmitterAddress()`

#### `internal/chain/submit.go`
**Role:** Broadcasts hub→chain relay transactions
**Exports:**
```go
func (c *GrpcClient) SubmitMemoryToChain(ctx, orgID string, mem BatchMemory) (string, error)
func (c *GrpcClient) SubmitMemoryBatch(ctx, orgID string, memories []BatchMemory) ([]string, error)
func (c *GrpcClient) SubmitServeBatch(ctx, orgID string, epoch uint64, entries []ServeEntryInput) (string, error)
func (c *GrpcClient) SubmitMemoryReport(ctx, orgID string, input ReportMemoryInput, contributorWalletAddress string) (string, error)
```
**Chain→Hub relay pattern (CO-213):** Each submit function now includes reputation increment messages:
- `SubmitMemoryToChain` broadcasts `MsgIncrementContribution` with contributor wallet (or pubkey fallback)
- `SubmitServeBatch` broadcasts `MsgIncrementServe` for each serve entry
- `SubmitMemoryReport` (ban path) broadcasts `MsgRecordBan` with contributor wallet when provided

#### `internal/chain/query.go`
**Role:** On-chain state queries for org/memory/reputation verification and retrieval metadata reconciliation
**Exports:** `IsOrgRegistered`, `GetOrgFromChain`, `GetEpochMerkleRoot`, `GetServeParams`, `GetEpochServeStats`, `GetAttestationParams`, `GetSessionAttestation`, `GetBandwidth`, `GetEmissionsParams`, `GetReputationStats`, `GetContributorProfile`, `GetMemoriesBatch`

**Memory batch parity (CO-224):** `MemoryBatchResult` now includes `ConfidenceBps` and `State` copied from chain `StoredMemoryCommitment` (`retrieval_confidence_bps`, `state`) so hub can compare and sync Qdrant payload metadata.

**Chain→Hub reputation wiring (CO-213):** `GetContributorProfile` queries the chain's x/reputation module for a contributor's on-chain profile (contribution_count, serve_count, first_seen_epoch). Nil-safe — returns nil if chain unreachable or contributor not found. Used by `retrieval.GetContributorStats` to merge chain and hub data.

#### `internal/chain/sync.go`
**Role:** Epoch metadata sync loop implementation for retrieval parity (CO-224)
**Exports:** `SyncEpochData(ctx, chainClient, qdrantClient, pool) error`

**Sync flow (CO-224):**
- Loads orgs with approved memories from PostgreSQL (`pending_submissions.status='approved'`)
- Scrolls org memory payloads from Qdrant (`cid`, `confidence_bps`, `lifecycle_state`)
- Batch-queries chain via `GetMemoriesBatch`
- Updates changed Qdrant payloads via `retrieval.UpdateMemoryConfidenceAndState`
- Logs sync summary (`Synced N memories across M orgs, updated K confidence/state values`)

#### `internal/chain/merkle.go`
**Role:** Binary SHA-256 Merkle tree computation over approved memory hashes
**Exports:** `ComputeMerkleRoot(leaves [][]byte) string`, `HashContribution(content []byte) []byte`

---

### Business Logic

#### `internal/orgs/orgs.go`
**Exports:**
```go
func CreateOrg(ctx, pool, req) (*OrgInfo, error)
func GetOrg(ctx, pool, orgID) (*OrgInfo, error)
func SetRotationPending(ctx, pool, orgID) error
func ClearRotationPending(ctx, pool, orgID) error
func IsRotationPending(ctx, pool, orgID) (bool, error)
func GetRotationPendingSince(ctx, pool, orgID) (*time.Time, error)
func BufferSubmission(ctx, pool, orgID, req) error
func FinalizeRotationBuffer(ctx, pool, orgID, newEpochID int) (int, error)
func RotateEpoch(ctx, pool, orgID, req) error
func GetEpochManifest(ctx, pool, orgID, epochID) (*EpochManifestResponse, error)
func GetLeaderPubkey(ctx, pool, orgID) (string, error)
func GetCurrentEpoch(ctx, pool, orgID) (int, error)
func OrgExists(ctx, pool, orgID) (bool, error)
func EpochExists(ctx, pool, orgID, epochID) (bool, error)
```
**Key detail:** `CreateOrg` calls `billing.EnsureOrgLedger()` after commit — new orgs get a credit ledger automatically  
**Known issues:** None

#### `internal/members/members.go`
**Exports:**
```go
func InviteMember(ctx, pool, orgID, currentEpoch, req) (*MemberRecord, error)
func SetPrePubkey(ctx, pool, orgID, pubkey string, prePubkey []byte) error   // CO-221
func GetPrePubkey(ctx, pool, orgID, pubkey string) ([]byte, error)            // CO-221
func GetMember(ctx, pool, orgID, pubkey) (*MemberRecord, error)
func RemoveMember(ctx, pool, orgID, pubkey, currentEpoch) error
func ListMembers(ctx, pool, orgID) ([]MemberRecord, error)
func VerifyMemberAccess(ctx, pool, orgID, pubkey, requestedEpoch) (bool, error)
func IsLeader(ctx, pool, orgID, pubkey) (bool, error)
func ListOrgsForMember(ctx, pool, pubkey) ([]MemberOrgEntry, error)
func RegisterDelegateKey(ctx, pool, orgID, req) error         // CO-214
func GetDelegateKey(ctx, pool, orgID, delegateAddress) (*protocol.DelegateKeyRecord, error) // CO-214
func ResolveDelegateToWallet(ctx, pool, delegateAddress) (walletAddress, orgID string, err error) // CO-214
func RevokeDelegateKey(ctx, pool, orgID, walletAddress) error  // CO-214
```
**Known issues:** None

#### `internal/moderation/moderation.go`
**Exports:**
```go
func SubmitToQueue(ctx, pool, req) error
  // verifies: Ed25519 sig over submission_hash bytes, SHA256(ciphertext||wrapped_dek) == submission_hash
  // stores ciphertext in ciphertext_hex column (encrypted, opaque)

func GetPendingQueue(ctx, pool, orgID, moderatorPubkey) ([]PendingQueueItem, error)
  // checks role: must be "moderator" or "leader"

func ApproveSubmission(ctx, pool, qdrantClient, orgID, submissionHash, moderatorPubkey, req) error
  // updates status, writes audit log, indexes in Qdrant
  // requires vector (768 dim), indexes keywords in org_keywords + memory_keywords

func DenySubmission(ctx, pool, orgID, submissionHash, moderatorPubkey, reason) error
  // updates status + denial_reason, checks moderator role
```
**Known issues:** None

#### `internal/retrieval/retrieval.go`
**Exports:**
```go
// Client
func NewQdrantClient(addr string, apiKey string) (*QdrantClient, error)  // strips http:// prefix, requires apiKey
func (c *QdrantClient) Close() error
func (c *QdrantClient) EnsureCollection(ctx, vectorSize) error
func (c *QdrantClient) UpsertPoint(ctx, entry) error  // injects Gaussian noise (σ=0.1) before storage
func (c *QdrantClient) QueryPoints(ctx, orgID, epochs, vector, keywordWeights, embeddingModelID, limit, includeDormant) ([]MemoryResult, bool, error)
func (c *QdrantClient) CountPoints(ctx) (int64, error)

// Package-level wrappers
func AddToIndex(ctx, client, entry) error
func EnsureCollection(ctx, client) error
func QueryByKeywords(ctx, client, orgID, accessibleEpochs, keywordWeights, vector, embeddingModelID, limit, includeDormant) ([]MemoryResult, bool, error)
func ScrollOrgMemoryPayloads(ctx, client, orgID) ([]OrgMemoryPayload, error)
func ScrollApprovedMemories(ctx, client, orgID, limit, offset) ([]MemoryResult, string, error)
func UpdateMemoryKeywords(ctx, client, orgID, oldKeywords, newKeyword) error
func UpdateMemoryConfidenceAndState(ctx, client, orgID, memoryCID, confidenceBps, lifecycleState) error
```
**Constants:** `EMBED_DIM = 768`, `contestedThreshold = 0.20`
**Key detail:** `UpsertPoint` applies Gaussian noise (σ=0.1 × L2 norm) before storing vectors. `QueryPoints` performs vector search, then applies keyword-overlap boost and confidence weighting before final ranking.
**Noise injection:** `injectGaussianNoise(vector, sigma)` — adds calibrated Gaussian noise at storage time only
**Lifecycle filtering (CO-224):** `ARCHIVED` is always excluded; `DORMANT` is excluded unless `includeDormant=true`
**Qdrant payload fields:** `cid`, `org_id`, `epoch_id`, `content_flags`, `keywords`, `confidence_bps`, `lifecycle_state`, `embedding_model_id`, `embedding_schema_version`, `vector_dim`
**Known issues:** None

#### `internal/retrieval/stats.go`
**Role:** Contributor statistics aggregation from hub DB and chain
**Exports:**
```go
type ChainQuerier interface {
    GetContributorProfile(ctx, contributorID string, epoch uint64) (*types.StoredContributorProfile, error)
}
func GetAcceptanceCount(ctx, pool, orgID, memoryCID string) (int, error)
func GetContributorStats(ctx, pool, chainClient ChainQuerier, orgID, contributorPubkey string) (*protocol.ContributorStats, error)
```
**Chain→Hub stats merge (CO-213):** `GetContributorStats` merges hub and chain data:
- Tries wallet address lookup first for chain queries, falls back to Ed25519 pubkey
- `contributions`: from chain `contribution_count` if available, else from hub `pending_submissions` count
- `serve_count`: from chain `serve_count`
- `account_age_days`: from chain `first_seen_timestamp` if available, else from hub `joined_at`
- `reports_upheld`/`false_reports_against`: always from hub (hub-only per CO-211)
**Known issues:** None

#### `internal/billing/billing.go`
**Exports:**
```go
const QueryCost = int64(1)
func EnsureOrgLedger(ctx, pool, orgID) error              // INSERT ON CONFLICT DO NOTHING
func GetBalance(ctx, pool, orgID) (int64, error)
func TopUp(ctx, pool, orgID, actor, amount) error         // transactional: update balance + record txn
func DeductQueryCredit(ctx, pool, orgID, receiptID) error  // transactional: decrement + record txn
func GetTransactions(ctx, pool, orgID, limit) ([]Transaction, error)
```
**Key detail:** `org_credits.balance` has `CHECK (balance >= 0)` at DB level — solvency enforced  
**Known issues:** None

#### `internal/receipts/receipts.go`
**Exports:**
```go
func CreateReceipt(ctx, pool, nodePrivkeyHex, orgID, billingEpoch, accessEpochs, agentPubkey, queryPayload, resultCIDs, agentSigHex) (*UsageReceipt, error)
```
**Key detail:** Signs with Ed25519. Falls back to all-zeros key if `nodePrivkeyHex` is empty.  
**Known issues:** None

#### `internal/verify/sig.go`
**Exports:**
```go
func RequestSignature(pubkeyHex, sigHex string, message []byte) error
```
**Key detail:** Standard Ed25519 verify. Accepts 32-byte pubkey hex + 64-byte sig hex. Message is raw bytes (typically the canonical message).
**Known issues:** None

#### `internal/verify/wallet_sig.go`
**Role:** secp256k1 wallet signature verification for leader-gated actions (CO-233)
**Exports:**
```go
func VerifyWalletSignature(walletAddress, signature, message []byte) error
```
**Key detail:** Verifies signatures produced by Cosmos wallet `signArbitrary` (EIP-712 structured data or plain bytes). Used for: report chain commitment (`POST /v1/orgs/{orgID}/reports/{reportID}/commit`), security config changes (`PATCH /v1/orgs/{orgID}/config` for `required_approvals` and `report_vote_threshold`). Canonical message format: `{action}|{orgID}|{field1}|{field2}|...`
**Known issues:** None

#### `internal/verify/canonical.go`
**Exports:**
```go
func CreateOrgMessage(orgID, leaderPubkey, leaderX25519Pubkey, orgName, domain, encEnvelope, searchEnvelope, modEnvelope, pkMod string, feeModel map[string]any) []byte
func InviteMemberMessage(orgID, pubkey, x25519Pubkey, role, signedBy, encEnvelope, searchEnvelope, modEnvelope string) []byte
func RotateEpochMessage(orgID, newPkMod, signedBy string, envelopes []protocol.MemberEnvelopePair) []byte
func RemoveMemberMessage(orgID, pubkey, signedBy string) []byte
func ApproveSubmissionMessage(orgID, submissionHash string, epochID int32, approvedCID, wrappedDekEnc, signedBy string, keywords []protocol.KeywordWithWeight) []byte
func DenySubmissionMessage(orgID, submissionHash, reason, signedBy string) []byte
func feeModelHash(feeModel map[string]any) string
func envelopesHash(envelopes []protocol.MemberEnvelopePair) string
func keywordsHash(keywords []protocol.KeywordWithWeight) string
```
**Known issues:** None

#### `internal/embed/embed.go`
**Exports:**
```go
func GetEmbedding(ctx, ollamaURL, text) ([]float32, error)
```
**Key detail:** Calls Ollama `/api/embeddings` with `nomic-embed-text`. Returns zero vector on failure (graceful fallback).  
**Constants:** `EMBED_DIM = 768`  
**Known issues:** None

#### `internal/envelopes/envelopes.go`
**Exports:**
```go
type Envelope struct { OrgID, Pubkey, EpochID int; EncEnvelope, SearchEnvelope string; ModEnvelope *string }
func Store(ctx, pool, orgID, pubkey, epochID, encEnv, searchEnv, modEnv) error
func Get(ctx, pool, orgID, pubkey) (*Envelope, error)
func BatchReplace(ctx, pool, orgID, epochID, pairs) error
func Delete(ctx, pool, orgID, pubkey) error
```
**Known issues:** None

#### `internal/auth/header.go`
**Exports:**
```go
type SignedTimestampAuth struct { Pubkey, Timestamp, Signature string }
var ErrMissingHeader, ErrInvalidScheme, ErrMalformedAuth
func ParseWeVibeSigned(r *http.Request) (*SignedTimestampAuth, error)
```
**Key detail:** Parses `WeVibe-Signed pubkey=...,timestamp=...,signature=...` header  
**Known issues:** None

#### `internal/db/db.go`
**Exports:**
```go
func NewPool(ctx, connStr) (*pgxpool.Pool, error)
func RunMigrations(ctx, pool) error  // creates org_keywords, memory_keywords tables
```
**Known issues:** None

---

### Protocol Types

#### `internal/protocol/types.go`
**All types:**
```
OrgInfo                  — org public info (org_id, org_name, domain, leader_pubkey, current_epoch, egress_mode, allowed_providers, status, rotation_status, created_at)
CreateOrgRequest         — org_id, leader_pubkey, leader_x25519_pubkey, org_name, domain, fee_model, pk_mod, signature, enc_envelope, search_envelope, mod_envelope (+ server-side umbral_pk storage field)
RotateEpochRequest       — new_pk_mod, signed_by, signature, envelopes []MemberEnvelopePair (+ server-side umbral_pk storage field)
EpochManifestResponse    — org_id, epoch_id, pk_mod, umbral_pk, signed_by, signature, created_at
MemberRecord             — org_id, pubkey, x25519_pubkey, role, join_epoch, history_access_from_epoch, authorized_until_epoch, active, joined_at
InviteMemberRequest     — pubkey, x25519_pubkey, pre_pubkey, role, signed_by, signature, enc_envelope, search_envelope, mod_envelope, epoch_sk
RemoveMemberRequest      — signed_by, signature
KeyEnvelopeResponse      — org_id, epoch_id, enc_envelope, search_envelope, mod_envelope *string
MemberEnvelopePair       — pubkey, enc_envelope, search_envelope, mod_envelope *string
SubmitMemoryRequest      — org_id, epoch_id, ciphertext, wrapped_dek_mod, submission_hash, contributor_pubkey, contributor_sig, stack_hint
SubmitMemoryResponse     — submission_hash, status
PendingQueueItem         — submission_hash, org_id, epoch_id, contributor_pubkey, ciphertext_hex, wrapped_dek_mod, stack_hint, created_at, status
KeywordWithWeight        — keyword string, weight float64
KeywordMatchDetail       — keyword, query_weight, memory_weight, product
ScoringBreakdown         — keyword_score, vector_score, gamma, delta, capped_boost, combined_score, keyword_matches, unmatched_query
ApproveRequest           — epoch_id, approved_cid, umbral_capsule, umbral_ciphertext, content_flags, keywords, keyword_weights, vector, embedding_model_id, embedding_schema_version, vector_dim, moderator_sig, signed_by
DenyRequest              — reason, signed_by, signature
QueryRequest             — org_id, agent_pubkey, pre_pubkey (CO-218), keyword_weights, vector, embedding_model_id, embedding_schema_version, limit, include_dormant (CO-224), agent_sig
MemoryResult             — cid, org_id, epoch_id, confidence_bps (CO-224), lifecycle_state (CO-224), wrapped_dek_enc, umbral_ciphertext (CO-221), cfrag (CO-218), capsule (CO-218), content_flags, retrieval_count, acceptance_count, contributor_stats, keywords, scoring_breakdown
QueryResponse           — results, contested, receipt_id, requires_reencryption (CO-218)
RejectRequest            — cid, org_id, reason, agent_pubkey, signature
RegisterPreKeyRequest    — pre_pubkey
MemberPreKeyResponse     — pre_pubkey
IndexEntry               — cid, org_id, epoch_id, keywords, keyword_weights, content_flags, vector, confidence_bps (CO-224), lifecycle_state (CO-224), embedding_model_id, embedding_schema_version, vector_dim
MemberOrgEntry           — org_id, org_name, role, current_epoch, history_access_from_epoch, egress_mode, allowed_providers, mod_pubkey
MemberOrgsResponse       — orgs []MemberOrgEntry
UsageReceipt             — receipt_id, org_id, billing_epoch, access_epochs, agent_pubkey, query_commitment, result_commitment, agent_signature, node_signature
StoreRecoverySharesRequest — shares, signed_by, signature
RecoveryShareEntry       — share_index, holder_pubkey, sealed_share
RecoveryShareResponse    — org_id, share_index, sealed_share
RegisterDashboardKeyRequest — pubkey, label, signed_by, signature
DashboardKeyRecord       — org_id, pubkey, label, registered_by, active, created_at
RegisterDelegateKeyRequest — wallet_address, delegate_address, delegate_pubkey, grant_tx_hash, grant_expiration, signed_by, signature (CO-214)
DelegateKeyRecord        — wallet_address, delegate_address, org_id, delegate_pubkey, grant_tx_hash, grant_expiration, active, created_at (CO-214)
```

---

### Database Schema

#### `db/schema.sql`
**Tables:**
```
orgs                  — PK: org_id. Includes stripe_customer_id, stripe_subscription_id, egress_mode CHECK, status CHECK, rotation_status CHECK, rotation_pending_since, chain_registered, trial_days (CO-266)
members               — PK: (org_id, pubkey). FK: orgs. Role CHECK (leader|moderator|member). Includes `pre_pubkey BYTEA`, `is_trial BOOLEAN`, `trial_expires_at TIMESTAMPTZ` (CO-266). ON DELETE CASCADE.
epoch_manifests       — PK: (org_id, epoch_id). FK: orgs. Includes `umbral_pk BYTEA`. ON DELETE CASCADE.
pending_submissions   — PK: submission_hash. FK: orgs. Includes `umbral_capsule BYTEA`, `umbral_ciphertext BYTEA`. Status CHECK (pending|approved|denied|escalated).
rotation_buffer       — PK: buffer_id (gen_random_uuid). FK: orgs. Submissions buffered during rotation_pending state.
usage_receipts        — PK: receipt_id (gen_random_uuid). FK: orgs.
audit_log             — PK: id (BIGSERIAL). FK: orgs.
org_credits           — PK: org_id. FK: orgs ON DELETE CASCADE. balance CHECK (>= 0).
credit_transactions   — PK: txn_id (BIGSERIAL). FK: orgs.
key_envelopes         — PK: (org_id, pubkey). Stores enc/search/mod envelopes per member.
recovery_shares       — PK: (org_id, share_index). Stores sealed Shamir shares.
dashboard_keys        — PK: (org_id, pubkey). Authorized dashboard identities per org.
delegate_keys         — PK: (org_id, delegate_address). UNIQUE: (org_id, wallet_address). Maps wallet to delegate key (CO-214)
org_keywords          — PK: id. UNIQUE: (org_id, keyword). Created via RunMigrations.
memory_keywords      — PK: (memory_cid, keyword). FK: (org_id, keyword) REFERENCES org_keywords.
```
**Indexes:** `idx_orgs_leader`, `idx_orgs_status`, `idx_members_active`, `idx_members_pubkey`, `idx_members_pubkey_active`, `idx_pending_org_status`, `idx_pending_contributor`, `idx_receipts_org_epoch`, `idx_audit_org_epoch`, `idx_credit_txn_org`, `idx_envelopes_org`, `idx_recovery_shares_holder`, `idx_dashboard_keys_pubkey`, `idx_delegate_keys_address`, `idx_delegate_keys_wallet`, `idx_org_keywords_org` (WHERE NOT deprecated), `idx_memory_keywords_keyword`

---

### Test Files Summary

| Test file | Tests | Requires |
|---|---|---|
| `handlers/health_test.go` | `TestHealthReturns200` | Nothing (in-memory) |
| `billing/billing_test.go` | 5 tests (EnsureOrgLedger, TopUp, Deduct, InsufficientBalance, GetTransactions) | DATABASE_URL |
| `moderation/moderation_test.go` | 5 tests (SubmitVerifySig, SubmitVerifyHash, GetPendingQueueRequiresMod, ApproveUpdatesStatus, DenyRecordsReason, HubNeverStoresPlaintext) | DATABASE_URL |
| `retrieval/retrieval_test.go` | 6 tests (AddToIndex, QueryByKeywords, AddAndQueryRoundtrip, NewQdrantClient_AddressParsing, Constants, ContestedThreshold) | Qdrant on localhost:6333 |
| `orgs/orgs_test.go` | 6 tests (CreateOrg_GetOrg, GetOrg_NotFound, OrgExists, GetLeaderPubkey, GetCurrentEpoch, FullOrgLifecycle) | DATABASE_URL |
| `members/members_test.go` | 7 tests (InviteMember_GetMember, GetMember_NotFound, RemoveMember, ListMembers, VerifyMemberAccess, IsLeader, FullMemberLifecycle) | DATABASE_URL |
| `receipts/receipts_test.go` | 1 test (ReceiptCreation) | DATABASE_URL |
| `verify/sig_test.go` | 5 tests (Valid, Tampered, InvalidPubkey, InvalidSig, WrongPubkey) | Nothing (in-memory) |
| `verify/canonical_test.go` | (not read) | DATABASE_URL |
| `embed/embed_test.go` | 2 tests (Fallback, Dim) | Nothing (in-memory) |
| `api/handlers/moderation_test.go` | (not read) | DATABASE_URL |
| `api/handlers/member_orgs_test.go` | (not read) | DATABASE_URL |
| `auth/header_test.go` | (not read) | Nothing (in-memory) |

---

## wevibe-chain — Cosmos SDK Appchain

**Module:** `github.com/wevibe-network/wevibe-chain`
**Go version:** 1.25.9
**Default ports:** 26657 (CometBFT RPC), 9090 (gRPC), 1317 (REST)
**Chain ID:** configurable via `wevibed init --chain-id`

### Entry Point

#### `cmd/wevibed/main.go`
**Role:** Cosmos SDK application CLI — init, start, keys, genesis, query, tx subcommands

### App Wiring

#### `app/app.go`
**Role:** Cosmos SDK BaseApp — registers all keepers, mounts KV stores, wires gRPC/RPC
**Key behavior:**
- 7 custom WeVibe modules + standard SDK modules (staking, auth, bank, gov, slashing, distribution, mint, epochs)
- Module ordering: InitGenesis, ExportGenesis, EndBlockers all explicitly set
- maccPerms includes operator module with Burner permission
- Chain foundation pins `github.com/cosmos/cosmos-sdk v0.53.5` and `github.com/cometbft/cometbft v0.38.20` (see `DECISIONS.md` D-S29-SDK-V053)

### Custom Modules (7)

| Module | Keeper Path | Proto Path | Tests | Purpose |
|--------|------------|-----------|-------|---------|
| x/attestation | x/attestation/keeper/ | proto/wevibe/attestation/v1/ | keeper + integration | Merkle root submission |
| x/bandwidth | x/bandwidth/keeper/ | proto/wevibe/bandwidth/v1/ | keeper + integration | Bandwidth throttling |
| x/emissions | x/emissions/keeper/ | proto/wevibe/emissions/v1/ | keeper | Daily pool, work scores |
| x/memory | x/memory/keeper/ | proto/wevibe/memory/v1/ | keeper + integration | Memory commitments |
| x/org | x/org/keeper/ | proto/wevibe/org/v1/ | keeper + integration | Org registration, membership |
| x/reputation | x/reputation/keeper/ | proto/wevibe/reputation/v1/ | keeper | Contributor reputation |
| x/serve | x/serve/keeper/ | proto/wevibe/serve/v1/ | keeper + integration | Serve attestations |

### Module Structure Pattern (all 8 modules follow this)

```
x/{module}/
├── keeper/
│   ├── keeper.go           # Keeper struct, state access, business logic
│   ├── msg_server.go       # MsgServer implementation
│   ├── grpc_query.go       # gRPC query handlers
│   ├── epoch_hooks.go      # Epoch hooks (emissions, challenge, serving only)
│   └── keeper_test.go      # Keeper tests
├── module/
│   ├── module.go           # AppModule, RegisterServices, EndBlocker
│   └── autocli.go          # AutoCLI command specs (partial)
└── types/
    ├── keys.go             # Store keys, module name
    ├── params.go           # DefaultParams, Validate (if hand-written)
    ├── expected_keepers.go # Inter-module keeper interfaces
    ├── errors.go           # Module-specific errors
    ├── codec.go            # Interface registration
    ├── tx.pb.go            # Proto-generated Msg types
    ├── query.pb.go         # Proto-generated Query types
    ├── types.pb.go         # Proto-generated domain types
    ├── state.pb.go         # Proto-generated state types
    └── params.pb.go        # Proto-generated Params
```

### Tests

| Test file | Tests | Requires |
|---|---|---|
| tests/integration/wevibe_txs_test.go | 9 tx pipeline tests | In-memory MemDB app |
| tests/integration/wevibe_queries_test.go | 11 gRPC query tests | In-memory MemDB app |
| x/*/keeper/keeper_test.go | Per-module keeper tests | In-memory |

### Scripts

| Script | Purpose |
|---|---|
| scripts/init-chain.sh | Idempotent genesis init with wevibe_epoch config |
| scripts/smoke-test.sh | RPC health + block production verification |

### Docker

- Dockerfile: multi-stage (golang:1.26-bookworm → debian:bookworm-slim)
- docker-compose.yml: single-validator with named volume
- Makefile: localnet-build/start/stop/reset/logs

---

## wevibe-server/wevibe-dashboard — Next.js UI

**Language:** TypeScript/React (Next.js)  
**Purpose:** Moderation + organization management dashboard

### `app/` — Next.js App Router

```
app/
├── layout.tsx                    # Root layout
├── page.tsx                      # Landing/redirect page
├── (auth)/
│   └── login/page.tsx           # Login page
├── (dashboard)/
│   ├── layout.tsx                # Dashboard layout with sidebar/topbar + OrgProvider
│   ├── billing/page.tsx
│   ├── health/page.tsx
│   ├── members/page.tsx
│   ├── memories/page.tsx
│   ├── moderation/page.tsx
│   ├── profile/page.tsx          # Own profile (CO-247)
│   ├── reports/page.tsx
│   ├── sessions/page.tsx          # Sessions with per-memory org destination (CO-247)
│   ├── settings/page.tsx
│   └── chain-submit/page.tsx
└── u/
    └── [wallet]/
        └── page.tsx              # Public profile (CO-247) — standalone, no sidebar
```

### `components/`

```
components/
├── layout/
│   ├── sidebar.tsx
│   ├── topbar.tsx
│   └── org-switcher.tsx          # Multi-org dropdown switcher (CO-247)
└── ui/
    ├── badge.tsx
    ├── button.tsx
    └── card.tsx
```

### `lib/`

```
lib/
├── types.ts          # TypeScript type definitions
├── hub-client.ts     # API client for wevibe-hub (includes getProfile since CO-247)
├── wevibe-auth.ts      # WeVibe signed auth utilities
├── wevibe-signing.ts   # Request signing utilities (includes registerDelegateKeyCanonical since CO-214)
├── mcp-client.ts     # MCP client for wevibe-mcp
├── wallet-connect.ts # Wallet connection (includes getOfflineSigner since CO-214)
├── delegate-key.ts   # secp256k1 delegate key generation and encrypted storage (CO-214)
├── chain-client.ts   # CosmJS SigningStargateClient wrapper for MsgGrant/MsgExec (CO-214)
├── delegation.ts     # Delegation orchestrator: setupDelegation, revokeDelegation (CO-214)
├── org-context.tsx   # Multi-org context provider + useOrgContext hook (CO-247)
└── settings.ts      # Dashboard settings persistence
```

### `e2e/` — Playwright tests

```
e2e/
├── global-setup.ts
├── fixtures.ts
├── author-memory.spec.ts
├── connection.spec.ts
├── mcp-tools.test.ts
└── navigation.spec.ts
```

### `package.json` — Key dependencies
- `next` (framework)
- `react`, `react-dom`
- `@radix-ui/*` (UI primitives)
- `tailwindcss` (styling)
- `@playwright/test` (e2e testing)
- `@cosmjs/stargate`, `@cosmjs/proto-signing`, `@cosmjs/amino`, `@cosmjs/crypto`, `@cosmjs/encoding`, `cosmjs-types` (CO-214: Cosmos chain signing for delegate key MsgGrant)

---

## WeVibe/wevibe-mcp — TypeScript MCP Client + HTTP API Server

**Language:** TypeScript
**Purpose:** MCP client for AI agents to interact with WeVibe Network. Also serves an HTTP API on `127.0.0.1:4450` for the OpenCode plugin (CO-244).

### `src/` — Main source

```
src/
├── server.ts              # MCP server entry point
├── dashboard-server.ts    # Dashboard integration server
├── types.ts              # TypeScript types
├── session.ts            # Session management
├── crypto.ts            # Cryptographic utilities
├── auth.ts              # Authentication + PRE identity lifecycle (CO-222)
├── contribution.ts      # Memory contribution flow
├── extraction.ts        # Content extraction (Pass 1: memory extraction, Pass 2: vocab-constrained keyword classification with new type exports — CO-236)
├── guard.ts             # Prompt injection detection
├── blacklist.ts         # Blacklist handling
├── llm.ts               # LLM interface
├── llm-ollama.ts        # Ollama LLM provider
├── llm-sampling.ts      # Sampling provider
├── embedding.ts         # Embedding generation
├── moderation.ts        # Moderation handling
├── sidecar.ts           # Umbral sidecar subprocess helper for encrypt/decrypt-reencrypted
├── vault.ts             # Vault management
├── pending-vault.ts     # Pending vault operations
├── key-store.ts         # Key storage
├── org-client.ts        # Org API client + PRE registration/query/decrypt/invite wiring (CO-222) + getOrgKeywords() for vocab-constrained extraction (CO-236)
├── artifact-policy.ts   # Artifact policy enforcement
├── artifact-extract.ts  # Artifact extraction
├── artifact-transform.ts # Artifact transformation
├── deserialize.ts        # Deserialization (PRE retrieval fields)
├── recovery.ts          # Recovery operations
├── canonical.ts         # Canonical message generation
├── buffer.ts            # Buffer operations
├── trust-panel.ts       # Trust panel formatting for contributor stats
├── retrieve-cli.ts      # Importable module for PRE retrieval (query → decrypt → sanitize → artifact policy → trust panel). Exports `retrieve(input: RetrieveInput): Promise<Output>`. No CLI wrapper.
├── http-server.ts       # HTTP API server on 127.0.0.1:4450 (CO-244)
└── admin.ts             # Admin operations (invite requires PRE pubkey + epoch SK)
```

### `wevibe_mcp/` — Python MCP implementation

```
wevibe_mcp/
├── __init__.py
├── server.py            # Python MCP server
├── server_legacy.py     # Legacy server
├── contribution.py
├── org_client.py
├── wallet.py
├── buffer.py
├── session.py
├── sanitize.py
├── blacklist.py
└── manifest.py
```

### `tests/`

```
tests/
├── integration/
│   ├── e2e-flow.test.ts
│   └── capstone.test.ts
├── production/
│   ├── hub-resilience.test.ts
│   ├── sampling-provider.test.ts
│   └── embedding-quality.test.ts
├── security/
│   └── (various)
├── wasm-crypto.test.ts
├── contribution.test.ts
├── guard.test.ts
├── llm.test.ts
├── moderation.test.ts
├── moderation-approval.test.ts
├── recovery.test.ts
├── rotation.test.ts
├── session.test.ts
├── vault.test.ts
├── steg-scan.test.ts
├── artifact-*.test.ts
├── deserialize.test.ts
├── key-store.test.ts
├── pending-vault.test.ts
├── buffer-encrypted.test.ts
├── egress-policy.test.ts
├── extraction.test.ts
├── threshold-recovery.test.ts
└── recovery-status.test.ts
```

---

## WeVibe/wevibe-guard — Rust YARA-X Scanner

**Language:** Rust  
**Purpose:** YARA-X prompt injection scanner

### `src/`

```
src/
├── lib.rs            # Library entry
├── main.rs          # Binary entry
├── scanner.rs       # Main scanner implementation
├── flags.rs         # Flag handling
├── credentials.rs   # Credential detection
└── exfiltration.rs  # Exfiltration detection
```

### `benches/`

```
benches/
└── scan_bench.rs    # Benchmarking
```

### `tests/`

```
tests/
├── unit_tests.rs
└── fixture_compliance.rs
```

---

## WeVibe/wevibe-sdk — Rust + Python Crypto SDK

**Languages:** Rust, Python  
**Purpose:** Crypto primitives + WASM bindings

### `crates/wevibe-sdk-core/` — Core Rust library

```
crates/wevibe-sdk-core/src/
├── lib.rs           # Library entry
├── crypto.rs        # Cryptographic operations
├── identity.rs      # Identity management
├── types.rs         # Type definitions
└── errors.rs        # Error types
```

### `crates/wevibe-sdk-wasm/` — WASM bindings

```
crates/wevibe-sdk-wasm/src/
└── lib.rs           # WASM bindings
```

### `wevibe_sdk/` — Python SDK

```
wevibe_sdk/
├── __init__.py
├── crypto.py
├── identity.py
├── key_store.py
├── pending_vault.py
└── keyword_filter.py
```

---

## WeVibe/protocol — Protocol Definitions

```
protocol/
├── openapi.yaml           # OpenAPI specification
├── README.md
├── contract_test.sh
└── test_vectors/          # Protocol test vectors
```

---

## WeVibe/anchor — Solana Anchor Programs

```
anchor/wevibe-identity/
├── src/lib.rs
├── tests/
└── Cargo.toml
```

---

## benchmark/ — Enrichment Benchmark Suite

**Language:** Node.js  
**Purpose:** Tests LLM enrichment quality across scenarios

### Structure

```
benchmark/
├── package.json
├── run.sh / run-all.sh / run-full.sh
├── README.md
├── lib/
│   ├── seed.mjs         # Test data seeding
│   ├── extract.py       # Result extraction
│   └── score.py         # Scoring logic
├── scenarios/
│   ├── SCHEMA.md
│   ├── 01-nginx-upload.json
│   ├── 02-sse-reverse-proxy.json
│   ├── 03-postgres-pgbouncer.json
│   ├── 04-docker-node-build.json
│   ├── 05-redis-session-failover.json
│   ├── 06-websocket-nginx.json
│   ├── 07-k8s-probes.json
│   ├── 08-graphql-dataloader.json
│   ├── 09-cors-credentials.json
│   └── 10-rate-limit-redis.json
├── backend/
│   ├── package.json
│   └── src/index.js     # Minimal Express server
└── results/             # Timestamped results
    └── 20260412-*/
```

---

## benchmark-adversarial/ — Adversarial Retrieval Benchmark

**Language:** Node.js  
**Purpose:** Tests security of retrieval system against adversarial inputs

### Structure

```
benchmark-adversarial/
├── package.json
├── run.sh
├── eval.mjs
├── test_small.mjs
├── scenarios.jsonl
├── results/
└── run.log / run_final.log
```

---

## backend/ — Minimal Express Backend

**Language:** Node.js/Express  
**Purpose:** File upload server for benchmark

```
backend/
└── src/routes/upload.js
```

---

## Dependencies Summary

| Package | Language | Direct Deps |
|---|---|---|
| wevibe-hub | Go | chi/v5, google/uuid, pgx/v5, qdrant/go-client |
| wevibe-dashboard | TypeScript | next, react, tailwindcss, @radix-ui/*, playwright, @cosmjs/stargate, @cosmjs/proto-signing, cosmjs-types |
| wevibe-mcp | TypeScript | (many npm packages) |
| wevibe-guard | Rust | yara-x |
| wevibe-sdk | Rust | (core crypto) |
| benchmark | Node.js | express, multer |

---

## Inter-Package Relationships

```
                     ┌─────────────────────────┐
                     │   wevibe-dashboard        │
                     │   (Next.js UI)          │
                     └───────────┬─────────────┘
                                 │ HTTP API
                     ┌───────────▼─────────────┐
                     │   wevibe-hub              │◄──── gRPC ────┐
                     │   (Go API server)       │               │
                     │   Port: 4440            │          ┌────▼────┐
                     └───────────┬─────────────┘          │wevibe-chain│
                                 │                        │(Cosmos)  │
           ┌─────────────────────┼──────────────┐         │Port:9090 │
           │                     │              │         └──────────┘
     ┌─────▼─────┐        ┌─────▼─────┐ ┌──────▼──────┐
     │wevibe-mcp   │        │wevibe-sdk   │ │wevibe-guard   │
     │(TypeScript)│        │(Rust/Py)  │ │(Rust)       │
     └───────────┘        └───────────┘ └─────────────┘

      benchmark/ ──────────► wevibe-hub (tests API)
      benchmark-adversarial/ ──────────► wevibe-mcp, wevibe-sdk (tests security)
```

## Consumer Path (post-CO-260)

**Canonical consumer chain (auth layers):**
1. `plugin` calls local `wevibe-mcp` HTTP API using `Authorization: Bearer <token>` loaded from `~/.wevibe/mcp-session-token` (D-12.5a, CO-260).
2. `wevibe-mcp` validates the Bearer token, performs canonicalization, and signs outbound hub requests with delegate auth (Option beta / D-12.5).
3. `wevibe-mcp` calls `wevibe-hub` with WeVibe-Signed delegate authentication.
4. `wevibe-hub` fans out to Qdrant, `wevibe-chain`, and `wevibe-umbral`, then returns through `wevibe-mcp` back to `plugin`.

**wevibe-mcp HTTP endpoints in this consumer path (Bearer token required):**
- `/v1/health`
- `/v1/recall`
- `/v1/serves`
- `/v1/reports`

**Architecture properties:**
- Plugin holds no long-lived secret; the token rotates on MCP restart; `~/.wevibe/mcp-session-token` must be file mode `0600`.
- Plugin makes no direct `wevibe-hub` calls for consumer operations.
- `wevibe-mcp` owns canonicalization + delegate signing (Option beta / D-12.5).
- `wevibe-hub` API contract remains unchanged; only caller identity changed.

**Cross-module references:**
- `WeVibe/wevibe-mcp/docs/TOPOLOGY.md`
- `wevibe-server/wevibe-hub/docs/TOPOLOGY.md`
- `DECISIONS.md` (`D-12.5`, `D-12.5a`)

### PRE Retrieval Data Flow (CO-216, CO-218, CO-221, CO-222)

```
wevibe-chain (MemberRemoved event)
         │
         ▼ (WebSocket / polling)
wevibe-hub (event consumer)
         │
         ▼ (gRPC DeleteKFrags)
wevibe-umbral (:4460)
         │
         ▼
KFrag store updated (kfrags deleted for removed member)
```

**Chain → Hub → Sidecar flow:**
1. `MsgRemoveMember` on wevibe-chain emits `member_removed` event
2. Hub subscribes via CometBFT WebSocket, receives event
3. Hub calls `DeleteKFrags` RPC on sidecar (org_id, member_pubkey)
4. Sidecar removes all kfrags matching (org_id, *, member_pubkey_hex)
5. Member can no longer have memories re-encrypted for them

**Approval + retrieval flow (CO-221/CO-222):**
1. Moderator approval in wevibe-mcp decrypts DEK, fetches epoch manifest (`umbral_pk`), and calls sidecar `encrypt`.
2. wevibe-mcp sends `umbral_capsule` + `umbral_ciphertext` in approve payload; hub stores both in `pending_submissions`.
3. Retrieval query uses Qdrant + chain for attestation, then loads capsule/ciphertext from PostgreSQL.
4. Hub calls sidecar `ReEncrypt` with stored capsule and member PRE pubkey to obtain cfrag.
5. Hub returns `cfrag + capsule + umbral_ciphertext`; wevibe-mcp calls sidecar `decrypt-reencrypted` with local PRE secret key and epoch delegating pubkey to recover DEK locally.

**Chain metadata parity flow (CO-224):**
1. On approval, hub writes chain-mirror retrieval metadata into Qdrant payload: `keywords`, `confidence_bps=10000`, `lifecycle_state=APPROVED`.
2. Keyword rename/merge operations update PostgreSQL vocabulary tables and then call `UpdateMemoryKeywords` to keep Qdrant payload keywords synchronized.
3. Query path (`QueryPoints`) uses Qdrant payload metadata for lifecycle filtering and ranking:
   - hard exclude `ARCHIVED`
   - exclude `DORMANT` unless `include_dormant=true`
   - apply keyword overlap boost and confidence weighting on top of vector score
4. Epoch poller (`SyncEpochData`) runs on a ticker, loads approved orgs from PostgreSQL, compares Qdrant payload vs chain `GetMemoriesBatch`, and updates changed `confidence_bps`/`lifecycle_state` via `UpdateMemoryConfidenceAndState`.
5. `ScrollApprovedMemories` reads `keywords`, `confidence_bps`, and `lifecycle_state` directly from Qdrant payload (no PostgreSQL keyword join).

**wevibe-mcp PRE identity + registration flow (CO-222):**
1. `auth.ts` loads/generates PRE identity and stores secret key hex in keystore account `pre-identity-key`.
2. On startup, wevibe-mcp loads memberships and calls `POST /v1/orgs/{orgID}/members/{pubkey}/pre-key` for each org.
3. Query calls send `pre_pubkey` from cached PRE identity.
4. Invite calls send invitee `pre_pubkey` and leader-supplied `epoch_sk`; create/rotate responses persist `epoch_sk` + `epoch_pk` locally.

**Retrieval flow (CO-218/CO-221 — PRE re-encryption):**
1. Consumer posts query with `pre_pubkey` (their secp256k1 PRE public key)
2. Hub verifies membership → queries Qdrant → fetches chain attestation data
3. Hub loads `umbral_capsule` + `umbral_ciphertext` from PostgreSQL per approved memory
4. For each memory with a capsule: hub calls `ReEncrypt` on sidecar (org_id, epoch_id, member_pk, capsule)
5. Sidecar applies stored kfrag → returns cfrag
6. Hub returns cfrag + capsule + umbral_ciphertext to consumer (hub never sees plaintext DEK)
7. Consumer decrypts locally with `decrypt-reencrypted`

**Hub-owned kfrag lifecycle (CO-218, Option C):**
- Hub triggers kfrag generation on member invite (calls sidecar `GenerateKFrags` with epoch SK)
- Hub triggers kfrag deletion on member removal (calls sidecar `DeleteKFrags`)
- No chain event subscription needed for Phase 1

**Internal PRE endpoints (CO-218, was 501):**
- `POST /v1/internal/reencrypt` — calls `umbralService.ReEncryptForMember`
- `POST /v1/internal/epoch-keypair` — calls `umbralService.GenerateEpochKeyPair`
- `POST /v1/internal/orgs/{orgID}/kfrags` — calls `umbralService.RegisterMember`

**Denial Attestation Flow (CO-225):**
```
Consumer/MCP Client
       │
       │ POST /v1/orgs/{orgID}/denials
       │ { memory_cid, reason, agent_sig }
       ▼
wevibe-hub ────────────────────────────────────────────
       │                                             │
       │ 1. Verifies sig + membership                │
       │ 2. Writes to serve_events table             │
       │    (event_type='denial', reason, org_id,    │
       │     epoch_id, memory_cid)                   │
       │ 3. Optionally calls POST /v1/orgs/{orgID}/  │
       │    denials/batch-submit                     │
       ▼                                             │
wevibe-chain ◄── MsgSubmitDenialBatch ─────────────────┘
       │
       │ x/serve msg_server
       ▼
StoredDenialAttestation stored
(keyed by org_id / epoch_id / memory_hash)
       │
       │ EndBlocker hook triggers x/memory decay
       ▼
x/memory epoch close:
  - idle decay (50 bps): all memories
  - negative-signal decay (500 bps): memories with denials
  - confidence_bps == 0 → ARCHIVED state
       │
       │ x/emissions ProcessOrgPayouts reads denials
       ▼
payout_per_memory counted (not payout_per_serve)
```

**Chain module changes (CO-225):**
- `x/memory`: dual-vector decay params (idle=50bps, negative=500bps), `confidence_bps=0` → `ARCHIVED`
- `x/serve`: `MsgSubmitDenialBatch`, `StoredDenialAttestation` (keyed org/epoch/memory-hash)
- `x/emissions`: `ProcessOrgPayouts` rewrite — `payout_per_memory` replaces `payout_per_serve`, counts approved memories per contributor

**API changes requiring wevibe-mcp updates (CO-218/CO-221/CO-222):**
- `QueryRequest`: new required field `pre_pubkey` (hex, 33-byte secp256k1)
- `EpochManifestResponse`: includes `umbral_pk` (hex)
- `ApproveRequest`: `wrapped_dek_enc` replaced by `umbral_capsule` + `umbral_ciphertext`
- `MemoryResult`: PRE retrieval fields `cfrag`, `capsule`, and `umbral_ciphertext` (wevibe-mcp retrieval path consumes PRE fields only)
- `QueryResponse`: new field `requires_reencryption` ([]string) — CIDs of old-format memories
- `InviteMemberRequest`: fields `epoch_sk` + `pre_pubkey`
- `POST/GET /v1/orgs/{orgID}/members/{pubkey}/pre-key`: member PRE key registration/lookup
- `CreateOrgResponse` / `RotateEpochResponse`: new fields `epoch_sk` and `epoch_pk` (hex)

**CO-216-F2 Resolution (CO-217):** Sidecar tests added (9 tests in `tests/integration.rs`)
**CO-216-F3 Resolution (CO-217):** gRPC stubs generated from proto, hand-written types removed
**CO-216-F4 Resolution (CO-217):** SecretKeyFactory workaround verified sound — full Umbral workflow passes
