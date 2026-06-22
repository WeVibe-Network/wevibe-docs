# WeVibe Network — Complete Repository Topology

**Generated:** 2026-06-14 (revised)  
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
| postgres | wevibe-postgres | postgres:5432 | 5433 | Hub operational DB |
| qdrant | wevibe-qdrant | qdrant:6333 | 6333, 6334 | Vector storage |
| wevibe-chain | wevibe-chain | chain:26657 (RPC), 9090 (gRPC), 1317 (REST) | 26656, 26657, 1317 | Cosmos appchain validator |
| hub | wevibe-hub | hub:4440 | 4440 | Go API server |
| dashboard | wevibe-dashboard | dashboard:3000 | 3000 | Next.js UI |
| wevibe-umbral | wevibe-umbral | wevibe-umbral:4460 | — | PRE encryption sidecar (gRPC) |
| wevibe-mcp | wevibe-mcp | wevibe-mcp:4450 | 4450 | MCP + local crypto + HTTP API |
| wevibe-faucet | wevibe-faucet | faucet:4470 | — | Dev/test VIBE faucet (dev-mode only) |
| wevibe-social-graph | wevibe-social-graph | social-graph:4470 | — | Public profile + badge rendering service |

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

**Module:** `github.com/wevibe-network/wevibe-server/wevibe-hub`  
**Go version:** 1.25.9  
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

**Route table:**
```
GET    /health
GET    /v1/members/{pubkey}/orgs
GET    /v1/profile/{wallet}                                      # Public profile

# Identity + pairing blobs (passkey custody)
POST   /v1/identity/blob
GET    /v1/identity/blob/{credentialId}
POST   /v1/pair
GET    /v1/pair/{pairingId}

GET    /v1/hub/serving-address                                    # Serving pubkey for response signing
GET    /v1/extraction-presets                                     # Leader's extraction preset configs
GET    /v1/balance/{address}                                      # On-chain VIBE balance

# Dev-only faucet (WEVIBE_DEV_ENDPOINTS=1)
POST   /v1/faucet/fund                                            # Fund a wallet from dev faucet

GET    /v1/orgs/discover                                          # List/search public orgs
POST   /v1/orgs                                                   # Create org

# Org-scoped (membership optional for some routes)
GET    /v1/orgs/{orgID}
PUT    /v1/orgs/{orgID}/extraction-profile                       # Leader sets extraction profile
GET    /v1/orgs/{orgID}/extraction-profile
POST   /v1/orgs/{orgID}/join                                      # Submit join request (D-12.8)
GET    /v1/orgs/{orgID}/epoch/{epochID}/manifest

# Membership-gated org routes
POST   /v1/orgs/{orgID}/epoch/rotate
POST   /v1/orgs/{orgID}/members                                  # Invite member
GET    /v1/orgs/{orgID}/members
GET    /v1/orgs/{orgID}/members/{pubkey}
POST   /v1/orgs/{orgID}/members/{pubkey}/enable-recall           # Enable recall for member
POST   /v1/orgs/{orgID}/members/{pubkey}/disable-recall          # Disable recall for member
POST   /v1/orgs/{orgID}/members/{pubkey}/pre-key                 # Register PRE pubkey
GET    /v1/orgs/{orgID}/members/{pubkey}/pre-key
POST   /v1/orgs/{orgID}/members/wallet                           # Link Cosmos wallet to passkey identity
DELETE /v1/orgs/{orgID}/members/{pubkey}                         # Remove member (leader-only)
GET    /v1/orgs/{orgID}/keys/envelope
POST   /v1/orgs/{orgID}/dashboard/keys
DELETE /v1/orgs/{orgID}/dashboard/keys/{pubkey}
POST   /v1/orgs/{orgID}/recovery/shares
GET    /v1/orgs/{orgID}/recovery/shares

POST   /v1/orgs/{orgID}/submit                                    # Contributor submits memory ciphertext
POST   /v1/orgs/{orgID}/submit/batch                              # Batch submit multiple memories

GET    /v1/orgs/{orgID}/reports
POST   /v1/orgs/{orgID}/reports
GET    /v1/orgs/{orgID}/reports/{reportID}
PATCH  /v1/orgs/{orgID}/reports/{reportID}
POST   /v1/orgs/{orgID}/reports/{reportID}/vote
POST   /v1/orgs/{orgID}/reports/{reportID}/commit                 # Leader wallet-commits report to chain

GET    /v1/orgs/{orgID}/moderation/queue
GET    /v1/orgs/{orgID}/moderation/history
POST   /v1/orgs/{orgID}/moderation/{submissionHash}/vote          # Advisory vote (approve/flag)
POST   /v1/orgs/{orgID}/moderation/{submissionHash}/approve       # Advisory approve
POST   /v1/orgs/{orgID}/moderation/{submissionHash}/deny          # Terminal deny (leader-only)
POST   /v1/orgs/{orgID}/submissions/{hash}/keyword-vote           # Per-keyword include/exclude vote
POST   /v1/orgs/{orgID}/moderation/batch-submit                   # Leader stages batch for chain

POST   /v1/orgs/{orgID}/serves                                    # Record serve event (CO-033a)
POST   /v1/orgs/{orgID}/denials                                   # Consumer denial attestation
GET    /v1/orgs/{orgID}/denials/pending-count                     # Leader panel count
GET    /v1/orgs/{orgID}/denials/pending                           # Leader pending list (newest-first, cap 200)

POST   /v1/orgs/{orgID}/query                                     # PRE retrieval query
GET    /v1/orgs/{orgID}/memories                                  # Scroll approved memories
GET    /v1/orgs/{orgID}/memories/{cid}

GET    /v1/orgs/{orgID}/keywords
GET    /v1/orgs/{orgID}/keywords/candidates                       # Pending keyword candidates for leader review
POST   /v1/orgs/{orgID}/keywords                                  # Leader adds manual keyword
PUT    /v1/orgs/{orgID}/keywords/merge                            # Leader merges two keywords
PUT    /v1/orgs/{orgID}/keywords/{keyword}/rename                 # Leader renames keyword
DELETE /v1/orgs/{orgID}/keywords/{keyword}                        # Leader deprecates keyword

POST   /v1/orgs/{orgID}/submit-keyword-results                    # Contributor submits verified keywords (CO-238)
POST   /v1/orgs/{orgID}/verify-keywords                           # Hub verifies and transitions pending_keyword→pending_chain
POST   /v1/orgs/{orgID}/submissions/{hash}/rerun-keywords         # Rerun keyword extraction via Ollama (CO-238)
PUT    /v1/orgs/{orgID}/submissions/{hash}/update-keywords        # Leader updates keyword set on submission
DELETE /v1/orgs/{orgID}/submissions/{hash}                        # Remove submission from pipeline (CO-238)
GET    /v1/orgs/{orgID}/submissions                               # List all submissions (leader/contributor view)
GET    /v1/orgs/{orgID}/my-submissions                            # Contributor-only own submission status

GET    /v1/orgs/{orgID}/health
GET    /v1/orgs/{orgID}/credits
GET    /v1/orgs/{orgID}/finances                                  # Credits + chain financial data (CO-266, GAP-O6)
GET    /v1/orgs/{orgID}/chain-config                              # Chain config read (leader-only)

GET    /v1/orgs/{orgID}/join-requests                             # List join requests
POST   /v1/orgs/{orgID}/join-requests/{requestID}/approve         # Approve join request
POST   /v1/orgs/{orgID}/join-requests/{requestID}/cancel-approval # Cancel pending approval
POST   /v1/orgs/{orgID}/join-requests/{requestID}/deny            # Deny join request

GET    /v1/profile/notifications                                  # Get notification preferences
PATCH  /v1/profile/notifications                                  # Update notification preferences
GET    /v1/notifications                                          # List notifications
GET    /v1/notifications/unread-count
POST   /v1/notifications/mark-read                                # Mark notifications as read
GET    /v1/notifications/ws                                       # WebSocket for real-time notifications

# PRE sidecar / leader-side kfrag delivery
POST   /v1/orgs/{orgID}/members/{pubkey}/kfrag                    # Leader-signed StoreKFrag (finished kfrag minted leader-side)
# (Internal reencrypt happens inside QueryMemories; the hub never mints epoch keys or kfrags —
#  GenerateEpochKeyPair/RegisterMember + /v1/internal/epoch-keypair + /v1/internal/orgs/{id}/kfrags were RIPPED, D-LEADER-SIDE-UMBRAL-MINT)

# Dev-only billing topup (WEVIBE_DEV_ENDPOINTS=1)
POST   /v1/billing/topup

# Test-mode routes (WEVIBE_TEST_MODE=true)
GET    /v1/test/health
POST   /v1/test/embed                                             # Test embedding with hub Ollama
GET    /v1/test/orgs/{orgID}/queue                                # Dump pending_submissions for org
GET    /v1/test/orgs/{orgID}/serve-queue                          # Dump serve_events queue depth
```

---

### Config

