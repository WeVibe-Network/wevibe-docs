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

### Empirical Replay Mode (CO-034, in flight)

A second compose mode for empirical replay against sim Steady-State will land in CO-034:

- **Overlay file:** `wevibe-server/docker-compose.fast.yml` — overrides chain epoch duration via `WEVIBE_EPOCH_DURATION_SECONDS` (default 2s) so 300 epochs complete in ~10 minutes instead of multi-day at production duration.
- **Activation:** `docker compose -f docker-compose.yml -f docker-compose.fast.yml up -d` via the `dogfood-fast` Makefile target in `wevibe-meta`.
- **Production mode:** unchanged. Production deployments leave `WEVIBE_EPOCH_DURATION_SECONDS` unset; the chain default applies.

Used by the empirical replay harness at `wevibe-meta/scripts/empirical_replay/` (CO-034) to measure the Sprint 32 contract: `chain.gap ≥ 75pp vs sim Steady-State, |Δ| ≤ 5pp`.

### Chain Broadcast (CO-258)

Leader/member chain tx broadcast is dashboard wallet-direct (`directBroadcast` to chain RPC). Per DECISIONS amendment 13 (D-ECON-STORAGE-MARKET), the hub does not relay leader-signed txs or expose a delegate-key relay path.

### Schema Bootstrap

Hub schema at `wevibe-server/db/schema.sql`. Applied on Postgres container init (D-13.10).

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
# POST /v1/orgs/{orgID}/members/delegate-key — REMOVED by DECISIONS amendment 13 (CO-214 delegate-key path retired)
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
POST   /v1/orgs/{orgID}/moderation/batch-submit                          # Hub-internal queue; NOT chain
POST   /v1/orgs/{orgID}/serves                                           # Record serve event (CO-033a: matched_keywords required, non-empty, validates 400 on empty per D-4.2 ingress)
POST   /v1/orgs/{orgID}/denials              # Record denial event (CO-225); increments
                                             # optimistic pending_denial_count per
                                             # D-2026-05-25-A (load-bearing for query
                                             # ranking)
GET    /v1/orgs/{orgID}/denials/pending-count  # D-2026-05-25-A: leader denial-batch panel
GET    /v1/orgs/{orgID}/denials/pending        # D-2026-05-25-A: leader-only list,
                                               # newest-first, capped at 200 rows,
                                               # includes total_count for batch UI
