# WeVibe Topology

**Generated:** 2026-08-18 — reflects current on-disk state of the live WeVibe workspace as of this date.

---

## Scope

- This edition **SUPERSEDES** the 2026-06-14 edition of `wevibe-docs/TOPOLOGY.md`.
- It is assembled exclusively from **validated read-only audit reports** of the live
  codebase — every fact was re-derived from current source, not carried forward
  from prior documentation.
- The audited architecture is the **RECALL-PIVOT / edge-standing** model, which is
  **LIVE**: the chain stores content-free, consumer-signed events, and standing is
  computed at the edge as a pure function of those events against the anchored
  policy version — never a frozen formula.

## Ports & identity

Port and identity facts throughout follow **AGENTS.md §2.1**: host MCP `:4450`
(operator daily driver, keychain identity, never bench-touchable) vs bench MCP
`:4550` (managed service, seed-derived identity).

## Assembly

This document is organized as numbered sections `01`–`25`, all inline in this single file; section `01` (this header) introduces the rest.

# 02 · Workspace Tooling — Proto Generation

All protobuf codegen in the workspace runs inside pinned Docker images — never a host-local
`protoc` or `buf`. The `wevibe-meta/Makefile` owns the umbrella target, and every runner image
is pinned by exact tag.

## Make-managed targets

`make proto-gen` is the umbrella and fans out to exactly two subtargets —
`proto-gen-chain` and `proto-gen-umbral`:

| Subtarget | Runner image (pinned) | Proto source | Generated output | Consumer |
|---|---|---|---|---|
| `proto-gen-chain` | `ghcr.io/cosmos/proto-builder:0.18.1` | `wevibe-chain` proto trees: `proto/wevibe/*/v1` (8 modules, incl. `identity/v1`'s 4 protos — omitted from the prior TOPOLOGY.md) | `wevibe-chain/x/*/types/*.pb.go` (33 files) | wevibe-chain itself |
| `proto-gen-umbral` | `bufbuild/buf:1.34.0` | `wevibe-umbral/proto/umbral/v1/sidecar.proto` | gRPC stubs for the sidecar API | wevibe-server (hub relay) |

A note on the server-side copy: `umbralpb/sidecar.proto` inside `wevibe-server` is a
**byte-identical mirror**, not a source. The real, authoritative proto lives in
`wevibe-umbral/proto/umbral/v1`; edits happen there and the mirror tracks it.

## wevibe-protocol — the fourth generation participant

`wevibe-protocol` generates TypeScript bindings on a **separate, non-Make path** that bypasses
the umbrella procedure entirely:

`npm run regen` → `codegen/regen.sh` → `bufbuild/buf:1.34.0` → generated output in
`wevibe-protocol/js/`.

Two caveats:

- Its ts-proto remote plugin is **unpinned** (`wevibe-protocol/buf.gen.yaml:3`) — the one
  deviation from the otherwise exact-tag pinning discipline.
- It currently has **no in-workspace consumer**; it is live tooling without a downstream
  dependent inside this workspace.

## Adding a new proto tree

For trees that belong under the umbrella, the procedure is unchanged: add a Make subtarget,
add it as a dependency of `proto-gen`, and pin its runner image by exact tag.
`wevibe-protocol`'s `regen.sh` path is the sole exception to this procedure — it runs
outside the Makefile rather than through it.

# 03 — Deployment topology

> Consolidated from the WO-TOP2-DEPLOY-PORTS second-pass validation (authoritative; 2026-08-18) and the
> WO-TOPOLOGY-DEPLOY first-pass audit. Where the two disagreed, the second pass wins. All docker host
> bindings are gated by `${WEVIBE_BIND_HOST:-127.0.0.1}` (local loopback by default).

## 3.1 Docker services

Authority = `wevibe-server/docker-compose.yml` (9 services):

| Service | Container | Internal addr | Host port | Purpose | Authority |
|---|---|---|---|---|---|
| wevibe-postgres | wevibe-postgres | `:5432` | `127.0.0.1:5433` | hub PostgreSQL (schema.sql applied by hub at startup) | compose:36; image postgres:16-alpine default 5432 |
| wevibe-qdrant | wevibe-qdrant | `:6333` | `127.0.0.1:6333` | vector store (retrieval); **6334 dropped** | compose:52; image qdrant/qdrant:v1.9.0 default 6333 |
| wevibe-chain | wevibe-chain | `26656/26657/1317/9090` | `127.0.0.1:26656/26657/1317/9090` | Cosmos chain (p2p/RPC/REST/gRPC) | compose:77–80; wevibe-chain/Dockerfile:20 EXPOSE; init-chain.sh binds 0.0.0.0 |
| wevibe-hub | wevibe-hub | `:4440` | `127.0.0.1:4440` | hub API | compose:148; Dockerfile.hub:43 EXPOSE 4440; `WEVIBE_HUB_PORT=4440` (131) |
| wevibe-dashboard | wevibe-dashboard | `:3000` | `127.0.0.1:3000` | docker Next.js dashboard (MCP default → `host.docker.internal:4450`) | compose:239; Dockerfile:17,22 `ENV PORT=3000` |
| wevibe-umbral | wevibe-umbral | `:4460` | **— (none)** | Umbral PRE sidecar (hub relay); container-only, no `ports:` block | compose:95–110; Dockerfile.umbral-sidecar:21,23 `EXPOSE 4460` + `--addr 0.0.0.0:4460` |
| wevibe-mcp | wevibe-mcp | `:4450` (NOT `:579`) | `127.0.0.1:4452` (off-route) | docker MCP (file keystore); 4452 unpublished | compose:267–268; Dockerfile.wevibe-mcp:22,33; config.ts:85–87 |
| wevibe-faucet | wevibe-faucet | `:4470` | **— (none)** | token faucet, in-network only | compose:201 `LISTEN_ADDR=:4470`; no `ports:` key; Dockerfile:15,21 |
| wevibe-social-graph | wevibe-social-graph | `:4470` | `127.0.0.1:4470` | profile + contributor-stats display (RPC) | compose:180,183; Dockerfile:21 EXPOSE 4470 |

The faucet/social-graph `:4470` duplication is on the **internal** address only — legal on the bridge
network, and there is no host-level collision: the only host-published `:4470` is social-graph
(compose:183, host-mapped `127.0.0.1:4470:4470`), while the faucet has no `ports:` key and is reachable
only inside the compose network.

**Cross-wiring** (container→container and container→host edges):

- **hub → faucet**: `FAUCET_URL=http://wevibe-faucet:4470` (compose:129) — in-network only.
- **dashboard → social-graph**: `WEVIBE_SOCIAL_GRAPH_URL=http://localhost:4470` (compose:228) — via the host-mapped port.
- **dashboard → MCP**: `WEVIBE_MCP_HTTP_URL=http://host.docker.internal:4450` (compose:236, `.env.example:59`) — the docker dashboard talks to the **host** MCP (keychain identity), not the docker MCP.

Related resolution: the dashboard's `getMcpHttpUrl()` default is `:4450`, env-overridable via
`WEVIBE_MCP_HTTP_URL` (`lib/config.ts:14`, `:95–100`). A `:4550` pointing was introduced (`4506b1f`) and
reverted (`6faac93`) in docker-compose.yml + `.env.example` — `:4550` is bench-only (AGENTS.md §2.1).

## 3.2 Host services

Non-docker services, or docker containers outside the wevibe compose:

| Service | Port | Identity / role | Authority |
|---|---|---|---|
| Host MCP (operator) | `:4450` | keychain identity `05c4b8cb…`, plugin-spawned (detached `dist/server.js`), **NOT docker** | wevibe-plugin.ts:1319–1324; config.ts:85–87; opencode.json:267–276 (MCP entry disabled) |
| Bench MCP | `:4550` | seed-derived ed fp `aa2aa706` (= Walter's benchmark wallet root); managed service `bench-mcp.sh start` | bench-mcp.sh:92–93; RUNBOOK §7; lconfig.py:34–35 |
| Local LLM proxy | `:4545` | **docker container** `local-llm-proxy`, project `localllmproxy` outside the workspace (`~/Desktop/Local LLM Proxy/`) → oMLX `:8001` | `~/Desktop/Local LLM Proxy/{docker-compose.yml,config/models.yaml}`; opencode.json:104 |
| Ollama | `:11434` | embeddings-only (nomic-embed-text:v1.5, 768-d); **the sole host exception** (D-13.10) | config.ts:91; `.env.example:70` (`host.docker.internal:11434`) |
| Bench dashboard | `:7717` | docker container `wevibe-bench-dashboard` (read-only board) | wevibe-bench/dashboard/docker-compose.yml:21 |
| Bench control plane | `:7718` | host node process; sole run-starter | wevibe-bench/control/server.mjs:92,1190 |
| Dev dashboard | `:3001` | host `npx next dev -H 127.0.0.1 -p 3001` | wevibe-dashboard/package.json:6 |
| Chain RPC / REST / gRPC | `:26657` / `:1317` / `:9090` | chain container (host-mapped) | init-chain.sh:25,38,42 + compose:78–80 |
| Qdrant | `:6333` | qdrant container (host-mapped) | compose:52 |
| **Retired (must be SILENT)** | `:4451`, `:4460–:4463` | dead services, enforced silent (see §3.5) | verify-clean.sh:600 + check 13 (:630); AGENTS.md §2.1 |

## 3.3 Local inference stack

```
OpenCode — daily driver (alias "auto (Local LLM Proxy - oMLX)")
  └─ provider baseURL http://127.0.0.1:4545/v1                 [opencode.json:104]
       └─ local-llm-proxy container (0.0.0.0:4545)             [~/Desktop/Local LLM Proxy/docker-compose.yml]
            ├─ omlx backend    → host.docker.internal:8001/v1  [oMLX — 4 concurrent lanes, models.yaml:81]
            └─ lmstudio backend→ host.docker.internal:1234/v1  [LM Studio — defaultBackend, models.yaml:49]

Ollama :11434 — embeddings only (nomic-embed-text:v1.5, 768-d), host exception [Metal GPU]
```

- The daily driver is **not** single-slot: the oMLX backend allows **4 concurrent lanes**
  (`models.yaml:81`). Strict serial (`maxInFlight:1`) applies only to the bench aliases, which pin it.
- oMLX also serves `jina-embeddings` at `:8001/v1/embeddings` — a second, non-Ollama embedding arm.
- Ollama holds embedding models only; it was configured but **down** at the time of the PASS-2 audit.

## 3.4 Bench topology

A bench run composes: bench MCP `:4550` (managed service commissioned from the leader seed), hub
`:4440`, model runtime `:1234` (LM Studio API, residency/context checks), contributor dashboard
`:3001`, control plane `:7718` (sole run-starter), bench dashboard `:7717`, and a per-cell persistent
`opencode serve` on `:4096`. Port `:4450` is the **operator's real host MCP** and is NEVER a bench
component — pointing any bench piece at it mints orgs under the operator's keychain identity and fails
the later membership check.

## 3.5 Retired ports — must be SILENT

`:4451` (former dashboard-server) and `:4460–:4463` are retired and **must not be listening**. Enforced
by `wevibe-meta/scripts/verify-clean.sh` check 13 (`retired_ports=(4451 4460 4461 4462 4463)` at
verify-clean.sh:600; gate at :630) — a listener on any of them fails verify-clean. The `:4451` launchd
plist is `.disabled` and the compose/env `:4451` references were dropped in wevibe-server `7203fb5`.

## 3.6 Notes and corrected mechanics

- **Schema bootstrap**: `db/schema.sql` is applied by the **hub at startup**
  (`cmd/wevibe-hub/main.go:84` → `internal/db/migrate.go:22–65`), **not** at Postgres container init —
  the postgres service has no `docker-entrypoint-initdb.d` mount (compose:37–38). D-13.10's "container
  init" wording is stale.
- **Empirical Replay Mode (CO-034) has LANDED**: `docker-compose.fast.yml` overlays
  `WEVIBE_EPOCH_DURATION_SECONDS=2` onto the chain's default 60 s epoch; `dogfood-fast` Makefile target;
  tooling in `wevibe-meta/scripts/empirical_replay/` (300 epochs ≈ **~5 h** at default cadence, not
  "multi-day").
- **Chain Broadcast**: the hub relays serve/denial event batches signed by hub-held per-org **serving
  keys** (`serves.go:496,611` → `submit.go:316` → `broadcast.go:626`); the dashboard broadcasts
  wallet-direct via `directBroadcast` (`wevibe-dashboard/lib/chain-client.ts:732`). The standing phrasing
  "the hub does not relay leader-signed txs" names the R-ONE-PATH constraint: there is **no
  delegate-key relay path**. (The "CO-258" label sometimes attached to this decision is misattributed —
  CO-258 is the umbral sidecar Docker-ization; the broadcast decision is D-ECON-STORAGE-MARKET amdt 13 /
  R-ONE-PATH.)

# 04. wevibe-hub

Hub API server: org registry, membership auth, moderation queue, recall/query surface,
chain-leg projection, and billing. Chi router, pgx + Postgres (schema applied by the hub
itself), Qdrant vectors, hubsgn response signing.

## Header

- **Module:** `github.com/wevibe-network/wevibe-server/wevibe-hub` (go.mod:1)
- **Go:** 1.25.9 (go.mod:3)
- **Default port:** 4440 (internal/config/config.go:40; consumed cmd/wevibe-hub/main.go:401)

## Entry point — cmd/wevibe-hub/main.go

**Startup failure model.** DB connect is FATAL (main.go:84-91); chain client setup is FATAL
(:100-109); Qdrant connect is NON-FATAL — the hub degrades and continues (:160-168). Three
further FATAL paths that the old topology omitted: edge-policy load (:64-67), on-chain anchor
mismatch (:112-144), and umbral client setup (:152-155).

Startup sequence facts:

- **Schema first:** `db.ApplySchema` runs BEFORE the pgx pool is opened (main.go:84
  `ApplySchema`, :88 `NewPool`) — the hub applies its own schema at startup; it does NOT
  rely on a pre-initialized Postgres.
- **Response signing:** hubsgn signs responses (main.go:78-82, :254;
  internal/hubsign/hubsign.go:16,24,56; middleware.go:13), surfaced via `X-Hub-Signature`.
- **Retrieval init** (main.go:160-167) plus ranker bootstrap (:46-54).
- **Startup sync:** `chain.SyncStandingFromEvents` (main.go:174;
  internal/chain/standing_projection.go:34) with `chain.SyncEpochData` (:171). The old doc's
  `chain.SyncKeywordWeightsFromChain` is DEAD — removed 9ccee0d, replaced by
  `SyncStandingFromEvents`.
- **60s epoch ticker** (main.go:182): each tick runs SyncEpochData (:187) and
  ReconcileMembership (:190).
- **7 handler setters** wired at startup: SetPool (pool.go:22) · SetRecallMode (:34) ·
  SetChainClient (:46) · SetFaucetURL (:54) · SetSocialClient (:62) · SetUmbralService (:58) ·
  SetNodePrivkey (:42).
- **Notifications wiring** (main.go:214-224); **chain watcher wiring** (:226-235).
- **CORS** from `CORS_ALLOWED_ORIGINS` (main.go:31-41, 245-253); exposes `X-Hub-Signature`.

## Route table — 101 registrations

Sole chi router: cmd/wevibe-hub/main.go:237 (101 route registrations = 94 unconditional +
2 dev-gated + 5 test-gated). ★NEW = present in code but absent from the old topology
(6 total). Line numbers cite main.go.

**Global middleware (all routes):** TraceID :238 · RequestID :239 · RealIP :240 ·
Recoverer :241 · CORS :246-253 · SigningMiddleware :254.

### UNGROUPED (14)

- `GET /health` :256
- `GET /v1/members/{pubkey}/orgs` :258
- `GET /v1/profile/{wallet}` :259
- `GET /v1/notifications/ws` :272 — sits OUTSIDE the RequireVerifiedIdentity group (after its close at :270)
- `POST /v1/orgs` :274
- `POST /v1/identity/blob` :275
- `GET /v1/identity/blob/{credentialId}` :276
- `POST /v1/pair` :277
- `GET /v1/pair/{pairingId}` :278
- `GET /v1/hub/serving-address` :279
- `GET /v1/extraction-presets` :280
- `GET /v1/balance/{address}` :281
- `POST /v1/faucet/fund` :283 — [DEV] dev-gated even though it sits outside the dev Route group
- `GET /v1/orgs/discover` :285

### IDENTITY — RequireVerifiedIdentity (5)

- `GET /v1/profile/notifications` :264
- `PATCH /v1/profile/notifications` :265
- `GET /v1/notifications` :267
- `GET /v1/notifications/unread-count` :268
- `POST /v1/notifications/mark-read` :269

### ORG-PUBLIC — no membership required (6)

- `GET /v1/orgs/{orgID}/` :289
- `GET /v1/orgs/{orgID}/extraction-profile` :290
- `POST /v1/orgs/{orgID}/join` :291
- `GET /v1/orgs/{orgID}/epoch/{epochID}/manifest` :292
- `GET /v1/orgs/{orgID}/epoch/current/chain` :293 ★NEW
- `PUT /v1/orgs/{orgID}/extraction-profile` :294

### MEMBERSHIP — RequireVerifiedMembership (70, all prefixed `/v1/orgs/{orgID}`)

- `POST /epoch/rotate` :300
- `POST /members` :302
- `GET /members` :303
- `GET /members/{pubkey}` :304
- `POST /members/{pubkey}/enable-recall` :305
- `POST /members/{pubkey}/disable-recall` :306
- `POST /members/{pubkey}/kfrag` :307
- `POST /members/{pubkey}/pre-key` :308
- `GET /members/{pubkey}/pre-key` :309
- `POST /members/wallet` :310
- `GET /keys/envelope` :311
- `POST /dashboard/keys` :313
- `DELETE /dashboard/keys/{pubkey}` :314
- `POST /recovery/shares` :316
- `GET /recovery/shares` :317
- `POST /submit` :319
- `POST /submit/batch` :320
- `POST /reports/` :322
- `GET /reports/` :323
- `GET /reports/{reportID}` :324
- `PATCH /reports/{reportID}` :325
- `POST /reports/{reportID}/vote` :326
- `POST /reports/{reportID}/commit` :327
- `GET /moderation/queue` :330
- `GET /moderation/history` :331
- `POST /moderation/{submissionHash}/vote` :332
- `POST /moderation/{submissionHash}/approve` :333
- `POST /moderation/{submissionHash}/deny` :334
- `POST /contributors/{contributorPubkey}/deny-pending` :335
- `POST /submissions/{submissionHash}/keyword-vote` :336
- `POST /moderation/batch-submit` :337
- `POST /serves` :339
- `GET /serves/confirm` :340 ★NEW
- `POST /events` :341 ★NEW
- `POST /decision-notes` :342 ★NEW
- `PUT /recall-rate-limit` :343
- `GET /recall-rate-limit` :344
- `GET /recall-queries` :345
- `GET /recall-queries/{queryID}` :346
- `GET /recall-health` :347
- `GET /pending-callbacks` :348 ★NEW
- `POST /extracted-sessions` :349
- `GET /extracted-sessions` :350
- `POST /denials` :351
- `GET /denials/pending-count` :352
- `GET /denials/pending` :353
- `POST /query` :355
- `GET /memories/{cid}` :356
- `GET /keywords` :358
- `GET /keywords/candidates` :359
- `POST /keywords` :360
- `PUT /keywords/merge` :361
- `PUT /keywords/{keyword}/rename` :362
- `DELETE /keywords/{keyword}` :363
- `POST /submit-keyword-results` :365
- `POST /verify-keywords` :366
- `PUT /submissions/{hash}/update-keywords` :367
- `GET /submissions/duplicate-clusters` :368
- `DELETE /submissions/{hash}` :369
- `GET /submissions` :370
- `GET /my-submissions` :371
- `GET /commit-status` :372
- `GET /health` :374
- `GET /credits` :376
- `GET /finances` :377
- `GET /chain-config` :378
- `GET /join-requests` :380
- `POST /join-requests/{requestID}/approve` :381
- `POST /join-requests/{requestID}/cancel-approval` :382
- `POST /join-requests/{requestID}/deny` :383

### DEV-gated — `WEVIBE_DEV_ENDPOINTS=true` (1)

- `POST /v1/billing/topup` :388

### TEST-gated — `WEVIBE_TEST_MODE=true` (5)

- `GET /v1/test/health` :393
- `POST /v1/test/embed` :395
- `POST /v1/test/redrive/approve` :396 ★NEW
- `GET /v1/test/orgs/{orgID}/queue` :397
- `GET /v1/test/orgs/{orgID}/serve-queue` :398

### DEAD route (removed)

`POST /v1/orgs/{orgID}/submissions/{hash}/rerun-keywords` — removed in d173331
(2026-06-14, leader-sovereign keyword curation). Zero source hits repo-wide; stale refs
remain only in the old TOPOLOGY.md docs.

## Config — 25 fields

internal/config/config.go:11-36. Env binding is MANUAL in `Load()` (:39-102) — there are
NO struct tags; a rewrite must say "env binding in Load()", never "env tags". ★NEW = the 4
fields missing from the old topology.

| Field | Env var | Default |
|---|---|---|
| `Port` | `WEVIBE_HUB_PORT` | 4440 |
| `DatabaseURL` | `DATABASE_URL` | "" |
| `QdrantAddr` | `QDRANT_ADDR` | "localhost:6333" |
| `QdrantAPIKey` | `QDRANT_API_KEY` | REQUIRED — panics if empty or < 32 chars |
| `StripeSecretKey` | `STRIPE_SECRET_KEY` | "" |
| `S3Bucket` | `WEVIBE_S3_BUCKET` | "wevibe-memories" |
| `NodePrivkey` | `HUB_NODE_PRIVKEY` | "" |
| `ChainGRPCURL` | `WEVIBE_CHAIN_GRPC_URL` | "" |
| `FaucetURL` | `FAUCET_URL` | "http://wevibe-faucet:4470" |
| `ChainRPCURL` | `WEVIBE_CHAIN_RPC_URL` | "" |
| `ChainID` | `WEVIBE_CHAIN_ID` | "" |
| `ChainSubmitterMnemonic` | `WEVIBE_CHAIN_SUBMITTER_MNEMONIC` | "" |
| `ChainEnabled` | `WEVIBE_CHAIN_ENABLED` | false (strict == "true") |
| `UmbralSidecarAddr` | `WEVIBE_UMBRAL_SIDECAR_ADDR` | "127.0.0.1:4460" |
| `SocialGraphURL` | `WEVIBE_SOCIAL_GRAPH_URL` | "http://wevibe-social-graph:4470" |
| `RetrievalTemperature` | `RETRIEVAL_TEMPERATURE` | 0.7 |
| `RetrievalNewMemBoostMult` | `RETRIEVAL_NEW_MEM_BOOST_MULT` | 0.5 |
| `RetrievalNewMemBoostWindow` | `RETRIEVAL_NEW_MEM_BOOST_WINDOW` | 30 |
| `RetrievalVectorNoiseSigma` | `RETRIEVAL_VECTOR_NOISE_SIGMA` | 0.0 |
| `RetrievalRecallDepth` | `RETRIEVAL_RECALL_DEPTH` | 5000 |
| `RetrievalOpenLoopFraction` ★NEW | `RETRIEVAL_OPEN_LOOP_FRACTION` | 0.0 (clamped [0,1], config.go:92-99) |
| `RetrievalCounterfactualLogging` ★NEW | `RETRIEVAL_COUNTERFACTUAL_LOGGING` | false |
| `RecallMode` | `WEVIBE_RECALL_MODE` | "prod" (only literal "test" passes) |
| `RelayHoldHours` ★NEW | `WEVIBE_RELAY_HOLD_HOURS` | 24 |
| `RelayHoldExemptOrgs` ★NEW | `WEVIBE_RELAY_HOLD_EXEMPT_ORGS` | nil |

## Handlers — 39 non-test files (internal/api/handlers/)

★NEW = the 2 files absent from the old topology (both added by 74ef23d, both with _test
siblings, both wired in main.go).

| File | Exports |
|---|---|
| balance.go | GetBalance |
| ban.go | DenyPendingForContributor |
| batch_submit.go | OrgHealth |
| billing.go | TopUpCredits/GetOrgCredits/GetOrgFinances |
| chain_config.go | GetOrgChainConfig |
| commit_status.go | CommitStatus |
| dashboard.go | RegisterDashboardKey/RevokeDashboardKey |
| decision_notes.go ★NEW | RecordDecisionNote |
| dedup.go | DuplicateClusters |
| denials.go | GetPendingDenialCount/GetPendingDenials |
| discovery.go | DiscoverOrgs |
| errors.go | WriteError |
| extracted_sessions.go | RecordExtractedSession/ListExtractedSessions |
| extraction.go | GetExtractionProfile/SetExtractionProfile/GetExtractionPresets |
| faucet.go | FundFromFaucet |
| health.go | Health |
| identity_blobs.go | StoreIdentityBlob/GetIdentityBlob |
| join.go | SubmitJoinRequest/ListJoinRequests/ApproveJoinRequest/CancelJoinApproval/DenyJoinRequest |
| keyword_extraction.go | SubmitKeywordResults/VerifyKeywords/UpdateKeywords/RemoveSubmission/ListSubmissions/ListMySubmissions |
| keywords.go | ListKeywords/ListKeywordCandidates/AddKeyword/MergeKeywords/RenameKeyword/DeprecateKeyword |
| members.go | InviteMember/GetMember/GetKeyEnvelope/ListMembers/GetMemberOrgs/LinkWallet/EnableMemberRecall/DisableMemberRecall/StoreMemberKFrag/RegisterPreKey/GetPreKey |
| moderation.go | SubmitMemory/SubmitMemoryBatch/GetPendingQueue/GetModerationHistory/VoteOnSubmission/VoteOnKeyword/ApproveSubmission/DenySubmission/PrepareBatchForChain |
| notification_emit.go | (no exports) |
| notifications.go | SetNotificationHub/SetNotificationDispatcher/ListNotifications/GetUnreadCount/MarkRead/NotificationWebSocket |
| orgs.go | CreateOrg/GetOrg/RotateEpoch/GetEpochManifest/GetCurrentChainEpoch |
| pairing_blobs.go | StorePairingBlob/GetPairingBlob |
| pending_callbacks.go ★NEW | GetPendingCallbacks |
| pool.go | SetPool/GetPool/SetQdrantClient/SetRecallMode/SetNodePrivkey/SetChainClient/SetChainWatcher/SetFaucetURL/SetUmbralService/SetSocialClient |
| profile.go | GetProfile |
| profile_notifications.go | GetNotificationPreferences (:32) / UpdateNotificationPreferences (:62) |
| ratelimit.go | SetRecallRateLimit/GetRecallRateLimit |
| recall_inspector.go | ListRecallQueries/GetRecallQueryDetail/GetRecallHealth |
| recovery.go | StoreRecoveryShares/GetRecoveryShare |
| reports.go | CreateReport/ListReports/GetReport/UpdateReport/VoteOnReport/CommitReport |
| retrieval.go | QueryMemories/GetMemory |
| serves.go | RecordServeEvent/ConfirmServeEvent/RecordEvent/SetRelayHoldConfig/EnqueueEligibleRelays/RecordDenialEvent |
| serving.go | SetResponsePubkeyHex/GetServingAddress |
| social_names.go | (no exports) |
| testing.go | TestHealth/TestEmbed/TestRedriveApproveMemory/TestServeQueueDepth/TestGetQueue |

Note: `GetNotificationPreferences`/`UpdateNotificationPreferences` live in
profile_notifications.go, NOT notifications.go (the old doc misattributed them to
notifications.go). Dead accessors `GetSocialClient` (pool.go), `IsDashboardKey`
(dashboard.go), and `GetNotificationHub` (notifications.go) were purged in 84fe5e7 and have
no entries here.

## Auth middleware — exactly two exports

- `auth.RequireVerifiedIdentity` — internal/auth/middleware.go:23; wired main.go:262
- `auth.RequireVerifiedMembership` — middleware.go:38; wired main.go:298

No other exported middleware; no renames.

## Status

Verified against report 1787047628 (first-pass audit) and re-validated by
1787048715 (second-pass, 29 CONFIRMED · 5 CORRECTED · 0 REJECTED). Counts re-checked for
this rewrite: **101 route registrations**, **25 config fields**, **39 non-test handler
files**.

# 05 — wevibe-hub Internal: Packages · Protocol · Schema · Tests

**Scope.** `wevibe-server/wevibe-hub/internal/**`, `internal/protocol/types.go`, `wevibe-server/db/schema.sql` (v3.1 RECALL-PIVOT), hub test suite.
**Authority.** Second-pass audit `1787048715-WO-TOP2-HUB-SURFACE.md` (every claim re-verified on-touch, file:line) wins over first-pass `1787047710-WO-TOPOLOGY-HUB-INTERNAL.md`. The old TOPOLOGY.md sections predate the RECALL-PIVOT (content-free event log on chain; standing computed at the edge as `f(events, policy)`, `edge-policy-v1` anchored at h45); this section describes the post-pivot reality.

## 5.1 Internal packages

25 internal directories; exactly **4 packages are new** since the last documented pass — **standing**, **memories**, **config**, **wlog** (§5.1.7). No undocumented package exists beyond those four.

### 5.1.1 internal/chain

| File | Role / exports |
|---|---|
| `grpc_client.go` | 16 accessors (:74–244) + 3 previously undocumented: `GetCurrentChainEpoch`, `BroadcastTxSync`, `GetOrgSigner` |
| `submit.go` | `SubmitRelayBatch` / `SubmitServeBatch` / `SubmitDenialBatch` / `BuildServeBatchMsg` / `BuildDenialBatchMsg`; pivot additions `OutcomeResolutionFromString`, `OutcomeSourceFromString`, `BuildEventBatchMsg` |
| `query.go` | 16 queries; `MemoryBatchResult` parity (CO-224, :402–413); CO-213 nil-safe with nuance — returns nil on ALL errors, not only unreachable/not-found (:365–389) |
| `sync.go` | `SyncEpochData` |
| `merkle.go` · `accounts.go` · `balance.go` · `broadcast.go` · `cometbft_subscriber.go` · `faucet.go` · `reconcile.go` | Merkle proofs · `OrgKeyRole` type + `OrgKeyServing` const · balances · tx broadcast · CometBFT subscription · `FundAddressFromFaucet` · `ReconcileMembership` |
| `watcher.go` · `watcher_memory.go` · `watcher_report_org.go` · `watcher_serve.go` | Chain watchers; `watcher_serve.go` gained pivot additions `processEventBatchBookkeeping`, `recomputeStandingForMemory` |
| `policy_anchor.go` ★new | Verifies the anchored policy hash against the chain |
| `redrive.go` ★new | `RedriveApproveMemory` heal path |
| `standing_chain_accept.go` ★new | Chain outcome-fingerprint acceptance gate |
| `standing_projection.go` ★new | `SyncStandingFromEvents` (:34) / `RecomputeMemoryStanding` — the pivot's edge projection engine |

### 5.1.2 orgs · members · moderation

- **orgs.go** — all 14 funcs; `CreateOrg` → `billing.ProvisionOrgLedger` after commit (:69); new `GetOrgIDByLeader` (:94).
- **members.go** — all 13 funcs; new sentinel `ErrInvalidPrePubkey`.
- **moderation.go** — all 9 funcs. Signature verification is over the **9-field canonical `SubmitMemoryMessage`** (moderation.go:47–58; canonical.go:149–172), not bare submission-hash bytes; the moderator-approve path uses the **5-field `ApproveSubmissionMessageSimple`** (canonical.go:190–199). Two extra hash checks (ciphertext, wrapped_dek). `ApproveSubmission`'s vector / embeddingModelID / embeddingSchemaVersion are accepted-but-ignored **dead params** (advisory-vote-only, D-MODERATION-ADVISORY). New types: `SubmissionVoteTally`, `KeywordVoteTally`, `KeywordVoteEntry`, `ModeratorRecommendation`.

### 5.1.3 retrieval — retrieval.go + ranking_core.go + querylog.go + stats.go

- Constants: `EMBED_DIM=768` (retrieval.go:43), `contestedThreshold=0.20` (:44).
- `NewQdrantClient` strips both http/https schemes, does NOT enforce apiKey, hardcodes port 6333 (:214–232).
- **Pivot re-basing:** per-keyword weights retired → `UpdateStanding` (:1314). The Qdrant payload now carries `keywords` (flat `[]string` labels) + `standing_bps` (default 10000) + `standing_archived` (retrieval.go:405–435). The `memory_keywords` table is orphaned by this (zero source refs; see §5.3).
- `injectGaussianNoise` inert at σ=0 default (D-9.5; env `RETRIEVAL_VECTOR_NOISE_SIGMA`).
- Lifecycle filtering: ARCHIVED excluded unconditionally; DORMANT gated on `includeDormant`.
- Env wiring: `RETRIEVAL_TEMPERATURE` / `RETRIEVAL_NEW_MEM_BOOST_MULT` / `RETRIEVAL_NEW_MEM_BOOST_WINDOW`; grace=20 hardcoded.
- **ranking_core.go ★new** — `ScoreAndRank` (:155); `ProbabilisticRanker` remains in retrieval.go:69 (ranking core stays in retrieval, NOT moved to standing).
- **querylog.go ★new** — `PersistRecallQuery`.
- **stats.go** — `ChainQuerier` / `GetAcceptanceCount` / `GetContributorStats`. The old "wallet-first" known issue is ALREADY FIXED (98783c7; chain reputation keyed by pubkey only). Field renames: contributions→`TotalApprovedMemories`, serve_count→`TotalServesReceived`, `account_age_days` derived from FirstContributionEpoch-as-days (no `first_seen_timestamp` exists).

### 5.1.4 billing · receipts · verify

- **billing.go** — all 7 funcs; ledger integrity via `CHECK(balance>=0)` (schema.sql:329).
- **receipts.go** — `CreateReceipt` 10-param list; stores SHA-256 commitments, never raw payloads; table `usage_receipts`.
- **verify/sig.go** — **Ed25519 contributor/moderator path** (:24).
- **verify/wallet_sig.go** — 5 exports; **NO EIP-712** (no typed-data path anywhere in repo or history): plain sha256 digest (:32) + `ecdsa.Verify` (:43), bech32 HRP `wevibe` (:52); the wallet path is ECDSA/secp256k1.
- **verify/canonical.go** — all 14 canonical builders, zero drift; `ApproveSubmissionMessage` carries umbralCapsule/umbralCiphertext. Serve/outcome event canonicalization (`CanonicalEventBody` / `ComputeEventFingerprint`) deliberately lives in **wevibe-chain `x/serve/types`**, not hub verify.

### 5.1.5 embed · envelopes · auth · db · hubsign

- **embed.go** — `EMBED_DIM=768` (:17).
- **envelopes.go** — **`Envelope.OrgID` / `Envelope.Pubkey` are `string`** (envelopes.go:11–12; the old doc's `int` is stale); `Store`/`Get`/`BatchReplace`/`Delete`. New files: `identity_blobs.go`, `pairing_blobs.go`.
- **auth/header.go** — the scheme prefix lives inside the Authorization header: **`Authorization: WeVibe-Signed …`** (:21–30, case-sensitive) — not a standalone header. New file: **auth/middleware.go** — `RequireVerifiedIdentity` (:23), `RequireVerifiedMembership` (:38).
- **db.go / migrate.go** — `ApplySchema`; **NO migrations** (schema.sql is the wipe-on-change SoT).
- **hubsign** — `NewFromEnv` / `PublicKeyHex` / `SignBody` / `SigningMiddleware`.

### 5.1.6 notifications · reports · serves · sanitize · social · umbral

- **notifications** — `NewHub` / `NewPreferenceStore` / `NewDispatcher` / `EmitUserNotification` + SMTP/webhook dispatch.
- **reports.go** — `Create`/`List`/`Get`/`Update`/`GetReportRecommendations`; escalation is a column (`escalation_votes`), not a func.
- **serves.go** — `RecordServe` / `RecordDenial` / `GetPendingServes` / `GetPendingDenials` / `HasPendingEvents` (:680); pivot exports `RecordOutcome`, `PendingOutcomeEvents`, `MarkOutcomeEvents`, `GetServeEventsByEpisode`, `ListOrgsWithEligiblePending`; `MarkServesSubmitted` (:648) / `MarkDenialsSubmitted` (:665). Serve + denial unified into `serve_events` (`event_type serve|denial`). Two live admission-gate call sites (serves.go:198, handlers/serves.go:299) → `memories.EnsureApproved`.
- **sanitize** — `ScanContent` (+ `homoglyphs.go`).
- **social** — `NewClient` / `ResolveNames` / `GetProfile`.
- **umbral** — all 6 exports (+ `umbralpb/` proto subdir).

### 5.1.7 NEW packages (4)

- **standing** (introduced 9ccee0d) — the pure edge-standing engine. `engine.go Compute(events, createdEpoch, currentEpoch, policy) Result` (:67): a deterministic fold, **stdlib-only imports**, no internal dependencies. `policy.go LoadPolicy` (:43) → `wevibe-hub/policy/edge-policy-v1.json` (anchored at chain h45). The policy file is loaded by **RELATIVE PATH** at startup (main.go:60–64) — **not** `go:embed`. `doc.go` states the `f(events, policy)` contract.
- **memories** (introduced 763a9c3) — `approval.go EnsureApproved` (:33): the **chain-first / DB-fallback admission gate** for serve + outcome intake.
- **config** — `Config` struct, **25 fields** (config.go:11–36) + `Load()` (:39–102). **ZERO struct tags** — env binding and defaults are manual inside `Load()` (never describe these as "env tags"). Not previously undocumented: the old TOPOLOGY.md L236–264 entry covers it but is stale by 4 fields — the new ones are `RetrievalOpenLoopFraction` (clamped [0,1]), `RetrievalCounterfactualLogging`, `RelayHoldHours`, `RelayHoldExemptOrgs`.
- **wlog** (introduced 1c0ab6d) — `wlog.go` structured slog + trace propagation: `Init` (:30) / `Op` (:36) / `Fingerprint` (:43) / `TraceID` (:69) / `UnaryClientInterceptor` (:82).

## 5.2 Protocol types — 50 structs, 0 DEAD, 0 renamed

`internal/protocol/types.go` declares **exactly 50 structs** — set-equal to the old doc's list (TOPOLOGY.md:774–786), which therefore remains the authoritative name set. Nothing was removed or renamed by the pivot.

**Name correction:** `Submission`, `Memory`, `BatchSubmission`, `ServeEvent`, `MemoryBatch` do **NOT** exist as protocol types. The only adjacent names are `Envelope` (in the envelopes package, not types.go) and the `SubmissionStatus` const group (types.go:12–17).

New non-type additions since the last doc pass: **10 package consts** — the `SubmissionStatus` group (4 consts) plus `MemoryTypeMemory`, `MaxKeywordsPerMemory`, `MaxMemoryChars`, `MaxNegativeSignalChars`, `KeywordFormatRegex`, `MaxBatchMemories` — and func `IsValidMemoryType`.

## 5.3 Database schema — db/schema.sql v3.1 RECALL-PIVOT

**Totals: 36 tables · 45 index objects (41 CREATE INDEX + 4 UNIQUE constraints).**

### Tables (36 — name, primary key, notable facts)

**Orgs & membership**

| Table | PK | Notes |
|---|---|---|
| `hub_instance` | `id` | |
| `orgs` | `org_id` | |
| `org_recall_rate_limits` | `org_id` | |
| `members` | `(org_id, pubkey)` | `trial_expires_at` is **TIMESTAMP, not TIMESTAMPTZ** (:108 — the only non-TZ timestamp in the file) |
| `epoch_manifests` | `(org_id, epoch_id)` | |
| `join_requests` | `request_id` | dates to first commit 90ef569 (NOT pivot-era) |

**Submissions & moderation**

| Table | PK | Notes |
|---|---|---|
| `pending_submissions` | `submission_hash` | status CHECK unchanged (`pending_keyword\|pending_chain\|committed\|denied`); `umbral_capsule`/`umbral_ciphertext` present (:149–150, leader-side mint) |
| `extracted_sessions` | `(org_id, contributor_pubkey, session_id)` | |
| `submission_mod_votes` | `(org_id, submission_hash, moderator_pubkey)` | |
| `keyword_mod_votes` | `(org_id, submission_hash, keyword, moderator_pubkey)` | |
| `decision_notes` | `id` | predates the pivot (74ef23d) — NOT pivot-era |

**Reports**

| Table | PK | Notes |
|---|---|---|
| `reports` | `id` | `resolution` column (upheld\|dismissed\|dismissed_malicious) |
| `report_votes` | `(org_id, report_id, voter_pubkey)` | |

**Billing & credits**

| Table | PK | Notes |
|---|---|---|
| `org_credits` | `org_id` | `CHECK(balance>=0)` :329 |
| `usage_receipts` | `receipt_id` | SHA-256 commitments |
| `credit_transactions` | `txn_id` | carries the **`delta BIGINT` (:345)** |
| `audit_log` | `id` | |

**Keys & identity**

| Table | PK | Notes |
|---|---|---|
| `key_envelopes` | `(org_id, pubkey)` | |
| `identity_blobs` | `(pubkey, credential_id)` | |
| `pairing_blobs` | `pairing_id` | |
| `recovery_shares` | `(org_id, share_index)` | |
| `dashboard_keys` | `(org_id, pubkey)` | |
| `org_extraction_profile` | `org_id` | |

**Serve/outcome events & standing (pivot core)**

| Table | PK | Notes |
|---|---|---|
| `serve_events` | `id` | pivot shape: `episode_ref` NOT NULL, `event_type` CHECK `serve\|denial`, `serve_fingerprint`, UNIQUE relay-dedup key |
| `outcome_events` | `id` | ★pivot-era (9ccee0d, :464): `resolution` CHECK `worked\|didnt_work\|unobserved` (:474), `fingerprint` UNIQUE (:479) |
| `memory_standing` | `(memory_cid, org_id)` | ★pivot-era (9ccee0d, :494): DERIVED projection — wipe-safe, recomputable from events |

**Recall / query path**

| Table | PK | Notes |
|---|---|---|
| `session_served_memories` | `(org_id, session_id, memory_cid)` | |
| `query_log` | `query_id` | **has NO gamma/delta columns** (full list :534–547) — the `delta BIGINT` belongs to `credit_transactions` |
| `query_candidate_scores` | `(query_id, memory_cid)` | |

**Keywords**

| Table | PK | Notes |
|---|---|---|
| `org_keywords` | `id` | |
| `keyword_candidates` | `(org_id, keyword, contributor_pubkey, submission_hash)` | |
| `memory_keywords` | `(memory_cid, keyword)` | **ORPHANED — 0 source refs** (payload now carries flat `keywords` labels); present in schema, not deleted |

**Notifications & infra**

| Table | PK | Notes |
|---|---|---|
| `notifications` | `id` | dates to first commit 90ef569 (NOT pivot-era) |
| `notification_preferences` | `recipient_pubkey` | dates to first commit 90ef569 (NOT pivot-era) |
| `rotation_buffer` | `buffer_id` | |
| `watcher_state` | `watcher_name` | |

**Correction on "new tables":** only `outcome_events` and `memory_standing` are pivot-era (both 9ccee0d). Prior first-pass labels of "6 NEW pivot-era tables" were wrong — `decision_notes` (74ef23d) and `notifications` / `notification_preferences` / `join_requests` (90ef569) all predate the pivot.

### UNIQUE constraints (4)

| Table | Columns | Line |
|---|---|---|
| `members` | `(org_id, wallet_address)` | :110 |
| `serve_events` | `(org_id, event_type, serve_key_pubkey, memory_content_hash, epoch_id)` | :453 |
| `outcome_events` | `(fingerprint)` | :479 |
| `org_keywords` | `(org_id, keyword)` | :589 |

### CREATE INDEX (41 — ★ marks the only 3 code-new indexes)

- `orgs`: idx_orgs_leader :68 · idx_orgs_status :69
- `members`: idx_members_active :113 · idx_members_membership_active :114 · idx_members_pubkey :115 · idx_members_wallet :116
- `pending_submissions`: idx_pending_org_status :189 · idx_pending_contributor :190
- `extracted_sessions`: idx_extracted_sessions_contributor :202
- `submission_mod_votes`: idx_submission_mod_votes_sub :225 · `keyword_mod_votes`: idx_keyword_mod_votes_sub :226
- `reports`: idx_reports_org_status :250 · idx_reports_memory :251 · idx_reports_created :252 · `report_votes`: idx_report_votes_report :266
- `rotation_buffer`: idx_rotation_buffer_org :292 · `usage_receipts`: idx_receipts_org_epoch :309 · `audit_log`: idx_audit_org_epoch :323 · `credit_transactions`: idx_credit_txn_org :352
- `key_envelopes`: idx_envelopes_org :367 · `identity_blobs`: idx_identity_blobs_credential :379 · `recovery_shares`: idx_recovery_shares_holder :403 · `dashboard_keys`: idx_dashboard_keys_pubkey :420
- `serve_events`: idx_serve_events_org_status :456 · idx_serve_events_org_epoch :457 · idx_serve_events_org_status_type :458 · idx_serve_events_pending_created :459 · **idx_serve_events_pairing_ref :460 ★ @2355743**
- `outcome_events`: **idx_outcome_events_org_status :489 ★ @9ccee0d** · **idx_outcome_events_pairing_ref :490 ★ @2355743**
- `decision_notes`: idx_decision_notes_org_member_memory :521 · `session_served_memories`: idx_session_served_served_at :531
- `query_log`: idx_query_log_org_created :564 · `query_candidate_scores`: idx_query_candidate_scores_query :565
- `org_keywords`: idx_org_keywords_org :592 · `keyword_candidates`: idx_keyword_candidates_org_kw :606 · `memory_keywords`: idx_memory_keywords_keyword :619
- `notifications`: idx_notifications_recipient_unread :637 · `notification_preferences`: idx_notification_preferences_updated_at :654
- `join_requests`: idx_join_requests_org_pending :678 · idx_join_requests_requester_org :679

**Correction on "new indexes":** the first-pass "13 NEW indexes" was a mislabel — all 13 names exist only relative to the stale doc, and just the 3 starred above are code-new; the remaining 8 (of those 13) date to first commit 90ef569.

## 5.4 Test files

- Old doc summary listed 32 entries: **31 verified intact** with zero summary/Requires drift.
- **DEAD:** `internal/retrieval/parity_test.go` — deleted in 9ccee0d (recall-pivot); its capability was absorbed into `ranking_core_test.go` (`TestScoreAndRank_SelftestParity`).
- **13 NEW test files:**
  - handlers/: `decision_notes`, `orgs_chain_epoch`, `pending_callbacks`, `retrieval_governor`, `serves_confirm`
  - chain/: `policy_anchor`, `query`, `redrive`, `standing_projection`, `watcher_register_org`
  - memories/: `approval`
  - standing/: `engine`, `policy`

## 5.5 DEAD list (verified removals, with commits)

Every absence verified by zero word-boundary source hits plus git diff/`--name-status` evidence.

| Dead item | Removed in | Replacement |
|---|---|---|
| `SubmitMemoryBatchAtomic` + `BatchMemory` (submit.go −108) | **cbaf289** ("MC-1 chain-leg … unwind") | leader-signed client-side batch (`SubmitMemoryBatch`) — a refactor, NOT the pivot |
| `MarkSubmitted` / `MarkFailed` / `CountPending` + `GetServeEventByIdentity` (serves.go −63) | **84fe5e7** ("purge dead accessors") | `MarkServesSubmitted` (:648) / `MarkDenialsSubmitted` (:665) / `HasPendingEvents` (:680) |
| `GetSocialClient` (pool.go) · `IsDashboardKey` (dashboard.go) · `GetNotificationHub` (notifications.go) | **84fe5e7** | none — dead accessors, no replacement |
| `SyncKeywordWeightsFromChain` (sync.go −114) | **9ccee0d** (recall-pivot) | `SyncStandingFromEvents` (chain/standing_projection.go:34) |
| retrieval keyword-weight funcs: `UpdateKeywordWeights`, `ApplyServeBoostLocal`, `ApplyDenialDecayLocal`, `GetKeywordWeights` | **9ccee0d** (recall-pivot) | `UpdateStanding` (retrieval.go:1314); per-keyword decay retired with the weights |
| `internal/retrieval/parity_test.go` | **9ccee0d** | absorbed into `ranking_core_test.go` |
| `POST /v1/orgs/{orgID}/submissions/{hash}/rerun-keywords` | **d173331** (2026-06-14) | leader-sovereign keyword curation — no server route |

## 06. wevibe-chain — Cosmos SDK Appchain (Events + Standing Model)

> Rewritten against current code truth; supersedes the 2026-06 TOPOLOGY section, including its "Earned-Trust Decay Code Anchors" subsection, which is DEAD and REMOVED (see §"Earned-Trust Decay — DEAD" below). Provenance: `wevibe-meta/workspace/reports/1787048788-WO-TOP2-RECALLPIVOT.md` (PASS-2, authoritative) and `1787047662-WO-TOPOLOGY-CHAIN.md`.

### Header

| Fact | Value | Anchor |
|---|---|---|
| Module path | `github.com/wevibe-network/wevibe-chain` | go.mod:1 |
| Go version | `go 1.25.9` (no toolchain directive) | go.mod:3 |
| Docker build | `golang:1.26-bookworm` → `debian:bookworm-slim` runtime | Dockerfile |
| Ports (all loopback, never 0.0.0.0) | 26657 CometBFT RPC (127.0.0.1) · 9090 gRPC (localhost) · 1317 API (localhost) | cometbft DefaultConfig; SDK DefaultGRPCAddress / DefaultAPIAddress |
| Chain ID | set via `wevibed init --chain-id <id>` (SDK genutil InitCmd flag) | cmd/wevibed/cmd/root.go:108 |
| Framework pins | cosmos-sdk v0.53.5 · cometbft v0.38.20 | go.mod:17,19 |

**Version skew note:** go.mod declares Go 1.25.9 while the Dockerfile builds with `golang:1.26-bookworm` — the image tag runs ahead of go.mod. This is a real version skew, not a doc error.

### Entry Point & App Wiring

- `cmd/wevibed/main.go` is a thin CLI wrapper over `cmd/wevibed/cmd/root.go` (init / start / keys / genesis / query / tx).
- `app/app.go` registers the **8 custom modules** — bandwidth, emissions, memory, org, identity, reputation, serve, attestation (module keys app.go:244-251; manager app.go:462-469) — plus standard SDK modules: auth, bank, staking, distribution, slashing, mint, consensus, genutil, epochs, gov, feegrant, authz, upgrade.
- x/serve is wired as the recall-pivot event log + `StoredPolicyAnchor` store (app.go:407).
- **Explicitly ordered:** `SetOrderInitGenesis` (app.go:482-505) and `SetOrderEndBlockers` (app.go:518-523). **Correction to the old doc's "all explicitly set" wording:** there is NO `SetOrderExportGenesis` call — export falls back to the SDK alphabetical default (`app/export.go:14` → `NewManagerFromMap`).
- Module-account perms: `"org": {Burner}` (app.go:126) — this is what lets the org module burn the 50% slot-auction fee.
- Epoch hooks explicitly ordered emissions → memory (app.go:418-423).
- Genesis seeding (`scripts/init-chain.sh`, idempotent, jq-based): `epochs.wevibe_epoch` (:127-134), a full CO-041 emission-pool struct (:175-188 — NOT `{}`), `reputation {"active": true}` (:194), `serve.policy_anchors` (:219-232, env-gated on `WEVIBE_EDGE_POLICY_FILE`), plus gov params and zero-inflation mint.

### Custom Modules (8)

All 8 modules follow `x/{module}/{keeper,types,proto}` and own a proto package at `proto/wevibe/{module}/v1/`.

| Module | Post-pivot purpose | Test coverage |
|---|---|---|
| `x/attestation` | Disabled-but-wired (see FORWARD NOTE) | keeper_test + integration |
| `x/bandwidth` | Memory-cap enforcement wired (see FORWARD NOTE) | keeper_test + integration |
| `x/emissions` | Flat daily mint + contributor rewards; Treasury removed | keeper_test + integration |
| `x/identity` | Identity registry | **NO keeper_test.go (never existed in git history) and NO integration coverage — only `msg_server_test.go`** |
| `x/memory` | Commitment / provenance / validity / merkle only (see recall-pivot model) | keeper_test + integration |
| `x/org` | Org registry, slot registry + auction (50% burn), serving/config msgs | keeper_test + integration |
| `x/reputation` | Reputation params/state | keeper_test + integration |
| `x/serve` | Recall-pivot event log (E1–E8) + receipts + `StoredPolicyAnchor`; serving-key gate | keeper_test + integration |

**FORWARD NOTE — all four sub-claims of the old doc are now VERIFIED BUILT:**

- **(a) x/org slot registry + auction:** `slotreg/` store key, `GetNextSlot`/`SetNextSlot`, `StoredSlotRegistry`, `computeAscendingPrice`; the slot fee is charged at org creation with **50% burned** (org module Burner perm).
- **(b) x/emissions Treasury removed:** tx.proto holds only `MsgMintDailyEmission` / `MsgUpdateParams` / `MsgClaimContributorReward`; `MsgWithdrawTreasury` is absent.
- **(c) x/attestation disabled-but-wired:** `SubmitSessionAttestation` returns `ErrAttestationDisabled`; storage + `UpdateParams` remain live.
- **(d) x/bandwidth memory-cap wired:** `MemoryUsed >= MemoryCap` rejects; `MsgSetBandwidthOverride` provided.

### The Recall-Pivot Model — Event Log + Policy Anchor (centerpiece)

This replaces the old TOPOLOGY subsection **"Earned-Trust Decay Code Anchors"**, which is DEAD and REMOVED. The chain holds **no trust, no decay, no weights, no scores, no content**: it is an append-only log of content-free, consumer-signed evidence, from which standing is computed at the edge as a pure function of (events, anchored policy version).

**What the chain stores** (x/serve + x/memory):

1. **An append-only, content-free, consumer-signed EVENT log, E1–E8** — `proto/wevibe/serve/v1/event.proto:10-25` (EventType enum, 9 values incl. `EVENT_TYPE_UNSPECIFIED=0`).
   - **E1 serve + E2 block are recorded as receipts** — `StoredServeReceipt` (serve/v1/state.proto:7-25) / `StoredDenialReceipt` (:28-40). E1/E2 `EventEntry` oneof bodies are codec-rejected (canonical.go:130-132): the receipts ARE the E1/E2 log records.
   - **E3 outcome, E6 validity-predicate, E7 cost-to-discover, E8 convergence are recorded as `StoredEvent`** (event.proto:107-125), via `ProcessEventBatch` (keeper.go:489-529).
   - **E4 contest + E5 sponsorship are PARKED** — enum slots only (event.proto:18, :20), no codec exists (canonical.go:77-133 default → `ErrInvalidEventType`); oneof body slots 10-13 only, with `reserved 14,15` + `reserved "contest","sponsorship"` (event.proto:103-104,123-124).
   - **E3 `OutcomeEventBody` is tri-state post-pivot:** `worked bool` is RESERVED (`reserved 2` + `"worked"`), replaced by `resolution` (`OutcomeResolution` WORKED / DIDNT_WORK / UNOBSERVED, event.proto:38-46) plus `source` and a `serve_ref` pairing field (commit b2d375e, WO-ATTRIB).
   - All stored records round-trip through genesis (InitGenesis keeper.go:662-706).
2. **`StoredPolicyAnchor` — lives in `event.proto:127-135` (NOT state.proto).** Fields: `policy_version` (string, f1), `policy_hash` (bytes, f2), `anchored_at_epoch` (uint64, f3), `anchored_at_height` (int64, f4). Immutable re-anchor (keeper.go:588-590); `anchored_at_epoch` is always 0 — "height is the authoritative on-chain ordinal" (keeper.go:598). Authority-gated via `MsgAnchorPolicyVersion` (tx.proto:81-86; msg_server.go:234-256). Genesis-seeded (init-chain.sh:219-232, env-gated on `WEVIBE_EDGE_POLICY_FILE`). **Live anchor:** `edge-policy-v1`, sha256 `2d2faa14461aa51bb72735b05debf30defff039750e5f90c1922ae813c87899e`, anchored at height 45.
3. **`StoredMemoryCommitment`** (`proto/wevibe/memory/v1/state.proto:49-81`): ciphertext + provenance anchors + `repeated string keywords = 27` — **flat labels, never weighted, never gating** (comment at :79-80). `serve_count_total` / `denial_count_total` / `archived_epoch` are RESERVED (fields 20-22, :70-71). There is no keyword-weight field anywhere.
4. **Standing is NEVER written on-chain** — boundary rule. Negative sweep: zero substantive `standing|weight|decay|score|trust` hits in proto/x (boundary-rule comments only: event.proto:8,128; msgs.go:191). Enforcement test: `TestStoredEventBoundaryRule_NoStandingVerdictFields` (keeper_test.go:556-557). No weights, standing, scores, trust values, or content on-chain; `git log -S` confirms `keyword_weights` / `trust_weight_bps` never existed.
5. **Consumer-signed gate:** every event submission passes `requireServingKeySigner` (`x/serve/keeper/msg_server.go:30-39`), which gates all three batch paths — `SubmitServeBatch` (:54), `SubmitDenialBatch` (:83), `SubmitEventBatch` (:198). An org with no registered serving address can never submit (:35). The org's serving key signs the tx envelope (gas via feegrant); the hub relay is the only submission path.

**Canonical signature strings** (`x/serve/types/canonical.go`):

- Serve: **`wevibe-serve-v3`** (:14,:29) — `episode_ref` is signed into the preimage (:20,:35); `CanonicalServeBody(orgID, memoryHash, epoch, serveKeyPubkey, nonce, episodeRef)` (:27). The v3 bump is commit `6c2ac8b` (WO-TRIGGER-BUILD A3, episode_ref added post-pivot); v1→v2 was `e6fcdae`. The keeper verifies against `serve.EpisodeRef` (keeper.go:231); golden vector `TestCanonicalServeBodyV3_GoldenVector` (types_test.go:126-146). Note: RECALL-PIVOT-SPEC §3.1 still reads v2 — the spec trails the code by one version.
- Event: `wevibe-event-v1` (canonical.go:64). Denial: `wevibe-denial-v1` (canonical.go:188).

### Earned-Trust Decay — DEAD and REMOVED

The entire earned-trust decay machinery was removed from consensus by **`e6fcdae`** ("replace per-keyword decay consensus with append-only event log + policy anchor (recall pivot)"). All of the following symbols have **zero code hits**: `applyDecay`, `ApplyEpochDecay`, `ApplyServeBoost`, `ApplyDenialDecay`, `KeywordWeight` (proto), `resolveOrgIdleDecayConfig`, `GetMatchedKeywordsForEpoch`, `calculateDenialRateAndTrust`, `GetActiveMemoryCountByOrg` — along with `getMatchedKeywords`, `minKeywordWeight`, `graceEpochsRemaining`, the x/memory decay constants (params.proto: reserved 5-26 minus 10), and the `matched_keywords` non-empty serve-gate.

- `ApplyEarnedTrustDecay` **never existed in any commit** — the name is a phantom originating from the old TOPOLOGY.md; the real machinery was the symbols listed above.
- Known stale residue: three doc hits for `ApplyEpochDecay` in chain `docs/PDP.md:5,75,199`.
- **x/memory's current purpose = commitment / provenance / validity / merkle only** — zero decay/trust/weight logic. `lifecycle.go` holds exactly 6 functions (`saveMemoryCommitment`:17, `loadMemoryByCID`:29, `GetContributorsWithApprovalsInEpoch`:63, `getCurrentEpoch`:93, `setCurrentEpoch`:102, `decodeCID`:112). Residual no-op: `applyConfidencePenalty` (relationships.go:100-102).
- **ARCHIVED is reached solely via validity-window expiry** — `CheckEpochExpiry` (validity.go:44; condition `ValidUntilEpoch != 0 && epoch > ValidUntilEpoch`, :81-82). `retrieval_threshold_bps` is reserved-only (params.proto:21); there is no threshold- or weight-based archival.
- **Epoch hook chain (emissions → memory, app.go:418-423):** memory `AfterEpochEnd` = `setCurrentEpoch` → `CheckEpochExpiry` → `getAllOrgsWithMemories` (epoch_hooks.go:57) → `ComputeAndStoreEpochMerkleRoot`. **`ApplyEpochDecay` is REMOVED from this chain.**

### Emissions

- **`MintDailyEmission`** (x/emissions/keeper/keeper.go:191, wired via epoch hook epoch_hooks.go:9→16): the validator portion is decremented from the pool (not minted); the contributor leg is a **flat even split** `(budget + rollover) / len(qualifying)` over approved contributors (qualification = approval count ≥ threshold, keeper.go:239-263). Serves are excluded from payouts.
- **Removed:** `ProcessOrgPayouts` (9bd601b); `payout_per_memory` / `RepTier` / `PayoutPerMemory` / `DebitTreasury` (926e8bb, "rip dead Treasury + RepTiers (D-ECON-STORAGE-MARKET)"). `MsgWithdrawTreasury` is absent. (`payout_per_serve` never existed in code — docs-only phantom, dropped.)
- tx.proto messages: exactly `MsgMintDailyEmission`, `MsgUpdateParams`, `MsgClaimContributorReward`.
- **Org creation = slot auction with 50% burn** (see FORWARD NOTE (a)); the burn is enabled by the org module's `Burner` perm (app.go:126).

### x/org — the old "design-only" line is wrong; all BUILT

The old TOPOLOGY's claim that x/org's `hub_endpoints` / `MsgSetServingInfo` were design-only is stale — **D-CHAIN-RESOLVED-HUB-ENDPOINT has landed** and everything is built:

- `hub_endpoints` — `StoredOrg` field 18 (state.proto:33).
- `MsgSetServingInfo` — tx.proto:22; msg_server.go:129-146; leader-only enforced.
- `MsgSetOrgConfig` — tx.proto:15; msg_server.go:200-237.

# 07 — wevibe-dashboard (`wevibe-server/wevibe-dashboard/`)

TypeScript/React Next.js dashboard. Pinned versions: `next` **15.5.23**, `react`/`react-dom` **^18.3.0**.

**Purpose — EXPANDED.** The prior doc's one-line purpose ("Moderation + organization management dashboard") under-reports the app: the dashboard now carries **moderation + organization management + extraction + sessions + recall-health + diagnostics + org-settings**. Extraction, sessions, recall-health, diagnostics, and org-settings appeared nowhere in the old trees.

Replaces the stale TOPOLOGY.md §"wevibe-server/wevibe-dashboard — Next.js UI" (lines 998–1174, dated 2026-06-14). Every tree below is PASS-2 authoritative (`1787048712-WO-TOP2-DASH-MCP.md`), re-verified at tree level against the code.

## `app/` — 58 files, 26 `route.ts`

```
app/
├── globals.css
├── layout.tsx
├── page.tsx                     → redirect('/my-org')
├── (auth)/
│   ├── connect-wevibe/page.tsx
│   └── login/page.tsx
├── (dashboard)/
│   ├── layout.tsx
│   ├── activity/page.tsx
│   ├── billing/page.tsx
│   ├── buy-org/page.tsx
│   ├── create-org/page.tsx
│   ├── diagnostics/page.tsx
│   ├── discover/
│   │   ├── page.tsx
│   │   └── [orgId]/page.tsx
│   ├── epochs/page.tsx
│   ├── faucet/page.tsx
│   ├── health/page.tsx
│   ├── join-requests/page.tsx
│   ├── members/page.tsx
│   ├── moderation/
│   │   ├── history/page.tsx
│   │   ├── new/page.tsx
│   │   └── reported/page.tsx
│   ├── my-org/page.tsx
│   ├── my-submissions/page.tsx
│   ├── notifications/page.tsx
│   ├── org-settings/page.tsx
│   ├── profile/page.tsx
│   ├── recall-health/page.tsx
│   ├── recovery/page.tsx
│   ├── sessions/
│   │   ├── page.tsx
│   │   └── extracted/page.tsx
│   └── settings/page.tsx
├── api/
│   ├── client-errors/route.ts
│   ├── errors/route.ts
│   ├── errors/clear/route.ts
│   ├── extract/route.ts
│   ├── extract/parked/route.ts
│   ├── extract/resume/route.ts
│   ├── extract/status/route.ts
│   ├── identity/adopt-local/route.ts
│   ├── lmstudio-models/route.ts
│   ├── mcp/decrypt-batch/route.ts
│   ├── mcp/embed-retrieval-card/route.ts
│   ├── mcp/history/route.ts
│   ├── mcp/queue/route.ts
│   ├── mcp-health/route.ts
│   ├── ollama-models/route.ts
│   ├── openrouter-embedding-models/route.ts
│   ├── openrouter-models/route.ts
│   ├── org-setup/route.ts
│   ├── org-setup/finalize/route.ts
│   ├── provision-recall/route.ts
│   ├── sessions/route.ts
│   ├── sessions/[id]/messages/route.ts
│   └── settings/
│       ├── route.ts
│       ├── embedding-readiness/route.ts
│       ├── readiness/route.ts
│       └── risk-appetite/route.ts
└── u/[wallet]/page.tsx          (public wallet profile)
```

Key corrections vs the old doc:

- **2 renames** (git `480038f`): `(dashboard)/epoch/` → `(dashboard)/epochs/`; `(dashboard)/recall-inspect/` → `(dashboard)/recall-health/`.
- **Root redirect**: `app/page.tsx:2` is `redirect('/my-org')` — NOT `/dashboard` or `/discover`. There is **no `/dashboard` URL** (the parenthesized `(dashboard)` route group contributes no URL segment). `/discover` **IS a real live URL** (`(dashboard)/discover/page.tsx` + `discover/[orgId]/page.tsx`) — it simply is not the redirect target.
- **11 NEW API routes** (the full 26-route `api/` tree is above; the old doc had 15): `client-errors`, `errors`, `errors/clear`, `extract/parked`, `extract/resume`, `extract/status`, `mcp-health`, `mcp/decrypt-batch`, `mcp/embed-retrieval-card`, `mcp/history`, `mcp/queue`.
- **2 NEW pages**: `(dashboard)/diagnostics/page.tsx` and `(dashboard)/org-settings/page.tsx`.
- `u/[wallet]/page.tsx` (public wallet profile) sits **outside both route groups**, oldest mtime in the tree — flagged as a leftover candidate (not removed).
- No dedicated wallet-sign-in or session-pickup pages exist: the split-card modal is a component rendered by `(auth)/login/page.tsx:78`, and session pickup is a lib feature surfaced via `notification-bell.tsx:30` + `extraction-queue.ts` resume.

## `components/` — 33 files, 12 subdirs (11 non-empty + EMPTY `backend/`)

```
components/
├── backend/                                  (EMPTY)
├── diagnostics/
│   ├── client-error-capture.tsx
│   ├── connection-error-modal.tsx
│   ├── error-fallback.tsx
│   ├── error-list.tsx
│   └── use-diagnostics-clear.ts              (.ts, not .tsx)
├── feed/my-feed.tsx
├── layout/
│   ├── keyword-seeding-banner.tsx
│   ├── notification-bell.tsx
│   ├── org-switcher.tsx
│   ├── sidebar.tsx
│   ├── tab-nav.tsx
│   └── topbar.tsx
├── memory/preference-score-card.tsx
├── moderation/
│   ├── leader-pipeline-panel.tsx
│   └── moderator-review-panel.tsx
├── notifications/notification-preferences-section.tsx
├── onboarding/
│   ├── identity-onboarding.tsx
│   └── split-card-signin.tsx
├── org-settings/keywords-section.tsx
├── pairing/pair-plugin.tsx
├── sessions/memory-review.tsx
├── ui/
│   ├── badge.tsx      ├── button.tsx     ├── card.tsx
│   ├── chip.tsx       ├── client-time.tsx├── modal.tsx
│   ├── searchable-model-combobox.tsx     ├── spinner.tsx
│   ├── states.tsx     ├── toggle.tsx     └── tooltip.tsx
└── wallet-connect-button.tsx
```

Key corrections vs the old doc:

- **12 NEW files** (exact): `diagnostics/` (client-error-capture, connection-error-modal, error-fallback, error-list, use-diagnostics-clear), `feed/my-feed.tsx`, `layout/keyword-seeding-banner.tsx`, `notifications/notification-preferences-section.tsx`, `onboarding/split-card-signin.tsx`, `org-settings/keywords-section.tsx`, `ui/chip.tsx`, `ui/toggle.tsx`.
- `diagnostics/use-diagnostics-clear.ts` is **`.ts`, not `.tsx`**.
- `onboarding/` is **pre-existing** (created 2026-06-06, `3b8754f`, holding `identity-onboarding.tsx`) — only `split-card-signin.tsx` inside it is new. New dirs are `diagnostics/`, `feed/`, `notifications/`, `org-settings/` (4) + the empty `backend/`.
- DEAD: `layout/mcp-connection-guard.tsx` (git-deleted `22326f5` 2026-07-08; the same commit also deleted `backend/dashboard-server-controls.tsx`, which is why `backend/` is now empty).

## `lib/` — 51 files, FLAT (no subdirs)

```
lib/
  canonical-body.ts          config.ts                    error-remediation.ts
  chain-client.ts            copy-error.ts                errors.ts
  copy-error.test.ts         diagnostics-clear-marker.ts  extract-shared.ts
  diagnostics-types.ts       draft-store.ts               extraction-queue.ts
  format.ts                  hub-client.ts                hub-error.ts
  identity-context.tsx       keyword-weights.ts           logger.ts
  mcp-errors.ts              mcp-rest.ts                  merkle.ts
  merkle.test.ts             nav-config.ts                opencode-session-events.ts
  opencode-session-events.test.ts   org-bridge.ts         org-context.tsx
  org-pricing.ts             org-role.ts                  passkey.ts
  preference-score.ts        provider-readiness.ts        role-colors.tsx
  session-model.ts           session-model.test.ts        session-types.ts
  settings.ts                settings-defaults.ts         sim-benchmark.json
  social-graph-client.ts     toast.ts                     types.ts
  use-dashboard-state.ts     verify-queue.ts              wallet-connect.ts
  wallet-seed-wrap.ts        wallet-seed-wrap.test.ts     wevibe-auth.ts
  wevibe-crypto.ts           wevibe-signing.ts            wevibe-submit.ts
```

Key corrections vs the old doc (git reconciliation: 39 documented − 2 dead + 1 wrong-path + 13 new = 51):

- DEAD: `deployment.ts` (git-deleted `c2f4395`, 2026-08-15 wallet-as-identity work).
- `mcp-client.ts` → `mcp-rest.ts`: a literal same-commit replace in `22326f5` (old deleted, new added together), with server proxy routes taking over the rest.
- `__tests__/` **never existed for `merkle.test.ts`** — git shows zero commits ever touching `__tests__/merkle.test.ts`; the file was added directly at `lib/merkle.test.ts` (`e98e5cf`). The old doc's `__tests__/merkle.test.ts` was a wrong path, not a relocated file. `__tests__/` only ever held `chain-client.test.ts` (deleted `864b9ba`, 06-29). **Delete `__tests__/` from the doc outright.**
- **13 NEW files** (exact, zero additional): `copy-error.ts` (+test), `diagnostics-clear-marker.ts`, `diagnostics-types.ts`, `extract-shared.ts`, `logger.ts`, `mcp-rest.ts`, `opencode-session-events.ts` (+test), `session-model.ts` (+test), `wallet-seed-wrap.ts` (+test).

## `e2e/` — 14 files (Playwright)

```
e2e/
├── billing.spec.ts
├── connection.spec.ts
├── fixtures.ts
├── global-setup.ts
├── helpers/
│   ├── mock-hub.ts
│   └── test-data.ts
├── leader-member-management.spec.ts
├── leader-org-management.spec.ts
├── leader-settings.spec.ts
├── moderation.spec.ts
├── navigation.spec.ts
├── page-objects/index.ts
├── reports.spec.ts
└── sessions.spec.ts
```

- DEAD: `mcp-tools.test.ts` (deleted `22326f5`, 07-08).
- 0 NEW — all 14 files date from the initial 05-21 import (`90ef569`); nothing added since.

## `package.json`

- **17/17 documented deps present** at: next 15.5.23; react/react-dom ^18.3.0; @cosmjs/stargate|proto-signing|amino|crypto|encoding 0.39.0; cosmjs-types 0.11.0; @noble/curves 2.2.0, @noble/ed25519 ^2.3.0, @noble/hashes ^1.8.0; better-sqlite3 ^11.0.0; sonner 2.0.7; wevibe-sdk-wasm; @playwright/test 1.60.0; tailwindcss ^3.4.0.
- NEW: `react-error-boundary` **6.1.2** (dependencies).
- `wevibe-sdk-wasm` is vendored: `file:./vendor/wevibe-sdk-wasm`.
- **NO external wallet/passkey npm libs** — wallet connect and passkeys are custom `lib/wallet-connect.ts` + `lib/passkey.ts`.
- `overrides: { "postcss": "^8.5.15" }`.

## Key wiring facts

- **Wallet-as-identity sign-in (Option A)** — `lib/wevibe-auth.ts`: `wrapSeedWithWallet` (:457–469) wraps the seed under the wallet identity with `credentialIdB64 = 'wallet:' + addr`, `kind: 'wallet'`; unlock at :335–340; cross-device adoption via `adoptIdentityFromWallet` (:716–769). Decision: DECISIONS.md:3904 (D-WALLET-AS-IDENTITY-SIGNIN-2026-08-15 — "no passkey root underneath", Option A).
- **Session pickup via `OPENCODE_DB_PATH`** — `lib/opencode-session-events.ts` `getDbPath()` (:6–9) reads `OPENCODE_DB_PATH`, falling back to `~/.local/share/opencode/opencode.db`; `app/api/extract/route.ts` forwards `session_db_path` to the MCP `/v1/extract` (:245, :261) and maps bench title → org_id (:270–276).
- **MCP endpoint is `:4450`, not `:4550`** — `getMcpHttpUrl()` defaults to `http://127.0.0.1:4450` (`lib/config.ts:14`, `DEFAULT_WEVIBE_MCP_HTTP_URL`), env-overridable via `WEVIBE_MCP_HTTP_URL` (:95–100); `docker-compose.yml:236` uses `host.docker.internal:4450`. A two-day `:4550` default was **reverted by `6faac93`** (2026-08-17, "decouple sign-in from the bench MCP") — `:4550` is the bench MCP only (AGENTS.md §2.1).
- **org-setup flow is LIVE end-to-end** — the `WEVIBE_DEPLOYMENT=server → 422 ORG_LOCAL_ONLY` gates were removed, but the flow remains wired and gated: `lib/org-bridge.ts:88,125` → `app/api/org-setup/route.ts:135-136` + `finalize/route.ts:103-104` → MCP `/v1/org-setup`, with the MCP-token and requester-identity gates intact. `buy-org`/`create-org` pages are not dead (wired at `sidebar.tsx:112-123`, `my-org:207`).

---

*Sources: PASS-2 authoritative `1787048712-WO-TOP2-DASH-MCP.md` (trees + verdicts, wins on any conflict); PASS-1 `1787047623-WO-TOPOLOGY-DASHBOARD.md` (wiring anchors + citations).*

# 08 — wevibe-mcp

**TypeScript** MCP client + HTTP API server. The local agent-side trust boundary of WeVibe: it owns
identity, crypto, extraction, recall, and serving — and talks to the hub over HTTP.

- Language: TypeScript (`package.json` typescript ^5.4.0, build = `tsc`)
- HTTP API binds **127.0.0.1:4450** by default, env-overridable via `WEVIBE_MCP_HTTP_PORT`
  (`src/config.ts:85-87`)
- Companion repo: `wevibe-opencode-plugin` (`plugins/wevibe-plugin.ts`, `tui/wevibe.tsx`,
  `bin/install-opencode.ts`; npm script `install-opencode`) — the plugin owns session state
  (`D-SIDECAR-PLUGIN-OWNS-STATE`), not the MCP

## src/ — 87 files (86 `.ts` + 1 `.md`); 65 top-level `.ts` + 4 dirs

```
src/
  admin.ts               artifact-extract.ts    artifact-policy.ts
  artifact-transform.ts  attestation.ts         auth.ts
  biometric.ts           blacklist.ts           canonical.ts
  config.ts              contribution.ts        crypto-utils.ts
  crypto.ts              denial-queue.ts        deserialize.ts
  embed-card.ts          embedding-config.ts    embedding.ts
  event-signing.ts       extract-jobs.ts        extraction-integrity.ts
  extraction-presets.ts  extraction.ts          failure-episodes.ts
  guard.ts               http-body.ts           http-server.ts
  hub-fetch.ts           hub-resolver.ts        identity-runtime.ts
  identity-sidecar.ts    key-store.ts           keychain.ts
  llm-ollama.ts          llm-openai-compat.ts   llm.ts
  logger.ts              manifest.ts            memory-admission.ts
  model-context.ts       moderation.ts          openrouter-catalog.ts
  orcarouter.ts          org-client.ts          pair-crypto.ts
  pairing-export.ts      pending-vault.ts       query-scrub.ts
  recovery.ts            retrieval-card.ts      retrieve-cli.ts
  retrieve-types.ts      risk-appetite.ts       serve-ref-store.ts
  serve-signing.ts       served-memory-store.ts server.ts
  session-db-substrate.ts  session-substrate.ts session-token.ts
  session.ts             trust-panel.ts         types.ts
  umbral.ts              vault.ts

  cli/     → bind.ts
  gstv/    → SYNC.md, chain.ts, engine.ts, episodes.ts, gitignore.ts, ops.ts,
             paths.ts, predicate.ts, receipts.ts, routes.ts, signal-keys.ts,
             spool.ts, store.ts, types.ts, unlock.ts, walk.ts   (15 .ts + 1 .md)
  mc1/     → index.ts, keywords.ts, path-regexes.ts, paths.ts, schema.ts   (5)
  types/   → EMPTY — no index.ts. The old TOPOLOGY's "shared MCP type exports"
             claim for this dir is false; it has zero entries.
```

**DEAD (removed, do not re-document):** `llm-sampling.ts` — git-deleted `d683baa` (R-13 one-path),
superseded by `llm-openai-compat.ts`; the old doc still listed it.

## tests/ — 101 `.test.ts` + 1 fixture

86 top-level + 10 `integration/` + 3 `security/` + 2 `production/`.

Prefix families (counts exact): `gstv-*` **17**, `mc1-*` **3**, `umbral-*` **4** (family incl.
`umbral.test.ts`; strict hyphen-glob = 3).

```
tests/   (top level — 86 .test.ts)

  artifact-extract.test.ts        artifact-policy.test.ts
  artifact-transform.test.ts      bind.test.ts
  biometric.test.ts               blacklist.test.ts
  canonical.test.ts               contribution.test.ts
  deserialize.test.ts             egress-policy.test.ts
  embed-card.test.ts              embedding-config.test.ts
  embedding-prefix.test.ts        embedding.test.ts
  env-flag.test.ts                event-signing-parity.test.ts
  extract-defaults.test.ts        extract-jobs.test.ts
  extraction-integrity.test.ts    extraction-presets.test.ts
  extraction-provider-selection.test.ts
  extraction-zero-progress-gate.test.ts
  extraction.test.ts              failure-episodes.test.ts
  free-lapse-classifier.test.ts   guard.test.ts
  hub-fetch.test.ts               hub-resolver.test.ts
  identity-sidecar.test.ts        key-store.test.ts
  leader-recall-invariant.test.ts llm-openai-compat.test.ts
  manifest.test.ts                model-context.test.ts
  moderation-approval.test.ts     moderation.test.ts
  openrouter-catalog.test.ts      orcarouter.test.ts
  org-client.test.ts              pair-crypto-reverse.test.ts
  pair-crypto.test.ts             pending-vault.test.ts
  recovery-status.test.ts         recovery.test.ts
  retrieval-card.test.ts          retrieve-cli-harvest.test.ts
  retrieve-embedding-dim.test.ts  risk-appetite.test.ts
  rotation.test.ts                seed-derivation-parity.test.ts
  serve-signing-parity.test.ts    served-memory-store.test.ts
  server-tools.test.ts            session-db-substrate.test.ts
  session-substrate.test.ts       session-token.test.ts
  session.test.ts                 signed-auth-header.test.ts
  steg-scan.test.ts               threshold-recovery.test.ts
  vault.test.ts                   wasm-crypto.test.ts

  gstv-*.test.ts (17): gstv-chain, gstv-engine, gstv-episodes, gstv-gitignore,
    gstv-ops-extended, gstv-ops, gstv-paths, gstv-predicate, gstv-receipts,
    gstv-routes, gstv-signal-keys, gstv-spool-conformance, gstv-spool,
    gstv-store, gstv-types, gstv-unlock, gstv-walk
  mc1-*.test.ts (3): mc1-core, mc1-envelope, mc1-keyword-boost
  umbral-* (4 family): umbral.test.ts, umbral-native-parity.test.ts,
    umbral-no-leak.test.ts, umbral-packaging.test.ts

  fixtures/   → spool-v1.plugin-produced.jsonl   (ONLY entry — no other fixtures exist)
  integration/ (10): capstone, e2e-flow, http-auth, http-body-guard,
    http-decision-notes, http-extract-async, http-reports, http-serves-confirm,
    http-serves, openrouter-catalog.integration
  security/ (3): attack-scenarios, query-scrub, recall-pipeline
  production/ (2): embedding-quality, hub-resilience
```

**DEAD (removed, do not re-document):** `sidecar.test.ts` + `sidecar-no-leak.test.ts` — deleted
`878e7cc` (Umbral WASM pivot; superseded by the `umbral-*` family) · `llm.test.ts` +
`production/sampling-provider.test.ts` — deleted `d683baa` (superseded by
`llm-openai-compat.test.ts`). `production/` still exists with the 2 files above.

## HTTP API — 26 routes (9 documented + 17 undocumented)

Sole dispatcher: `handleRequest` @ `src/http-server.ts:3192`. Bearer session-token auth
(`session-token.ts:68`, `~/.wevibe/mcp-session-token`; `extractBearer` :85; `authorize`
http-server.ts:152).

| Path | Method | file:line | Class |
|---|---|---|---|
| `/v1/health` | GET | 3201 | documented |
| `/v1/recall` | POST | 3206 | documented |
| `/v1/serves` | POST | 3261 | documented |
| `/v1/reports` | POST | 3278 | documented |
| `/v1/org-setup` | POST | 3246 | documented |
| `/v1/org-setup/finalize` | POST | 3251 | documented |
| `/v1/provision-recall` | POST | 3256 | documented |
| `/v1/identity/pubkeys` | GET | 3329 | documented · gated `WEVIBE_BENCH_ENDPOINTS=1` |
| `/v1/submit` | POST | 3334 | documented · gated `WEVIBE_BENCH_ENDPOINTS=1` |
| `/v1/extract` | POST | 3226 | undocumented — the missing 17th (primary extraction-submit) |
| `/v1/extract/status/{jobId}` | GET | 3211 (prefix; slice :1182) | undocumented |
| `/v1/extract/parked` | GET | 3216 | undocumented |
| `/v1/extract/resume` | POST | 3221 | undocumented |
| `/v1/extract/defaults` | GET | 3231 | undocumented |
| `/v1/identity/export-pairing` | POST | 3236 | undocumented |
| `/v1/shutdown` | POST | 3241 | undocumented |
| `/v1/orgs/{id}/serves/confirm` | GET | 3266–3267 (regex) | undocumented |
| `/v1/orgs/{id}/outcome-events` | POST | 3272–3273 (regex) | undocumented |
| `/v1/decision-notes` | POST | 3283 | undocumented |
| `/v1/denials` | POST | 3288 | undocumented |
| `/v1/mod/queue` | POST | 3293 | undocumented |
| `/v1/mod/decrypt-batch` | POST | 3298 | undocumented |
| `/v1/mod/embed-retrieval-card` | POST | 3303 | undocumented |
| `/v1/mod/history` | GET **+ POST** | 3308 | undocumented |
| `/v1/gstv/goal` | GET | 3313 (prefix; `?repo_root=`) | undocumented |
| `/v1/gstv/seal` | POST | 3321 | undocumented |

Notes: `/v1/identity/pubkeys` + `/v1/submit` are bench-only, gated on
`WEVIBE_BENCH_ENDPOINTS==='1'` (`:92`) — never present in a daily-driver install.
`/v1/mod/history` accepts both GET and POST. `/v1/extract/status/` and `/v1/gstv/goal` are prefix
matches; the two `/v1/orgs/{id}/...` routes are regex matches. No route exists outside this table.

## Key pivots & features

**Umbral PRE — in-process WASM, no subprocess.** The old sidecar model is dead: `sidecar.ts` deleted,
`WEVIBE_UMBRAL_SIDECAR_BIN` dropped (0 executable references remain). Umbral loads the vendored
WASM in-process: `vendor/umbral-wasm/wevibe_umbral_wasm.js` require'd at `src/umbral.ts:53`
(wasm-bindgen glue). Pivot commit `878e7cc` (DECISIONS.md:229).

**Secret-storage split — keychain vs encrypted file.** The identity seed
(`identity-seed-v1`, service `wevibe-network`) lives in the OS keychain via **`@napi-rs/keyring`**
(`keychain.ts:20`). The PRE secret key lives in an **AES-256-GCM encrypted file keystore** at
`~/.wevibe/keys/keys.json` (`key-store.ts`; env override `WEVIBE_KEYSTORE_PATH`; v1 format:
nonce/ciphertext/tag). **Keytar never existed in this codebase** — `KEYTAR_SERVICE` at `auth.ts:16`
is a legacy constant name only; any doc saying keytar is false.

**orcarouter.ts — cloud extraction routing (undocumented in old TOPOLOGY).** API key read from
`~/.local/share/opencode/auth.json` (`orcarouter.ts:75`, `parsed['orcarouter']['key']` :99);
live `GET /v1/models` for limits (:212) with a 15 s timeout, effective limit = min(live, static).

**Tool roster — exactly 3 tools.** `setup_org`, `wevibe_status`, `wevibe_set_provider_policy`.
Nothing else is registered (DECISIONS.md:3196's claim that `wevibe_set_risk_appetite` "stays
registered" is stale — it is NOT registered). `getProviderPolicy`/`setProviderPolicy` live
(`risk-appetite.ts:69,73`); `getRiskAppetite`/`setRiskAppetite` removed.

**Ripped recall graveyard — all 0 src matches.** `recallTimeScan`, `gateMemories`,
`rerankByRelevance`, `disambiguateMemories`, `buildElicitationPreview`,
`formatMemoryPresentation`, `FormattedMemory`, `WEVIBE_ALLOW_UNREVIEWED` — dead, do not document.

**Lazy boot.** No biometric prompt at boot (`server.ts:330-338` explicit no-boot-biometric block);
biometric unlock is first-use only (`key-store.ts:262,290,336`). PRE membership
sync/registration is deferred to first use as well.

# 9. wevibe-guard

Rust YARA-X scanning service: the content/secret guard that scans for prompt injection,
embedded credentials, and exfiltration patterns before content reaches an agent's context.

## Layout

All 13 files verified at their stated paths (report 1787047705, §wevibe-guard — NO DRIFT);
zero undocumented files in any of the three trees:

```
src/
├── lib.rs            # Library entry
├── main.rs           # Binary entry
├── rules/
│   └── injection.yar # YARA rule source
├── scanner.rs        # Main scanner implementation
├── flags.rs          # Flag handling
├── credentials.rs    # Credential detection
└── exfiltration.rs   # Exfiltration detection

benches/
└── scan_bench.rs     # Benchmarking

tests/
├── unit_tests.rs
├── fixture_compliance.rs
├── fixtures_good.json
├── fixtures_bad.json
└── fixtures_redteam.json
```

Note: `injection.yar` lives under `src/rules/` — there is no top-level `rules/` directory.

## Role in WeVibe

Invoked by the MCP (`spawnSync` against the binary path in `WEVIBE_GUARD_BIN`) to scan
recalled content and secrets. It is non-blocking on the recall path — a guard, not a gate
that halts retrieval.

## Status

Verified accurate as of the 1787047705 audit; no doc/code drift, no dead files. The prior
TOPOLOGY section required no changes beyond this rewrite.

# 10. wevibe-sdk

Rust + WASM SDK: the crypto/identity primitives (secp256k1, ed25519, hashes) shared across
WeVibe, compiled to WebAssembly for browser-side use.

## Layout

Cargo workspace with exactly two members (Cargo.toml:1-3):

| Member | Contents |
|---|---|
| `crates/wevibe-sdk-core` | `src/{lib,crypto,identity,secp256k1,types,errors}.rs` |
| `crates/wevibe-sdk-wasm` | `src/lib.rs` (wasm-bindgen wrapper over core) |

7/7 source files verified against the audit (report 1787047705, §wevibe-sdk) — NO DRIFT in the tree.

Additional top-level artifacts (additions noted by the audit, not contradictions — they are
generated output and fixtures, not Rust crates):

- `pkg/`, `pkg-nodejs/` — wasm-bindgen JS build output
- `protocol/test_vectors/` — shared crypto test vectors
- `crates/wevibe-sdk-core/tests/crypto_tests.rs` — core-crate crypto tests
- `docs/`

## Role in WeVibe

Provides the client-side cryptographic primitives used during local decrypt/sanitize of recalled
memories. The WASM build is vendored into the dashboard at
`wevibe-dashboard/vendor/wevibe-sdk-wasm` and consumed via a `file:` dependency, so all curve and
hash operations run in the browser — ciphertext never reaches the dashboard server for decryption.

## Status

Verified accurate as of the 1787047705 audit; no doc/code drift, no dead files, no changes
required beyond this rewrite.

## 11. wevibe-faucet

**LIVE, hub-required infrastructure — NOT "dev-only".** The old framing ("dev-only, never ships
to mainnet") survives only for the *external HTTP endpoint* the hub optionally exposes. The
faucet service itself is a hard startup dependency of the hub: the hub does not start unless
the faucet is healthy, and the hub's ungated relay path calls it in production flow to gas-fund
organization signers. Reframe verified on-touch (PASS-2, report 1787048710): all three claims —
hard `depends_on`, container-only port, ungated top-up path — CONFIRMED.

### 11.1 What it is

A small Go service that holds a funded wallet and signs `bank/MsgSend` transactions of `uvibe`
to requested addresses, talking raw CometBFT JSON-RPC (no Cosmos client library server). Entry
point is `wevibe-faucet/main.go`: `config.Load` → `NewCosmosBroadcaster` → `NewService` +
`Initialize` (pre-fetches the faucet account number/sequence) → HTTP server on `LISTEN_ADDR`,
graceful shutdown on SIGINT/SIGTERM. The service itself has **no authentication** — no API key,
no middleware; protection is the per-address rate limit plus deployment topology (no host port).

### 11.2 Repo layout

```
wevibe-faucet/
├── main.go                        # entry point, wiring + graceful shutdown
└── internal/
    ├── config/config.go           # 6 env fields, see §11.5
    ├── faucet/faucet.go           # Service (rate limiter + fund flow) + CosmosBroadcaster
    ├── faucet/faucet_test.go      # unit tests
    └── server/server.go           # HTTP surface: /v1/health, /v1/fund
```

Five Go files, nothing else.

### 11.3 HTTP surface (internal/server/server.go)

Two routes, registered via Go 1.22 method-pattern mux (`server.go:21-22`). All responses are
`application/json`.

**`GET /v1/health`** — used as the Docker healthcheck.

- 200: `{"status":"ok","faucet_address":"wevibe1…","chain_id":"wevibe-local-1"}`
- 500: `{"error":"<chain-id fetch failure>"}`

**`POST /v1/fund`** — request `{"address":"wevibe1…","amount":<int64 uvibe>}`. The decoder uses
`DisallowUnknownFields`.

- 200: `{"tx_hash":"<hex>","funded":true}`
- 400: `{"error":"invalid JSON body"}` (malformed or unknown fields)
- 400: `{"error":"address and amount are required"}` — the amount>0 guard (empty address is
  rejected by the same check at the HTTP layer; `Fund()` additionally enforces `amount > 0`
  at the service boundary, `faucet.go:114`)
- 429: `{"error":"rate limit exceeded for address <address>"}`
- 500: `{"error":"<message>"}` for any other failure (invalid address, RPC/broadcast errors)

### 11.4 Rate limiter semantics (internal/faucet/faucet.go)

Per-address, in-memory, **1 request per 60 s** with defaults (config: `RATE_LIMIT_MAX=1`,
`RATE_LIMIT_WINDOW_SECONDS=60`). Three properties worth knowing precisely:

1. **Anchored window, not sliding.** The window is anchored to the first request's timestamp:
   a funded request increments the count but keeps the original `requested` time
   (`faucet.go:161-166`), so the window never slides forward on activity.
2. **Same-amount repeat within the window is idempotent.** A repeat for the same address and
   same amount returns the stored record — `{"tx_hash":"<stored hash>","funded":true}` — without
   broadcasting again (`faucet.go:128-132`). At `max=1` any *different* amount within the window
   hits 429.
3. **Records never evict.** The `perAddress` map grows monotonically for the process lifetime;
   no TTL/eviction path exists. Fine for a local-scale service; a bounded-cache concern only at
   very large address counts.

Sequence handling is defensive: on a broadcast failure matching "incorrect account sequence",
the signer state is re-queried from chain and the send retried once (`faucet.go:143-157`).
`CHAIN_ID` resolution caches after first successful RPC `status` fetch (`faucet.go:274-301`).

### 11.5 Configuration (internal/config/config.go) — exactly 6 fields

| Env var | Default | Notes |
|---|---|---|
| `CHAIN_RPC` | `tcp://wevibed:26657` | `tcp://` scheme is normalized to `http://`; docker-compose overrides with `tcp://wevibe-chain:26657` (compose:202) |
| `CHAIN_ID` | (optional) | if unset, fetched once from the node's RPC `status` and cached |
| `FAUCET_MNEMONIC` | — **required**, startup fails without it | derived in-memory via HD path `m/44'/118'/0'/0/0`, keyring backend = memory only |
| `LISTEN_ADDR` | `:4470` | |
| `RATE_LIMIT_WINDOW_SECONDS` | `60` | non-positive / unparsable values fall back to 60 |
| `RATE_LIMIT_MAX` | `1` | non-positive / unparsable values fall back to 1 |

There is **no auth configuration at all** — no token, key, or allowlist field exists.

### 11.6 Deployment (wevibe-server/docker-compose.yml)

- **Service block `wevibe-faucet`** (compose:194-215): `LISTEN_ADDR ":4470"`, `CHAIN_RPC
  tcp://wevibe-chain:26657`, `CHAIN_ID wevibe-local-1`, `RATE_LIMIT_WINDOW_SECONDS 60`,
  `RATE_LIMIT_MAX 1`, `FAUCET_MNEMONIC` from env with a local-dev fallback mnemonic. Itself
  `depends_on: wevibe-chain (condition: service_healthy)`. Healthcheck hits
  `http://127.0.0.1:4470/v1/health`.
- **Container-only port — NO host `ports:` key.** The faucet is reachable only inside the
  compose network. Port-parity hazard: host port **4470 IS published** — by
  `wevibe-social-graph` (compose:183, `${WEVIBE_BIND_HOST:-127.0.0.1}:4470:4470`). Same port
  number, different service, different network namespace; don't confuse them.
- **Hard startup dependency of the hub** (compose:160-161):
  `wevibe-faucet: condition: service_healthy`. The hub container will not start without a
  healthy faucet.

### 11.7 Two hub consumption paths (wevibe-server)

**(1) External endpoint — dev-gated.** `POST /v1/faucet/fund` on the hub, registered inside the
`WEVIBE_DEV_ENDPOINTS == "true"` gate (`wevibe-hub/cmd/wevibe-hub/main.go:243` reads the env,
route registered at 282-284). This is the path the old "dev-only" verdict describes; the gate
commit `fbc5549` (2026-06-12) touched only docker-compose.yml + main.go.

**(2) Org-signer gas top-up — UNGATED, production path.** The relay worker tops up each
organization's signing wallet before broadcasting:
`serveRelayWorker` → `relayPendingEventsByOrg` → `SubmitRelayBatch` →
`BroadcastMsgsForOrgServingCommit` → `broadcastMsgsForOrg` calls `loadOrgSignerState`
(`chain/broadcast.go:680`, faucet call at :697) and `ensureOrgSignerBalance` (func at
`broadcast.go:738`, faucet calls at :745/:752/:762) → `fundFromFaucet` (`chain/faucet.go:13`,
`POST /v1/fund` at :33). Thresholds: `MIN_GAS_BALANCE = 5_000_000` uvibe,
`TOPUP_AMOUNT = 20_000_000` uvibe (`broadcast.go:41-42`). A full negative sweep confirmed NO
dev-gate on this path: `devEndpointsEnabled` is referenced ONLY at `main.go:282/387`.

### 11.8 Verdict

- "dev-only, never ships to mainnet" is true **ONLY for the external HTTP endpoint**
  (`POST /v1/faucet/fund`, gated by `WEVIBE_DEV_ENDPOINTS`).
- The **service itself is hub-required infrastructure**: hard `depends_on` gates hub startup,
  and the ungated org-signer top-up path consumes it on every relay cycle. It is reachable in
  production deployments regardless of any dev flag.

# 12. wevibe-umbral — hybrid workspace, dual-live WASM + sidecar

**Centerpiece:** `wevibe-umbral` is a HYBRID WORKSPACE, and BOTH decrypt/re-encrypt paths are LIVE.
The hub-side re-encryption relay runs as a native gRPC sidecar container (`wevibe-umbral:4460`); the
MCP runs the same crypto in-process from a vendored WASM build. Neither path is legacy — they serve
different trust boundaries and divide work along a hard split: the MCP derives, mints, encrypts, and
decrypts locally; the hub sidecar performs the ONLY `ReEncrypt`. The MCP never calls the hub's
ReEncrypt.

## 12.1 Repo crate layout — NOT a single crate

The old "single crate with lib + bin" description is superseded. Current layout:

- **Root package `wevibe-umbral`** — lib + bin still at `src/`, which holds exactly six files:
  `lib.rs`, `main.rs`, `service.rs` (gRPC impl), `store.rs` (kfrag store), `cli.rs`, `generated.rs`
  (tonic-generated server trait + client).
- **`[workspace] members = ["crates/core"]`** (Cargo.toml:18) — `wevibe-umbral-core`, holding
  `crypto.rs` + `ops.rs`. Root `src/crypto.rs` was MOVED to `crates/core/src/crypto.rs`;
  `src/lib.rs:9` re-exports `wevibe_umbral_core::crypto` so old paths keep resolving.
- **`exclude = ["crates/wasm"]`** (Cargo.toml:23) — `wevibe-umbral-wasm` is a standalone workspace
  root (its own `[workspace]` at crates/wasm/Cargo.toml), built as a `cdylib` and vendored into the
  MCP (`vendor/umbral-wasm`).
- Rust deps: `umbral-pre`, `tonic`, `tonic-prost`, `prost`, `dashmap`, `clap`. Dockerfile: release
  build, `EXPOSE 4460`, `CMD serve 0.0.0.0:4460`.

The WASM and the :4460 service compile from the SAME `crates/core` source — byte-compatible
(`wevibe-mcp/src/umbral.ts:15-17`).

## 12.2 The two live paths + callers

### (a) Hub-side re-encryption relay — native gRPC sidecar container `wevibe-umbral:4460`

NOT retired. Compose service (docker-compose.yml:95, built from `Dockerfile.umbral-sidecar`),
started before the hub (`depends_on`). Hub callers:

| Hub call site | Operation |
|---|---|
| `retrieval.go:388` (inside QueryMemories) | `ReEncryptForMember` — recall path; returns cfrag + capsule + umbral_ciphertext (retrieval.go:379,395,396) |
| `members.go:588` | `StoreKFrag` — leader kfrag delivery |
| `watcher.go:514` (from MsgRemoveMember case :398) | `OnMemberRemoved` → `DeleteKFrags` |

Proto contract `proto/umbral/v1/sidecar.proto:10-31` = exactly 5 RPCs: **StoreKFrag / ReEncrypt /
DeleteKFrags / DeleteOrgKFrags / Health** — all implemented (`service.rs:38-218`; generated server
trait at `generated.rs:307`, client at `:86`). Note: `DeleteOrgKFrags` and `Health` are wired but
have no live production caller; the hub-side gRPC client wrapper exposes 4 methods and StoreKFrag goes
through the raw generated client (`internal/umbral/service.go:30`, methods at :29/:56/:92).

### (b) MCP in-process crypto — WASM

`vendor/umbral-wasm/wevibe_umbral_wasm_bg.wasm` (279,045 bytes), loaded at `src/umbral.ts:53` via
plain `require` — **no child_process/execFile, no subprocess**. Callers:

- `moderation.ts:242` → `umbralEncrypt` — approval/embed DEK encrypt to the epoch PK.
- `org-client.ts` — epoch-keypair derive (:633, :811), kfrag mint (:824), recall
  `decrypt-reencrypted` (:529).

### The split

- MCP derives the epoch keypair, mints kfrags, encrypts, and decrypts-reencrypted **locally** via WASM.
- The hub sidecar does the **only** `ReEncrypt`.
- MCP never calls hub ReEncrypt (no grpc import anywhere in MCP `src/`); the WASM `reencrypt` export
  is interface-only — no MCP wrapper (`.d.ts:27-29`: "Not currently called by the MCP — the hub
  relays re-encryption").

## 12.3 RIPPED/DEAD — D-LEADER-SIDE-UMBRAL-MINT

The leader-side mint path was removed (commits `811f0fb` proto/RPC, `66b42a1` hub service). All of
the following are ABSENT — absence verified in proto, service.rs, generated.rs, and hub router
(`cmd/wevibe-hub/main.go:256-399` has ZERO `/v1/internal/` routes):

- `umbralService.GenerateEpochKeyPair` / `RegisterMember` (zero occurrences in hub Go; `66b42a1`
  deleted `reencrypt.go`).
- `POST /v1/internal/epoch-keypair`, `POST /v1/internal/orgs/{orgID}/kfrags`.
- Sidecar gRPC `GenerateKeyPair` / `GenerateKFrags` RPCs.

**Replacement:** kfrag delivery is `POST /v1/orgs/{orgID}/members/{pubkey}/kfrag`
(main.go:307 → members.go:500 `StoreMemberKFrag`) — leader-signed, role=leader gate.

Also dead/unreachable:

- MCP `src/sidecar.ts` — DELETED (commit `878e7cc`); `WEVIBE_UMBRAL_SIDECAR_BIN` dropped. The env
  name survives only in 11 doc/comment/regression occurrences, 0 executable (regression assert at
  `tests/umbral.test.ts:19-20`). `dist/sidecar.js` is a stale compiled leftover — exists but
  unreachable, imported by nothing (purge by rebuilding dist).
- MCP `rotateEpoch` has a DEAD defensive read of optional `respBody.epoch_sk`/`epoch_pk`
  (org-client.ts:1156-1176) — the hub returns neither field, so it never fires.

## 12.4 KFrag store

DashMap (`store.rs:49`) + disk persistence. Env `WEVIBE_UMBRAL_KFRAG_STORE`, default
`/data/kfrags.json` (store.rs:12,56-60). Atomic persist sequence (store.rs:327-376):
write temp file → chmod 0600 → write_all → fsync → rename over target → chmod 0600 → fsync parent
dir. Docker volume `wevibe_umbral_kfrags:/data` (docker-compose.yml:104, :294). Kfrags exist ONLY on
sidecar disk — never in the hub DB.

## 12.5 Epoch key derivation

```
epoch_sk = HKDF-SHA256(K_master, salt = "" , info = "wevibe-umbral-epoch-" + epoch, len = 32)
```

- Derivation lives UPSTREAM in the MCP, not the Rust repo: `org-client.ts:53-63` uses
  `hkdfSync('sha256', masterKey, salt=empty, info="wevibe-umbral-epoch-"+epoch, 32)` — info string
  is decimal-ASCII epoch (:59), salt empty (:58), 32-byte output. The Rust repo contains zero HKDF
  code; it consumes an already-derived 32-byte hex `--seed`.
- The 32 bytes become a canonical secp256k1 scalar via `SecretKey::try_from_be_bytes`
  (`crates/core/src/crypto.rs:60`) — NOT `SecretKeyFactory::from_secure_randomness` (zero
  occurrences; any doc claiming a "SecretKeyFactory workaround" is wrong).
- **The hub NEVER receives or holds `epoch_sk`**: zero `epoch_sk`/`EpochSk`/`epoch_secret` in hub
  Go; the hub RotateEpoch response returns only `{status, buffered_moved}` (orgs.go:356-359); the hub
  stores `umbral_pk` hex + capsule/ciphertext only.

## 12.6 Secret material — keychain correction

- OS-keychain backend is **`@napi-rs/keyring`** v1.3.0 (package.json:52) — **NOT keytar**; keytar
  never existed in this repo (0 occurrences, empty `git log -S keytar`; `KEYTAR_SERVICE`/`KeytarLike`
  are legacy identifiers only).
- But the keychain does NOT hold the PRE key: `@napi-rs/keyring` stores the **identity seed**
  (`identity-seed-v1`, keychain.ts:20, key-store.ts:230/241).
- The **PRE secret key** (`pre-identity-key`) lives in the AES-256-GCM encrypted FILE keystore
  `~/.wevibe/keys/keys.json` via `fileStore` (key-store.ts:54-65,87-149; auth.ts:47,63).
- Prior TOPOLOGY.md text ("via keytar") was wrong twice over: wrong package AND wrong medium/which-key.

## 12.7 CLI commands (non-RPC)

`main.rs:18-78` `Commands` enum → `cli.rs` `cmd_*` handlers → core `ops.rs` hex ops:
`derive-epoch-keypair`, `generate-kfrags`, `encrypt`, `reencrypt`, `decrypt-reencrypted`, `serve`.
All crypto commands run locally; none invoke the gRPC contract.

## 12.8 Deployment note — port 4460 is CONTAINER-ONLY

Sidecar port 4460 has **no `ports:` mapping** in docker-compose; the hub reaches it over compose DNS
as `wevibe-umbral:4460`. The Go default target `127.0.0.1:4460` applies only to a host-process
sidecar, not the containerized deployment.

**Stale docs to ignore** (still describe mint RPCs as live): `wevibe-umbral/README.md:35-36`,
`wevibe-umbral/docs/TOPOLOGY.md:98-99` and `:173`.

# 13. wevibe-social-graph — D-SG-2 Reference Display Client

> Section 13 of the TOPOLOGY.md rewrite. Facts verified on-disk by WO-TOP2-REPO-LEVEL (PASS-2, authoritative) and WO-TOPOLOGY-SERVICES-SOCIAL (PASS-1); PASS-2 wins on any conflict.

## 13.1 Framing — D-SG-2 reference display client (supersession of D-13.4)

**This repo is NOT a centralized service.** The old TOPOLOGY described it as a "Public Profile Service", the implementation of D-13.4's centralized "Social Graph Service". That decision is dead:

- D-13.4 carries the banner `SUPERSEDED (DMO-030, 2026-06-10) by D-SG-2 + D-MISSION-INVARIANT` (`DECISIONS.md:2153-2155`).
- D-SG-2 (`DECISIONS.md:2665-2668`) redefines the social graph as an **OPEN-SOURCE, forkable, self-hostable DISPLAY client** that **reads chain state via RPC** — D-SG-2 is the social-graph read contract; D-MISSION-INVARIANT supplies the exit-guarantee constraint it must not violate.

Canonical framing: **the `wevibe-social-graph` repo is the D-SG-2 reference display client.** It is a display layer over chain state — not an authority and not a store of record. Anyone may fork and self-host it; the chain remains the sole authority for whatever it renders, consistent with the Four Exit Guarantees (no party can read/withhold/rewrite/kill an organization's knowledge through this component — it holds only self-declared profile data and public chain reads).

The code itself still matches the old description functionally (SQLite profile CRUD + chain-REST contributor-stats enrichment), so the section survives — only the role label changes. Never describe this repo as "centralized Social Graph Service" or "Public Profile Service" again.

## 13.2 Language & module

- **Go 1.25.9**; module `github.com/wevibe-network/wevibe-social-graph` (`go.mod:1`).
- **No cosmos-sdk dependency** — chain data is fetched over plain REST/RPC via a small hand-rolled client, not a Cosmos SDK.

## 13.3 Routes (6, verbatim)

| Method | Path | Notes |
|---|---|---|
| GET | `/v1/health` | |
| POST | `/v1/profiles` | create profile |
| GET | `/v1/profiles/batch?wallets=` | batch fetch, comma-joined `wallets` query param (`server.go:178`) |
| GET | `/v1/profiles/{wallet}` | public read |
| PATCH | `/v1/profiles/{wallet}` | gated — wallet-ownership proof, §13.6 |
| GET | `/v1/stats/contributor/{pubkey}` | contributor stats from chain |

Registrations at `internal/server/server.go:26-30` (`/v1/health`, `/v1/profiles/batch`, `/v1/profiles`, `/v1/profiles/`, `/v1/stats/contributor/`). GET and PATCH on `/v1/profiles/{wallet}` share one handler registration — six routes, five registrations. All reads are ungated; only PATCH is gated.

## 13.4 Structure

```
wevibe-social-graph/
├── cmd/server/main.go          — entrypoint; SOCIAL_GRAPH_DB_PATH (main.go:17), SOCIAL_GRAPH_PORT (main.go:18)
├── internal/server/
│   ├── server.go               — route registration + handlers
│   ├── signature.go            — PATCH wallet-ownership verifier (§13.6)
│   └── signature_test.go       — 7 tests (contradicts README's "no *_test.go" claim, §13.9)
├── internal/store/store.go     — SQLite `profiles` table
└── internal/chain/client.go    — GetContributorStats (client.go:67) + GetContributorReward (client.go:151)
```

## 13.5 Store methods — `GetProfilesByWallets`, NOT `ListBatch`

Full exported set of `internal/store/store.go`: `New` / `Close` / `CreateProfile` (`:81`) / `GetProfile` (`:103`) / `UpdateProfile` (`:136`) / `GetProfilesByWallets` (`:183`).

**Correction:** the batch method is `GetProfilesByWallets`. `ListBatch` **never existed** — the name appears nowhere in the repo; it was a misnomer introduced by the old TOPOLOGY.md:1535 and is retired here.

## 13.6 PATCH auth — custom secp256k1 scheme (NOT ADR-036 signArbitrary)

`PATCH /v1/profiles/{wallet}` requires `wallet_pubkey` + `wallet_signature` (`server.go:139-153`). The verifier (`internal/server/signature.go:16-56`) is a **custom scheme**:

- `sha256(message)` digest — **no Amino envelope/prefix** (signature.go:41);
- `btcec/v2` `ParsePubKey` — 33-byte compressed secp256k1 pubkey;
- stdlib `ecdsa.Verify` over a 64-byte `r||s` signature (signature.go:48-50);
- bech32 address binding against the `wevibe` HRP (signature.go:80) — the signature must be proven by the pubkey that derives the claimed wallet address.

**Correction:** the old doc's label "Cosmos `signArbitrary` proof" is wrong. This is **not** ADR-036-wire-compatible — there is no Amino envelope, and a Keplr `signArbitrary` signature would **NOT** verify here. Repo-wide `rg -i "signArbitrary|ADR-036|amino"` = zero hits.

## 13.7 Port, storage, CORS

- **Port 4470 (default)**, from `SOCIAL_GRAPH_PORT` (`main.go:18`); docker-compose host-maps `127.0.0.1:4470:4470` (lines 180-183). Dashboard consumes `WEVIBE_SOCIAL_GRAPH_URL=http://localhost:4470` (line 228).
- **Port 4470 is shared with the faucet without collision:** the faucet's `LISTEN_ADDR=":4470"` (line 201) has **no `ports:` key** — container/network-namespace-local only (hub reaches it via `FAUCET_URL=http://wevibe-faucet:4470`, line 129). Only social-graph publishes 4470 to the host.
- **SQLite** via `SOCIAL_GRAPH_DB_PATH`, default `/data/social-graph.db` (`main.go:17`) — holds the `profiles` table only.
- **CORS wraps all routes** (`server.go:31`; `corsMiddleware` at server.go:301-314) with `Access-Control-Allow-Origin: *`.
- Access posture: **all reads public; only PATCH is gated** (§13.6).

## 13.8 Scope — contributor stats ONLY

The display surface is scoped to **CONTRIBUTOR-scoped data**:

- The only stats route is `GET /v1/stats/contributor/{pubkey}`, backed by `GetContributorStats` (+ `GetContributorReward`) — contributor-level totals (including contributor-level serve/self-serve counts carried in the stats payload, client.go:52-53).
- **No org serve counts, no per-org breakdown** — no org-scoped route or renderer exists.
- **No badges.** The serve-milestone, rarity-tier, and contribution-volume badge families are design-stage only (`ROADMAP.md`: badge rendering and rarity-tier logic are alpha/design-stage; rendering those three families from chain RPC inputs is future work). No badge entity or rendering code exists in the repo today.

## 13.9 Corrections carried from the old TOPOLOGY

| Old claim | Correction |
|---|---|
| "Public Profile Service" / centralized Social Graph Service (D-13.4) | D-SG-2 reference display client — open-source, forkable, self-hostable RPC display layer (DMO-030, 2026-06-10) |
| Store method `ListBatch` | `GetProfilesByWallets` (`store.go:183`); `ListBatch` never existed |
| "Cosmos signArbitrary proof" for PATCH | Custom sha256 + btcec/v2 + stdlib ecdsa.Verify (r\|\|s), bech32 `wevibe` binding; NOT ADR-036, Keplr signArbitrary would not verify |
| (repo README:53) "no `*_test.go` module tests" | False — `signature_test.go` exists with 7 passing tests |

## 14. wevibe-protocol — Protocol Definitions

Shared contract repository: the OpenAPI spec for hub HTTP endpoints, the protobuf codegen
pipeline producing JS/TS bindings from `wevibe-chain/proto`, and the fixed test vectors
used by contract tests and the recall parity harness. No running service; content only.

### File layout

`openapi.yaml` · `README.md` · `contract_test.sh` · `codegen/` · `js/` · `docs/` ·
`buf.gen.yaml` · `package.json`, plus the two test-vector directories below.
(Also present, not part of the contract surface: `CHANGELOG.md`, `CONTRIBUTING.md`,
`LICENSE`, `ROADMAP.md`, `SECURITY.md`, `runs/`, `node_modules/`.)

### Test vectors — BOTH directories are real (correction to old doc)

The old TOPOLOGY listed only the dash form. In fact **two** test-vector directories
exist, **both git-tracked**, holding disjoint content:

| Dir | Contents |
|---|---|
| `test-vectors/` (dash) | Exactly 1 file: `recall-ranking-parity.json` — the shared recall parity fixture consumed by wevibe-sim's parity harness. |
| `test_vectors/` (underscore) | 7 crypto fixtures + `README.md`: `epoch_key_derivation.json`, `fee_model_hash.json`, `hub_response_signing_v1.json`, `mnemonic_roundtrip.json`, `relay_envelope_v1.json`, `seal_open_envelope.json`, `shamir_roundtrip.json`. |

Verified with `git ls-files` (9 tracked paths: 1 dash-form + 8 underscore-form entries).

### Proto-gen pipeline

`npm run regen` (`package.json:11`) → `bash codegen/regen.sh` → Docker image
`bufbuild/buf:1.34.0` (pinned per D-S29-PROTO-BUF-IMG; regen.sh:41-46). buf runs inside
`wevibe-chain/proto` (so its `buf.lock` resolves the Cosmos SDK deps) with
`wevibe-protocol/buf.gen.yaml` as the template, emitting regenerated bindings into
`wevibe-protocol/js/`; hand-authored `js/index.ts` and `js/registry.ts` survive the wipe
of the generated `js/wevibe/` tree.

The generator is the **remote community plugin `buf.build/community/stephenh-ts-proto`**
(`buf.gen.yaml:3`) — unpinned: no plugin version is declared, so regenerated output can
drift with upstream ts-proto releases.

**No in-workspace consumer imports the generated bindings.** Repo-wide grep finds zero
references to `wevibe-protocol` or `wevibe-protocol/js` in wevibe-mcp and the dashboard
(`wevibe-server/dashboard`) package manifests or source. Note the contradiction:
`wevibe-protocol/docs/TOPOLOGY.md:35` still claims "Dashboard and MCP consumers compile
against the regenerated field contract" — that internal doc statement is drift, not
current fact. `js/` stands as published-but-unconsumed bindings plus the contract-test
artifact basis.

### STALE — openapi.yaml `ApproveRequest` (F2, reconcile item; not a live bug)

`openapi.yaml` still models the pre-umbral approval flow. `ApproveRequest`
(openapi.yaml:299-305) lists `wrapped_dek_enc` in both `required` and `properties`, and
the spec contains **zero** umbral fields file-wide. The live hub contract
(`wevibe-server/wevibe-hub/internal/protocol/types.go:244-260`) carries instead
`umbral_capsule`, `umbral_ciphertext`, `memory_type`, `mc_version` — and has **no**
`wrapped_dek_enc` field at all.

This is a doc/spec reconcile item, not a live defect: wire traffic follows the Go types,
not the stale spec. The same legacy field lingers in two further schemas —
`MemoryResult.wrapped_dek_enc` (:382) and `MemoryDetail.wrapped_dek_enc` (:406) — which
belong to the same reconcile sweep.

## 15. wevibe-biometric — Native Biometric Gate

> NEW — absent from the 2026-06-14 TOPOLOGY.md. Verified on-touch 2026-08-18 (source: `wevibe-meta/workspace/reports/1787048697-WO-TOP2-MISSING-REPOS.md`).

### What it is

A Rust **napi-rs v3** native addon (single `src/lib.rs`; napi-rs v3 declared in `Cargo.toml:14`) providing a cross-platform biometric prompt gate: **macOS Touch ID** and **Windows Hello** (`lib.rs:100-200`). It replaces the macOS-only `node-mac-auth` dependency. On **Linux there is no prompt floor — the gate falls open**.

The napi build emits a generated JS wrapper plus **4 per-platform npm subpackages**: `darwin-arm64`, `darwin-x64`, `linux-x64-gnu`, `win32-x64-msvc`. A darwin-arm64 prebuilt is intended to be tracked, but that is intent-only: there is no `.git` to track it (`.gitignore:21-24` merely documents the intent).

Build: `npm run build` (= `napi build --platform --release`).

### Fail-open invariant

The gate is advisory, never a lockout — constructors fail open (`lib.rs:48-62`), and the error map (`lib.rs:238-261`) draws the line:

| Outcome | Behavior |
|---|---|
| verified | proceed |
| biometrics unavailable | proceed (fail-open) |
| not enrolled | proceed (fail-open) |
| explicit user cancel | **block** |
| explicit auth failure | **block** |
| 180 s no-response timeout | **block, retryably** — hardware was available, so this is a *deliberate exception* to fail-open (`lib.rs:226-232`) |

### Sole consumer

`wevibe-mcp/src/biometric.ts` is the **only consumer** (import at `:34`; declared as an `optionalDependency` in `wevibe-mcp/package.json:59-61`, wired via npm-link symlink). It gates three key-material operations — **identity create**, **recovery export**, and **identity unlock** — at `key-store.ts:262/290/336`, and only when the **keychain backend** is active.

### Distribution status — NOT a git repo, NOT on npm

`wevibe-biometric` is **NOT a git repository** and is **NOT published to the npm registry** (`npm view` returns E404). It is consumed solely as an **npm-linked local sibling** of wevibe-mcp.

**Registry-publish gap:** because it is declared as an optional dependency that is absent from the registry, an install-from-registry (e.g. a fresh clone without the local link) resolves to *no package* — and per the fail-open invariant, the gate then **silently falls open**. Publishing the platform subpackages (or vendoring the prebuilt) is the durable fix.

# 16. wevibe-bench

The project's current sole focus: the deterministic cumulative memory-lift campaign harness.
One of the 14 git repos at workspace root. Entirely absent from the old TOPOLOGY.md — this
section is new.

## Role

Runs a backgammon task twice per campaign cycle — OFF cell and ON cell — and measures the
cumulative memory lift between them:

- **OFF** = the no-memory floor. The model plays with no recall; backend is `NoneBackend`
  (`wevibe_bench/backends/none_backend.py`).
- **ON** = recall injected. The backend (`wevibe_bench/backends/wevibe_backend.py`) serves
  memories through the bench MCP (`:4550`), which queries the hub (`:4440`).

Everything the campaign needs to be reproducible — seed, manifest, roster hash, per-cell logs —
is frozen under `runs/`.

## Layout

| Path | What it is |
|---|---|
| `wevibe_bench/` | Python harness package (below) |
| `scripts/` | Run drivers: `run_cumulative.py` (the main driver), `run_backgammon.py`, `bench_preflight.py`, plus ladder/transport/measurement/cleanup scripts |
| `control/` | Host control plane on `:7718` — Node, stdlib-only (`control/server.mjs:6-7,92`); the ONLY tier that starts runs |
| `dashboard/` | The `:7717` board — read-only by construction (GET-only, no docker socket; `dashboard/server.mjs:5,45`) |
| `tasks/backgammon/` | Task definitions: `bench/`, `gates/`, `golden/`, `prompts/`, `runs/` |
| `docker/worker/` | Worker container build |
| `config/` | Bench env files (`bench.env`, `cloud.env`) |
| `runs/` | Cell logs + `cumulative/manifest.json` + `profiles/` + `mcp4550.pid`/`.log` |

`wevibe_bench/` package internals:

- `config.py` — arbitrary schedule schema (`config.py:23`) plus the scored-ladder roster SoT
  (`config.py:330,345`) and the worker registry `WORKER_MODEL_REGISTRY`. (The old fixed
  `model_ladder` is gone — replaced by this schema.)
- `runner.py` — cell runner.
- `cumulative/` — sequencer + manifest (roster hash re-verified per launch, frozen in manifest).
- `lifecycle/` — `mcp_rest`, `lconfig`, `m2_proof`.
- `adapters/` — `docker_worker`, `backgammon`, `openrouter_proxy`.
- `backends/` — `wevibe` (ON) and `none` (OFF).
- Top-level modules: `preflight.py`, `scorecard.py`, `recall_gold.py`, `seeding.py`,
  `spend_key.py`, `proxy_meter.py`, `contention.py`.

Profiles: memory profiles are frozen in `runs/profiles/` (declared, NOT enforced —
`enforced:false`). Model roster SoT is `wevibe_bench/config.py` with the JS mirror
`control/roster.mjs`. "Org profiles" is NOT a bench concept (org extraction profiles are
hub-side).

## Run driver — `scripts/run_cumulative.py`

Main-parser flags, which must go BEFORE the subcommand (argparse exits 2 otherwise):
`--org --model --roster-model --task --seed --manifest --on-budget --cloud --router --provider`
(registered at `run_cumulative.py:1324-1390`).

Subcommands (registered `:1396-1423`):

- `run` — with `--mode off|on` (`:1402-1410`); `--mode on` requires `--org`
  (enforced `:1278-1282`). `--model` must match the manifest on every subcommand or the
  driver aborts with "roster hash drift detected".
- `state` — campaign state inspection.

DEAD (removed by `ba2947a`): the `extract` subcommand and the `--until-review` flag.
Extraction is now dashboard-driven via `OPENCODE_DB_PATH`. Known doc drift:
`control/contract.mjs:68,84` still documents a removed `resume` subcommand — argparse
registers only `run` + `state`.

## Scoring

- The **gates suite** `tasks/backgammon/gates/` (Playwright core / edges / conformance) does
  the scoring.
- `scorecard.py` RECORDS integrity-enforced artifacts — it never scores a `not_scored` run.
- `golden/` is the reference/mock implementation, NOT a comparator.

## Ports & transport

| Port | What |
|---|---|
| `:4550` | Bench MCP — a managed service started via `bench-mcp.sh start`; the run path NEVER calls `bring_up()`, it only connects |
| `:4545` | Worker/relay spend proxy (transport path to the resident model) |
| `:7717` | Bench dashboard (read-only board) |
| `:7718` | Host control plane (sole run-starter) |
| `:4440` | Hub (queried by the bench MCP for recall) |
| `:1234` | Model runtime |
| `:3001` | Contributor dashboard |
| `:4096` | `opencode serve` host port published by the bench (`serve_host_port`, `run_cumulative.py:1435-1436`) |
| `:4450` | **NOT a bench component** — the operator's REAL host `wevibe-mcp` (daily driver). Never point a bench component at it; preflight forbids it (`bench_preflight.py:50,53`) |

Note: `bench-mcp.sh` and `verify-clean.sh` live in `wevibe-meta/scripts/`, not in this repo —
wevibe-bench only references them (RUNBOOK.md:645,667,775).

## Operative doc

`RUNBOOK.md` v7 at repo root (`RUNBOOK.md:1-2`) is the ONLY operative document for running the
campaign: read it and nothing else to operate. All other bench documents are history with no
authority; on any disagreement the RUNBOOK wins. Operator-facing commands are also summarized in
`USER-BENCHMARKING-GUIDE.md`. Launch gotcha: `nohup … &` requires `< /dev/null`, or zsh suspends
the job the moment it touches stdin.

## Status

Verified on-touch 2026-08-18 (report 1787048697-WO-TOP2-MISSING-REPOS, §wevibe-bench; PASS-2
authoritative over the 1787047618 chart). Known drift carried forward, not fixed here:
`control/contract.mjs` still documents the dead `resume` subcommand.

# 17. wevibe-meta

The orchestration repo: a Makefile-driven control surface for the full WeVibe lifecycle
(deploy, wipe, verify, dogfood, replay), the operative scripts that manage bench and host
services, an integration test suite, and the local-only workspace knowledge base that every
orchestration tier reads from and appends to. IS a git repo (clean tree, HEAD `e635515`).

Verified against report 1787048697 (§ wevibe-meta), on-touch 2026-08-18.

## Makefile — 36 targets drive the full lifecycle

Proto-gen gets its own section in this document, but here it is just ONE of 36 targets. The
named lifecycle targets, grouped by function:

| Group | Targets |
|---|---|
| Docker stack | `docker-up`, `docker-down`, `docker-up-fast`, `docker-build-fast` |
| Contributor dashboard | `contributor-up`, `contributor-down`, `contributor-restart`, `contributor-status` |
| Host MCP | `host-mcp-build`, `stop-host`, `served-cache-clear` |
| Bench MCP (managed service) | `bench-mcp-stop`, `bench-mcp-start`, `bench-mcp-restart`, `bench-mcp-status` |
| Wipes | `bench-wipe`, `host-state-wipe` |
| Verification | `verify-clean` (delegates to `scripts/verify-clean.sh`, 13 checks) |
| One-button | `redeploy` (wipe + rebuild + bring-up in one invocation) |
| Health | `health` |
| Dogfood | `dogfood`, `dogfood-fast`, `dogfood-health`, `dogfood-pipeline`, `dogfood-fast-down` |
| Replay / parity | `replay-gate`, `parity-check`, `parity-fixtures` |
| Cross-repo sync | `wevibe-mcp-token`, `sync-sdk-wasm`, `sync-extraction-prompts` |
| Proto | `proto-gen`, `proto-gen-chain`, `proto-gen-umbral` |
| Housekeeping | `clean`, `help` |

## scripts/ — the operative tooling

- `bench-mcp.sh` — manages the commissioned bench MCP service on `:4550` (`start` /
  `stop` / `restart` / `status`), seeded from `~/.wevibe/bench/leader-seed.txt`. **Lives
  HERE, not in wevibe-bench** — wevibe-bench only references it (RUNBOOK.md:645,667,775 and
  USER-BENCHMARKING-GUIDE.md).
- `verify-clean.sh` — the 13-check verify-clean gate (`verify-clean.sh:618-630`), likewise
  referenced by wevibe-bench but owned here.
- `contributor-dashboard.sh` — contributor dashboard lifecycle.
- `lib.sh` — shared shell helpers for the above.
- `replay-gate.sh` + `empirical_replay.sh` + `empirical_replay/` — the replay gate driver,
  its shell entrypoint, and the self-contained Go module implementing empirical replay.
- `sim-baseline-extract.js`, `sim-baseline-perseed.js`, `sim-calibration.js`,
  `sim-trajectory.js` — recall-sim baseline/calibration/trajectory scripts.

## tests/ — vitest/tsx integration suite

Subdirectories: `consumer`, `contributor`, `leader`, `moderator`, `e2e`, `lib`, `scripts`,
`runs`. One known stub: `tests/e2e/full-lifecycle.test.ts` is a delete-verdict stub with an
empty body (`tests/e2e/full-lifecycle.test.ts:12`, CO-266) — scheduled for removal, not a
passing test.

## workspace/ — local-only knowledge base

- `docs/`, `templates/` — orchestration docs and report templates.
- `reports/` — the reports KB: ~1870 files (1871 at last count), the durable record of every
  completed work order. **Gitignored wholesale** (`.gitignore:8`; `git ls-files workspace`
  returns zero files) — local-only by design, never pushed.
- `runs/`, `spikes/`, `timeline-chunks/` — run artifacts, spike material, timeline sources.

## Status

IS a git repo, clean tree, HEAD `e635515`. The reports KB grows continuously, so its file
count is a moving target (~1870 at the 2026-08-18 audit, correcting an earlier chart's 1853).

# 18. Workspace-Root Docs & Directories — Canon / Operative / Artifact

The workspace root (`/Users/jerrysmith/Desktop/wevibe-workspace/`) is **NOT a git repo**; it holds loose
documentation and directories that belong to no repo. Some of these files have tracked copies in
`wevibe-docs` (which **IS** a git repo), and those mirrors can drift. Every item below was verified
present on-touch 2026-08-18 (WO-TOP2-MISSING-REPOS).

Everything at the root falls into exactly one of three classes:

- **canon** — authoritative specs and design documents; the statement of what the system is.
- **operative** — live working documents that steer day-to-day execution; read these to know what to do.
- **artifact** — backups, archives, run outputs, indexes, and exports; never authoritative, regenerable or disposable.

## 18.1 Canon

| File | What it is |
|---|---|
| `CANONICALUX.md` | Canonical UX spec. Local orchestration doc — gitignored by design, **never committed, NO wevibe-docs copy exists** (the root copy is the only copy). |
| `RECALL-PIVOT-SPEC.md` | The conclusive recall-pivot event-schema + boundary spec (LIVE): content-free consumer-signed events on-chain, standing computed at the edge against an anchored policy (edge-policy-v1 @ height 45), the T1–T8 measurable contract, sim→production mirror. Root copy is operative; the tracked wevibe-docs copy is a stale mirror (see §18.4). |
| `RECALL-PIVOT.md` | The recall-pivot decision document. Byte-identical twin of its wevibe-docs copy. |

## 18.2 Operative

| File / dir | What it is |
|---|---|
| `RECALL-SYSTEM.md` | Operative recall-system reference. Root copy is operative; the tracked wevibe-docs copy is a stale mirror (see §18.4). |
| `RECALL-SYSTEM2.md` | Successor working draft on the recall system. |
| `EXTRACTION-FLOW.md` | Ground-truth reference for the extraction pipeline: UX flow, 23-stage table, keyword sub-pipeline crux, canon mapping, per-stage measurement guide. Byte-identical twin of its wevibe-docs copy; verify line anchors on touch (extraction files are uncommitted and drifting). |
| `EXTRACTION-FLOW2.md` | Continuation working draft of the extraction flow. |
| `BENCHMARK-DIARY.md` | The benchmarking diary — details calibrated to the memory-pipeline benchmarking process. |
| `PROJECT-HISTORY-TIMELINE.md` | Chronological project history. |
| `AUDIT.md` | Audit record. |
| `SESSIONCONTINUANCE.md` | Live session-continuance ledger (local orchestration doc, gitignored, never committed). |
| `directive.md` | Current directive doc (local orchestration doc, gitignored, never committed). |
| `session-prompts/` | Session-prompt corpus. |
| `extraction-prompts/` | Extraction-prompt corpus (synced from wevibe-meta via `make sync-extraction-prompts`). |
| `landing/` | Landing-page assets. |

## 18.3 Artifact

- `BENCHMARK-DIARY.md.bak` — backup of the diary.
- `SESSIONCONTINUANCE-ARCHIVE-3.md` — archived continuance ledger.
- `runs/` — run artifact outputs.
- `.codegraph/` — workspace-wide code-graph index (auto-syncs; backs the codegraph tools).
- `cosmos-x/`, `noble142/` — vendored/support dependency trees.
- `node_modules/` — installed packages.
- `.logs/` — log directory.
- `*.html` / `*.log` / `*.tgz` / `*.json` — one-off exports and dumps from runs.

## 18.4 Operative-copy ruling — root vs wevibe-docs

Where a doc exists in BOTH the root and tracked `wevibe-docs`, authority is:

| Doc | Ruling | Evidence (2026-08-18) |
|---|---|---|
| `RECALL-PIVOT-SPEC.md` | **Root copy operative**; wevibe-docs copy a STALE mirror | root copy newer AND larger; diffs differ |
| `RECALL-SYSTEM.md` | **Root copy operative**; wevibe-docs copy a STALE mirror | root copy newer AND larger; diffs differ |
| `CANONICALUX.md` | **Root-only** — no wevibe-docs copy | full-tree negative sweep; gitignored by design |
| `EXTRACTION-FLOW.md` | Byte-identical twin (root ≡ wevibe-docs) | `cmp -s` exit 0 |
| `RECALL-PIVOT.md` | Byte-identical twin (root ≡ wevibe-docs, both 7576 B) | `cmp -s` / size match |

Rule of thumb: trust the **root copy** for canon and operative docs until a tracked mirror is proven byte-identical.

## 18.5 Root scripts — exactly one file

Root `scripts/` contains **exactly one file: `store-cloud-key.sh`**. It is NOT an orchestration home —
all orchestration scripts live in **`wevibe-meta/scripts/`** (bench-mcp.sh, verify-clean.sh,
contributor-dashboard.sh, lib.sh, replay-gate.sh, empirical_replay/, sim-baseline/calibration scripts).

## 18.6 Root git status

The root is **not** a git repo — each repo under it is its own git boundary, and `wevibe-docs` **is** one.
That asymmetry is exactly why tracked mirrors of root docs drift: nothing at the root syncs them.
Local orchestration docs (`AGENTS.md`, `SESSIONCONTINUANCE.md`, `directive.md`, `CANONICALUX.md`)
are gitignored everywhere and never committed by design.

## 19. Personal Memory Layer

The personal memory layer is the workspace-local recall substrate beneath WeVibe's org-level memory: a CodeGraph index over the entire workspace that agents query for attributed, code-level context. Architecturally current as of the 2026-08 audit; its graph stats below are CORRECTED against a fresh re-count (the old TOPOLOGY.md figures were stale).

### 19.1 The `.codegraph/` index

- `.codegraph/` exists at the workspace root (`/Users/jerrysmith/Desktop/wevibe-workspace/.codegraph/`), daemon live and auto-syncing (file watcher active; auto-sync on change; index lags writes by ~1s).
- `codegraph.db` ~195 MB (195,424,256 B at audit time).
- **CORRECTED stats (fresh sqlite re-count, PASS-2 authoritative): 1,323 files / 31,110 nodes / 86,970 edges** — NOT the stale 768 files / 19.5k nodes / 53.7k edges previously documented. Node count cross-confirmed via `nodes_fts` = 31,110.
- **Coverage: 20 top-level paths indexed** — all core repos + bench/biometric/faucet/opencode-plugin + aux dirs.

### 19.2 CodeGraph itself

- External tool, not a WeVibe component: `codegraph` CLI + MCP server, MIT-licensed, on PATH at `/opt/homebrew/bin/codegraph` (v1.0.1, `@colbymchenry/codegraph`).
- Wired as a **SEPARATE opencode MCP** in `~/.config/opencode/opencode.json` (`mcp.codegraph`), distinct from the WeVibe MCP entry.
- Telemetry **off** — triple-corroborated: persisted `~/.codegraph/telemetry.json` disabled since 2026-06-19, `DO_NOT_TRACK=1` env, and the separate `mcp.wevibe` entry independently disabled.

### 19.3 Stage 2 — `PersonalMemoryProvider` (NOT started)

- Stage 2 is a planned `PersonalMemoryProvider` interface in `wevibe-opencode-plugin` that would generalize personal recall beyond the CodeGraph index.
- Status: **DESIGN-LOCKED, NOT started** — the interface is absent from the plugin repo AND from its full git history (strong negative); `D-PERSONAL-MEMORY` / `GAP-PERSONAL-MEMORY` remain design-locked / open. No pivot has occurred.

### 19.4 Known stale copy (out of scope)

- The same stale graph stats (768 / 19.5k / 53.7k) also live in `wevibe-docs/DECISIONS.md:3248`. Correcting that copy is tracked separately and is NOT part of this rewrite.

# 20. wevibe-opencode-plugin — OpenCode Plugin + TUI

> Corrects the stale duplicate formerly at TOPOLOGY.md:1489–1509 (the accurate structure was already carried at TOPOLOGY.md:1240–1243). Authoritative fact base: reports `1787048710-WO-TOP2-REPO-LEVEL.md` (PASS-2, authoritative) and `1787047604-WO-TOPOLOGY-PLUGIN-SIM.md` (Surface 2).

The OpenCode-side surface of WeVibe: an engine plugin (recall injection, telemetry, org binding) plus the TUI onboarding/status card. Separate from the `mcp.wevibe` MCP server (registered in `~/.config/opencode/opencode.json`), telemetry off.

## Authoritative structure (real — larger than the old 2+1+1 shorthand)

```
wevibe-opencode-plugin/
├── plugins/            # 32 files — engine plugin
│   ├── wevibe-plugin.ts    # entry (recall injection, telemetry, gates)
│   ├── funnel-counters.ts
│   ├── binding.ts          # org binding ({root}/.wevibe/org.json|org.local.json)
│   ├── gstv-*, metrics.ts, org-join-gate.ts,
│   ├── outcome-*, predicate-*, recall-harvest.ts, wevibe-paths.ts, …
│   └── + 15 test files + fixture generator
├── tui/                # 4 files — TUI card
│   ├── wevibe.tsx                  # card (moved in from wevibe-mcp/opencode-plugin/ by 7b0a2ff, 2026-06-18)
│   └── _render_{compact,harness,responsive}.tsx
├── bin/                # 1 file
│   └── install-opencode.ts         # installer (628 lines)
└── tui.json            # ROOT registration: {"plugin": [["./tui/wevibe.tsx", {}]]}
```

## DEAD / never-was (explicit, swept via dir listing + full git history + workspace-wide grep)

| Claim | Verdict |
|---|---|
| `plugins/wevibe.tsx` | **Never existed** (git log empty) — the TUI card is `tui/wevibe.tsx` |
| `tui/tui.json` | **Never existed** — registration is the repo-ROOT `tui.json` |
| `wevibe-admin install-opencode` | **Removed** from wevibe-mcp by `f97037f` (added `264a29f`; zero current hits) |
| `~/.wevibe/config` | **Never existed** — 0 references workspace-wide (the only hit was the stale doc itself) |

## Installer

`bin/install-opencode.ts` via `npm run install-opencode` → `tsx bin/install-opencode.ts install-opencode` (package.json:24). It copies the source `tui/`, merges `tui.json` + `opencode.json`, and npm-links the `wevibe` CLI. The old note ("wevibe-mcp `npm run build` copies it") was wrong — that build is `tsc` only and copies nothing.

## Runtime reads (actual)

| Path | Reader |
|---|---|
| `~/.wevibe/identity.json` | `tui/wevibe.tsx:21` (TUI; also org-join gate) |
| `~/.wevibe/plugin-config.json` | `plugins/wevibe-plugin.ts:219` |
| `{root}/.wevibe/org.json` \| `org.local.json` | `binding.ts:65-66` (engine plugin) |
| `~/.wevibe/served-memories.json`, `blacklist.json` | plugin |
| `~/.wevibe/mcp-session-token` | read-only (written by MCP) |

**NOT** `~/.wevibe/config` (never existed). No long-lived secrets are stored by the plugin.

## Recall engine + session tie

- **Mode flag:** `WEVIBE_RECALL_MODE` — `prod` (default) / `test` — read INDEPENDENTLY by all three tiers: plugin (`wevibe-plugin.ts:388-391`), MCP (`retrieve-cli.ts:75-77`, `http-server.ts:3355`), hub (`config.go:53-57` → `main.go:94 SetRecallMode` → `pool.go:34-38 recallModeIsTest` → `retrieval.go`; `docker-compose.yml:137`). Test mode also disables Earned-Trust auto-accept (`:653-655`). Active as D-RECALL-MODE-FLAG (DECISIONS.md:3160).
- **Session tie:** the session-id is the REAL OpenCode `sessionID`, captured from `chat.message` + `experimental.chat.system.transform`, and threaded to `/v1/recall` + `/v1/serves`. Injection is gated once per session via `sessionInjectedCids` (`[inject]` logging at `wevibe-plugin.ts:2004,2007`) — a post-July fix; RC-035/054 had it ungated. D-SESSION-SERVE-DEDUP is ACTIVE but its session-id clause is superseded by D-RECALL-MODE-FLAG.

## Personal Memory Layer (adjacent, unchanged verdict)

Stage 2 `PersonalMemoryProvider` remains DESIGN-LOCKED (D-PERSONAL-MEMORY / GAP-PERSONAL-MEMORY), NOT started — absent from the plugin repo and its full git history. Its `.codegraph/` stats: 1,323 files / 31,110 nodes / 86,970 edges (see §Personal Memory).

# 21. wevibe-sim

Recall-alignment simulation suite: hand-ported re-implementations of the hub's retrieval/ranking
logic (Go `retrieval.go`) exercised against a synthetic corpus to measure recall behavior. This
section replaces the old §"wevibe-sim/recall-sim/ — Recall-Alignment Simulation Suite" and
corrects three of its claims, per the PASS-2 audit (report 1787048710, authoritative over PASS-1
report 1787047604).

## Corrections to the old doc

| Old claim | Reality |
|---|---|
| "wevibe-sim is NOT a git repo" | **STALE.** It IS a git repo: exactly 2 commits — `977bdf7` (2026-07-30) and `c78cf46` (2026-08-13, "bump next to 14.2.35"). The AGENTS.md workspace table carries the same stale wording. |
| Parity harness "planned" | **BUILT.** See §Parity harness below. |
| Parity target implied pivot-era | **Directionally wrong.** The parity fixture is PRE-pivot and pins the RETIRED D-9.3-era ranker. See §Parity harness below. |

## Layout

The core suite is unchanged from the old doc and still lives under `recall-sim/`:

```
wevibe-sim/                      # git repo (977bdf7, c78cf46)
├── recall-sim/
│   ├── config.mjs
│   ├── lib/
│   ├── pipeline/                # rank.mjs, retrieve-c3.mjs
│   ├── corpus/                  # + augment-arm-b.mjs
│   ├── eval/
│   ├── results/<timestamp>/
│   ├── parity/                  # NEW: check-parity.mjs, gen-parity-fixtures.mjs
│   ├── rb1a/                    # NEW
│   ├── runs/armb-ablation/      # NEW
│   ├── data/                    # NEW: + 4 snapshots
│   └── scrub-sessions.mjs       # NEW
├── runs/                        # NEW: second (policy-sim) harness, see below
└── Next.js dashboard            # next 14.2.35
```

Additions beyond the old doc: `parity/`, `rb1a/`, `runs/armb-ablation/`, `data/` + 4 snapshots,
`scrub-sessions.mjs`, `corpus/augment-arm-b.mjs`.

**Second harness (absent from the old doc):** a newer policy harness sits at the `wevibe-sim/`
repo root — `runs/policy-sim*` plus `runs/{audit,eval-point,policy-sim}.js`. The repo also ships
a Next.js dashboard (`next` bumped to 14.2.35 by commit `c78cf46`).

## The two evals

| Command | What it is |
|---|---|
| `npm run sim:eval` | CPU-only retrieval eval (`run-eval.mjs`) |
| `npm run sim:behavioral` | 3-arm behavioral eval (`run-behavioral.mjs:188-191`); judge model `openrouter/anthropic/claude-opus-4.8-fast` (`config.mjs:86`) |

## Parity harness — BUILT (not "planned")

All three pieces exist and are git-tracked:

- `recall-sim/parity/check-parity.mjs` + `recall-sim/parity/gen-parity-fixtures.mjs`
- shared fixture `wevibe-protocol/test-vectors/recall-ranking-parity.json` (tracked in
  wevibe-protocol; latest commit `322bc65`)
- wired as `npm run sim:parity` (`wevibe-sim/package.json:14`)

The Go side uses inline self-tests, not the JSON fixture.

### Directional correction — the fixture is PRE-pivot

`322bc65` is dated 2026-07-16 — 13 days BEFORE the RECALL-PIVOT (2026-07-29) — and its schema is
D-9.3 keyword-weight semantics with NO standing field. The parity harness therefore pins the
RETIRED D-9.3-era ranker, which is exactly why `rank.mjs` still implements keyword weights:

- keyword gate (`rank.mjs:81-83`)
- γ boost (`rank.mjs:61/85/119`)
- new-memory boost (`rank.mjs:66/105-111`)
- denial decay (`rank.mjs:64/101-103`)
- power-law sampler (`rank.mjs:195-257`)

The hub source citation remains intact: `retrieval.go:812` — "D-9.4 power-law sampler.
Source: wevibe-sim/ranking-fix.js:73-111".

## Ranker re-base (RECALL-PIVOT)

The old "exact mirror of product" framing is pivot-superseded. DECISIONS.md:1279 was AMENDED
2026-07-29 (RECALL-PIVOT): the product re-based per-keyword weight ranking onto the edge-standing
scalar, retiring the keyword weights. Sim's `rank.mjs` retains keyword weights — that retained
behavior is the divergence the parity harness pins. D-RECALL-ALIGNMENT (DECISIONS.md:720) was
amended, not superseded.

## Execution policy

Runs execute under per-call kill, an in-process watchdog, 15s heartbeats, a spawn-budget cap, and
a `--dir` isolated working cwd. Session scrubbing is MANUAL — `npm run sim:scrub` — not automatic
post-run.

## Status

Verified against the PASS-2 audit (report 1787048710; PASS-1 report 1787047604). Latest
recall-sim run 2026-07-03; latest behavioral run 2026-06-09.

# 22. Inter-Package Relationships & Dependency Summary

Synthesis across the per-package sections: where the packages interlock at runtime, which chain modules are live vs parked, and the complete workspace repo set + language split. All claims verified on-touch against code, 2026-08-18.

## 22.1 The chain-directory + hub-response-signing family is BUILT

**Key cross-cutting finding:** the entire chain-directory and hub-response-signing family is **built and wired end-to-end — it is NOT design-only**. The old `TOPOLOGY.md` (Layers 1 and 2, Consumer Path step 0) AND `DECISIONS.md:3270` ("design-only, multi-repo build not started") are BOTH stale on this point.

Chain side (`wevibe-chain`, `x/org`):

| Artifact | What it is | Authority |
|---|---|---|
| `hub_endpoints` | 1–3 ordered hub URLs per org (state field 18) | `proto/wevibe/org/v1/state.proto` |
| `hub_serving_address` | org serving address (state field 14) | `proto/wevibe/org/v1/state.proto` |
| `hub_response_pubkey` | pubkey hub responses are signed with (state field 19) | `proto/wevibe/org/v1/state.proto` |
| `MsgSetServingInfo` | publishes the above (extends `MsgSetServingKey`) | `proto/wevibe/org/v1/tx.proto:60-68` |
| `MsgSetOrgConfig` | org config publication | `proto/wevibe/org/v1/tx.proto` |
| `SetServingInfo` + `ValidateHubEndpoints` | keeper enforcement | `x/org/keeper/keeper.go:588-623` |

Consumer side (`wevibe-mcp`):

- `hub-resolver.ts:82-124` — chain-resolves an org's hub endpoints and walks the ordered list via `pickActiveEndpoint` failover.
- `hub-fetch.ts:94-98` — verifies the `X-Hub-Signature` header on hub responses against the chain-published `hub_response_pubkey`.

Both ends are wired: orgs publish their hub directory on-chain; the MCP resolves the hub from the chain (never a hardcoded constant) and cryptographically verifies every hub response against the published key.

## 22.2 Cross-module status (chain modules)

- `x/attestation` — **disabled-but-wired** (`msg_server.go:24-26`). The proof tier is re-expressed by `D-PROVENANCE-ADMISSIBILITY-2026-07-23` as P1 (`tee_receipt`/CommitLLM) + P2 (`zktls_proof`/blind-witness), mostly unbuilt; CO-282 spike path `wevibe-meta/workspace/spikes/zktls-attestation/RESULTS.md` exists.
- `x/bandwidth` — **LIVE**, consumed on the serve path (`x/serve/keeper/keeper.go:221`).
- `x/identity` — **LIVE**.
- `x/reputation` — **LIVE**.
- Recall-pivot event log: the 8 content-free, consumer-signed event types are live in `x/serve` (`serve/v1/event.proto`, E1–E8; E4 contest + E5 sponsorship PARKED), with `StoredPolicyAnchor` + `MsgAnchorPolicyVersion` and a genesis-seeded anchor (`edge-policy-v1`, hash `2d2faa14…87899e`, anchored at height 45).

## 22.3 Cross-wiring map (runtime edges between packages)

| Edge | Mechanism | Authority |
|---|---|---|
| hub → faucet | `FAUCET_URL=http://wevibe-faucet:4470` (in-network, no host port) | `docker-compose.yml:129` |
| dashboard → social-graph | `WEVIBE_SOCIAL_GRAPH_URL=http://localhost:4470` | `docker-compose.yml:228` |
| dashboard → MCP | `WEVIBE_MCP_HTTP_URL=http://host.docker.internal:4450` | `docker-compose.yml:236`, `.env.example:59` |
| MCP → hub | `POST /v1/orgs/{orgID}/query`, signed with `X-Agent-Signature` (ed25519) | hub router + MCP hub client |
| hub → chain | gRPC `:9090`; plus CometBFT `NewBlock` tx-decode watch (e.g. `MsgRemoveMember` → sidecar `DeleteKFrags`) | `watcher.go:464,514` |
| hub → Qdrant | HTTP REST `:6333` (retrieval store) | `docker-compose.yml:52` |
| hub → umbral sidecar | gRPC — exactly 5 RPCs: StoreKFrag / ReEncrypt / DeleteKFrags / DeleteOrgKFrags / Health | `sidecar.proto` |
| MCP → guard | `spawnSync` into the YARA-X scanner | MCP scan path |
| MCP → biometric | napi-rs addon gating identity create / recovery export / identity unlock | `biometric.ts:34`, `key-store.ts:262/290/336` |
| dashboard → chain | wallet-direct `directBroadcast` (no hub relay) | dashboard chain client |

Note the trust shape: the hub is the sole relay between MCP and chain (the MCP never broadcasts to the chain directly), while the dashboard does broadcast directly with the leader wallet; hub responses back toward the MCP are signature-verified per §22.1.

## 22.4 Workspace repo set

**14 git repos** at the workspace root:

`wevibe-bench`, `wevibe-chain`, `wevibe-docs`, `wevibe-faucet`, `wevibe-guard`, `wevibe-mcp`, `wevibe-meta`, `wevibe-opencode-plugin`, `wevibe-protocol`, `wevibe-sdk`, `wevibe-server`, `wevibe-sim`, `wevibe-social-graph`, `wevibe-umbral`.

- `wevibe-sim` **IS** a git repo since 2026-07-30 (first commit `977bdf7`) — the old TOPOLOGY.md "not a git repo" claim (and the matching AGENTS.md §3 line) is stale.
- Upstream `WeVibe`/anchor is documented but **not cloned** in this workspace.

**Non-git directories** at root: `wevibe-biometric/` (Rust napi-rs addon, npm-link-only local sibling — not a git repo, not on the npm registry), `cosmos-x/`, `noble142/`, `node_modules/`, `runs/`, `scripts/` (single file: `store-cloud-key.sh`; orchestration scripts live in `wevibe-meta/scripts/`), `session-prompts/`, `extraction-prompts/`, `landing/`.

The workspace **root itself is NOT a git repo** — each package is versioned independently.

## 22.5 Language & dependency split

| Language | Packages |
|---|---|
| **Go 1.25.9** | `wevibe-chain`, hub (`wevibe-server`), `wevibe-social-graph`, `wevibe-faucet` |
| **TypeScript** | `wevibe-mcp`, `wevibe-opencode-plugin`, `wevibe-dashboard` |
| **Rust** | `wevibe-guard`, `wevibe-sdk` (+ WASM), `wevibe-umbral`, `wevibe-biometric` (napi-rs v3) |
| **Python** | `wevibe-bench` (campaign harness + run driver) |
| **Next.js** | `wevibe-dashboard` (`:3000` docker / `:3001` dev) and the `wevibe-sim` dashboard (next 14.2.35) |

`wevibe-meta` sits outside the language split as the orchestration repo (36-target Makefile, `scripts/`, `tests/`, gitignored `workspace/` reports KB).

# 23. Canonical 5-Layer Architecture (corrected)

The stale TOPOLOGY.md (2026-06-14) described the five layers before two amendments reshaped them:
the RECALL-PIVOT joint amendment (D-4.1+D-4.2+I-7+R-DECAY-FROZEN, ratified 2026-07-30, LIVE —
`edge-policy-v1` hash `2d2faa14…87899e` anchored at height 45) and D-ECON-STORAGE-MARKET
(`ProcessOrgPayouts` removed). Every claim below is re-labeled against current code per the
WO-TOPOLOGY-FLOWS audit (report 1787047783, Region 1). Verdict vocabulary: **VERIFIED** (built as
described), **DEVIATED→BUILT** (doc said design-only; it has shipped), **DEAD** (never built or
removed), **PIVOT-SUPERSEDED** (an amendment replaced the design).

## Layer 1 — Chain state

Post-pivot, the chain stores content-free evidence only: 8 consumer-signed serve/denial EVENT
types in `x/serve` (`serve/v1/event.proto`, E1–E8; E4 contest + E5 sponsorship PARKED) plus an
anchored policy version (`StoredPolicyAnchor` + `MsgAnchorPolicyVersion`). Standing is a pure
function of (events, anchored policy_version) computed at the EDGE — per-keyword weights, decay
formulas, and matched-keyword gates never touch consensus. State it actually holds:

| Component | Status | Evidence |
|---|---|---|
| Aggregate serve/denial counters (contributor + org; NOT per-memory cards) | VERIFIED | `x/serve` `StoredEpochServeStats` / `StoredContributorEpochServes`, `keeper.go:323-403` |
| Approved-memory contribution counts | VERIFIED | `StoredOrg.total_committed_memories`, `org/v1/state.proto:9` |
| Org membership + roles | VERIFIED | `StoredMemberRecord`, `org/v1/state.proto:48-55` |
| Economic state | VERIFIED | `StoredEmissionPool`, `emissions/v1/state.proto:6-18` |
| Hub directory: `hub_endpoints` (1–3 ordered URLs), `hub_serving_address`, `MsgSetServingInfo` | **DEVIATED→BUILT** | see below |
| Per-memory rarity tier computed at commit, frozen on-chain | **DEAD** | see below |

**Hub directory — DEVIATED→BUILT.** The old doc (and `DECISIONS.md:3270`, "design-only,
multi-repo build not started") are BOTH stale. Built: `MsgSetServingInfo`
(`wevibe-chain/proto/wevibe/org/v1/tx.proto:60-68`) + `hub_endpoints` (state.proto field 18) and
`hub_serving_address` (field 14); keeper `SetServingInfo` + `ValidateHubEndpoints`
(`x/org/keeper/keeper.go:588-623`). The MCP consumes it: `hub-resolver.ts:82-124` chain-resolves
org hub endpoints with `pickActiveEndpoint` failover. Both ends wired.

**Per-memory rarity — DEAD.** No rarity field exists in `StoredMemoryCommitment`
(`memory/v1/state.proto:49-81`). The only residue is the `rarity_multiplier_cap` param and
`ComputeRarityMultiplier`, which has zero production callers (`x/emissions/keeper/keeper.go:626`).
Never built, and pivot-retired regardless (the per-keyword supply/demand model is gone).

## Layer 2 — RPC / API

| Component | Status | Evidence |
|---|---|---|
| Hub-response-signing | **DEVIATED→BUILT** | `hub_response_pubkey` (state.proto field 19) stored on-chain; MCP verifies `X-Hub-Signature` against it (`hub-fetch.ts:94-98`) |
| "RPC exposes rarity tier" | **DEAD** | rarity was never built (Layer 1) |

The doc listed hub-response-signing as design-only; it is live: the org's `hub_response_pubkey`
rides the same `SetServingInfo` path as the hub directory, and the MCP rejects hub responses whose
signature does not verify against it. Hub is the consumer's sole relay — MCP has no direct chain
broadcast.

## Layer 3 — Social graph (NARROWED)

The social graph renders **CONTRIBUTOR stats only**. The doc's larger claims do not match code:

- NO org serve counts (serve/retrieval counts are excluded from economics entirely — social-only, D-SG-1).
- NO badges — serve-milestone, rarity-tier, and contribution-volume badges are all design-stage.
- NO per-org breakdown; NO canonical-spec application.
- NO amendment-12 fundedness surfacing: the chain-side query exists
  (`GET /wevibe/org/v1/account/{org_id}`), the social-graph half is not built.

## Layer 4 — Emissions / economics

| Component | Status | Evidence |
|---|---|---|
| "Contribution-only VIBE payout per approved memory" | **PIVOT-SUPERSEDED** | flat even split (below) |
| Contributor payout = flat even split from network pool | BUILT | `MintDailyEmission`, `x/emissions/keeper/keeper.go:191-311` |
| Emissions → validators/stakers | VERIFIED | SDK `mint` + `distr` modules; the emissions keeper mints the contributor portion only |
| Org-creation burn sink | VERIFIED | slot auction burns 50%, `x/org/keeper/keeper.go:230-243` |
| Leader revenue demand leg | not yet built | 0 `revenue` hits in the chain (verified absent as documented) |

Per D-ECON-STORAGE-MARKET, `ProcessOrgPayouts` + `RepTier`/`PayoutPerMemory` + the
`DebitTreasury` payout path were REMOVED (commit `9bd601b`). Per-memory and per-serve payout fields
return zero hits anywhere in the chain — there is no payout dimension tied to individual memories
or serves.

## Layer 5 — Attestation / trust

- Terminology (D-ATTEST-TERMINOLOGY-SPLIT): "attestation" = provenance only; the serve concept uses
  "serve receipts" (`StoredServeReceipt` / `StoredDenialReceipt`).
- `D-ATTEST-PROOF-TIER` (defer-and-keep-warm, PENDING-SPIKE) is **PIVOT-SUPERSEDED**:
  re-expressed by `D-PROVENANCE-ADMISSIBILITY-2026-07-23` as a P1 tier (`tee_receipt` / CommitLLM)
  and a P2 tier (`zktls_proof` / blind-witness). Both tiers remain mostly UNBUILT.
- `x/attestation` is disabled-but-wired: `SubmitSessionAttestation` returns
  `ErrAttestationDisabled` (`msg_server.go:24-26`).
- The CO-282 spike record is VERIFIED present at
  `wevibe-meta/workspace/spikes/zktls-attestation/RESULTS.md`.

## Status

All verdicts above carry file:line + git-history evidence in report 1787047783 (WO-TOPOLOGY-FLOWS,
Region 1). Net corrections vs the 2026-06-14 doc: hub directory + hub-response-signing flipped
design-only → BUILT; per-memory rarity deleted as DEAD; social graph narrowed to contributor stats;
per-approved-memory payout replaced by the flat even split; proof-tier design re-expressed under
D-PROVENANCE-ADMISSIBILITY-2026-07-23.

# 24. Consumer Path, PRE Retrieval Data Flow, and Denial Receipt Flow

> Rewritten section — replaces the 2026-06-14 TOPOLOGY.md text. State corrected per read-only audits
> `1787047783-WO-TOPOLOGY-FLOWS` and `1787048733-WO-TOP2-UMBRAL` (all verdicts carry file:line and
> git-history evidence). Pivot context: RECALL-PIVOT (live 2026-07-30, edge-policy-v1 anchored h45,
> hash `2d2faa14…87899e`) supersedes the per-keyword-weight decay model; D-ECON-STORAGE-MARKET
> supersedes per-memory payouts.

## 24.A Consumer Path

Four-step auth chain (step numbers follow the request):

1. **Bearer token.** The consumer agent holds a session token at `~/.wevibe/mcp-session-token`
   (mode 0600). Every MCP HTTP endpoint is Bearer-gated (`http-server.ts:3192-3351`, `authorize()`).
2. **MCP request normalization + signing.** The MCP canonicalizes each outbound hub request and signs
   it in-process.
3. **Hub dispatch with resolved endpoint.** The MCP chain-resolves the org's hub endpoints
   (see below) — NOT a hardcoded URL — then POSTs the signed request.
4. **Hub verification + response.** The hub verifies and executes; signed hub responses are
   verified by the MCP on the way back (`X-Hub-Signature` against the org's `hub_response_pubkey`,
   `hub-fetch.ts:94-98`).

**CORRECTION — "delegate signing" is DEAD.** The MCP signs requests itself: consumer-signed header
`WeVibe-Signed: pubkey,timestamp,signature` (`auth.ts:97-113`) with org-serve-key canonical
signatures (`serve-signing.ts:95-128`). Delegate-identity helper code was removed (git `9b7465c`).
The MCP has no direct chain broadcast — the hub is the sole relay; the only chain data the MCP reads
is the org's serving info.

**CORRECTION — hub endpoint resolution is BUILT (was "design-only").** `hub-resolver.ts:82-124`
chain-resolves the org's `hub_endpoints` (1–3 ordered URLs, `MsgSetServingInfo` /
`state.proto` field 18) with `pickActiveEndpoint` failover; hub-fetch verifies the hub's response
signature. Both ends wired and live.

**MCP consumer endpoints** (all Bearer-gated):

| Endpoint | Purpose |
|---|---|
| `/v1/health` | liveness |
| `/v1/recall` | recall (hub relay + local decrypt of re-encrypted memories) |
| `/v1/serves` | serve receipt submission |
| `/v1/reports` | wallet-signed reports |
| `/v1/denials` | denial receipts (consumer ladder, §24.C) |
| `/v1/org-setup` | B2 org-setup bridge (start) |
| `/v1/org-setup/finalize` | B2 org-setup bridge (finalize) |
| `/v1/provision-recall` | B2 org-setup bridge (provisioning recall) |

**Consumer-path DEVIATED→BUILT summary (old doc fixes):** chain directory
(`hub_endpoints`, `hub_serving_address`, `hub_response_pubkey`, `MsgSetServingInfo`), hub-response
signature verification, and endpoint failover are all implemented — the old doc's and
DECISIONS.md:3270's "design-only / multi-repo build not started" wording was stale in both places.
Cross-module references `wevibe-mcp/docs/TOPOLOGY.md` and
`wevibe-server/wevibe-hub/docs/TOPOLOGY.md` still exist; only the old doc's `WeVibe/` repo-root
prefix was stale.

## 24.B PRE Retrieval Data Flow

### Member-removal → kfrag purge

`MsgRemoveMember` → `member_removed` event (attrs `{org_id, member_pubkey, removed_by,
block_height}`, `x/org` `msg_server.go:149-185`) → hub watches CometBFT `NewBlock` and decodes txs;
on `MsgRemoveMember` it calls `OnMemberRemoved` → sidecar gRPC `DeleteKFrags` (`watcher.go:464,514`;
`sidecar.proto:20-23`). Nuance: the hub subscribes via tx-decode on NewBlock, not a dedicated
`member_removed` WebSocket channel — same effect, different mechanism.

### Approval + retrieval (the two Umbral paths)

Two PRE paths are both live and have a strict caller split (canon `D-LEADER-SIDE-UMBRAL-MINT`):

- **MCP in-process WASM** (`vendor/umbral-wasm`, `src/umbral.ts:53` — no subprocess): moderator
  approval decrypts the DEK and re-encrypts in-process to the epoch public key
  (`moderation.ts:242`); epoch-keypair derivation and kfrag minting also run locally
  (`org-client.ts:529,633,811,824`), and the consumer decrypts re-encrypted recall results locally
  (`decrypt-reencrypted`).
- **Hub-side gRPC sidecar** (`wevibe-umbral:4460`, container-only port via compose DNS): the hub
  calls `ReEncryptForMember` inside `QueryMemories` (`handlers/retrieval.go:388`), which re-encrypts
  the stored `capsule` under the requesting member and returns `cfrag` + `capsule` +
  `umbral_ciphertext` (retrieval.go:379,395,396). **The sidecar performs the ONLY re-encryption**;
  the MCP never calls a hub gRPC. The WASM `reencrypt` export is interface-only (no MCP wrapper).

Retrieval sequence: hub recall → sidecar `ReEncrypt` → response carries cfrag/capsule/umbral
ciphertext → consumer decrypts locally. Approval/retrieval is therefore: moderator decrypts DEK and
re-encrypts in-process via WASM; hub re-encrypts on recall; consumer finishes decryption.

### Metadata parity (hub → Qdrant)

- `SyncEpochData` 60-second ticker — lifecycle-only metadata sync (`sync.go:20`).
- `UpdateMemoryKeywords` — LIVE (`keywords.go:306,400`).
- `ScrollApprovedMemories` — **DEAD** (removed `575a1ac`); replaced by `ScrollOrgMemoryPayloads`.
- **Qdrant payload = `standing_bps` + flat `keywords` list** (`retrieval.go:405-419`).
  The old doc's `keyword_weights` payload is DEAD — per-keyword weights left consensus in the
  RECALL-PIVOT; standing now enters search as the single edge-computed scalar.

### PRE identity

`pre-identity-key` stored in the encrypted file keystore (`auth.ts:16-17`); per the Umbral second
pass, that is the AES-256-GCM file `~/.wevibe/keys/keys.json` — NOT the OS keychain (the keychain's
`@napi-rs/keyring` backend stores the identity seed `identity-seed-v1`; keytar never existed).
Registration is **lazy first-use** (`server.ts:330-338`, `identity-runtime.ts:22-41`) — the old
"on startup, calls pre-key per org" wording is wrong.

### B2 org-setup bridge

MCP exposes `/v1/org-setup`, `/v1/org-setup/finalize`, and `/v1/provision-recall` with a pending
TTL `PENDING_ORG_SETUP_TTL_MS = 30*60*1000` (30 min, `http-server.ts:101-104`); finalize returns
`{setup_id, payload, recovery_phrase}`.

### Leader-side kfrag minting + delivery

The LEADER mints kfrags locally from its master key (`D-LEADER-SIDE-UMBRAL-MINT`):

- Epoch key derivation (MCP `org-client.ts:53-63`):
  `epoch_sk = HKDF-SHA256(K_master, salt="", info="wevibe-umbral-epoch-{epoch}", len=32)` —
  canonical secp256k1 scalar (`SecretKey::try_from_be_bytes`, `crypto.rs:60`).
- **The hub never receives or holds `epoch_sk`** — hub stores `umbral_pk` (hex) plus
  capsule/ciphertext; kfrags live ONLY in the sidecar's disk-backed store
  (`WEVIBE_UMBRAL_KFRAG_STORE`, default `/data/kfrags.json`, Docker volume
  `wevibe_umbral_kfrags:/data`, atomic 0600 persist, `store.rs:327-376`).
- **Kfrag delivery:** leader-signed `POST /v1/orgs/{orgID}/members/{pubkey}/kfrag`
  (`main.go:307` → `members.go:500` StoreMemberKFrag, role=leader gate) → sidecar `StoreKFrag`.

### PRE endpoints — sidecar surface (5 RPCs, nothing more)

`proto/umbral/v1/sidecar.proto:10-31` defines exactly five RPCs: `StoreKFrag`, `ReEncrypt`,
`DeleteKFrags`, `DeleteOrgKFrags`, `Health`. **Everything else was RIPPED** (D-LEADER-SIDE-UMBRAL-MINT):
sidecar gRPC `GenerateKeyPair`/`GenerateKFrags` (removed `811f0fb`); hub
`umbralService.GenerateEpochKeyPair`/`RegisterMember`; hub routes `/v1/internal/epoch-keypair` and
`/v1/internal/orgs/{orgID}/kfrags` (removed `66b42a1`; zero `/v1/internal/` routes remain in the
router). Absence verified against proto, service.rs, generated.rs, and hub router — not assumed.

### API changes (hub `protocol/types.go`)

Live umbral fields: `pre_pubkey` (`:279`), `umbral_pk` (`:77,96`),
`umbral_capsule`/`umbral_ciphertext` (`:247-248`), `cfrag`/`capsule` (`:299-301`),
`requires_reencryption` (`:323`); `InviteMemberRequest` carries `PrePubkey`. **There is no
`epoch_sk` anywhere in the hub API.** Known staleness: `wevibe-protocol/openapi.yaml` still lists
`wrapped_dek_enc` on ApproveRequest and omits the umbral fields — a separate reconcile item.

## 24.C Denial Receipt Flow (CO-225, corrected)

### Dashboard: chain-submit page + denial-batch panel — DEAD

The `chain-submit` dashboard page and its denial-batch panel are **removed** (page deleted `da32c0b`;
no `MsgSubmitDenialBatch` path remains in the dashboard). The successor UX is
`/moderation/new`'s `LeaderPipelinePanel`, which commits `MsgSubmitCommitment` batches — denial
batches reach the chain through the leader/moderator path only.

### Consumer denial ladder (MCP side)

1. Agent records a denial → `POST /v1/denials` → entry appended to
   `~/.wevibe/pending-denials.json`, flushed to the hub in the background.
2. **Response is 200, NOT 202** (`http-server.ts:3189`).
3. Queued record shape (`denial-queue.ts:24-31`):
   `{id, org_id, epoch_id, memory_hash, reason, created_at}` — includes `epoch_id`, has
   **NO `consumer_pubkey`** (the old doc's record shape is stale).
4. **4xx responses ALSO dequeue** (`denial-queue.ts:153-161`) — malformed submissions are not
   retried forever.
5. **`~/.wevibe/blacklist.json` local-suppression filter is DEAD (MCP tier)** — the call was removed
   in `92c02a5`; the import survives in `server.ts:9` as dead code. The plugin tier consults the file
   live: `seedDeniedFromLocalBlacklist` reads `~/.wevibe/blacklist.json` at init and every transform
   turn to seed `deniedCids`, which gates injection.

### Hub intake

`POST /v1/orgs/{orgID}/denials` → `serve_events` row with `event_type='denial'`, `status='pending'`
(`main.go:351`, `serves.go:934`; table columns and CHECK constraints match) →
**response 201, NOT 200** (`serves.go:1035`). This is the load-bearing status split of the flow:
the consumer-facing MCP returns **200**; the hub intake returns **201**. Companion query
`GET /v1/orgs/{orgID}/denials/pending` (`main.go:353`).

> **⚠ LIVE BUG (F1) — flagged by the audit, still open:** the hub denial INSERT omits
> `episode_ref`, which `db/schema.sql:436` declares NOT NULL. On a fresh database (pre-MVP
> wipe-on-change semantics — no migrations) every denial write fails with **SQLSTATE 23502**
> (not_null_violation). Needs a fix in the hub denial insert or a schema default.

### Hub → chain batching (watcher)

`processDenialBatchBookkeeping` → `status='submitted'` (`watcher_serve.go:69`). **The
`chain_commit_events` bookkeeping insert is DEAD — the table was dropped in `72f91a2`; no migration
path exists.**

### PIVOT-SUPERSEDED: optimistic_weight formula

The old `optimistic_weight = chain_weight − pending_denial_count × DenialDecayBPS / 10000` formula
was **removed** (`9ccee0d`). Pending denials now enter scoring as a **flat −0.05 penalty per denial**
on the edge-standing base (`ranking_core.go:210-212`) — consistent with the RECALL-PIVOT move of all
weight/decay math to the edge policy.

### Chain: MsgSubmitDenialBatch + receipts

`x/serve` `MsgSubmitDenialBatch` → `StoredDenialReceipt` keyed
`org/epoch/denial-fingerprint` with `memory_hash` embedded
(`x/serve/keeper/msg_server.go:60-183`). Emitted event `denial_batch_submitted` with attrs
`{org_id, submitter, epoch, accepted_count, rejected_count, block_height}` — verified exact match.
Denial receipts are part of the 8 content-free consumer-signed event types the chain stores
(RECALL-PIVOT).

### PIVOT-SUPERSEDED: Earned Trust decay, payout_per_memory, archival

- `applyDecay` / `ApplyEpochDecay` / `calculateDenialRateAndTrust` — **DEAD** (removed `e6fcdae`).
- "archives when all weights ≤ retrievalThreshold" — **DEAD**; `MEMORY_STATE_ARCHIVED` is now reached
  via validity-window expiry (`validity.go:81-82`), not a weight threshold; `retrieval_threshold_bps`
  is dead. Archival is an edge-computed visibility outcome, not a chain weight gate.
- `x/emissions ProcessOrgPayouts` + `payout_per_memory` — **DEAD** (removed `9bd601b`,
  D-ECON-STORAGE-MARKET); replaced by `MintDailyEmission` flat even contributor split. Denial
  receipts no longer route into a per-memory payout path.

## Cross-cutting corrections carried by this section

| Old claim | Corrected state |
|---|---|
| Delegate signing on consumer path | DEAD — consumer-signed `WeVibe-Signed` + org-serve-key signatures |
| Hub endpoint resolution "design-only" | BUILT — `hub-resolver.ts` + `X-Hub-Signature` verification |
| Startup pre-key registration per org | LAZY first-use |
| Sidecar as Umbral "minting" service | DEAD — sidecar = 5 RPCs incl. the only ReEncrypt; mint RPCs + `/v1/internal/*` RIPPED |
| MCP denial response 202 | 200 (hub intake is 201) |
| blacklist.json suppression filter | DEAD (call removed 92c02a5) |
| `optimistic_weight` DenialDecayBPS formula | PIVOT-SUPERSEDED — flat −0.05/denial |
| `chain_commit_events` bookkeeping | DEAD (table dropped 72f91a2) |
| `applyDecay` / `ApplyEpochDecay` / `calculateDenialRateAndTrust` + weight-threshold archival | DEAD — validity-window expiry; edge-computed |
| `ProcessOrgPayouts` / payout_per_memory | DEAD (9bd601b) — `MintDailyEmission` flat split |

# 25 — RECALL → INJECTION PIPELINE (edge-standing · repeat-failure trigger · once-per-session injection)

> Charted against live code; every claim is `file:line`-cited — treat citations as load-bearing.
>
> Citations verified on-touch **2026-08-18** by two read-only audits: PASS-2 `1787048689-WO-TOP2-TRIGGER-CADENCE` (AUTHORITATIVE — wins on any conflict) and PASS-1 `1787047606-WO-TOPOLOGY-PIPELINE`. Exact lines drift with edits.
>
> **Layer ownership:** Stage 1 = `wevibe-server/wevibe-hub` (Go) · Stage 2 = `wevibe-mcp` (TS) · Stage 3 = `wevibe-opencode-plugin` (TS) · Stage 4 = `opencode` runtime (homebrew binary).
>
> **CONTRADICTION RESOLVED:** the old TOPOLOGY.md disagreed with itself — line 1507 ("once per session, not every turn") vs line 2048 ("EVERY-TURN PUSH"). **Once-per-session is the current truth** (D-INJECTION-CADENCE-2026-07-24, DECISIONS.md:3720); the "EVERY-TURN PUSH" text was stale pre-2026-07-24 behavior. This section states the live pipeline only.
>
> **Two pivots make this section what it is:** RECALL-PIVOT (2026-07-29/30 — edge-standing scalar replaces per-keyword chain weights in Stage 1 scoring) and D-RECALL-TRIGGER-2026-08-08 (repeat-failure GSTV trigger replaces per-`chat.message` prewarm in Stage 3). Neither pivot changed the transport hop chain (§5 below).

## Pipeline at a glance (4 stages, root → leaf)

```
FAILURE EPISODE — second red under a stable failureKey (opencode session)
   │  plugin hook tool.execute.after — the SOLE recall trigger (wevibe-plugin.ts:2227)
   ▼
[Stage 3a] triggerRecall(sid, query, "repeat_failure") ── POST /v1/recall ──┐
                                                                              ▼
[Stage 2]  wevibe-mcp handleRecall (http-server.ts:422)
           → retrieve() → queryOrgMemories — ed25519-sign body, X-Agent-Signature
           ── POST /v1/orgs/{org}/query ──┐
                                          ▼
[Stage 1]  wevibe-hub QueryMemories (handlers/retrieval.go:30)
           → body-sig verify (five 401 paths) → Qdrant REST points/search
           → ScoreAndRank (overlap × edge-standing) → floor + budget
           → power-law sampler → chain attest + Umbral ReEncrypt → QueryResponse
                                                                     ▼
[Stage 2]  per-memory: in-process WASM Umbral decrypt (umbral.ts, no subprocess)
           → wevibe-guard spawnSync scan (non-blocking; guard.passed=false on fail)
           → {status, memories[]}  ─────────────────────────────┐
                                                                ▼
[Stage 3b] experimental.chat.system.transform fires EVERY turn, but the
           once-per-session gate (injectedCids) lets each cid through ONCE
           → insertAtStableEarlyPosition(output.system, block) = splice(1,0)
                                                                ▼
[Stage 4]  opencode 1.18.15 LLMRequestPrep.prepare → {role:"system"} message #2
           (OR providerOptions.instructions on OpenAI+OAuth) → MODEL
```

---

## Stage 1 — Hub retrieval ROOT (`wevibe-server/wevibe-hub`, Go)

**Route:** `POST /v1/orgs/{orgID}/query` (`cmd/wevibe-hub/main.go:355`), behind `auth.RequireVerifiedMembership` (`:298`). **There is NO `/v1/recall` route on the hub** — `/v1/recall` is the MCP endpoint (Stage 2).

### Scoring math — AS IT IS NOW (RECALL-PIVOT re-based this)

`ScoreAndRank` (`internal/retrieval/ranking_core.go`) — the per-keyword `Σ(queryWeight·memoryWeight)` of the old doc is **GONE**. Current math:

1. **`keywordOverlap = |query ∩ memory| / |query|`** (`ranking_core.go:107-129`) — set-overlap ratio, no chain keyword weights.
2. **`standingFactor = StandingBps / 10000`** (`:182-191`) — the edge-standing scalar.
3. **`gammaBoost = 0.1 · keywordOverlap · standingFactor`** — KeywordBoostFactor γ = 0.1, local const (`retrieval.go:512-513`).
4. **`cappedBoost = min(gammaBoost, 0.15 · VectorScore)`** — KeywordBoostDelta δ = 0.15 (`retrieval.go:512-513`).
5. **`final = VectorScore + cappedBoost`**.

- **Admission gate:** drops `Archived` memories and any memory with `StandingBps ≤ threshold` (threshold = **1500 bps** from `policy/edge-policy-v1.json`; `must_not` ARCHIVED encoded in the Qdrant filter of the search request).
- **Pending-denial:** flat **`max(0, final − PendingDenials · 0.05)`** (`ranking_core.go:210-212`).
- **New-memory boost SURVIVES** (`:214-221`): multiplier **0.5**, grace **20**, window **30**.
- **`DenialDecayBPS` / `ServeBoostBPS` are DEAD** — 0 matches anywhere in `internal/retrieval`. γ/δ constants survive but now scale overlap·standing, not per-keyword chain weights.
- Wire note: `ScoringBreakdown.keyword_score` (`internal/protocol/types.go:233`) still exists on the wire but now carries the overlap ratio — wire-compatible, semantically misleading.

### Edge-standing (hub-local, policy-anchored)

Standing is computed **at the edge** by the hub, not read as a per-keyword weight from the chain: `internal/standing/{engine,policy}.go` + `policy/edge-policy-v1.json` (**initial_standing_bps = 10000, threshold = 1500**). Startup runs a **fatal on-chain anchor verification** (`main.go:112-144`); `SyncStandingFromEvents` (`:174`) keeps it current; Qdrant points carry the `standing_bps` payload.

### X-Agent-Signature (replaces dead `agent_sig`)

`QueryRequest.agent_sig` body field is **DEAD** (removed hub `575a1ac` / MCP `2fcd60b`; word-boundary grep = 0 matches — the only `agent_sig*` hits are the `agent_signature` receipt column). The hub instead verifies **ed25519 over the raw request body** (`handlers/retrieval.go:39` read → `:46` header) with **FIVE 401 paths**: missing header `:47-51`, bad format `:53-58`, missing pubkey `:73-77`, bad pubkey `:79-84`, verify-fail (`ed25519.Verify` `:86`) `:87-90`. The verified signature is stored into **`usage_receipts.agent_signature`** (`receipts.go:43-52`; call sites `retrieval.go:277` zero-result, `:456` normal).

### Qdrant layer

Pure **HTTP REST** client (no gRPC/SDK): `QdrantClient` (`retrieval.go:235`, client `:262-270`); **fresh `&http.Client{Timeout: 10s}` per call** (10 construction sites, no pooling/keep-alive). Search = `POST points/search` (`:540`), search limit = `recallDepth` (5000).

---

## Stage 2 — MCP recall MIDDLE (`wevibe-mcp`, TS)

**Entry:** `POST /v1/recall` (`http-server.ts:3206-3207`, Bearer auth `authorize()` `:152`) → `handleRecall` (`:422`) → `retrieve()` (`retrieve-cli.ts:255`) → `queryOrgMemories` (`org-client.ts:126`, POST at `:163`).

- **Request signing:** `queryOrgMemories` JSON-stringifies the exact body (`org-client.ts:143`), signs it (ed25519, `:149`), and sends body `:166` + header **`X-Agent-Signature`** `:165`. `rg "agent_sig"` over `wevibe-mcp/src` = 0 matches — the dead field left no residue.
- **Decrypt loop** (`retrieve-cli.ts:426-524`): per-memory, ciphertext accepted as `ciphertext_hex ?? encrypted_blob` (`:449-450`, chain-first shape).
- **Umbral is IN-PROCESS WASM** — `umbral.ts:53` `require('../vendor/umbral-wasm/…')`. No env var, no binary, **no subprocess**.
- **Guard is non-blocking:** `wevibe-guard` runs via `spawnSync` (`guard.ts:43`, `WEVIBE_GUARD_BIN` fallback `:19-29`); a failing memory is still returned, annotated **`guard.passed=false`** (`http-server.ts:521-543`).
- **Decrypt failure silently `continue`s** (`retrieve-cli.ts:518-523`); reason_code `decrypt_failed` is set **only if ALL memories fail** (`:567-575`).
- **`provisionRecall`** (`org-client.ts:786`, was `:644`; HTTP `http-server.ts:3256`, admin `cmdProvisionRecall` `:551-564`) — tail §6 re-mint recovery.

---

## Stage 3 — Plugin recall + inject LEAF (`wevibe-opencode-plugin/plugins/wevibe-plugin.ts`)

### 3a — Recall TRIGGER is repeat-failure (D-RECALL-TRIGGER-2026-08-08)

Recall fires **ONLY on the second red under a stable `failureKey`**, once per key per session, re-arming only when the failureKey changes. **Sole trigger hook: `tool.execute.after`** (`wevibe-plugin.ts:2227`) → `assessRecallNeed` (`:2239`) → `computeFailureKey` → `episodeTracker.openOrTouch` (`:2294`/`:2308`) → arm → `triggerRecall(sid, need.query, "repeat_failure")` (`:2301`/`:2317`). `RecallTrigger = "repeat_failure"` — the type appears at `:109` and nowhere else.

- **`chat.message` NEVER fires recall** (`:1832`): it is session-tracking only — "Recall never fires on user prompts" (`:1833-1835`), it just records `activeSessionId` (`:1836`).
- **Arm predicate (exact):** `!episode.opened && !episode.fired && editSeen && wevibeAvailable && bindingState.active && !recallInFlight` (`:2297` tripwire / `:2313` cascade). `!episode.opened` = second-red gate; `!episode.fired` = once-per-key-per-session; `editSeen` = C3b flake guard; `!recallInFlight` = single-arm (latch `:1606`).
- **Key split (D-RECALL-KEY-SPLIT):** `computeFailureKey` (`failure-key.ts:24-26`) = `sha256Hex("wevibe-failure-v1\n" + repoBinding + "\n" + predicateId + "\n" + identity)`, `identity = obs.failingTest ?? "cmd:"+commandFp8` — the **stable episode identity**. Contrast `needSignature` (`metrics.ts:200`) — **volatile**, recomputed per attempt, query material only.
- **GSTV predicate declared ONCE** at `${repoRoot}/.wevibe/predicate.json` (`predicate-binding.ts:5-6`), resolved at bind (`resolvePredicateForRepo` `:119-138`), cached per repo. (Fidelity note: the declaration's `command` is never executed — live red/green measurement is the machine-readable reporter parse, `bench-fixture-adapter.ts:23-88`; host non-zero exit is the cheap tripwire.)
- **Episode lifecycle (`outcome-episode.ts`):** open on first red `openOrTouch` (`:182-245`), accumulate repeat red (`:210-216`), close on green `observeToolResult` (`:286-319`), close on idle/expiry `onSessionIdle` (`:321-331`, **`EXPIRY_IDLE_TURNS=2`** `:78`). (Fidelity note: `closeSession` `:333-340` has no live caller — idle-expiry is the operative "unobserved" closer.)
- **Cascade + flake-guard:** C3a — only the first sorted failing id arms, the rest `markFired` (`:2304-2323`); C3b — `editSeenBySession` read-and-reset (`:2275-2276`).
- **Prewarm IIFE KILLED** (git `72a3204`): init IIFE (`:1705-1719`) is only `getRecallMode` + `ensureWeVibeMcpRunning` + `gcServedMemories`; `"prewarm"` survives solely as an inert `currentSessionId()` fallback (`:632`).

### 3b — Injection is ONCE-PER-SESSION (D-INJECTION-CADENCE-2026-07-24)

`experimental.chat.system.transform` (`:1839-2124`) fires every turn, but each cid injects **at most once per session**:

- **Gate:** `candidates = eligible.filter(m => !injectState.injectedCids.has(m.cid))` (`:1924`); empty → `[inject] steady_state … cadence=once` return (`:1925-1943`).
- **Push:** `insertAtStableEarlyPosition(output.system, memoryBlock)` (`:1994`) = `system.splice(1, 0, block)` (`:138`) → **system message #2** — a stable early position, immune to later append churn.
- **Session identity = OpenCode's real `sessionID`:** per-session state keyed `sid = currentSessionId()` (`:1921`) — `sessionInjectedCids` (`:598`, served-cid ledger) and `sessionInjectState` (`:607`, injection gate). The old 40-hex process-global session id is **gone** (removed `72a3204`).
- Compaction: verbatim restore via `output.context.unshift` under `experimental.session.compacting` (`:2126-2144`).
- **This resolves the old TOPOLOGY.md contradiction:** line 1507 ("once per session") was current truth; line 2048 ("EVERY-TURN PUSH") was stale pre-2026-07-24 behavior.

---

## Stage 4 — Inject → LLM boundary (`opencode` runtime — homebrew 1.18.15)

**Runtime is homebrew opencode 1.18.15** (`/opt/homebrew/bin/opencode` → `Cellar/opencode/1.18.15`; `opencode --version` = `1.18.15`). **NOT a vendored 1.16.0 binary** — no such binary exists anywhere in the workspace. The "1.16.0" label refers to the **npm `@opencode-ai/plugin` TYPE package** under `.opencode/node_modules`, whose hook signature is exact at `dist/index.d.ts:265-270`: `(input: { sessionID?: string; model: Model }, output: { system: string[] }) => Promise<void>`. (Aside: `wevibe-opencode-plugin/package.json` pins `@opencode-ai/plugin: 1.4.10` — two package versions, identical hook signature, no delivery impact.)

- **`LLMRequestPrep.prepare`** — **10 recovered steps** from the real binary (de-minified; prior audits under-enumerated 5). Key behaviors: `instructions` joined **only** when `provider.id === "openai" && auth.type === "oauth"`; message-prepend skipped on `isWorkflow`.
- **Delivery verdict (definitive):** plugin `splice(1,0)` → **system message #2 IS delivered** on the normal path; OpenAI+OAuth → `providerOptions.instructions` (verbatim pass-through); workflow → `systemPrompt = system.join("\n")`. The old `opencode-ai/opencode#885` citation is DEAD (404) — retired.

---

## §5 — End-to-end hop chain (UNCHANGED by the trigger pivot)

The trigger/cadence pivots live entirely in the plugin; the transport below is identical before and after (no recall-trigger route exists in either MCP route table or hub route block):

```
plugin  ──POST /v1/recall──────────────▶ MCP      http-server.ts:3206
                                        handleRecall :422 → retrieve() retrieve-cli.ts:255
MCP     ──POST /v1/orgs/{orgID}/query─▶ hub      org-client.ts:126 (signs body :149,
                                                   X-Agent-Signature :165) → main.go:355
                                        QueryMemories handlers/retrieval.go:30
                                                   (ed25519 body-sig verify :46-90)
hub     ──HTTP REST points/search──────▶ Qdrant   retrieval.go:540
hub     ──gRPC GetMemoriesBatch────────▶ chain    internal/chain/query.go:415
hub     ──gRPC ReEncrypt───────────────▶ Umbral   internal/umbral/service.go:56
hub     ──verified sig stored──────────▶ usage_receipts.agent_signature (receipts.go:46)
```

---

## §6 — Tail: kfrag persistence + wipes

- **`KFragStore` is disk-backed** (`wevibe-umbral/src/store.rs:48`; atomic persist `:307-393`); path via `WEVIBE_UMBRAL_KFRAG_STORE`, default `/data/kfrags.json` (`docker-compose.yml:102,104`).
- **`make dogfood` = `docker compose down -v` STILL WIPES** the volume (`wevibe-meta/Makefile:77,180`) — kfrags included. Recovery after a wipe = `provisionRecall` re-mint (`org-client.ts:786`, see Stage 2).

---

*Sources (PASS-2 authoritative on conflict): `wevibe-meta/workspace/reports/1787048689-WO-TOP2-TRIGGER-CADENCE.md` (AUTHORITATIVE CURRENT-STATE BLOCK), `wevibe-meta/workspace/reports/1787047606-WO-TOPOLOGY-PIPELINE.md`. Supersedes old TOPOLOGY.md §"RECALL → INJECTION PIPELINE" (lines 1934–2124).*