#### `internal/config/config.go`
**Exports:** `Config` struct, `Load() Config`
**Fields:**
```go
Port                       int       // env: WEVIBE_HUB_PORT, default: 4440
DatabaseURL                string    // env: DATABASE_URL
QdrantAddr                 string    // env: QDRANT_ADDR, default: "localhost:6333"
QdrantAPIKey              string    // env: QDRANT_API_KEY (required; panics if < 32 chars)
OllamaURL                  string    // env: OLLAMA_URL, default: "http://localhost:11434"
StripeSecretKey            string    // env: STRIPE_SECRET_KEY
S3Bucket                   string    // env: WEVIBE_S3_BUCKET, default: "wevibe-memories"
NodePrivkey                string    // env: HUB_NODE_PRIVKEY (Ed25519, hex)
ChainGRPCURL               string    // env: WEVIBE_CHAIN_GRPC_URL
FaucetURL                  string    // env: FAUCET_URL, default: "http://wevibe-faucet:4470"
UmbralSidecarAddr          string    // env: WEVIBE_UMBRAL_SIDECAR_ADDR, default: "127.0.0.1:4460"
SocialGraphURL             string    // env: WEVIBE_SOCIAL_GRAPH_URL, default: "http://wevibe-social-graph:4470"
// Optional in Phase 1; required for Sprint 23 WebSocket subscription.
ChainRPCURL                string    // env: WEVIBE_CHAIN_RPC_URL
ChainID                    string    // env: WEVIBE_CHAIN_ID
ChainSubmitterMnemonic     string    // env: WEVIBE_CHAIN_SUBMITTER_MNEMONIC
ChainEnabled               bool      // env: WEVIBE_CHAIN_ENABLED
RetrievalTemperature       float64   // env: RETRIEVAL_TEMPERATURE, default: 0.7
RetrievalNewMemBoostMult   float64   // env: RETRIEVAL_NEW_MEM_BOOST_MULT, default: 0.5
RetrievalNewMemBoostWindow uint64    // env: RETRIEVAL_NEW_MEM_BOOST_WINDOW, default: 30
RetrievalVectorNoiseSigma  float64   // env: RETRIEVAL_VECTOR_NOISE_SIGMA, default: 0.0 (D-9.5)
RetrievalRecallDepth       uint64    // env: RETRIEVAL_RECALL_DEPTH, default: 5000
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
var umbralService *umbral.Service        // CO-218 — PRE sidecar service (gRPC)
func SetPool(p *pgxpool.Pool)
func SetQdrantClient(c *retrieval.QdrantClient)
func SetNodePrivkey(key string)
func SetChainClient(c *chain.GrpcClient)
func SetUmbralService(s *umbral.Service) // CO-218
```

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
func GetPendingQueue(w, r)     // GET — caller pubkey from auth header (leader or can_moderate)
func VoteOnSubmission(w, r)    // POST — advisory vote (approve/flag); advisory only, no quorum
func ApproveSubmission(w, r)   // POST — advisory approve (leader or can_moderate)
func DenySubmission(w, r)      // POST — calls moderation.DenySubmission() with reason
func VoteOnKeyword(w, r)       // POST — per-keyword include/exclude vote on a submission
func GetModerationHistory(w,r) // GET — last 24h of moderation decisions (leader or can_moderate)
```
**Known issues:** None

#### `internal/api/handlers/billing.go`
**Exports:**
```go
func TopUpCredits(w, r)   // POST /v1/billing/topup — reads {org_id, amount, signed_by} (dev-only)
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
func GetMemory(w, r)          // GET — retrieves a single memory by CID
```
**Known issues:** None

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
func ListKeywords(w, r)         // GET — any active member can read org keyword vocabulary
func AddKeyword(w, r)          // POST — leader only
func MergeKeywords(w, r)       // PUT — leader only, merges source into target
func RenameKeyword(w, r)        // PUT — leader only, renames keyword
func DeprecateKeyword(w, r)    // DELETE — leader only
```
**Known issues:** None

#### `internal/api/handlers/submissions.go` (CO-238)
**Exports:**
```go
func SubmitKeywordResults(w, r)  // POST /submit-keyword-results — contributor submits verified keywords
func VerifyKeywords(w, r)        // POST /verify-keywords — hub validates format/weights (no vocab gate) and transitions pending_keyword→pending_chain
func UpdateKeywords(w, r)        // PUT /submissions/{hash}/update-keywords — leader persists curated keyword selection
func RemoveSubmission(w, r)      // DELETE /submissions/{hash} — remove submission from pipeline
func ListSubmissions(w, r)       // GET /submissions — all submissions (leader sees queue; contributor sees own)
func ListMySubmissions(w, r)     // GET /my-submissions — contributor-only own submission status view
```
**Known issues:** None

#### `internal/api/handlers/notifications.go`
**Exports:**
```go
func GetNotificationPreferences(w, r)   // GET/PATCH /profile/notifications
func UpdateNotificationPreferences(w, r)
func ListNotifications(w, r)            // GET /v1/notifications (paginated)
func GetUnreadCount(w, r)              // GET /v1/notifications/unread-count
func MarkRead(w, r)                    // POST /v1/notifications/mark-read
func NotificationWebSocket(w, r)       // GET /v1/notifications/ws — real-time WebSocket push
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
```
**Known issues:** None

#### `internal/moderation/moderation.go`
**Exports:**
```go
func SubmitToQueue(ctx, pool, req) error
  // verifies: Ed25519 sig over submission_hash bytes, SHA256(ciphertext||wrapped_dek) == submission_hash
  // stores ciphertext in ciphertext_hex column (encrypted, opaque)

func GetPendingQueue(ctx, pool, orgID, moderatorPubkey) ([]PendingQueueItem, error)
  // checks: must be leader or have can_moderate capability

func ApproveSubmission(ctx, pool, qdrantClient, orgID, submissionHash, moderatorPubkey, req) error
  // advisory approve (leader or can_moderate); updates status, writes audit log, indexes in Qdrant
  // requires vector (768 dim), indexes keywords in org_keywords + memory_keywords

func DenySubmission(ctx, pool, orgID, submissionHash, moderatorPubkey, reason) error
  // updates status + denial_reason; leader-only (deny is terminal)
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
**Known issues:** `GetContributorStats` currently PREFERS a member's linked wallet address over their Ed25519 pubkey when querying chain reputation — this strands pubkey-earned reputation the instant a wallet is linked. Locked fix (DESIGN-LOCKED, `D-REPUTATION-KEYED-BY-PUBKEY`): query by passkey pubkey and become alias-aware (resolve the pubkey→wallet alias post-migration) rather than wallet-preferring.

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
> **FORWARD NOTE (Sprint 32, identity overhaul — DECISIONS.md `D-IDENTITY-PROGRESSIVE-CUSTODY`, brief `wevibe-meta/workspace/reports/design-identity-onboarding-migration.md`):** the Ed25519 `WeVibe-Signed` verification below stays, but the identity it authenticates is being reworked. Incoming: identity is a **client-held key created at first run and protected by a passkey (WebAuthn)** — no wallet required to participate; the dashboard moves from its current Keplr-signature-derived Ed25519 identity (`lib/wevibe-auth.ts`) and the plugin/MCP from its random on-device keypair onto a **single shared passkey-wrapped client-key scheme** (the two must mint the same identity). The key's ciphertext may be backed up to the hub but the **hub never holds a usable signing key** (non-custodial). A Cosmos wallet becomes an **optional linked authority** (staged handover, not migration) needed only to claim rewards / pay mainnet fees. Members are keyed by pubkey (`wallet_address` nullable), so wallet-free contribution already works at the hub layer. This section updates as it lands. Reputation/XP is keyed by the **passkey pubkey**, not the wallet: memory-contribution XP already keys by the contributor pubkey on-chain, and serve XP must be fixed to do the same (`x/serve` currently keys serve XP by the wallet address and skips it when none is linked — a bug against wallet-free participation). Carrying reputation onto a wallet is a deliberate **on-chain, dual-signed alias** (passkey pubkey → wallet address) with an `is_migrated` flag, gated by the contributor's memory-contribution trail — NOT a hub-DB record (DECISIONS.md `D-REPUTATION-KEYED-BY-PUBKEY`, `D-MIGRATION-ONCHAIN-ALIAS`). Privileged roles always have a wallet (`D-LEADER-REQUIRES-WALLET`). Earnings follow reputation: pay-per-approved-memory emissions must credit the passkey `contributor_id` recorded on the memory (today `x/emissions` credits the wallet `contributor_address`), accruing a claim-later balance keyed by the passkey pubkey that becomes withdrawable to the wallet only after the dual-signed `is_migrated` migration (`D-MIGRATION-ONCHAIN-ALIAS`; folds the SEC-FLAG-4 / GAP-ECON-BUILD withdrawal-claim path).

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
**Key detail:** Verifies signatures produced by Cosmos wallet `signArbitrary` (EIP-712 structured data or plain bytes). Used for: report chain commitment (`POST /v1/orgs/{orgID}/reports/{reportID}/commit`). (The former `PATCH /v1/orgs/{orgID}/config` for `required_approvals`/`report_vote_threshold` was removed with the moderation-quorum model — D-MODERATION-ADVISORY.) Canonical message format: `{action}|{orgID}|{field1}|{field2}|...`
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
InviteMemberRequest     — pubkey, x25519_pubkey, pre_pubkey, role, signed_by, signature, enc_envelope, search_envelope, mod_envelope  (NO epoch_sk — kfrags are minted leader-side and delivered via StoreKFrag)
StoreMemberKFragRequest  — epoch_id, pre_pubkey, kfrag  (leader-signed; POST /v1/orgs/{orgID}/members/{pubkey}/kfrag)
CreateOrgRequest         — …, umbral_pk (hex, leader-derived epoch public key), org_id, tx_hash  (NO epoch_sk_envelope)
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
```

---

### Database Schema