# POST /v1/orgs/{orgID}/serves/batch-submit  — DELETED CO-011a.4: dashboard relays MsgSubmitServeBatch
# POST /v1/orgs/{orgID}/denials/batch-submit — DELETED CO-011a.4: dashboard relays MsgSubmitDenialBatch directly to chain (Category A per D-2026-05-25-A; supersedes the earlier Category B framing)
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
# POST /v1/relay/broadcast — REMOVED by DECISIONS amendment 13 (hub relay removed; wallet-direct chain writes only)
GET    /v1/orgs/{orgID}/my-submissions                                   # Contributor-only submission status view (CO-265)
GET    /v1/orgs/{orgID}/submissions/keywords/pending                     # List pending keyword verification (CO-238)
GET    /v1/orgs/{orgID}/submissions/keywords/pending-chain               # List ready for chain submit (CO-238)
# POST /v1/orgs/{orgID}/submissions/batch-chain-submit — DELETED CO-011a.4: dashboard relays MsgApproveMemory directly
# POST /v1/orgs/{orgID}/serves/batch-submit            — DELETED CO-011a.4: dashboard relays MsgSubmitServeBatch
# POST /v1/orgs/{orgID}/denials/batch-submit           — DELETED CO-011a.4: dashboard relays MsgSubmitDenialBatch directly to chain (Category A per D-2026-05-25-A; supersedes earlier Category B framing)
GET    /v1/orgs/{orgID}/denials/pending-count                            # Leader denial-batch panel count (D-2026-05-25-A)
GET    /v1/orgs/{orgID}/denials/pending                                  # Leader-only pending list (newest-first, capped at 200, includes total_count)
GET    /v1/orgs/{orgID}/health
GET    /v1/orgs/{orgID}/finances                                         # Credits + chain financial data (CO-266, GAP-O6)
GET    /v1/orgs/{orgID}/chain-config                                     # Chain config read (CO-266, GAP-O7, leader-only; CO-011a.4 deleted the PUT counterpart)
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
RetrievalTemperature       float64 // env: RETRIEVAL_TEMPERATURE, default: 0.7
RetrievalNewMemBoostMult   float64 // env: RETRIEVAL_NEW_MEM_BOOST_MULT, default: 0.5
RetrievalNewMemBoostWindow uint64  // env: RETRIEVAL_NEW_MEM_BOOST_WINDOW, default: 30
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
func CreateOrg(w, r)            // POST /v1/orgs — sig verified, leader-only; persists epoch umbral_pk and synchronously calls RegisterOrgOnChain
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
// RegisterDelegateKey removed by DECISIONS amendment 13 (CO-214 path retired)
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
func ListMemories(w, r)        // GET — scroll through approved memories with pagination (keyword_weights/lifecycle_state from Qdrant payload)
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
- Query ranking combines vector score, keyword overlap boost, optimistic pending-denial decay, and new-memory boost before D-9.4 position assignment
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
**Role:** Internal hub chain submission helpers for operational flows (not a dashboard delegate relay path)
**Exports:**
```go
func (c *GrpcClient) SubmitMemoryToChain(ctx, orgID string, mem BatchMemory) (string, error)
func (c *GrpcClient) SubmitMemoryBatch(ctx, orgID string, memories []BatchMemory) ([]string, error)
func (c *GrpcClient) SubmitServeBatch(ctx, orgID string, epoch uint64, entries []ServeEntryInput) (string, error)
func (c *GrpcClient) SubmitMemoryReport(ctx, orgID string, input ReportMemoryInput, contributorWalletAddress string) (string, error)
```
**Chain-side reputation fanout (CO-213):** Each submit function now includes reputation increment messages:
- `SubmitMemoryToChain` broadcasts `MsgIncrementContribution` with contributor wallet (or pubkey fallback)
- `SubmitServeBatch` broadcasts `MsgIncrementServe` for each serve entry
- `SubmitMemoryReport` (ban path) broadcasts `MsgRecordBan` with contributor wallet when provided

#### `internal/chain/query.go`
**Role:** On-chain state queries for org/memory/reputation verification and retrieval metadata reconciliation
**Exports:** `IsOrgRegistered`, `GetOrgFromChain`, `GetEpochMerkleRoot`, `GetServeParams`, `GetEpochServeStats`, `GetAttestationParams`, `GetSessionAttestation`, `GetBandwidth`, `GetEmissionsParams`, `GetReputationStats`, `GetContributorProfile`, `GetMemoriesBatch`

**Memory batch parity (CO-224):** `MemoryBatchResult` includes `Keywords`, `Epoch`, `State`, and `MemoryType` copied from chain `StoredMemoryCommitment` so hub can reconcile Qdrant payload metadata.

**Chain→Hub reputation wiring (CO-213):** `GetContributorProfile` queries the chain's x/reputation module for a contributor's on-chain profile (contribution_count, serve_count, first_seen_epoch). Nil-safe — returns nil if chain unreachable or contributor not found. Used by `retrieval.GetContributorStats` to merge chain and hub data.

#### `internal/chain/sync.go`
**Role:** Epoch metadata sync loop implementation for retrieval parity (CO-224)
**Exports:** `SyncEpochData(ctx, chainClient, qdrantClient, pool) error`