#### `db/schema.sql`
**Tables:**
```
orgs                  — PK: org_id. Includes stripe_customer_id, stripe_subscription_id, egress_mode CHECK, status CHECK, rotation_status CHECK, rotation_pending_since, chain_registered, trial_days (CO-266)
members               — PK: (org_id, pubkey). FK: orgs. Role CHECK (leader|member). Includes `can_contribute BOOLEAN`, `can_moderate BOOLEAN` (per-member capabilities, chain-mirrored), `pre_pubkey BYTEA`, `is_trial BOOLEAN`, `trial_expires_at TIMESTAMPTZ` (CO-266). ON DELETE CASCADE.
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

> **FORWARD NOTE (Sprint 32, storage-market overhaul — DECISIONS.md `D-ECON-STORAGE-MARKET`, brief `wevibe-meta/workspace/reports/design-storage-market-economy.md`):** landed: `x/org` slot registry + ascending-price acquisition auction; `x/emissions` org-treasury payout subsystem removed (Treasury/`MsgWithdrawTreasury` removed); `x/attestation` neutered (disabled-but-wired); `x/bandwidth` memory-cap wired as the testnet DDoS guard. Remaining unbuilt: on-chain demand-leg router; self-assessed-value (Harberger) rent + forced-sale-in-window; Dutch resale of freed slots; full per-memory storage-deposit activation (parameterized ~0 on testnet).

| Module | Keeper Path | Proto Path | Tests | Purpose |
|--------|------------|-----------|-------|---------|
| x/attestation | x/attestation/keeper/ | proto/wevibe/attestation/v1/ | keeper + integration | Session-attestation storage (NOT merkle — merkle roots live in x/memory). Being neutered: disabled/no-op until verification infra (D-ATTEST-ROADMAP). Optional TEE model-provenance (D-ATTEST-TEE-TIER) is verified OFF-CHAIN first (Phase 0, no chain change); on-chain anchoring here is Phase 1. |
| x/bandwidth | x/bandwidth/keeper/ | proto/wevibe/bandwidth/v1/ | keeper + integration | Bandwidth throttling |
| x/emissions | x/emissions/keeper/ | proto/wevibe/emissions/v1/ | keeper | Emission pool, epoch emission, work scores (32-yr schedule scheduled CO-041) |
| x/identity | x/identity/keeper/ | proto/wevibe/identity/v1/ | keeper + integration | Passkey identity management; wallet linking aliasing |
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
├── layout.tsx                    # Root layout + providers
├── page.tsx                      # Landing/redirect → /dashboard or /discover
├── globals.css
├── api/                          # Next.js API routes (server-side)
│   └── [...route]/               # Catch-all for hub proxy if needed
├── (auth)/
│   └── login/page.tsx           # Login page
├── (dashboard)/
│   ├── layout.tsx                # Dashboard layout with sidebar/topbar + OrgProvider
│   ├── activity/page.tsx         # Activity feed / notifications log
│   ├── billing/page.tsx          # Billing + credits management
│   ├── buy-org/page.tsx          # Purchase / create new org
│   ├── chain-submit/page.tsx     # Leader batch commit to chain (denial batches, memory commits)
│   ├── create-org/page.tsx       # Org creation flow
│   ├── discover/page.tsx         # Public org discovery / browse
│   ├── epoch/page.tsx            # Epoch rotation management
│   ├── faucet/page.tsx           # Dev faucet access page
│   ├── health/page.tsx           # Hub/chain health diagnostics
│   ├── join-requests/page.tsx    # Pending join request queue (leader)
│   ├── keywords/page.tsx         # Vocabulary + keyword management
│   ├── members/page.tsx          # Member list with capability pills
│   ├── memories/page.tsx         # Committed memory browser
│   ├── moderation/page.tsx       # Moderation queue with expandable per-memory dropdown
│   ├── my-org/page.tsx           # Current org overview
│   ├── my-submissions/page.tsx   # Contributor's own submission status view
│   ├── notifications/page.tsx    # Notification center
│   ├── profile/page.tsx          # Own profile settings
│   ├── recovery/page.tsx         # Shamir recovery share management
│   ├── reports/page.tsx          # Memory report queue + voting
│   ├── sessions/page.tsx         # Session list with per-memory org destination (CO-247)
│   └── settings/page.tsx         # Extraction model, extraction profile config
└── u/
    └── [wallet]/
        └── page.tsx              # Public profile — standalone, no sidebar (served by hub /v1/profile/{wallet})
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
├── dashboard-server.ts    # Dashboard integration server — exposes tools for wevibe-dashboard Next.js
├── config.ts              # Central URLs/ports/models configuration
├── types.ts               # TypeScript shared type definitions
├── session.ts             # Session management (contribution sessions + extraction pass)
├── crypto.ts              # Core cryptographic utilities (encrypt/decrypt, key derivation)
├── crypto-utils.ts        # Additional crypto helpers (hashing, encoding)
├── pair-crypto.ts         # Pairing v1 crypto (import/export, both directions)
├── auth.ts                # Authentication + PRE identity lifecycle (CO-222)
├── biometric.ts           # Biometric prompt integration (passkey/WebAuthn)
├── identity-sidecar.ts    # Non-secret `~/.wevibe/identity.json`: public keys + timestamps + per-org map
├── identity-runtime.ts    # Memoized `ensureIdentity()` runtime gate; passkey enrollment
├── contribution.ts        # Memory contribution flow (contributor-side extraction + submit)
├── extraction.ts          # Content extraction (Pass 1: memory extraction, Pass 2: vocab-constrained keyword classification — CO-236); routeKeywordCandidates + normalizeOrRankDecay for fallback weights
├── extraction-presets.ts  # Named extraction preset profiles (leader-configurable model selection per use-case)
├── guard.ts               # YARA-X prompt injection detection via wevibe-guard subprocess
├── blacklist.ts           # Consumer denial blacklist management (~/.wevibe/blacklist.json)
├── llm.ts               # LLM provider interface + getLlmProvider/setLlmProvider singleton
├── llm-ollama.ts        # Ollama REST API chat/embeddings provider
├── llm-openai-compat.ts  # OpenAI-compatible API provider (OpenRouter, etc.)
├── llm-sampling.ts      # MCP sampling-based LLM provider (for OpenCode integration)
├── embedding.ts         # nomic-embed-text embeddings via Ollama /api/embeddings; EMBEDDING_MODEL pinned in config
├── embed-card.ts        # buildRetrievalCard + buildAnticipatedNeed + computeLocalEmbedding; C1/C2 at Verify
├── retrieval-card.ts    # Q-B retrieval card construction (serializeMemoryText, parseMemoryText, buildRetrievalCard, buildAnticipatedNeed); NeedHarvest for query need-card
├── ocr-sanitize.ts      # OCR artifact sanitization for embedding input
├── moderation.ts        # Moderation handling — vote, approve, deny; embed-card call with {strictAnticipated:false}
├── sidecar.ts           # Umbral PRE sidecar wrapper — CLI: encrypt/decrypt-reencrypted/derive-epoch-keypair/generate-kfrags (leader-side mint); gRPC client to hub sidecar for StoreKFrag/ReEncrypt/DeleteKFrags only
├── vault.ts             # Persistent encrypted credential vault (~/.wevibe/vault)
├── pending-vault.ts     # In-flight submission vault with encryption + retry
├── key-store.ts         # Disk-backed keystore for PRE identity, epoch secrets, Shamir shares
├── keychain.ts          # OS keychain integration (macOS Keychain / Linux secret-service) for biometric-gated keys
├── org-client.ts        # Hub API client — submit, query, moderation, keyword ops + getOrgKeywords() vocab lookup; PRE registration flow
├── hub-fetch.ts         # Typed HTTP client helpers for wevibe-hub REST API
├── hub-resolver.ts      # Chain-RPC → hub endpoint resolution (D-CHAIN-RESOLVED-HUB-ENDPOINT design-only)
├── artifact-policy.ts   # Artifact policy enforcement — which memories can be served to consumer
├── artifact-extract.ts  # Artifact extraction from session context + LLM enrichment
├── artifact-transform.ts # Artifact transformation for different output formats
├── deserialize.ts       # Deserialization of PRE retrieval payloads (cfrag, capsule, umbral_ciphertext)
├── recovery.ts          # Shamir recovery share operations; epoch rotation recovery flow
├── canonical.ts         # Canonical message generation for WeVibe-Signed auth headers
├── buffer.ts            # Submission buffering with rotation-pending staging
├── denial-queue.ts      # Denial queue management — ~/.wevibe/pending-denials.json flush to hub
├── trust-panel.ts       # Trust panel formatting for contributor stats (serve count, XP, badges)
├── retrieve-cli.ts      # Importable retrieval module: query → decrypt → sanitize → artifact policy → trust panel; exports retrieve()
├── http-server.ts       # Standalone HTTP API server on 127.0.0.1:4450 (CO-244); Bearer token auth; serves /v1/recall, /v1/serves, /v1/reports
├── session-token.ts     # Ephemeral MCP session token management (~/.wevibe/mcp-session-token)
├── risk-appetite.ts     # Consumer recall risk appetite filter ("lowest" vs "neutral")
├── serve-signing.ts     # Serve event signing utilities for consumer-side attestation
└── admin.ts             # Admin CLI: install-opencode/uninstall-opencode (idempotent cross-OS), identity-status, setup-identity --json, export-pairing
```

- **Repo-root (NOT under `src/`):** `opencode-plugin/wevibe.tsx` — OpenCode 1.16 TUI onboarding plugin; canonical source, installed by `wevibe-admin install-opencode` to `~/.config/opencode/tui/wevibe.tsx` and registered in `~/.config/opencode/tui.json`.

- **Built path runtime note:** no biometric prompt at process boot (LAZY boot), and PRE membership sync/registration are first-use deferred (`D-SIDECAR-PLUGIN-OWNS-STATE`, `D-PLUGIN-ONBOARDING-HOOK`).

### `wevibe_mcp/` — Python MCP (legacy path, not actively used)

```
wevibe_mcp/
├── __init__.py
├── server_legacy.py     # Legacy Python MCP wrapper
├── contribution.py      # Contribution helpers
└── ...
```

### `tests/`

```
tests/
├── integration/
│   ├── e2e-flow.test.ts       # Full contribution → moderation → approval → query pipeline
│   └── capstone.test.ts       # End-to-end milestone test
├── production/
│   ├── hub-resilience.test.ts  # Hub downtime graceful degradation
│   ├── sampling-provider.test.ts
│   └── embedding-quality.test.ts
├── wasm-crypto.test.ts         # WASM crypto bindings (Rust SDK ↔ TS)
├── contribution.test.ts        # Extraction pass, keyword candidates, submit flow
├── guard.test.ts               # YARA-X prompt injection detection
├── llm.test.ts                 # LLM provider interface / Ollama fallback behavior
├── moderation.test.ts          # Submit → queue → vote → approve/deny flow
├── moderation-approval.test.ts # Full approval with embed-card + PRE re-encryption
├── retrieval-card.test.ts      # buildRetrievalCard, buildAnticipatedNeed, parseMemoryText
├── retrieve-cli-harvest.test.ts # NeedHarvest building from session context
├── risk-appetite.test.ts       # Risk appetite filtering (lowest vs neutral)
├── serve-signing-parity.test.ts # Serve event signing parity with hub canonical.go
├── session-token.test.ts       # Ephemeral token rotation
├── session.test.ts             # Contribution session lifecycle
├── vault.test.ts               # Encrypted credential vault CRUD
├── pending-vault.test.ts       # In-flight submission buffer
├── key-store.test.ts           # Disk keystore account management
├── pair-crypto.test.ts         # Pairing v1 import/export round-trips
├── pair-crypto-reverse.test.ts # Reverse-direction pairing crypto
├── rotation.test.ts            # Epoch rotation + recovery share lifecycle
├── threshold-recovery.test.ts  # Shamir 2-of-3 threshold reconstruction
├── recovery-status.test.ts     # Recovery status / share verification
├── blacklist.test.ts           # Denial queue → hub flush cycle
├── canonical.test.ts           # WeVibe-Signed header canonical message generation
├── deserialize.test.ts         # PRE payload deserialization (cfrag, capsule fields)
├── egress-policy.test.ts       # Artifact policy + provider leakage enforcement
├── extraction.test.ts          # Full Pass 1 memory extraction via Ollama
├── extract-defaults.test.ts    # Extraction weight defaults and normalizeOrRankDecay fallback
├── extraction-presets.test.ts  # Named preset CRUD
├── artifact-extract.test.ts    # Artifact extraction from session context
├── artifact-policy.test.ts     # Memory serve eligibility policy
├── artifact-transform.test.ts  # Output format transformation
├── hub-fetch.test.ts           # Hub API client request/response handling
├── hub-resolver.test.ts        # Chain-RPC hub endpoint resolution (design-only)
├── install-opencode.test.ts    # OpenCode plugin installer idempotency
├── manifest.test.ts            # Org manifest loading / caching
├── ocr-sanitize.test.ts        # OCR artifact sanitization for embedding input
├── org-client.test.ts          # Org API client — submit, query, moderation calls
└── steg-scan.test.ts           # Steganography detection in session artifacts
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

## wevibe-faucet — Dev/Test VIBE Faucet

**Language:** Go  
**Purpose:** Development-mode token faucet for local/testnet VIBE distribution  
**Default port:** 4470 (container), not exposed to host by default (dev-only)

### Entry Point

#### `main.go`
**Role:** Minimal HTTP faucet server; funds a given wallet address from a pre-seeded mnemonic up to a configured maximum per-request and per-day cap.

```
GET  /health              — returns {"status":"ok","balance":...}
POST /fund?address=<addr> — sends VIBE tokens, returns tx hash or error
```

**Auth:** Dev-only endpoint (the hub gates access via `WEVIBE_DEV_ENDPOINTS=1` before forwarding); faucet itself has no auth in dev mode.
**Known issues:** Not a production service; never ships to mainnet.

---

## Personal Memory Layer — Bounded, Pull-Mode Local Memory (Stage 1 LIVE via CodeGraph; Stage 2 design-locked)

**Implementation:** Stage 1 = **CodeGraph** (external `codegraph` CLI + MCP server, MIT, on PATH; index at workspace-root `.codegraph/`). Stage 2 (planned) = a `PersonalMemoryProvider` interface in `wevibe-opencode-plugin`.  
**Purpose:** Deterministic, on-demand (pull-mode) recall of KNOWN facts about the user's own repo/work — repo/code map, durable project facts/decisions, session-handoff state. Distinct from shared org memory and explicitly NOT a predictive context-injector (that lane is the org recall pipeline, D-RECALL-ALIGNMENT).  
**Status:** Stage 1 LIVE (CodeGraph wired as a separate opencode MCP, 2026-06-19, telemetry off); Stage 2 DESIGN-LOCKED (D-PERSONAL-MEMORY, GAP-PERSONAL-MEMORY) — not started.

### Design
- **Partition (load-bearing):** runs OUTSIDE the org-memory security path (PRE decrypt, guard, gate); no third-party engine is fused into the security-critical plugin/hub.
- **Boundary:** pull/deterministic only; never speculative session-start injection (the over-injection failure mode the boundary exists to prevent).
- **Stage 1 (LIVE):** **CodeGraph** is the deterministic layer — workspace-root `.codegraph/` graph (768 files / 19.5k nodes / 53.7k edges, all repos, auto-syncs); module `TOPOLOGY.md` files remain the curated narrative layer on top; usage steered in AGENTS.md "CodeGraph — code navigation". Manager + delegates inherit the `codegraph_codegraph_*` MCP tools (verified on `gather`).
- **Stage 2 (planned):** a stable `PersonalMemoryProvider` interface (store/retrieve/forget/health); predictive-personal recall served through WeVibe's own aligned pipeline as a private corpus (NOT a bolt-on). BYO opt-in note: agentmemory/mem0/supermemory are *predictive* engines (excluded push lane) — a discouraged, out-of-support hatch, NOT the deterministic default (CodeGraph fills that).
- **Invariant:** local/personal scratch, NOT hub-durable, NOT chain-anchored; excluded from the rebuildable contract (CANONICALUX §15 disposability drill unaffected); exit = re-index from local source.

---

## wevibe-opencode-plugin — OpenCode TUI Onboarding Plugin

**Language:** TypeScript/React  
**Purpose:** WeVibe-branded onboarding TUI shown inside OpenCode when the plugin is installed. Drives first-run identity creation, org pairing, and extraction model selection.

### Structure

```
plugins/
└── wevibe.tsx            # React component — TUI card rendered inside OpenCode prompt shell
tui/
├── tui.json              # OpenCode TUI registration: "wevibe" → ~/.config/opencode/tui/wevibe.tsx
└── wevibe.tsx            # Installed canonical source (installed by `wevibe-admin install-opencode`)
```

**Installation path:** `npm run build` in wevibe-mcp root copies the compiled plugin to `~/.config/opencode/tui/` and registers it in `tui.json`.
**Runtime note:** Plugin reads from `~/.wevibe/identity.json` (identity-sidecar) and `~/.wevibe/config` for org pairing state. No long-lived secrets stored by the plugin itself.

**Recall engine (`plugins/wevibe-plugin.ts`) — recall-mode + session-tie (D-RECALL-MODE-FLAG, 2026-06-21):** the recall/inject engine reads **`WEVIBE_RECALL_MODE`** from `process.env` (`prod` default | `test`); the same flag is read independently by the MCP (`http-server.ts`/`retrieve-cli.ts`) and the hub (`config.go` → `handlers.SetRecallMode` → `recallModeIsTest()` in `retrieval.go`, wired into `docker-compose.yml` as `WEVIBE_RECALL_MODE: ${WEVIBE_RECALL_MODE:-prod}`). Mode selects the governor base — `prod` floor 0.55 / budget 3 / limit 3 / hub throttles on; `test` floor 0 / budget 1000 / limit 1000 / hub trial-daily + rate-limit bypassed (trial-EXPIRY still enforced). The plugin no longer mints a process-global random-hex session id (REVERSES D-SESSION-SERVE-DEDUP): it captures OpenCode's real `sessionID` from the `chat.message` and `experimental.chat.system.transform` hook inputs, threads it to `/v1/recall` + `/v1/serves`, and gates injection through a per-session `sessionInjectedCids` set so each memory is injected **once per session** (not every turn). Every injection is logged (`[inject] <ISO> sid=… injected N: …`). In `test` only, persisted Earned-Trust auto-accept is disabled so every recalled candidate re-enters the review gauntlet and is re-counted.

---

## wevibe-social-graph — Public Profile + Badge Service

**Language:** Go  
**Purpose:** Serves public contributor/org profiles at `/:wallet` and renders badges (serve-milestone, rarity-tier, contribution-volume). Reads exclusively from chain RPC; no hub DB dependency.

### Structure

```
cmd/server/
└── main.go               # HTTP server entry point; serves profile pages + badge SVGs
internal/
├── chain/                # Chain RPC query client for on-chain state (contribution_count, serve_count, reputation tier)
└── store/                # Badge criteria computation from chain data; in-memory cache with TTL
```

**Default port:** 4470 inside Docker (`wevibe-social-graph:4470`); hub config `SocialGraphURL` points here.
**CORS:** public read endpoint — no auth required for profile/badges.
**Badge families (from canonical spec):**
- serve-milestone: derived from chain `serve_count`
- rarity-tier: derived from on-chain rarity tier (per-memory keyword supply/demand at commit time)
- contribution-volume: derived from chain `contribution_count`

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

## wevibe-sim/recall-sim/ — Recall-Alignment Simulation Suite

**Language:** Node.js (ES modules)  
**Purpose:** Offline, decision-grade validation of recall/extraction quality (DECISIONS.md `D-RECALL-ALIGNMENT`). Lives under `wevibe-sim/` (NOT a git repo). Mirrors the REAL extract→keyword→embed→rank pipeline and reads the REAL shipping extraction prompts from `wevibe-mcp/prompts/`. The ranker (`pipeline/rank.mjs`) is an exact mirror of the hub `retrieval.go` scoring (cosine + γ keyword boost + keyword gate + power-law sampling + new-memory boost + denial decay); `pipeline/retrieve-c3.mjs` replicates the C3 (full-proposal) cell. Embeddings use local `nomic-embed-text` (768-d), identical to production.

### Structure

```
recall-sim/
├── config.mjs            # single source of truth: models, scale, watchdog caps, ablation CELLS
├── lib/                  # prng, parallel worker-pool, isolated-opencode LLM client, watchdog
├── pipeline/             # prompts, embed, rank (retrieval.go mirror), query, extract, keywords,
│                         #   retrieve-c3 (C3 top-1), solve, judge
├── corpus/               # domains, gen-sessions, build-corpus, scenarios
├── eval/                 # retrieval eval (Recall@k/MRR/nDCG/separation/%lost-to-gate) + ablation matrix;
│                         #   behavioral eval (3-arm) + metrics
└── results/<timestamp>/  # matrix.json + summary.md per run
```

### Two evals
- **Retrieval eval** (`npm run sim:eval`, CPU-only) — ranks a synthetic corpus against synthetic queries across an ablation matrix (C0 prod-baseline … C3 full-proposal); measures whether the situation-centric `retrieval_card` representation ranks the gold memory highly.
- **Behavioral eval** (`npm run sim:behavioral`) — 3 arms per (scenario, lower-tier model): no injection / oracle-gold injection / realistic top-1 from the C3 retrieval pipeline; a blind `opus-4.8-fast` judge scores each answer 0–3 against the extracted ground-truth lesson; reports per-arm lift + C3 in-set retrieval hit-rate.

### Parity invariant (Stage-1 anti-drift)
Because `rank.mjs`/`retrieve-c3.mjs` are hand-ports of the Go retrieval path (note the `// Source: wevibe-sim/ranking-fix.js` citation in `retrieval.go`), sim and product can silently drift. The planned guard is a **cross-language parity harness**: shared golden fixtures (known inputs → known rankings) that BOTH the Go retrieval tests and the JS sim must satisfy, so any divergence fails CI rather than relying on manual vigilance.