**Sync flow (CO-224):**
- Loads orgs with approved memories from PostgreSQL (`pending_submissions.status='approved'`)
- Scrolls org memory payloads from Qdrant (`cid`, `keyword_weights`, `lifecycle_state`, `memory_type`)
- Batch-queries chain via `GetMemoriesBatch`
- Updates changed Qdrant payloads via `retrieval.UpdateMemoryState`
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
// Delegate-key member helpers removed by DECISIONS amendment 13 (CO-214 path retired)
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
func (c *QdrantClient) UpsertPoint(ctx, entry) error  // stored-vector noise DISABLED by default (σ=0, D-9.5); injectGaussianNoise is a no-op at σ=0
func (c *QdrantClient) QueryPoints(ctx, orgID, epochs, vector, keywordWeights, embeddingModelID, limit, includeDormant) ([]MemoryResult, bool, error)
func (c *QdrantClient) CountPoints(ctx) (int64, error)

// Package-level wrappers
func AddToIndex(ctx, client, entry) error
func EnsureCollection(ctx, client) error
func QueryByKeywords(ctx, client, orgID, accessibleEpochs, keywordWeights, vector, embeddingModelID, limit, includeDormant) ([]MemoryResult, bool, error)
func ScrollOrgMemoryPayloads(ctx, client, orgID) ([]OrgMemoryPayload, error)
func ScrollApprovedMemories(ctx, client, orgID, limit, offset) ([]MemoryResult, string, error)
func UpdateMemoryKeywords(ctx, client, orgID, oldKeywords, newKeyword) error
func UpdateMemoryState(ctx, client, orgID, memoryCID, lifecycleState) error
```
**Constants:** `EMBED_DIM = 768`, `contestedThreshold = 0.20`
**Key detail:** `UpsertPoint` calls `injectGaussianNoise`, but stored-vector noise is **DISABLED by default (σ=0)** per D-9.5 (it was inherited Echo code that cost ~20pp good-memory recall). `QueryPoints` fetches up to `recallDepth` (default 5000) candidates, then applies keyword-overlap boost, optimistic pending-denial decay, and new-memory boost, then assigns positions with D-9.4 tempered power-law sampling (strict top-1; positions 2..N sampled without replacement).
**Noise injection:** `injectGaussianNoise(vector, sigma)` — present but **inert by default (σ=0, D-9.5)**; configurable via `RETRIEVAL_VECTOR_NOISE_SIGMA`
**Lifecycle filtering (CO-224):** `ARCHIVED` is always excluded; `DORMANT` is excluded unless `includeDormant=true`
**Qdrant payload fields:** `cid`, `org_id`, `epoch_id`, `content_flags`, `keyword_weights`, `lifecycle_state`, `memory_type`, `embedding_model_id`, `embedding_schema_version`, `vector_dim`
**Retrieval env wiring (`cmd/wevibe-hub/main.go` + compose):** `RETRIEVAL_TEMPERATURE`, `RETRIEVAL_NEW_MEM_BOOST_MULT`, `RETRIEVAL_NEW_MEM_BOOST_WINDOW` configure `ProbabilisticRanker`.
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
> **FORWARD NOTE (Sprint 32, identity overhaul — DECISIONS.md `D-IDENTITY-PROGRESSIVE-CUSTODY`, brief `wevibe-meta/workspace/reports/design-identity-onboarding-migration.md`):** the Ed25519 `WeVibe-Signed` verification below stays, but the identity it authenticates is being reworked. Incoming: identity is a **client-held key created at first run and protected by a passkey (WebAuthn)** — no wallet required to participate; the dashboard moves from its current Keplr-signature-derived Ed25519 identity (`lib/wevibe-auth.ts`) and the plugin/MCP from its random on-device keypair onto a **single shared passkey-wrapped client-key scheme** (the two must mint the same identity). The key's ciphertext may be backed up to the hub but the **hub never holds a usable signing key** (non-custodial). A Cosmos wallet becomes an **optional linked authority** (staged handover, not migration) needed only to claim rewards / pay mainnet fees. Members are keyed by pubkey (`wallet_address` nullable), so wallet-free contribution already works at the hub layer. This section updates as it lands.

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
(REMOVED by amendment 13) RegisterDelegateKeyRequest / DelegateKeyRecord (CO-214 delegate-key relay contract retired)
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
# delegate_keys        — REMOVED by DECISIONS amendment 13 (CO-214 delegate-key storage retired)
org_keywords          — PK: id. UNIQUE: (org_id, keyword). Created via RunMigrations.
memory_keywords      — PK: (memory_cid, keyword). FK: (org_id, keyword) REFERENCES org_keywords.
```
**Indexes:** `idx_orgs_leader`, `idx_orgs_status`, `idx_members_active`, `idx_members_pubkey`, `idx_members_pubkey_active`, `idx_pending_org_status`, `idx_pending_contributor`, `idx_receipts_org_epoch`, `idx_audit_org_epoch`, `idx_credit_txn_org`, `idx_envelopes_org`, `idx_recovery_shares_holder`, `idx_dashboard_keys_pubkey`, `idx_org_keywords_org` (WHERE NOT deprecated), `idx_memory_keywords_keyword`

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

> **FORWARD NOTE (Sprint 32, storage-market overhaul — DECISIONS.md `D-ECON-STORAGE-MARKET`, brief `wevibe-meta/workspace/reports/design-storage-market-economy.md`):** the file:line details below reflect CURRENT (pre-overhaul) code. Incoming changes: `x/org` → slot registry + acquisition auction + self-assessed-V rent + demand-leg router (replaces `DeriveOrgID`/`DynamicPrice`); `x/memory` → per-memory storage deposit + keeper-prune + report-state transition; `x/emissions` → removal of the org-treasury payout subsystem + WorkScore/operator machinery; `x/bandwidth` memory-cap wired as the testnet DDoS guard (deleted at mainnet); `x/attestation` neutered (disabled-but-wired). This section is updated as those land.

| Module | Keeper Path | Proto Path | Tests | Purpose |
|--------|------------|-----------|-------|---------|
| x/attestation | x/attestation/keeper/ | proto/wevibe/attestation/v1/ | keeper + integration | Session-attestation storage (NOT merkle — merkle roots live in x/memory). Being neutered: disabled/no-op until verification infra (D-ATTEST-ROADMAP). Optional TEE model-provenance (D-ATTEST-TEE-TIER) is verified OFF-CHAIN first (Phase 0, no chain change); on-chain anchoring here is Phase 1. |
| x/bandwidth | x/bandwidth/keeper/ | proto/wevibe/bandwidth/v1/ | keeper + integration | Bandwidth throttling |
| x/emissions | x/emissions/keeper/ | proto/wevibe/emissions/v1/ | keeper | Emission pool, epoch emission, work scores (32-yr schedule scheduled CO-041) |
| x/memory | x/memory/keeper/ | proto/wevibe/memory/v1/ | keeper + integration | Memory commitments |
| x/org | x/org/keeper/ | proto/wevibe/org/v1/ | keeper + integration | Org registration, membership |
| x/reputation | x/reputation/keeper/ | proto/wevibe/reputation/v1/ | keeper | Contributor reputation |
| x/serve | x/serve/keeper/ | proto/wevibe/serve/v1/ | keeper + integration | Serve attestations |

- **Design-only (not yet built):** `x/org` `StoredOrg` gains `hub_endpoints` (ordered list of 1–3 transport URLs for failover redundancy) + leader-signed setter (`MsgSetServingInfo` extending `MsgSetServingKey`, or `MsgSetOrgConfig`); proto updates regenerate via Docker `make proto-gen` (never hand-edit `.pb.go`). Decision: `D-CHAIN-RESOLVED-HUB-ENDPOINT`.

### Genesis Seeding & Epoch Hooks (Sprint 32 / CO-040)

**module.HasGenesis wiring (CO-040, DECISIONS D-S32-HASGENESIS-CUSTOM-MODULES).**
`x/emissions` and `x/reputation` implement `cosmos-sdk/types/module.HasGenesis`
(`DefaultGenesis`/`ValidateGenesis`/`InitGenesis`/`ExportGenesis`) in their
`module/module.go`. The SDK `ModuleManager.InitGenesis` dispatches this; before
CO-040 these modules implemented only the `appmodule.AppModule` marker, so their
genesis path was silently skipped (the cause behind app.go's CO-005d sentinel
comment). Genesis Go structs are JSON-marshaled (`encoding/json`), not via the codec.