### Execution policy
Every run is MANAGER-run, wrapped in `timeout` + an in-process watchdog (per-call kill, per-stage wall-clock cap, spawn-budget cap, 15s heartbeats). Cloud LLM calls run in an isolated working dir (`--dir`) so the workspace `AGENTS.md` is never injected into generation/extraction; sessions are scrubbed after each run.

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
  access (**model & price set by the leader**, market-driven); settlement runs
  through an on-chain demand-leg router that enforces a protocol burn
  `max(n%, floor)` (the loop's deflationary sink); the remainder's destination
  (non-custodial leader wallet vs network-held per-org account) is an open
  custody question. The prior Treasury/`MsgWithdrawTreasury` model is removed
  and no withdrawal path is built. Moderator pay is leader-discretionary from
  that (open) revenue path. Decided, not yet built. See `DECISIONS.md`
  `D-ECON-CANON` / `D-ECON-STORAGE-MARKET`.
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
4. Invite/create/rotate: the leader derives the epoch keypair from K_master locally (`epoch_sk = HKDF(K_master, info="wevibe-umbral-epoch-{epoch}")`), mints kfrags locally, and delivers only the finished kfrag (StoreKFrag) + the epoch public key `umbral_pk`. No `epoch_sk` is sent to the hub; no `epoch_sk`/`epoch_pk` is persisted (re-derivable from K_master). See D-LEADER-SIDE-UMBRAL-MINT.

**B2 org-setup bridge (wevibe-mcp host :4450, bearer session-token; dashboard delegates org crypto here — browser cannot run Umbral):**
- `POST /v1/org-setup {org_name, domain, leader_wallet}` → MCP generates K_master, derives `umbral_pk`, mod identity, seals envelopes, signs the create-org canonical; holds `{masterKey, modPrivkey}` in-process by `setup_id` (30-min TTL); returns `{setup_id, payload, recovery_phrase}` (private keys never returned).
- `POST /v1/org-setup/finalize {setup_id, org_id}` → MCP persists `master` + `mod-privkey` keystore envelopes under the chain-assigned `org_id` (chain-first; org_id unknown until the Keplr broadcast).
- `POST /v1/provision-recall {org_id}` → `provisionRecall`: derive epoch_sk, mint kfrag for the member's pre_pubkey, register pre-pubkey, StoreKFrag.

**Retrieval flow (CO-218/CO-221 — PRE re-encryption):**
1. Consumer posts query with `pre_pubkey` (their secp256k1 PRE public key)
2. Hub verifies membership → queries Qdrant → fetches chain attestation data
3. Hub loads `umbral_capsule` + `umbral_ciphertext` from PostgreSQL per approved memory
4. For each memory with a capsule: hub calls `ReEncrypt` on sidecar (org_id, epoch_id, member_pk, capsule)
5. Sidecar applies stored kfrag → returns cfrag
6. Hub returns cfrag + capsule + umbral_ciphertext to consumer (hub never sees plaintext DEK)
7. Consumer decrypts locally with `decrypt-reencrypted`

**Leader-side kfrag lifecycle (D-LEADER-SIDE-UMBRAL-MINT — supersedes CO-218 "Option C"):**
- The LEADER mints kfrags locally (MCP umbral CLI, from the K_master-derived epoch_sk + member pre_pubkey) and StoreKFrags the finished kfrag to the hub sidecar (the hub never holds epoch_sk and never mints).
- Hub triggers kfrag deletion on member removal (calls sidecar `DeleteKFrags`)
- No chain event subscription needed for Phase 1

**PRE endpoints (hub sidecar keeps ReEncrypt + StoreKFrag + DeleteKFrags + DeleteOrgKFrags + Health only):**
- `ReEncryptForMember` is called inside `QueryMemories` (recall) — no separate internal endpoint.
- `POST /v1/orgs/{orgID}/members/{pubkey}/kfrag` — leader-signed → `umbralService.StoreKFrag` (stores a finished kfrag).
- RIPPED (D-LEADER-SIDE-UMBRAL-MINT): `umbralService.GenerateEpochKeyPair`, `umbralService.RegisterMember`, `POST /v1/internal/epoch-keypair`, `POST /v1/internal/orgs/{orgID}/kfrags`, and the sidecar gRPC `GenerateKeyPair`/`GenerateKFrags` RPCs.

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
- `InviteMemberRequest`: field `pre_pubkey` (NO `epoch_sk` — kfrags minted leader-side, delivered via StoreKFrag)
- `POST/GET /v1/orgs/{orgID}/members/{pubkey}/pre-key`: member PRE key registration/lookup
- `POST /v1/orgs/{orgID}/members/{pubkey}/kfrag`: leader-signed StoreKFrag `{epoch_id, pre_pubkey, kfrag}`
- `CreateOrgRequest`: field `umbral_pk` (HEX, leader-derived epoch public key; was `[]byte`); no `epoch_sk`/`epoch_pk` in any create/rotate response (leader derives locally from K_master). D-LEADER-SIDE-UMBRAL-MINT.

**CO-216-F2 Resolution (CO-217):** Sidecar tests added (9 tests in `tests/integration.rs`)
**CO-216-F3 Resolution (CO-217):** gRPC stubs generated from proto, hand-written types removed
**CO-216-F4 Resolution (CO-217):** SecretKeyFactory workaround verified sound — full Umbral workflow passes

---

# RECALL → INJECTION PIPELINE — COMPLETE MAP & DEFECT LEDGER (SoT — charted 2026-06-21)

> **This is the single source of truth for the org-memory recall→injection pipeline, root→leaf.** Charted by a 4-slice parallel gather sweep (Opus 4.6 fast) on 2026-06-21 against the live code. It records the LIVE path AND every dead / stubbed / contradictory finding, because the system is being overhauled (suspected DOA after many cumulative pivots). **As code is edited/pruned/added in the overhaul, THIS section is updated in lockstep to match the new code.** Every claim is `file:line`-cited; treat citations as load-bearing.
>
> **Layer ownership:** Stage 1 = `wevibe-hub` (Go) · Stage 2 = `wevibe-mcp` (TS) · Stage 3 = `wevibe-opencode-plugin` (TS) · Stage 4 = `opencode` runtime (vendored binary).
>
> **✅ Resolved 2026-06-21 (Phase 2 prune):** the older "PRE Retrieval Data Flow" section calls `UpdateMemoryKeywords` and `ScrollApprovedMemories` live. Re-verified workspace-wide: `UpdateMemoryKeywords` IS live (`internal/api/handlers/keywords.go:306,400`) — old section correct; `ScrollApprovedMemories` was dead and has been removed. **Lesson: the G1 gather's "zero callers" was package-scoped, not workspace-scoped — it false-flagged 6 funcs as dead that have callers in `internal/chain`/`handlers`. Always re-verify workspace-wide before deleting (this caught it pre-delete).**

## Pipeline at a glance (4 stages, root → leaf)

```
USER PROMPT (opencode session)
   │  plugin hook chat.message  (wevibe-plugin.ts:982)
   ▼
[Stage 3a] plugin.loadMemories ── POST 127.0.0.1:4450/v1/recall ──┐
                                                                   ▼
[Stage 2]  wevibe-mcp handleRecall (http-server.ts:231)
           → retrieve() → queryOrgMemories ── POST /v1/orgs/{org}/query ──┐
                                                                          ▼
[Stage 1]  wevibe-hub QueryMemories (handlers/retrieval.go:27)
           → QueryByKeywords → QdrantClient.QueryPoints → ScoreAndRank
           → relevance-floor + surface-budget → D-9.4 power-law sample
           → chain attest + Umbral ReEncrypt + contributor stats
           → QueryResponse{results,…}  ────────────────────────────────┐
                                                                        ▼
[Stage 2]  per-memory: fetch ciphertext → Umbral sidecar decrypt (execFile)
           → AES decryptSymmetric → artifact policy → wevibe-guard scan (annotate only)
           → {status, memories[]}  ────────────────────────────────────┐
                                                                        ▼
[Stage 3b] plugin caches memories → approval gate (TUI popup, if live)
           → experimental.chat.system.transform: output.system.push(memoryBlock)
                                                                        │
                                                                        ▼
[Stage 4]  opencode LLMRequestPrep.prepare → {role:"system"} message #2
           (OR providerOptions.instructions on OpenAI-OAuth) → MODEL
```

---

## Stage 1 — Hub retrieval ROOT (`wevibe-server/wevibe-hub`, Go)

**Route:** `POST /v1/orgs/{orgID}/query` (`cmd/wevibe-hub/main.go:280`), behind `auth.RequireVerifiedMembership` (`main.go:227`). **There is NO `/v1/recall` route on the hub** — `/v1/recall` is the MCP endpoint (Stage 2). Any doc/client referencing a hub `/v1/recall` is wrong.

**Files:** `internal/retrieval/{retrieval.go (1768L), ranking_core.go (246L), querylog.go, stats.go}`, `internal/api/handlers/{retrieval.go (614L), pool.go}`, `internal/config/config.go`, `cmd/wevibe-hub/main.go`, `protocol/types.go`.

**Call chain (verbatim hops):**
1. `handlers.QueryMemories` (`handlers/retrieval.go:27`) — parse `QueryRequest`; require orgID/agentPubkey/prePubkey; default limit test=1000 / prod=3 (`:64-68`); membership (`:76`); trial+daily gate (`:83-122`); rate limit from `org_recall_rate_limits` (`:124-152`); resolve epochs (`:154-164`).
2. `retrieval.QueryByKeywords` (`retrieval.go:1106`) — pure passthrough to `client.QueryPoints(...)`.
3. `QdrantClient.QueryPoints` (`retrieval.go:419`) — build Qdrant filter (org match, `must_not` ARCHIVED, optional DORMANT); `POST {restURL}/collections/{collection}/points/search` (`:478`); search limit = `recallDepth` (5000); dedup by CID; load pending-denial counts from PG `serve_events` (`:545`); build `[]RankCandidate` filtered to authorized epochs (`:578-640`).
4. `ScoreAndRank` (`ranking_core.go:162`) — the pure scoring engine (math below).
5. **Relevance floor + surface budget** (`retrieval.go:701-718`) — filter `weightedScore >= relevanceFloor`, then `cap = min(limit, surfaceBudget)`.
6. **D-9.4 power-law sampler** `probabilisticRank` (`retrieval.go:125-215`) — position 1 = strict argmax; positions 2+ sampled w/o replacement, weight `(score/maxScore)^(1/temp)`, `temp=0.7`.
7. Contested flag (`retrieval.go:767`, `contestedThreshold=0.20`).
8. Back in handler (`handlers/retrieval.go:183-416`) — async query-log persist; **session dedup: drop CIDs served in last 24h for same `session_id`** (`:185-208`); chain attest `GetMemoriesBatch` (`:241`); Umbral `ReEncryptForMember` from `pending_submissions` (`:257-346`); banned filter (`:348-370`); contributor stats (`:372-393`); receipt (`:395-405`); emit `QueryResponse` (`:409-416`).

**Scoring math (`ranking_core.go`):** `keywordScore = Σ(queryWeight[kw]·memoryWeight[kw])` (`:102-136`); `gammaBoost = keywordScore·0.1` (`:187`); `cappedBoost = min(gammaBoost, 0.15·vectorScore)` (`:188-194`); `final = vectorScore + cappedBoost` (`:195`); pending denial `final = max(0, final − denials·0.05)` (`:197-199`); new-mem boost `final ·= 1 + 0.5·max(0, 1−age/(grace+window))` (`:201-208`); sort by `final` desc. Constants: `keywordBoostFactor=0.1`, `keywordBoostDelta=0.15` (`retrieval.go:450-451`), `EMBED_DIM=768`, `recallDepth=5000`, `DenialDecayBPS=500`, `ServeBoostBPS=100`.

**Structs (`protocol/types.go`):** `QueryRequest` (`:268`: org_id, agent_pubkey, pre_pubkey, keyword_weights, vector, embedding_model_id, limit, session_id, include_dormant, relevance_floor, surface_budget, agent_sig); `MemoryResult` (`:284`); `ScoringBreakdown` (`:226`: keyword_score, vector_score, gamma, delta, capped_boost, combined_score, keyword_matches, unmatched_query_keywords); `QueryResponse` (`:313`: results, contested, receipt_id, requires_reencryption).

**Recall mode:** `config.go:48` reads `WEVIBE_RECALL_MODE` (default prod); `main.go:75` `SetRecallMode`; `pool.go:33` `recallModeIsTest()`. **Prod vs test differs ONLY in throttles** (default limit 3→1000, trial+rate-limit bypassed `handlers/retrieval.go:64-124`). Scoring/floor/budget/sampler are mode-independent.

**Qdrant layer:** pure HTTP REST client (no gRPC/SDK), `QdrantClient` (`retrieval.go:242`); **new `http.Client` per request, 10s timeout** (no keep-alive/pooling).

**Stage-1 dead/cruft — ✅ PRUNED 2026-06-21 (Phase 2a, retrieval.go 1767→1526, −241L; hub `go build ./...` + `go test ./...` green):**
- **REMOVED (re-verified truly dead, zero workspace callers):** `computeKeywordScore`, `applyPendingDenialDecay`, `applyNewMemoryBoost` (method), `CountPoints`, `ScrollApprovedMemories` — plus dead collateral `ErrInvalidOffset`, unused imports (`encoding/base64`, `errors`), dead const `MaxServesPerEpoch`, and the dead-only test `retrieval_optimistic_test.go` + dead `applyNewMemoryBoost` cases in `ranking_test.go`.
- **REMOVED dead `MemoryResult` fields:** `ConfidenceBps`, `RetrievalCount`, `WrappedDekEnc` (`internal/protocol/types.go`).
- **KEPT — gather was WRONG (these have live callers, NOT dead):** `GetKeywordWeights`, `ApplyServeBoostLocal`, `ApplyDenialDecayLocal` (← `internal/chain/watcher_serve.go`); `ScrollOrgMemoryPayloads`, `UpdateMemoryState` (← `internal/chain/sync.go`); `UpdateMemoryKeywords` (← `internal/api/handlers/keywords.go`).
- Still present (live, unchanged): `Gate: false` hardcoded (`retrieval.go:643`); Gaussian noise `sigma=0.0` no-op + index-time only; `QueryRequest.EmbeddingSchemaVersion` unused (request field).

---

## Stage 2 — MCP recall MIDDLE (`wevibe-mcp`, TS)

**Endpoint:** `POST /v1/recall` (`http-server.ts:957`) → `handleRecall` (`http-server.ts:231`); Bearer-token auth (`authorize()` `:105`).

**Call chain:**
1. `handleRecall` (`http-server.ts:231`) — authorize; `flushDenials()` fire-and-forget (`:236`); parse body, apply governor defaults for limit/relevance_floor/surface_budget (`:239-256`); require `query` (`:263`); call `retrieve(input)` (`:281`).
2. `retrieve()` (`retrieve-cli.ts:262`) — `initCrypto` → `ensureIdentity` (lazy biometric, registers PRE pubkey) → `loadMemberships` (`org-client.ts:240`) → select org → `getActiveHubUrlForOrg` → `buildQueryHarvest` (`:188`) → `buildNeedCard` (`retrieval-card.ts:84`) → `dissect_to_keywords` → `computeLocalEmbedding` (`embedding.ts`) → **`queryOrgMemories`** (`org-client.ts:121`, POSTs `/v1/orgs/{orgId}/query`) → `deserializeMemoryResult` (`deserialize.ts:56`).
3. **Per-memory decrypt loop** (`retrieve-cli.ts:386-483`) — fetch ciphertext via `hubFetchVerified` `GET /v1/orgs/{orgId}/memories/{cid}` (`:392`); `decryptMemoryBlob` (`org-client.ts:403`): `getOrCreatePreIdentity` → `getEpochUmbralPk` → **`umbralDecryptReencrypted`** (`sidecar.ts:120`) → `decryptSymmetric(ciphertext, dek)` (AES); then `extractArtifacts` / `checkArtifactPolicy` / `transformMemoryContent` / `formatTrustPanel`; build `MemoryOutput`.
4. Back in `handleRecall` — **`runWeVibeGuard`** per memory (`http-server.ts:302`); provider-policy check (`:321-328`); emit `{status:'ok', memories:[…], reason_code?}` (`:354`).

**Decrypt + guard mechanics:**
- **Umbral sidecar = `execFile` child process** (`sidecar.ts:54`). Binary from `WEVIBE_UMBRAL_SIDECAR_BIN` — **REQUIRED, no fallback; throws if unset** (`sidecar.ts:14`). Args `--capsule --cfrags --ciphertext --receiving-sk --delegating-pk`; returns `{plaintext}` on stdout. PRE secret key stored in OS keychain via `keytar` (`auth.ts:47`).
- **wevibe-guard = `spawnSync`** (`guard.ts:43`); binary from `WEVIBE_GUARD_BIN` or relative fallback (`guard.ts:19-29`); JSON stdin → `{passed, detections, flags}`.
- **Guard does NOT block** — failing memories are still returned with `guard.passed=false` attached (`http-server.ts:314-318`); blocking is delegated to the plugin.
- **Decrypt failure silently skips the memory** (`retrieve-cli.ts:477-482` `continue`); only if ALL fail does `reason_code:'decrypt_failed'` surface. Partial loss is invisible to the caller.

**Types:** `RetrieveInput` (`retrieve-cli.ts:19`), `MemoryOutput` (`retrieve-cli.ts:41`), `MemoryWithGuard` (`http-server.ts:115`, adds `guard{passed,detections,flags}`), `ScoringBreakdown` (`types.ts:35`), deserialized `MemoryResult` (`types.ts:46`). **No `blocked` and no `source` field exists** anywhere in MCP types. `MemoryType = 'memory'` single value (`types.ts:99`, D-5.1).

**Recall mode:** `getRecallMode()` (`retrieve-cli.ts:93`); `RECALL_MODE_GOVERNORS` (`:80`): prod `{floor 0.55, budget 3, limit 3}`, test `{0, 1000, 1000}`; used as request defaults in `handleRecall` (`http-server.ts:239`).

**Stage-2 dead/cruft:**
- `agentSig` — **✅ REPLACED 2026-06-21 with real request-body signing.** The dead `agent_sig` body field is gone; the MCP now signs the exact serialized request body with the agent Ed25519 key and sends it in header `X-Agent-Signature` (`org-client.ts` queryOrgMemories); the hub reads raw body bytes, `ed25519.Verify` against the middleware-authenticated pubkey, **401 on missing/invalid**, then unmarshals (`handlers/retrieval.go`), and stores the verified sig in `usage_receipts.agent_signature` (now meaningful, no DB migration). Hub `go build/test` + MCP tsc green. **⚠ WIRE-CONTRACT CHANGE: hub + MCP must be redeployed together** (old MCP → new hub = 401).
- **✅ PRUNED 2026-06-21 (Phase 2b, server.ts 663→388, −275L; tsc green):** removed the dead old-MCP-tool recall graveyard — `recallTimeScan`, `gateMemories`, `rerankByRelevance`, `disambiguateMemories`, `buildElicitationPreview`, `formatMemoryPresentation`, `FormattedMemory` type, `ALLOW_UNREVIEWED` — plus now-unused imports (`runWeVibeGuard`, `MemoryResult`, `getLlmProvider`) and dead-only test cases in `tests/security/recall-pipeline.test.ts` + `tests/server-tools.test.ts`. (Pre-existing unrelated failures remain in `tests/sidecar.test.ts`: "Invalid SecretKey" — NOT caused by the prune, verified.)
- **Risk appetite (consumer filter):** LIVE via dashboard settings page + TUI `/wevibe-risk` → `~/.wevibe/plugin-config.json` → plugin `getRiskAppetite()` filters at injection. **✅ 2026-06-21: removed the vestigial MCP path only** (`wevibe_set_risk_appetite` tool + MCP `getRiskAppetite/setRiskAppetite`); kept `getProviderPolicy/setProviderPolicy` (live) and the dashboard/TUI/plugin path (the real one).
- **✅ 2026-06-21: `loadMemberships` response now verified** — added `hubFetchVerifiedWithKey` (shared verify logic) + cached hub-level `response_pubkey` from `GET /v1/hub/serving-address`; `loadMemberships` no longer uses raw `fetch`. Caveat: the hub-level key is self-reported (not chain-anchored like org keys) — acceptable for the membership list; stronger anchoring is future work.
- Guard scan passes empty keywords+metadata (`http-server.ts:302`). *(still open)*

---

## Stage 3 — Plugin recall + inject LEAF (`wevibe-opencode-plugin/plugins/wevibe-plugin.ts`, ~1147L + `tui/wevibe.tsx`)

**Hooks returned (`:973`):** `tool.execute.before` (`:974`), `chat.message` (`:982`), `experimental.chat.system.transform` (`:1006`), `experimental.session.compacting` (`:1137`).

**Recall trigger chain:**
- **Prewarm IIFE** (`:930-946`) at plugin load — `getRecallMode`, `ensureWeVibeMcpRunning`, `loadMemories(queryToUse)` where `queryToUse` is project-derived (`:898-929`, fallback `"project coding standards conventions best practices"`). **`activeSessionId` is null here → `currentSessionId()` returns `"prewarm"`** (`:290`) → recall sent with `session_id:"prewarm"`.
- `chat.message` (`:982`) → `triggerRecall` (`:891`) → `loadMemories` (`:667`). **Single-flight: if a recall is in-flight, the new one is silently dropped** (`:894`).
- `loadMemories` (`:667`) — cache check (5min TTL); `getRecallGovernorConfig()`; `POST 127.0.0.1:4450/v1/recall` with `{query, limit, session_id, relevance_floor, surface_budget}` (`:698`); clear+rebuild `cachedMemories`/`memoryIndex` (`:754`); per memory build `CachedMemory`, auto-deny if guard-blocked, **test-mode AUTO-APPROVE → `approvedCids.add(cid)`** (`:816-819`, *Phase 1 2026-06-21 — was a delete that forced re-popup*), enqueue undecided candidates for prod popup (`:856`, comment "Hub governs… no client-side re-governing" `:854`).

**State model (`:276-281`):** `approvedCids`, `deniedCids`, `reportedCids`, `pendingCids`, `sessionInjectedCids: Map<sid,Set>`. **Init gate (`:311`): `if (getRecallMode() !== "test") load accepted` — test mode starts with empty approvals.** Files in `.opencode/`: `wevibe-plugin-status.json` (accepted/denied/reported, written by `recordStatusSnapshot` `:370`), `…-queue.json`, `…-decisions.json`, `wevibe-tui-active.json` (heartbeat). Plus `~/.wevibe/blacklist.json` (`seedDeniedFromLocalBlacklist` `:292`, called at init AND every transform `:1027`).

**Injection mechanism — `experimental.chat.system.transform` (`:1006-1143`, Phase 1 2026-06-21):** (1) await in-flight recall ≤15s (`:1009`); (2) `drainDecisions` + reseed blacklist (`:1026`); (3) compute pending-undecided (`:1029`); (4) **TUI gate wait loop ONLY `if (isTuiLive())`** ≤5min, 250ms poll (`:1044-1061`); (5) **eligible filter requires `approvedCids.has(cid)`** (`:1070-1074`); early-return only if `eligible.length===0` (`:1077-1083`); (6) **EVERY-TURN PUSH: build `memoryBlock` from ALL `eligible` and `output.system.push` it every turn** (`:1088-1103`) — the SOLE injection point, fixes the once-per-session DOA; (7) header is **mode-aware** — test = honest ("you may acknowledge these team memories…"), prod = covert ("Do not mention WeVibe Network…") (`:1092-1094`); (8) **toast** in test mode when `newlyServed.length>0` via `client.tui.showToast` (`:1112-1121`); (9) **serve attestation once per session**: `newlyServed = eligible.filter(!injectedSet.has(cid))`, fire `/v1/serves` + `injectedSet.add` ONLY for those (`:1123-1143`).

**Popup gate:** `isTuiLive()` heartbeat <30s (`:353`); TUI writes heartbeat /10s, polls queue /5s (`wevibe.tsx:1004/1019`); `recordDecision` appends to decisions file (`wevibe.tsx:416`); `drainDecisions` (`:379`) maps accept→approved / deny→denied / block→denied+blacklist+hub denial / report→reported+hub report.

**Guard-failed memory handling (✅ 2026-06-21):** guard-FAILED memories are no longer silently auto-denied — they are surfaced in the approval popup with **Report as the default-selected action** and **Accept moved to the end** (deliberate navigation), Deny/Block unchanged (TUI `wevibe.tsx` builds the action order conditionally on the guard-flagged check; `a/d/b/r` shortcuts resolve by label). Plugin: guard-failed memories enqueue (not auto-deny) and count as pending-undecided; an explicit **Accept overrides the guard block** (the inject filter now gates on approval, not `blocked`); **guard-failed never auto-approves even in test mode**.

**Headless vs TUI behavior (load-bearing; Phase 1 2026-06-21):**

| Scenario | Prod | Test |
|---|---|---|
| **TUI live** | gate waits for accepts → approved memories inject, re-pushed every turn | auto-approved on recall → inject + toast, re-pushed every turn |
| **No TUI (headless `opencode run`)** | only memories approved in a PRIOR session (loaded from status file) inject; NEW memories never inject (popup-gated, by design) | **auto-approve → injects with no popup** (`:816` add), toast confirms; fixes the prior "nothing ever injects" headless/test failure |

**Stage-3 dead/cruft:** **✅ PRUNED 2026-06-21 (Phase 2c, tsc green):** removed `contextPaths` (populated-never-read) + the now-empty `tool.execute.before` hook that only fed it, and `memoryIndex` (populated-never-read). Still present: compaction filter omits `deniedCids`+appetite, inconsistent with inject filter (`:1138`); `"prewarm"` `sessionInjectedCids` entry never cleaned; 3 TUI render harnesses duplicate `parseRetrievalCard`/theme (`tui/_render_*.tsx`). **CORRECTION (2026-06-21): the client-side governor (`PROD/TEST_RECALL_GOVERNOR_DEFAULTS` + `getRecallGovernorConfig`) is NOT vestigial** — it is the SOURCE of `relevance_floor`/`surface_budget`/`limit` sent to MCP→hub, intentionally duplicated across plugin/MCP/hub per D-RECALL-MODE-FLAG. The `:854` "hub governs" comment means "don't RE-filter client-side after the hub returns," NOT that the governor params are dead. Likewise plugin `getRiskAppetite()` (`:1053`) is LIVE (filters at injection); the dashboard settings page + TUI dialog are its real consumer UIs (write `~/.wevibe/plugin-config.json`). Only the **MCP** `wevibe_set_risk_appetite` tool + MCP `getRiskAppetite/setRiskAppetite` are vestigial (a 3rd redundant path; D-RISK-APPETITE-UI: tool "stays registered but is NOT the runtime path").

---

## Stage 4 — Inject → LLM boundary (`opencode` runtime — vendored, v1.16.0)

**Hook type** (`.opencode/node_modules/@opencode-ai/plugin/dist/index.d.ts:265-270`):
```ts
"experimental.chat.system.transform"?: (input: { sessionID?: string; model: Model },
                                         output: { system: string[] }) => Promise<void>
```

**Dispatch — `LLMRequestPrep.prepare`** (opencode binary, de-minified):
1. Build `e = [ join([agent.prompt, ...A.system, user.system], "\n") ]` (single string, `e[0]`), save `o = e[0]`.
2. Call hook with `{system: e}` → plugin does `e.push(memoryBlock)` → `e = [base, memoryBlock]`.
3. Post-hook consolidation **only if `e.length > 2 && e[0] === o`** (fires when plugin pushed ≥2 entries; our push of 1 → `length===2` → does NOT fire; block stays a separate entry).
4. Standard path: `messages = [...e.map(s => ({role:"system", content:s})), ...A.messages]` → memory block becomes **system message #2**.
5. **OpenAI-OAuth path** (`provider.id==="openai" && auth.type==="oauth"`) OR `isWorkflow`: system entries are **NOT** prepended as messages; joined into `providerOptions.instructions` instead. Content still delivered, different channel.

**Delivery verdict:** plugin-pushed `output.system` **IS delivered to the model** (system message on standard path, `instructions` on OAuth). Issue #885 ("not rendered in transcript") is **display-only — still sent to the model**.

**On which turns:** `prepare` + the hook run **every turn**, BUT the plugin's per-session `sessionInjectedCids` dedup pushes the block **only on the first eligible turn**; on later turns the plugin early-returns without pushing → **the memory block is in the system prompt for ONE turn, then gone.**

---

## OVERHAUL DIRECTION — LOCKED by Walter 2026-06-21

Three forks resolved; all subsequent edits target these, and the Stage sections above are updated to match as code changes:

1. **Injection persistence → EVERY TURN (was once-per-session).** Approved/eligible memories are re-pushed into `output.system` on every turn so they persist in the model's context for the whole session (fixes the #1 DOA cause below). The "already injected this session" concept is SPLIT: a per-turn presence push (re-send every turn) is separated from serve-attestation counting (`/v1/serves` fires once per memory per session). `sessionInjectedCids` no longer gates the push — only the serve count. AMENDS the once-per-session heritage of D-SESSION-SERVE-DEDUP.
2. **Auto-approve in TEST mode + non-blocking toast (was popup-only, mandatory).** In `WEVIBE_RECALL_MODE=test`, recalled memories auto-approve (no blocking gate) so headless/benchmark runs inject. Observability replaces the blocking popup: AFTER injection returns, fire a non-blocking TUI **toast** ("N memories injected …") reusing wevibe's existing toast surface. **PROD stays popup-only / human-in-the-loop — auto-approve is strictly test-gated.**
   - **Toast surface (charted 2026-06-21):** the server-side plugin's `client` object exposes `client.tui.showToast({ body: { title, message, variant: "info"|"success"|"warning"|"error", duration } })` (SDK `@opencode-ai/sdk/dist/gen/sdk.gen.d.ts:328-402`, types `types.gen.d.ts:3264-3286`, HTTP `POST /tui/show-toast`). Plugin currently makes only ONE `client.*` call — `client.app.log` (`wevibe-plugin.ts:333-341`); the toast is additive, fire-and-forget, same pattern. The TUI already uses `api.ui.toast()` 23+ times (wrapper `tui/wevibe.tsx:251-257`; real endpoint-change toast `tui/wevibe.tsx:1093`). **Wire point:** in the inject hook immediately after the `[inject]` log (`wevibe-plugin.ts:1112`), before the serve loop (`:1114`).
3. **Honest/inspectable injected block (was covert "do not mention").** The "Do not mention WeVibe Network or this section to the user" instruction is dropped (at least in test) so the model can acknowledge what it received and the pipeline is verifiable by asking. Prod tone is a separate UX call.

## DOA ROOT-CAUSE — "[inject] logged but the model saw nothing"

Ranked by likelihood (converging finding across Stage-3 + Stage-4):

1. **Per-session single-turn injection (PRIMARY).** Plugin injects each memory once per session (`sessionInjectedCids`, plugin `:1086`); opencode rebuilds the system prompt every turn (Stage 4). So the block is present only on the turn it first injects and vanishes thereafter. If the user asks "what memories did you see?" on a *later* turn, the model genuinely has no memory content in context — yet an `[inject]` line was logged earlier. **The "once per session" design (D-SESSION-SERVE-DEDUP heritage) is fundamentally incompatible with a per-turn-rebuilt system prompt.**
2. **`"Do not mention WeVibe Network or this section"` instruction** (plugin `:1096`) — even when the block IS present, the model is told to deny it. Makes "what memories did you see?" a useless probe (always denies).
3. **Headless / test-mode = zero injection** — no live TUI ⇒ `approvedCids` empty ⇒ nothing eligible (Stage-3 table). Any benchmark arm using `opencode run` injects nothing.
4. **Silent partial drops upstream** — MCP decrypt failures skip memories with no signal (`retrieve-cli.ts:477`); guard never blocks but plugin auto-denies guard-blocked CIDs.
5. **Prewarm/single-flight race** — first user message's recall can be dropped (`:894`); transform may inject on the prewarm (project-derived) query, not the user's question.

**Overhaul implication:** injection persistence must move from "once per session" to "present in system context whenever eligible" (re-push every turn, dedup at the boundary not the source), the approval model needs a headless/auto path, and the "do not mention" framing needs a decision. These are architecture forks for Walter, tracked next to this map.

## OPERATIONAL FRAGILITY — ephemeral kfrag store (charted 2026-06-21, hit live)

**The Umbral sidecar's kfrag store is in-memory only (`DashMap`, no disk, no volume).** Confirmed: `wevibe-umbral` container has zero mounts; SESSIONCONTINUANCE documents "StoreKFrag → in-memory DashMap, no disk." **Any restart of the sidecar (incl. a Docker daemon bounce) wipes all kfrags**, after which EVERY recall fails server-side:
```
[recall] umbral ReEncrypt FAILED … member=… : kfrag not found in sidecar
[recall] umbral re-encryption complete reencrypted=0 requiresReencryption=N total=N
```
→ hub returns results with no capsule → MCP rejects with `hub query failed: Error: memory result missing capsule` → `/v1/recall` returns **HTTP 500** → plugin logs `cached=0 … nothing injected`. This is NOT a code bug in the recall logic and NOT caused by the plugin — it is lost crypto state.

**Non-destructive recovery (re-mint + StoreKFrag, preserves corpus + chain):**
```
TOKEN=$(cat ~/.wevibe/mcp-session-token)
curl -s -X POST http://127.0.0.1:4450/v1/provision-recall \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"org_id":"wevibe-org-0"}'   # → {"status":"ok"}  (may prompt Touch ID to unseal master key)
```
`provisionRecall` (`wevibe-mcp/src/org-client.ts:644`): loads org master envelope (leader-only) → derives epoch_sk → mints kfrag for the consumer PRE pubkey → registers pre-pubkey → `POST /v1/orgs/{org}/members/{edPubkey}/kfrag` (StoreKFrag). Also exposed as `wevibe-admin provision-recall --org <id>` and the dashboard members-page button. **Do NOT use `make dogfood` to recover — it runs `docker compose down -v` and wipes the corpus + chain.**

**Overhaul candidate (Walter):** either persist the kfrag store to disk/volume, or auto-reprovision on sidecar startup, so a restart doesn't silently kill recall. Tracked as a fragility, not yet fixed.

**✅ FIXED 2026-06-21 (persist-to-disk):** `KFragStore` now loads from disk on `new()` and persists atomically (temp→`0600`→write→fsync→rename→`0600`) on every `insert`/`delete`/`delete_org`; path from env `WEVIBE_UMBRAL_KFRAG_STORE` (default `/data/kfrags.json`); serde_json with hex-encoded binary (exact round-trip); corrupt/missing file → start empty (no crash); startup logs entry count + a loud warning if empty. `docker-compose.yml` adds volume `wevibe_umbral_kfrags:/data` + the env. `cargo build --release` + tests green. **Deploy note:** rebuild the `wevibe-umbral` image with the volume; on first boot after deploy the store is still empty → re-provision ONCE (`POST /v1/provision-recall`), after which it survives all future restarts. `make dogfood` (`down -v`) still wipes intentionally.