**Genesis seeding path.** `wevibed init` builds genesis.json from
`app.ModuleBasics` (app/encoding.go), which contains ONLY SDK modules — the custom
modules are absent, so `ModuleManager.InitGenesis` would skip any module whose
`app_state` key is nil. Therefore `scripts/init-chain.sh` jq-seeds:
- `app_state.emissions = {}` → emissions `InitGenesis` derives the pool from
  `DefaultParams()` (`DefaultEmissionPool()`), so DefaultParams is the single
  source of truth (DECISIONS D-S32-EMISSION-POOL-GENESIS).
- `app_state.reputation = {"active": true}` → reputation active at genesis
  (DECISIONS D-S32-REPUTATION-DEFAULTGENESIS-ACTIVE, reinforcing D-13.5).

**Epoch-hook chain.** The epochs module fires `AfterEpochEnd` for the
`wevibe_epoch` identifier via MultiEpochHooks: emissions first (mint + payouts),
then memory (`setCurrentEpoch` → `CheckEpochExpiry` → `ApplyEpochDecay` → merkle
roots). Both hooks obey **R-EPOCH-HOOK-RESILIENCE** (DECISIONS
D-S32-EPOCH-HOOK-RESILIENCE): every recoverable failure logs a warning and returns
nil, because the SDK epoch dispatcher discards the entire cached-write batch if any
hook returns a non-nil error.

**cachekv iterator correctness (DECISIONS D-S32-CACHEKV-ITER) — LOAD-BEARING.**
Under the cache-wrapped store used by epoch hooks / BeginBlock,
`cacheMergeIterator.Error()` returns non-nil at normal end-of-iteration. The legacy
`for iter.Valid(){…}; if err := iter.Error(); err != nil { return err }` pattern
(24 sites across emissions/memory/org/reputation keepers) therefore mis-reads
exhaustion as failure on the live chain — `ApplyEpochDecay`, `CheckEpochExpiry`,
`getAllOrgsWithMemories`, and emissions `GetAllOrgs` all error every epoch. This was
the true cause of the Sprint-31 "zero decay" symptom; unit tests missed it because
`rootmulti`/IAVL iterators return nil at end. The fix (remove post-loop
`iter.Error()` checks; collect-then-mutate for iterate-and-modify paths; cachekv-
wrapped regression test) is scoped to CO-041 Task A.


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
├── wevibe-signing.ts   # Request signing utilities
├── mcp-client.ts     # MCP client for wevibe-mcp
├── wallet-connect.ts # Wallet connection (includes getOfflineSigner since CO-214)
├── chain-client.ts   # CosmJS direct-broadcast + WeVibe message builders
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
- `@cosmjs/stargate`, `@cosmjs/proto-signing`, `@cosmjs/amino`, `@cosmjs/crypto`, `@cosmjs/encoding`, `cosmjs-types` (Cosmos chain signing + direct broadcast from dashboard)

---

## WeVibe/wevibe-mcp — TypeScript MCP Client + HTTP API Server

**Language:** TypeScript
**Purpose:** MCP client for AI agents to interact with WeVibe Network. Also serves an HTTP API on `127.0.0.1:4450` for the OpenCode plugin (CO-244).
**Status (built + committed, `6ceac5d..264a29f`):** plugin onboarding + identity-sidecar/pairing/installer path landed (`D-SIDECAR-PLUGIN-OWNS-STATE`, `D-PLUGIN-ONBOARDING-HOOK`).

### `src/` — Main source

```
src/
├── server.ts              # MCP server entry point; LAZY boot in main() (initCrypto + non-prompting existence check; membership sync + PRE-pubkey registration deferred to first use)
├── dashboard-server.ts    # Dashboard integration server
├── config.ts              # Central URLs/ports/models
├── types.ts              # TypeScript types
├── session.ts            # Session management
├── crypto.ts            # Cryptographic utilities
├── pair-crypto.ts        # Pairing v1 crypto (import/export, both directions)
├── auth.ts              # Authentication + PRE identity lifecycle (CO-222)
├── identity-sidecar.ts   # Non-secret `~/.wevibe/identity.json`: public keys + createdAt/adoptedAt/extractedAt/lastPairingId + per-org `orgs` map keyed by chain `org_id`
├── identity-runtime.ts   # Memoized `ensureIdentity()` runtime gate
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
└── admin.ts             # Admin CLI: install-opencode/uninstall-opencode (idempotent cross-OS installer), identity-status, setup-identity --json, export-pairing
```

- **Repo-root (NOT under `src/`):** `opencode-plugin/wevibe.tsx` — OpenCode 1.16 TUI onboarding plugin; canonical source, installed by `wevibe-admin install-opencode` to `~/.config/opencode/tui/wevibe.tsx` and registered in `~/.config/opencode/tui.json`.

- **Built path runtime note:** no biometric prompt at process boot (LAZY boot), and PRE membership sync/registration are first-use deferred (`D-SIDECAR-PLUGIN-OWNS-STATE`, `D-PLUGIN-ONBOARDING-HOOK`).

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

## Canonical 5-Layer Architecture (Chain → RPC → Social Graph → Economy → Attestation)

This section defines the canonical system layering for public profile display and
economics. It complements (rather than replacing) the module-level details in
`## wevibe-chain — Cosmos SDK Appchain` and the runtime auth flow in
`## Consumer Path (post-CO-260)`.

**Canonical flow:** `Layer 1 Chain` → `Layer 2 RPC` → `Layer 3 Social Graph` →
`Layer 4 Economic Layer`, with `Layer 5 Future Pluggable Attestation` as a
post-mainnet extension.

### Layer 1 — Chain (source of truth)

- Immutable state + economics.
- Canonical on-chain state includes:
  - memory provenance + author attribution
  - serve/denial counts as **aggregate counters** for both contributor and org
    (not per-memory cards)
  - approved-memory (contribution) counts
  - org membership + roles
  - **(design-only; not yet built)** org directory fields: `hub_endpoints`
    (ordered list of 1–3 transport URLs, failover) and `hub_serving_address`
    (serve/deny AUTH key) as distinct values, with leader-signed setter path
    (`MsgSetServingInfo` extending `MsgSetServingKey`, or `MsgSetOrgConfig`)
  - per-memory rarity tier (computed once at commit from keyword
    supply/demand, then frozen on-chain)
  - economic state
- **Design-only (not yet built):** `hub_endpoints` proto/state changes use Docker
  `make proto-gen` (no hand-edited `.pb.go`). Decision:
  `D-CHAIN-RESOLVED-HUB-ENDPOINT`.
- Economics consumes **only** contribution counts + the network threshold;
  serve counts are never economic inputs.

### Layer 2 — RPC (the read contract)

- Query interface exposing Layer 1 chain state to any client.
- This is the contract between chain and consumers (notably the social graph).
- RPC exposes raw counts/inputs for rendering, including serve counts, rarity
  tier, contribution counts, and roles.
- **Design-only (not yet built):** org-details query is the org directory:
  consumers/plugins resolve `org_id → hub_endpoints` from chain RPC (no manual
  hub URL configuration). Decision: `D-CHAIN-RESOLVED-HUB-ENDPOINT`.
- **Design-only (not yet built):** hubs sign responses with the serving key;
  clients verify against on-chain `hub_serving_address`; signature contract is
  published in `wevibe-protocol` so self-hostable hubs conform. Decision:
  `D-HUB-RESPONSE-SIGNED`.

### Layer 3 — Social Graph (display client)

- Open-source, forkable, self-hostable public display client that reads Layer 2
  RPC.
- Renders public profiles: serve counts (contributor + org), reputation, and
  badges.
- Badge families:
  - serve-milestone
  - rarity-tier (from on-chain rarity tier)
  - contribution-volume
- Badge scope/behavior:
  - scoped per-org with profile breakdown
  - no cross-org leaderboard
  - serve-milestone + contribution-volume criteria come from a canonical spec
    applied by the reference social graph so tiers remain consistent across
    forks
  - badges are status-only (no reward)

### Layer 4 — Economic Layer

- Contribution-only VIBE payout: per approved memory at the network threshold,
  paid to contributors.
- Emissions payout path: validators/stakers.
- Org-creation burn is a sink.
- Leader revenue path: org demand leg — members pay the org in VIBE for recall
  access (hub-accounted, **model & price set by the leader**, market-driven),
  settling to the org treasury; a **small protocol burn** is taken on
  subscription revenue at settlement (the loop's deflationary sink), remainder
  to the leader (`MsgWithdrawTreasury`). Moderator pay is leader-discretionary
  from treasury. Decided, not yet built. See `DECISIONS.md`
  `D-ECON-CANON`.
- Serve counts are deliberately excluded from economics (anti-game).
- Decision locks: `DECISIONS.md` `D-ECON-CANON` and
  `D-S32-TOKENOMICS-LOCKED`.

### Layer 5 — Future Pluggable Attestation (post-mainnet roadmap)

- Separate components plug into the chain to validate claims cryptographically
  or via API session claims (for example: "user X, model Y, N turns, problem
  Z").
- This is the planned evolution of whitepaper §3.10 Session Attestation +
  §3.11 Difficulty Scoring.
- Enhancement target for the social/economic layers remains TBD.
- Infrastructure is not yet present; this is a major roadmap item.
- Decision reference: `DECISIONS.md` `D-ATTEST-ROADMAP`.

## Consumer Path (post-CO-260)

**Canonical consumer chain (auth layers):**
0. **Design-only (not yet built):** ONCE per session start (biometric-free),
   `plugin` reads cached chain `org_id`s from sidecar, queries chain RPC org
   details for each org's `hub_endpoints` (priority-ordered, 1–3, failover), and
   updates local hub URL + per-org sidecar entry if changed; consumer never
   configures a URL. Endpoint-list change emits a one-time passive toast
   (`D-CHAIN-RESOLVED-HUB-ENDPOINT`, `D-HUB-ENDPOINT-CHANGE-TOAST`).
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
- **Design-only (not yet built):** hub endpoint resolution + response-signature
  verification are chain-resolved (`D-CHAIN-RESOLVED-HUB-ENDPOINT`,
  `D-HUB-RESPONSE-SIGNED`) with one-time passive endpoint-change toast
  (`D-HUB-ENDPOINT-CHANGE-TOAST`).

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
1. On approval, hub writes retrieval metadata into Qdrant payload: `keyword_weights`, `lifecycle_state`, `memory_type`, and embedding metadata.
2. Keyword rename/merge operations update PostgreSQL vocabulary tables and then call `UpdateMemoryKeywords` to keep Qdrant payload keyword weights synchronized.
3. Query path (`QueryPoints`) uses Qdrant payload metadata for lifecycle filtering and ranking:
   - hard exclude `ARCHIVED`
   - exclude `DORMANT` unless `include_dormant=true`
   - apply keyword overlap boost, optimistic pending-denial decay, and D-9.4 new-memory boost
   - assign positions using tempered power-law sampling (strict top-1, sampled positions 2..N)
4. Epoch poller (`SyncEpochData`) runs on a ticker, loads approved orgs from PostgreSQL, compares Qdrant payload vs chain `GetMemoriesBatch`, and updates changed lifecycle state values.
5. `ScrollApprovedMemories` reads `keyword_weights` and `lifecycle_state` directly from Qdrant payload (no PostgreSQL keyword join).

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

**Denial Attestation Flow (CO-225; consumer loop finalized 2026-05-25 per D-2026-05-25-A):**
```
Consumer plugin (wevibe-plugin.ts)
       │
       │ User clicks Deny on a memory in the recall approval UI.
       │ Plugin does TWO actions, both fire-and-forget:
       │   (a) appends memory_hash to ~/.wevibe/blacklist.json (local suppression)
       │   (b) POST /v1/denials to wevibe-mcp with {memory_hash, org_id, reason?}
       ▼
wevibe-mcp HTTP API (loopback 127.0.0.1:4450, Bearer token per D-12.5a)
       │
       │ 1. Append {memory_hash, org_id, reason?, consumer_pubkey, ts}
       │    to ~/.wevibe/pending-denials.json (atomic write).
       │ 2. Return 202 Accepted to the plugin immediately.
       │ 3. Flush queue to hub opportunistically:
       │      — on next /v1/recall call (piggyback),
       │      — on periodic timer,
       │      — survives plugin/wevibe-mcp restarts.
       │ 4. On hub 200 OK, remove the entry from the queue.
       ▼
wevibe-hub POST /v1/orgs/{orgID}/denials (D-2026-05-25-A)
       │
       │ 1. Verify consumer signature + org membership.
       │ 2. Insert into serve_events (event_type='denial', status='pending',
       │    org_id, epoch_id, memory_content_hash, nullifier, reason).
       │ 3. Pending counts are derived from serve_events at query time;
       │    no separate counter table is required.
       │ 4. Return 200 OK.
       │
       ├─► QUERY PATH (load-bearing invariant — D-2026-05-25-A):
       │   Hub query handler ranks results using
       │     optimistic_weight = chain_weight
       │                         − (pending_denial_count × DenialDecayBps/10000)
       │   applied equally to all keywords on the memory (mirrors chain logic).
       │   A denial received at T is reflected in any query at T+ε. No batching,
       │   no caching delay.
       │
       │ Pending denials accumulate at the hub. Leader chooses when to settle.
       ▼
Dashboard denial-batch panel (on /chain-submit)
       │
       │ Shows: "N denials awaiting on-chain submittal."
       │ No per-denial review. No automatic trigger. Leader's choice.
       │ If N > 200: "200 of N are shown; submit additional batches as needed."
       │
       │ Leader clicks Submit:
       │   GET /v1/orgs/{orgID}/denials/pending  (newest-first, max 200)
       │   Dashboard builds MsgSubmitDenialBatch.
       │   Leader signs with WALLET (Category A per D-1.3 — same pattern as
       │     MsgApproveMemory).
       │   Dashboard broadcasts directly to chain RPC (NOT via hub relay).
       ▼
wevibe-chain x/serve MsgSubmitDenialBatch handler
       │
       │ Per accepted denial entry:
       │   StoredDenialAttestation persisted (keyed org_id / epoch_id / memory_hash)
│   Calls memoryKeeper.ApplyEarnedTrustDecay (D-4.2): updates per-keyword
        │     weight using denial_rate-scaled decay; transitions to
        │     MEMORY_STATE_ARCHIVED if all weights ≤ retrievalThreshold (1500 bps).
        │ Chain emits `denial_batch_submitted` event {org_id, submitter, epoch,
        │   accepted_count, rejected_count, block_height} — queryable via CometBFT
        │   `tx_search` as `denial_batch_submitted.org_id='<org_id>'` (CO-016).
       ▼
ChainWatcher (hub) observes the block
       │
       │ processDenialBatchBookkeeping:
       │   - Marks matching serve_events rows status='submitted'
       │   - Pending denial counts then drop automatically from query-time
       │     scoring because only status='pending' rows are counted
       │   - Applies the new chain weight as the baseline for future optimistic
       │     calculations
       │   - Inserts chain_commit_events row (action_type='denial_batch_submitted')
       │
       │ x/emissions ProcessOrgPayouts reads serves and denials at epoch close.
       ▼
payout_per_memory counted (not payout_per_serve)
```

**Chain module changes (CO-225):**
- `x/memory`: Earned Trust decay params (D-4.2: serveD=220, denialD=900, idleD=600, grace=20, trustMinServes=1, trustMaxRate=0.30, idleProtect=0.05, idleUntrusted=1.0, retrievalThreshold=1500); archive when all keyword weights ≤ retrievalThreshold
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
