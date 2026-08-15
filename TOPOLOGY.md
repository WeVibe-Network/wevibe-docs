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
R-PROTO-REGEN.

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

Used by the empirical replay harness at `wevibe-meta/scripts/empirical_replay/` (CO-034). See DECISIONS for the Sprint 32 replay contract.

### Chain Broadcast (CO-258)

Leader/member chain tx broadcast is dashboard wallet-direct (`directBroadcast` to chain RPC). Hub does not relay leader-signed txs or expose a delegate-key relay path (see DECISIONS D-ECON-STORAGE-MARKET).

### Schema Bootstrap

Hub schema at `wevibe-server/db/schema.sql`. Applied on Postgres container init (D-13.10).

---

## wevibe-server/wevibe-hub — Go API Server

**Module:** `github.com/wevibe-network/wevibe-server/wevibe-hub`  
**Go version:** 1.25.9  
**Default port:** 4440

### Entry Point

#### `cmd/wevibe-hub/main.go`
**Role:** HTTP server entry point — loads config, applies schema, wires DB/chain/Qdrant/middleware/routes, starts watcher + HTTP server

**Key behavior:**
- Startup failure model: DB/chain failures are fatal (`db.NewPool`, chain enabled/config/client checks); only Qdrant degrades gracefully.
- Response signing: initializes signer (`hubsign.NewFromEnv`), publishes serving pubkey (`handlers.SetResponsePubkeyHex`), and signs all responses via `hubsign.SigningMiddleware`.
- Schema bootstrap: runs `db.ApplySchema(cfg.DatabaseURL)` before creating the pgx pool.
- Retrieval init: `retrieval.NewQdrantClient(...)` then `SetRetrievalConfig(...)` + `SetPendingDenialDB(pool)` + `handlers.SetQdrantClient(...)` when available.
- Startup sync: `chain.SyncEpochData(...)` + `chain.SyncKeywordWeightsFromChain(...)` (when Qdrant is available).
- Epoch ticker loop: every 60s runs both `chain.SyncEpochData(...)` and `chain.ReconcileMembership(...)`.
- Handler wiring: `SetPool`, `SetRecallMode`, `SetChainClient`, `SetFaucetURL`, `SetSocialClient`, `SetUmbralService`, `SetNodePrivkey`.
- Notifications wiring: creates notifications Hub + Dispatcher; registers SMTP channel (if configured) and webhook channel.
- Chain watcher wiring: builds tx decoder, injects dispatcher, starts watcher goroutine.
- CORS: reads `CORS_ALLOWED_ORIGINS` env var (comma-separated), defaults to `http://*`, `https://*`; exposes hub signature header.

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
POST   /v1/orgs/{orgID}/contributors/{contributorPubkey}/deny-pending
POST   /v1/orgs/{orgID}/submissions/{hash}/keyword-vote           # Per-keyword include/exclude vote
POST   /v1/orgs/{orgID}/moderation/batch-submit                   # Leader stages batch for chain

POST   /v1/orgs/{orgID}/serves                                    # Record serve event (CO-033a)
PUT    /v1/orgs/{orgID}/recall-rate-limit
GET    /v1/orgs/{orgID}/recall-rate-limit
GET    /v1/orgs/{orgID}/recall-queries
GET    /v1/orgs/{orgID}/recall-queries/{queryID}
GET    /v1/orgs/{orgID}/recall-health
POST   /v1/orgs/{orgID}/extracted-sessions
GET    /v1/orgs/{orgID}/extracted-sessions
POST   /v1/orgs/{orgID}/denials                                   # Consumer denial receipt
GET    /v1/orgs/{orgID}/denials/pending-count                     # Leader panel count
GET    /v1/orgs/{orgID}/denials/pending                           # Leader pending list (newest-first, cap 200)

POST   /v1/orgs/{orgID}/query                                     # PRE retrieval query
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
GET    /v1/orgs/{orgID}/submissions/duplicate-clusters
DELETE /v1/orgs/{orgID}/submissions/{hash}                        # Remove submission from pipeline (CO-238)
GET    /v1/orgs/{orgID}/submissions                               # List all submissions (leader/contributor view)
GET    /v1/orgs/{orgID}/my-submissions                            # Contributor-only own submission status
GET    /v1/orgs/{orgID}/commit-status

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
POST   /v1/test/embed                                             # Test embedding with configured provider
GET    /v1/test/orgs/{orgID}/queue                                # Dump pending_submissions for org
GET    /v1/test/orgs/{orgID}/serve-queue                          # Dump serve_events queue depth
```

**Auth middleware:**
- `auth.RequireVerifiedIdentity` wraps profile-notification + notification-read routes.
- `auth.RequireVerifiedMembership` wraps membership-gated org routes under `/v1/orgs/{orgID}`.

---

### Config

#### `internal/config/config.go`
**Exports:** `Config` struct, `Load() Config`
**Fields:**
```go
Port                       int       // env: WEVIBE_HUB_PORT, default: 4440
DatabaseURL                string    // env: DATABASE_URL
QdrantAddr                 string    // env: QDRANT_ADDR, default: "localhost:6333"
QdrantAPIKey               string    // env: QDRANT_API_KEY (required; panics if < 32 chars)
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
RecallMode                 string    // env: WEVIBE_RECALL_MODE, default: "prod" ("test" accepted)
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
var faucetURL string
var umbralService *umbral.Service        // CO-218 — PRE sidecar service (gRPC)
var socialClient *social.Client
var recallMode string

func SetPool(p *pgxpool.Pool)
func GetPool() *pgxpool.Pool
func SetQdrantClient(c *retrieval.QdrantClient)
func SetRecallMode(m string)
func SetNodePrivkey(key string)
func SetChainClient(c *chain.GrpcClient)
func SetFaucetURL(url string)
func SetUmbralService(s *umbral.Service) // CO-218
func SetSocialClient(c *social.Client)
func GetSocialClient() *social.Client
```

#### `internal/api/handlers/health.go`
**Exports:** `Health(w, r)` — returns JSON `{"status":"ok","timestamp":...,"version":"0.2.0","db":"connected|disconnected"}`
**Known issues:** None

#### `internal/api/handlers/orgs.go`
**Exports:**
```go
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
func GetKeyEnvelope(w, r)   // GET — reads from key_envelopes table, auth required
func ListMembers(w, r)      // GET — no auth
func GetMemberOrgs(w, r)    // GET /v1/members/{pubkey}/orgs — lists all orgs a member belongs to
func LinkWallet(w, r)       // POST /v1/orgs/{orgID}/members/wallet
func EnableMemberRecall(w, r)
func DisableMemberRecall(w, r)
func StoreMemberKFrag(w, r) // POST /v1/orgs/{orgID}/members/{pubkey}/kfrag
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
func SubmitMemoryBatch(w, r)   // POST /submit/batch — batch ingest path
func GetPendingQueue(w, r)     // GET — caller pubkey from auth header (leader or can_moderate)
func VoteOnSubmission(w, r)    // POST — advisory vote (approve/flag); advisory only, no quorum
func ApproveSubmission(w, r)   // POST — advisory approve (leader or can_moderate)
func DenySubmission(w, r)      // POST — calls moderation.DenySubmission() with reason
func VoteOnKeyword(w, r)       // POST — per-keyword include/exclude vote on a submission
func GetModerationHistory(w,r) // GET — last 24h of moderation decisions (leader or can_moderate)
func PrepareBatchForChain(w,r) // POST /moderation/batch-submit
```
**Known issues:** None

#### `internal/api/handlers/billing.go`
**Exports:**
```go
func TopUpCredits(w, r)   // POST /v1/billing/topup — reads {org_id, amount, signed_by} (dev-only)
func GetOrgCredits(w, r)  // GET /v1/orgs/{orgID}/credits — returns balance + transactions
func GetOrgFinances(w, r) // GET /v1/orgs/{orgID}/finances — hub credits + chain treasury
```
**Known issues:**
- `TopUpCredits` does NOT verify signature — anyone can top up any org (low risk: adds credits, doesn't remove them)
- No actual Stripe integration — `TopUpCredits` is manual credit injection, not payment processing
- `GetOrgCredits` does not verify caller is a member of the org — balance visible to anyone who knows orgID

#### `internal/api/handlers/retrieval.go`
**Exports:**
```go
func QueryMemories(w, r)       // POST — keyword + vector query, enforces recall gates/rate limits, returns PRE payload + usage receipt
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
func ListKeywordCandidates(w,r) // GET — leader review queue for suggested keywords
func AddKeyword(w, r)          // POST — leader only
func MergeKeywords(w, r)       // PUT — leader only, merges source into target
func RenameKeyword(w, r)        // PUT — leader only, renames keyword
func DeprecateKeyword(w, r)    // DELETE — leader only
```
**Known issues:** None

#### `internal/api/handlers/keyword_extraction.go` (CO-238)
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

#### Additional handler files (previously undocumented)
- `balance.go` — `GetBalance` endpoint proxying chain bank balance lookups.
- `ban.go` — leader `DenyPendingForContributor` moderation shortcut.
- `batch_submit.go` — `OrgHealth` endpoint summarizing org pipeline/queue health.
- `chain_config.go` — `GetOrgChainConfig` chain/org config read endpoint.
- `commit_status.go` — authenticated pending-chain/commit-error status endpoint.
- `dedup.go` — `DuplicateClusters` near-duplicate cluster inspection over stored embeddings.
- `denials.go` — pending denial count/list endpoints.
- `discovery.go` — public org discovery endpoint.
- `errors.go` — shared JSON error envelope helper (`WriteError`).
- `extracted_sessions.go` — record/list extracted contributor sessions.
- `extraction.go` — extraction profile read/write + extraction preset catalog.
- `faucet.go` — dev faucet funding endpoint.
- `identity_blobs.go` — passkey identity blob store/retrieve endpoints.
- `join.go` — join-request submit/list/approve/cancel/deny flow.
- `keyword_extraction.go` — keyword submission/verification/update/removal/listing.
- `notification_emit.go` — shared helper that emits persisted + live notifications.
- `pairing_blobs.go` — pairing blob store/retrieve endpoints.
- `profile_notifications.go` — get/update per-user notification preferences.
- `ratelimit.go` — set/get org recall rate-limit policy.
- `recall_inspector.go` — recall query list/detail/health inspection endpoints.
- `reports.go` — report create/list/get/update/vote/commit handlers.
- `serves.go` — serve/denial ingest + asynchronous relay batching to chain.
- `serving.go` — serving-address endpoint + response-pubkey setter.
- `social_names.go` — social-graph display-name resolution helper for wallets.
- `testing.go` — `WEVIBE_TEST_MODE`-gated health/embed/queue debug endpoints.

---

### Internal Packages

#### `internal/chain/grpc_client.go`
**Role:** gRPC connection to wevibe-chain node, signing key derivation, query client stubs
**Exports:** `NewGrpcClient(grpcURL, chainID, mnemonic)`, `Close()`, `SubmitterAddress()`, and query/signing accessors (`GetOrgQueryClient`, `GetMemoryQueryClient`, `GetServeQueryClient`, `GetAttestationQueryClient`, `GetBandwidthQueryClient`, `GetEmissionsQueryClient`, `GetReputationQueryClient`, `GetCodec`, `GetRegistry`, `GetKeyring`, `GetSubmitterAddress`, `GetTxConfig`, `GetChainID`).

#### `internal/chain/submit.go`
**Role:** Internal hub chain submission helpers for operational flows (not a dashboard delegate relay path)
**Exports:**
```go
func (c *GrpcClient) SubmitMemoryBatchAtomic(ctx context.Context, db *pgxpool.Pool, faucetURL, orgID string, memories []BatchMemory) (txHash string, committedCIDs []string, err error)
func (c *GrpcClient) SubmitRelayBatch(ctx context.Context, db *pgxpool.Pool, faucetURL, orgID string, msgs []types.Msg) (txHash string, err error)
func (c *GrpcClient) SubmitServeBatch(ctx context.Context, db *pgxpool.Pool, faucetURL, orgID string, epoch uint64, entries []ServeEntryInput) (string, error)
func (c *GrpcClient) SubmitDenialBatch(ctx context.Context, db *pgxpool.Pool, faucetURL, orgID string, epoch uint64, entries []DenialEntryInput) (string, error)
func (c *GrpcClient) BuildServeBatchMsg(orgID string, epoch uint64, entries []ServeEntryInput) (*servetypes.MsgSubmitServeBatch, error)
func (c *GrpcClient) BuildDenialBatchMsg(orgID string, epoch uint64, entries []DenialEntryInput) (*servetypes.MsgSubmitDenialBatch, error)
```

#### `internal/chain/query.go`
**Role:** On-chain state queries for org/memory/reputation verification and retrieval metadata reconciliation
**Exports:** `IsOrgRegistered`, `GetOrgFromChain`, `GetOrgMembersFromChain`, `GetOrgConfigFromChain`, `GetOrgAccountFromChain`, `GetOrgTreasuryBalanceFromChain`, `GetEpochMerkleRoot`, `GetServeParams`, `GetEpochServeStats`, `GetAttestationParams`, `GetSessionAttestation`, `GetBandwidth`, `GetEmissionsParams`, `GetReputationStats`, `GetContributorProfile`, `GetMemoriesBatch`

**Memory batch parity (CO-224):** `MemoryBatchResult` includes `Keywords`, `Epoch`, `State`, and `MemoryType` copied from chain `StoredMemoryCommitment` so hub can reconcile Qdrant payload metadata.

**Chain→Hub reputation wiring (CO-213):** `GetContributorProfile` queries the chain's x/reputation module for a contributor's on-chain profile (contribution_count, serve_count, first_seen_epoch). Nil-safe — returns nil if chain unreachable or contributor not found. Used by `retrieval.GetContributorStats` to merge chain and hub data.

#### `internal/chain/sync.go`
**Role:** Epoch metadata sync loop implementation for retrieval parity (CO-224)
**Exports:** `SyncEpochData(ctx, chainClient, qdrantClient, pool) error`, `SyncKeywordWeightsFromChain(ctx, chainClient, qdrantClient, pool) error`

**Sync flow (CO-224):**
- Loads orgs with committed memories from PostgreSQL (`pending_submissions.status='committed'`)
- Scrolls org memory payloads from Qdrant (`cid`, `keyword_weights`, `lifecycle_state`, `memory_type`)
- Batch-queries chain via `GetMemoriesBatch`
- Updates changed Qdrant payloads via `retrieval.UpdateMemoryState`
- Logs sync summary (`Synced N memories across M orgs, updated K confidence/state values`)

#### `internal/chain/merkle.go`
**Role:** Binary SHA-256 Merkle tree computation over approved memory hashes
**Exports:** `ComputeMerkleRoot(leaves [][]byte) string`, `HashContribution(content []byte) []byte`

#### Additional chain files (previously undocumented)
- `accounts.go` — `OrgKeyRole`/`OrgKeyServing` signer-role constants used by broadcast paths.
- `balance.go` — chain balance query helper (`GetBalance`).
- `broadcast.go` — tx simulation/broadcast engine (gas strategy, signer state, faucet top-up, commit/sync modes).
- `cometbft_subscriber.go` — CometBFT subscribe/block/status wrapper used by the chain watcher.
- `faucet.go` — faucet funding client (`FundAddressFromFaucet`).
- `reconcile.go` — periodic hub↔chain membership reconciliation (`ReconcileMembership`).

#### Watcher subsystem (`internal/chain/watcher*.go`)
- `watcher.go` — main chain watcher loop: subscribe/catch-up/reconnect, per-tx decode + routing, last-seen cursor persistence (`watcher_state`).
- `watcher_memory.go` — approved-memory commit bookkeeping + commit-error recording.
- `watcher_serve.go` — serve/denial batch bookkeeping after on-chain confirmation.
- `watcher_report_org.go` — report bookkeeping and org-level side effects.

---

### Business Logic

#### `internal/orgs/orgs.go`
**Exports:**
```go
func CreateOrg(ctx, pool, orgID string, req protocol.CreateOrgRequest) (*OrgInfo, error)
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
**Key detail:** `CreateOrg` calls `billing.ProvisionOrgLedger(ctx, pool, orgID, req.FeeModel.MonthlyCredits, req.LeaderPubkey)` after commit.
**Known issues:** None

#### `internal/members/members.go`
**Exports:**
```go
func InviteMember(ctx, pool, orgID, currentEpoch, req) (*MemberRecord, error)
func SetPrePubkey(ctx, pool, orgID, pubkey string, prePubkey []byte) error   // CO-221
func GetPrePubkey(ctx, pool, orgID, pubkey string) ([]byte, error)            // CO-221
func GetMember(ctx, pool, orgID, pubkey) (*MemberRecord, error)
func RemoveMember(ctx, pool, orgID, pubkey, currentEpoch) error
func LinkWallet(ctx, pool, orgID, pubkey, walletAddress string) error
func ListMembers(ctx, pool, orgID) ([]MemberRecord, error)
func VerifyMemberAccess(ctx, pool, orgID, pubkey, requestedEpoch) (bool, error)
func GetMemberCapabilities(ctx, pool, orgID, pubkey) (canContribute bool, canModerate bool, err error)
func GetMemberRole(ctx, pool, orgID, pubkey) (string, error)
func GetTrialStatus(ctx, pool, orgID, pubkey) (bool, *time.Time, error)
func IsLeader(ctx, pool, orgID, pubkey) (bool, error)
func ListOrgsForMember(ctx, pool, pubkey) ([]MemberOrgEntry, error)
```
**Known issues:** None

#### `internal/moderation/moderation.go`
**Exports:**
```go
func SubmitToQueue(ctx, pool, req, sanitizationFindings []byte) error
  // verifies: Ed25519 sig over submission_hash bytes, SHA256(ciphertext||wrapped_dek) == submission_hash
  // stores ciphertext in ciphertext_hex column (encrypted, opaque)

func GetPendingQueue(ctx, pool, orgID, moderatorPubkey) ([]PendingQueueItem, error)
  // checks: must be leader or have can_moderate capability

func CastApprovalVote(ctx, pool, orgID, submissionHash, moderatorPubkey, vote string) (approveCount int, flagCount int, err error)
func ApproveSubmission(ctx, pool, orgID, submissionHash, moderatorPubkey, memoryType string, vector []float32, embeddingModelID string, embeddingSchemaVersion string) error
func CastKeywordVote(ctx, pool, orgID, submissionHash, keyword, moderatorPubkey, vote string) (includeCount int, excludeCount int, err error)
func GetSubmissionVoteTallies(ctx, pool, orgID string, submissionHashes []string) (map[string]SubmissionVoteTally, error)
func GetKeywordVoteTallies(ctx, pool, orgID string, submissionHashes []string) (map[string]map[string]KeywordVoteTally, error)
func GetModeratorRecommendations(ctx, pool, orgID string, submissionHashes []string) (map[string][]ModeratorRecommendation, error)

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
func (c *QdrantClient) SetRetrievalConfig(vectorNoiseSigma float64, recallDepth uint64)
func (c *QdrantClient) SetPendingDenialDB(db DBQueryer)
func (c *QdrantClient) EnsureCollection(ctx, orgID string, vectorSize uint64) error
func (c *QdrantClient) UpsertPoint(ctx, entry) error
func (c *QdrantClient) QueryPoints(ctx, orgID string, epochs []int32, vector []float32, keywordWeights []protocol.KeywordWithWeight, embeddingModelID string, limit uint64, includeDormant bool, relevanceFloor float64, surfaceBudget int) ([]protocol.MemoryResult, bool, []CandidateScore, error)
func (c *QdrantClient) NearestExistingMemories(ctx, orgID string, vector []float32, limit int) ([]NearDupMatch, error)
func (c *QdrantClient) DeletePointByCID(ctx, orgID, memoryCID string) error
func (c *QdrantClient) UpdateKeywordWeights(ctx, orgID, memoryCID string, weights map[string]float64) error

// Package-level wrappers
func SetRetrievalRanker(r *ProbabilisticRanker)
func AddToIndex(ctx, client, entry) error
func EnsureCollection(ctx, client, orgID string) error
func QueryByKeywords(ctx context.Context, client *QdrantClient, orgID string, accessibleEpochs []int32, keywordWeights []protocol.KeywordWithWeight, vector []float32, embeddingModelID string, limit uint64, includeDormant bool, relevanceFloor float64, surfaceBudget int) ([]protocol.MemoryResult, bool, []CandidateScore, error)
func ScrollOrgMemoryPayloads(ctx, client, orgID) ([]OrgMemoryPayload, error)
func UpdateMemoryKeywords(ctx, client, orgID, oldKeywords, newKeyword) error
func UpdateMemoryState(ctx, client, orgID, memoryCID, lifecycleState) error
func ApplyServeBoostLocal(ctx, db, memoryCID, orgID string) error
func ApplyDenialDecayLocal(ctx, db, memoryCID, orgID string) error
func GetKeywordWeights(ctx, db, orgID, memoryCID string) (map[string]float64, error)
```
**Constants:** `EMBED_DIM = 768`, `contestedThreshold` (see DECISIONS D-9.4)
**Key detail:** `UpsertPoint` calls `injectGaussianNoise`, but stored-vector noise is **DISABLED by default (σ=0)** per D-9.5. `QueryPoints` fetches up to `recallDepth` (default per DECISIONS D-RECALL-MODE-FLAG) candidates, then applies keyword-overlap boost, optimistic pending-denial decay, and new-memory boost, then assigns positions with D-9.4 tempered power-law sampling (strict top-1; positions 2..N sampled without replacement).
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
**Known issues:** `GetContributorStats` currently PREFERS a member's linked wallet address over their Ed25519 pubkey when querying chain reputation — this strands pubkey-earned reputation the instant a wallet is linked. See DECISIONS D-REPUTATION-KEYED-BY-PUBKEY.

#### `internal/billing/billing.go`
**Exports:**
```go
func ProvisionOrgLedger(ctx, pool, orgID string, initialBalance int64, actor string) error
func Subscribe(ctx, pool, orgID, memberPubkey, actor string) error
func GrantFreeRecall(ctx, pool, orgID, memberPubkey, actor string) error
func RevokeRecall(ctx, pool, orgID, memberPubkey, actor string) error
func GetBalance(ctx, pool, orgID) (int64, error)
func TopUp(ctx, pool, orgID, actor, amount) error         // transactional: update balance + record txn
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
> **FORWARD NOTE:** Identity model is being reworked (see DECISIONS D-IDENTITY-PROGRESSIVE-CUSTODY, D-REPUTATION-KEYED-BY-PUBKEY, D-MIGRATION-ONCHAIN-ALIAS, D-LEADER-REQUIRES-WALLET).

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
func VerifyCosmosArbitrarySignature(signerAddress string, message []byte, pubkeyBytes, signatureBytes []byte) error
func ParsePubkeyBytes(pubkeyBase64 string) ([]byte, error)
func ParseSignatureBytes(signatureBase64 string) ([]byte, error)
func BuildCommitCanonicalMessage(orgID, reportID, txHash string) string
func BuildConfigUpdateCanonicalMessage(orgID string, updates map[string]interface{}) string
```
**Key detail:** Verifies signatures produced by Cosmos wallet `signArbitrary` (EIP-712 structured data or plain bytes). Used for report chain commitment (`POST /v1/orgs/{orgID}/reports/{reportID}/commit`); canonical builders emit newline-delimited envelopes (e.g., `wevibe.commit_report.v1` + keyed lines).
**Known issues:** None

#### `internal/verify/canonical.go`
**Exports:**
```go
func CreateOrgMessage(leaderPubkey, leaderX25519Pubkey, orgName, domain, encEnvelope, searchEnvelope, modEnvelope, pkMod string, feeModel protocol.FeeModel) []byte
func InviteMemberMessage(orgID, pubkey, x25519Pubkey, role, signedBy, encEnvelope, searchEnvelope, modEnvelope string, canContribute, canModerate bool) []byte
func RotateEpochMessage(orgID, newPkMod, signedBy string, envelopes []protocol.MemberEnvelopePair) []byte
func RemoveMemberMessage(orgID, pubkey, signedBy string) []byte
func TransferLeadershipMessage(orgID, newLeaderPubkey, signedBy string) []byte
func CloseOrgMessage(orgID, signedBy string) []byte
func SubmitMemoryMessage(orgID string, epochID int, submissionHash string, contributorPubkey string, memoryType string, ciphertextHash string, plaintextHash string, salt string, wrappedDekHash string) []byte
func ApproveSubmissionMessage(orgID, submissionHash string, epochID int32, approvedCID, umbralCapsule, umbralCiphertext, memoryType, signedBy string, keywords []protocol.KeywordWithWeight) []byte
func ApproveSubmissionMessageSimple(orgID, submissionHash string, epochID int32, memoryType, signedBy string) []byte
func DenySubmissionMessage(orgID, submissionHash, reason, signedBy string) []byte
func BanContributorMessage(orgID, contributorPubkey, signedBy string) []byte
func feeModelHash(feeModel protocol.FeeModel) string
func envelopesHash(envelopes []protocol.MemberEnvelopePair) string
func keywordsHash(keywords []protocol.KeywordWithWeight) string
```
**Known issues:** None

#### `internal/embed/embed.go`
**Exports:**
```go
type EmbeddingConfig struct {
    BaseURL string
    APIKey  string
    Model   string
}
func ResolveEmbeddingConfig() (EmbeddingConfig, error)
func GetEmbedding(ctx context.Context, text string) (vector []float32, modelID string, err error)
```
**Key detail:** Embedding config resolves from dashboard config (`WEVIBE_DASHBOARD_CONFIG` or `~/.config/wevibe/dashboard.json`), then calls the selected provider endpoint.  
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
```
#### `internal/db/migrate.go`
**Exports:** `ApplySchema(databaseURL string) error`
**Known issues:** None

#### `internal/hubsign/`
**Role:** Hub response-signature subsystem.
**Core exports:** `hubsign.NewFromEnv()`, `Signer.PublicKeyHex()`, `Signer.SignBody(...)`, `hubsign.SigningMiddleware(...)`.

#### `internal/notifications/`
**Role:** Notification persistence + websocket fanout + channel dispatch.
**Core exports:** `notifications.NewHub()`, `notifications.NewPreferenceStore(...)`, `notifications.NewDispatcher(...)`, `notifications.EmitUserNotification(...)`, plus SMTP/webhook channels.

#### `internal/reports/reports.go`
**Role:** Report domain operations behind handler layer.
**Core exports:** `Create`, `List`, `Get`, `Update`, `GetReportRecommendations`.

#### `internal/serves/serves.go`
**Role:** Serve/denial event persistence + relay bookkeeping helpers.
**Core exports:** `RecordServe`, `RecordDenial`, `GetPendingServes`, `GetPendingDenials`, `MarkSubmitted`, `MarkFailed`, `CountPending`, `HasPendingEvents`.

#### `internal/sanitize/`
**Role:** Plaintext sanitization scanner used during submission processing.
**Core exports:** `sanitize.ScanContent(text)` returning findings for control chars, bidi overrides, homoglyphs, etc.

#### `internal/social/client.go`
**Role:** Hub client for wevibe-social-graph profile/display-name lookups.
**Core exports:** `social.NewClient(...)`, `ResolveNames(...)`, `GetProfile(...)`.

#### `internal/umbral/`
**Role:** Hub-side Umbral sidecar client/service wrapper (gRPC) for PRE kfrag storage + re-encryption.
**Core exports:** `umbral.NewClient(...)`, `umbral.NewService(...)`, `StoreKFrag`, `ReEncryptForMember`, `OnMemberRemoved`, `RemoveOrgKFrags`, `Health`.

---

### Protocol Types

#### `internal/protocol/types.go`
**All types:**
```
OrgInfo, DiscoverOrg, CreateOrgRequest, RotateEpochRequest, EpochManifestResponse
MemberRecord, InviteMemberRequest, RemoveMemberRequest, TransferLeadershipRequest, CloseOrgRequest
LinkWalletRequest, EnableMemberRecallRequest, RegisterPreKeyRequest, MemberPreKeyResponse
KeyEnvelopeResponse, MemberEnvelopePair
SubmitMemoryRequest, SubmitMemoryBatchRequest, SubmitMemoryResponse
Finding, PendingQueueItem, KeywordWithWeight, KeywordCandidate, KeywordMatchDetail, ScoringBreakdown
ApproveRequest, DenyRequest, FeeModel
QueryRequest, MemoryResult, ContributorStats, QueryResponse, RejectRequest
IndexEntry, MemberOrgEntry, MemberOrgsResponse, UsageReceipt
StoreRecoverySharesRequest, RecoveryShareEntry, RecoveryShareResponse
RegisterDashboardKeyRequest, DashboardKeyRecord
CreateReportRequest, EscalationVote, ReportRecommendation, ReportRecord
UpdateReportRequest, VoteOnReportRequest, VoteOnReportResponse, ReportListResponse
```

---

### Database Schema

#### `db/schema.sql`
**Tables:**
```
hub_instance             — singleton hub instance UUID for client reset detection across DB wipes.
orgs                  — PK: org_id. Includes stripe_customer_id, stripe_subscription_id, egress_mode CHECK, status CHECK, rotation_status CHECK, rotation_pending_since, chain_registered, trial_days (CO-266)
org_recall_rate_limits   — per-org recall throttle policy (max_requests/window_seconds).
members               — PK: (org_id, pubkey). FK: orgs. Role CHECK (leader|member). Includes `can_contribute BOOLEAN`, `can_moderate BOOLEAN` (per-member capabilities, chain-mirrored), `pre_pubkey BYTEA`, `is_trial BOOLEAN`, `trial_expires_at TIMESTAMPTZ` (CO-266). ON DELETE CASCADE.
epoch_manifests       — PK: (org_id, epoch_id). FK: orgs. Includes `umbral_pk BYTEA`. ON DELETE CASCADE.
pending_submissions   — PK: submission_hash. FK: orgs. Includes `umbral_capsule BYTEA`, `umbral_ciphertext BYTEA`. Status CHECK (`pending_keyword|pending_chain|committed|denied`), default `pending_keyword`.
extracted_sessions       — contributor session IDs that already completed extraction.
submission_mod_votes     — per-submission moderator approve/flag votes.
keyword_mod_votes        — per-keyword include/exclude votes on pending submissions.
reports                  — report records + resolution state.
report_votes             — per-report votes from moderators/leaders.
rotation_buffer       — PK: buffer_id (gen_random_uuid). FK: orgs. Submissions buffered during rotation_pending state.
usage_receipts        — PK: receipt_id (gen_random_uuid). FK: orgs.
audit_log             — PK: id (BIGSERIAL). FK: orgs.
org_credits           — PK: org_id. FK: orgs ON DELETE CASCADE. balance CHECK (>= 0).
org_extraction_profile   — per-org extraction profile (prompt/ctx/model/preset).
credit_transactions   — PK: txn_id (BIGSERIAL). FK: orgs.
key_envelopes         — PK: (org_id, pubkey). Stores enc/search/mod envelopes per member.
identity_blobs           — passkey identity blob storage keyed by (pubkey, credential_id).
pairing_blobs            — pairing blob storage keyed by pairing_id.
recovery_shares       — PK: (org_id, share_index). Stores sealed Shamir shares.
dashboard_keys        — PK: (org_id, pubkey). Authorized dashboard identities per org.
serve_events             — pending/submitted serve + denial receipts for relay batching.
session_served_memories  — per-session dedupe memory ledger for recall injection.
query_log                — recall query telemetry rows.
query_candidate_scores   — per-candidate scoring/disposition rows for each query.
watcher_state            — chain watcher resume cursor (`chain_watcher` row).
org_keywords          — PK: id. UNIQUE: (org_id, keyword). Declared in `schema.sql` and applied via `db.ApplySchema`.
keyword_candidates       — contributor-suggested non-vocabulary keywords.
memory_keywords      — PK: (memory_cid, keyword). FK: (org_id, keyword) REFERENCES org_keywords.
```
**Indexes:** `idx_orgs_leader`, `idx_orgs_status`, `idx_members_active`, `idx_members_membership_active`, `idx_members_pubkey`, `idx_members_wallet`, `idx_pending_org_status`, `idx_pending_contributor`, `idx_extracted_sessions_contributor`, `idx_submission_mod_votes_sub`, `idx_keyword_mod_votes_sub`, `idx_reports_org_status`, `idx_report_votes_report`, `idx_receipts_org_epoch`, `idx_audit_org_epoch`, `idx_credit_txn_org`, `idx_envelopes_org`, `idx_identity_blobs_credential`, `idx_recovery_shares_holder`, `idx_dashboard_keys_pubkey`, `idx_serve_events_org_status`, `idx_query_log_org_created`, `idx_query_candidate_scores_query`, `idx_org_keywords_org`, `idx_keyword_candidates_org_kw`, `idx_memory_keywords_keyword`, `idx_notifications_recipient_unread`, `idx_join_requests_org_pending`

---

### Test Files Summary

| Test file | Summary | Requires |
|---|---|---|
| `internal/api/handlers/errors_test.go` | Verifies JSON error envelope with/without optional detail fields. | Nothing (in-memory) |
| `internal/api/handlers/health_test.go` | Health endpoint smoke test. | Nothing (in-memory) |
| `internal/api/handlers/keyword_extraction_verify_test.go` | Verifies `/verify-keywords` fail-closed gates and pending_keyword→pending_chain transition persistence. | DATABASE_URL |
| `internal/api/handlers/member_orgs_test.go` | Verifies signed member-org listing auth, timestamp window checks, inactive filtering, and multi-org aggregation. | DATABASE_URL |
| `internal/api/handlers/moderation_test.go` | Verifies submit epoch validation and advisory moderation vote tallies remain advisory-only. | DATABASE_URL |
| `internal/api/handlers/reports_test.go` | Exercises report create/list/get/update/auth paths (including escalation/resolution flows). | DATABASE_URL |
| `internal/api/handlers/serves_test.go` | Verifies serve-relay batching, epoch ordering, tx-size cap flushing, and status marking behavior. | Nothing (in-memory fakes) |
| `internal/auth/header_test.go` | Verifies `WeVibe-Signed` header parser for valid, malformed, and scheme/error cases. | Nothing (in-memory) |
| `internal/billing/billing_test.go` | Covers ledger provisioning/top-up/subscription and transaction history behaviors. | DATABASE_URL |
| `internal/chain/broadcast_test.go` | Validates gas-estimation/retry strategy and invalid gas strategy handling. | Nothing (in-memory mocks) |
| `internal/chain/merkle_test.go` | Validates Merkle root behavior (empty/single/even/odd/deterministic). | Nothing (in-memory) |
| `internal/chain/submit_test.go` | Verifies commit/serve/denial message builders map fields and reject invalid entries. | Nothing (in-memory) |
| `internal/chain/watcher_test.go` | Covers watcher initialization/resume cursor behavior. | Nothing (in-memory mocks) |
| `internal/embed/embed_test.go` | Covers embedding config resolution and retry behavior for transient provider failures. | Config fixtures; optional live provider for live subtest |
| `internal/hubsign/hubsign_test.go` | Verifies deterministic signer derivation + Ed25519 sign/verify roundtrip. | Nothing (in-memory) |
| `internal/hubsign/middleware_test.go` | Verifies response-signing middleware preserves response body/status and emits signature header. | Nothing (in-memory) |
| `internal/members/members_test.go` | Covers invite/get/remove/list/access/leader/member-org lifecycle behavior. | DATABASE_URL |
| `internal/moderation/moderation_test.go` | Covers queue admission checks, vote/deny persistence, and plaintext non-retention guarantees. | DATABASE_URL |
| `internal/orgs/orgs_test.go` | Covers org create/read/exists/leader/epoch lifecycle behavior. | DATABASE_URL |
| `internal/receipts/receipts_test.go` | Receipt creation/signing persistence checks. | DATABASE_URL |
| `internal/retrieval/matched_keywords_test.go` | Verifies per-result matched keyword extraction/overlap handling. | Nothing (mocked Qdrant responses) |
| `internal/retrieval/parity_test.go` | Verifies ranking parity against fixture corpus expectations. | Fixture files only |
| `internal/retrieval/ranking_core_test.go` | Self-test parity for the extracted ranking core. | Nothing (in-memory) |
| `internal/retrieval/ranking_test.go` | Exercises probabilistic ranker determinism/temperature/limit edge cases. | Nothing (in-memory) |
| `internal/retrieval/retrieval_test.go` | Qdrant integration + query behavior tests (including contested threshold and model filter behavior). | Qdrant on localhost:6333 |
| `internal/sanitize/scanner_test.go` | Validates scanner findings for zero-width chars, bidi controls, homoglyphs, and clean text. | Nothing (in-memory) |
| `internal/serves/serves_test.go` | Verifies serve-event persistence and matched-keyword validation. | DATABASE_URL |
| `internal/umbral/client_test.go` | Sidecar integration tests for leader-minted kfrag lifecycle + org purge idempotence. | Built `wevibe-umbral` binary + local sidecar |
| `internal/umbral/service_edge_test.go` | Edge-case tests for service wrappers (reencrypt/store/delete/health/close). | Nothing (stubbed sidecar client) |
| `internal/verify/canonical_test.go` | Verifies deterministic canonical message bodies/hashes and cross-language fee-model vectors. | Fixture vectors (`wevibe-sdk/protocol/test_vectors`) |
| `internal/verify/noble_compat_test.go` | Confirms hub signature verification compatibility with Noble-generated signatures. | Nothing (in-memory) |
| `internal/verify/sig_test.go` | Ed25519 request-signature verification success/failure cases. | Nothing (in-memory) |

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
- 8 custom WeVibe modules + standard SDK modules (staking, auth, bank, gov, slashing, distribution, mint, epochs)
- Module ordering: InitGenesis, ExportGenesis, EndBlockers all explicitly set
- maccPerms includes org module with Burner permission
- Chain foundation pins `github.com/cosmos/cosmos-sdk v0.53.5` and `github.com/cometbft/cometbft v0.38.20` (see `DECISIONS.md` D-S29-SDK-V053)

### Custom Modules (8)

> **FORWARD NOTE:** Landed code state: `x/org` slot registry + ascending-price acquisition auction; `x/emissions` Treasury/`MsgWithdrawTreasury` removed; `x/attestation` disabled-but-wired; `x/bandwidth` memory-cap wired. See DECISIONS D-ECON-STORAGE-MARKET.

| Module | Keeper Path | Proto Path | Tests | Purpose |
|--------|------------|-----------|-------|---------|
| x/attestation | x/attestation/keeper/ | proto/wevibe/attestation/v1/ | keeper + integration | Session-attestation storage (NOT merkle). Disabled/no-op (see DECISIONS D-ATTEST-ROADMAP, D-ATTEST-TEE-TIER, D-ATTEST-PROOF-TIER). |
| x/bandwidth | x/bandwidth/keeper/ | proto/wevibe/bandwidth/v1/ | keeper + integration | Bandwidth throttling |
| x/emissions | x/emissions/keeper/ | proto/wevibe/emissions/v1/ | keeper | Emission pool, epoch emission, work scores |
| x/identity | x/identity/keeper/ | proto/wevibe/identity/v1/ | keeper + integration | Passkey identity management; wallet linking aliasing |
| x/memory | x/memory/keeper/ | proto/wevibe/memory/v1/ | keeper + integration | Memory commitments |
| x/org | x/org/keeper/ | proto/wevibe/org/v1/ | keeper + integration | Org registration, membership |
| x/reputation | x/reputation/keeper/ | proto/wevibe/reputation/v1/ | keeper | Contributor reputation |
| x/serve | x/serve/keeper/ | proto/wevibe/serve/v1/ | keeper + integration | Serve receipts |

- **Design-only (not yet built):** `x/org` `StoredOrg` gains `hub_endpoints` + leader-signed setter (`MsgSetServingInfo` extending `MsgSetServingKey`, or `MsgSetOrgConfig`); proto updates regenerate via Docker `make proto-gen` (never hand-edit `.pb.go`). See D-CHAIN-RESOLVED-HUB-ENDPOINT.

### Genesis Seeding & Epoch Hooks (Sprint 32 / CO-040)

**module.HasGenesis wiring (CO-040).**
`x/emissions` and `x/reputation` implement `cosmos-sdk/types/module.HasGenesis`
(`DefaultGenesis`/`ValidateGenesis`/`InitGenesis`/`ExportGenesis`) in
`module/module.go`; SDK dispatch is via `ModuleManager.InitGenesis`.
See DECISIONS D-S32-HASGENESIS-CUSTOM-MODULES.

**Genesis seeding path.** `wevibed init` builds genesis.json from
`app.ModuleBasics` (`app/encoding.go`); custom modules are absent unless
`app_state` keys are seeded. `scripts/init-chain.sh` jq-seeds:
- `app_state.emissions = {}`
- `app_state.reputation = {"active": true}`
See DECISIONS D-S32-EMISSION-POOL-GENESIS and
D-S32-REPUTATION-DEFAULTGENESIS-ACTIVE.

**Epoch-hook chain.** The epochs module fires `AfterEpochEnd` for
`wevibe_epoch` via MultiEpochHooks: emissions first (mint + payouts), then
memory (`setCurrentEpoch` → `CheckEpochExpiry` → `ApplyEpochDecay` → merkle
roots). See DECISIONS D-S32-EPOCH-HOOK-RESILIENCE.

**cachekv iterator correctness.** Under cache-wrapped stores used by epoch
hooks / BeginBlock, the legacy
`for iter.Valid(){…}; if err := iter.Error(); err != nil { return err }`
pattern at 24 sites (emissions/memory/org/reputation keepers) mis-reads normal
iterator exhaustion as failure. Affected functions include
`ApplyEpochDecay`, `CheckEpochExpiry`, `getAllOrgsWithMemories`, and
emissions `GetAllOrgs`. See DECISIONS D-S32-CACHEKV-ITER.

### Module Structure Pattern (all 8 modules follow this)

```
x/{module}/
├── keeper/
│   ├── keeper.go           # Keeper struct, state access, business logic
│   ├── msg_server.go       # MsgServer implementation
│   ├── grpc_query.go       # gRPC query handlers
│   ├── epoch_hooks.go      # Epoch hooks (emissions, memory only)
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

### Earned-Trust Decay Code Anchors

- `x/serve/keeper/keeper.go` — `GetMatchedKeywordsForEpoch(ctx, orgID, memoryCID, epoch)` returns the matched-keyword set for one memory+epoch; memory decay consumes this as the keyword-activity signal.
- `x/memory/keeper/lifecycle.go` — `applyDecay(...)` is the canonical earned-trust decay function (serve/denial/idle deltas, trust-weighting, clamp/archive behavior).
- `x/memory/keeper/lifecycle.go` — `ApplyEpochDecay(ctx, epoch)` iterates active memories and applies `applyDecay` once per memory at epoch end.
- `proto/wevibe/memory/v1/state.proto` — `StoredMemoryCommitment` is the persisted memory-state proto (including `keywords`, `serve_count_total`, `denial_count_total`, `last_active_epoch`, `state`, `archived_epoch`, and provenance/hash fields).

### Tests

| Test file | Tests | Requires |
|---|---|---|
| tests/integration/wevibe_txs_test.go | 9 tx pipeline tests | In-memory MemDB app |
| tests/integration/wevibe_queries_test.go | 11 gRPC query tests | In-memory MemDB app |
| tests/integration/wevibe_modules_test.go | Cross-module integration/query wiring tests (attestation, bandwidth, memory, serve) | In-memory MemDB app |
| tests/integration/helpers.go | Shared integration harness/bootstrap (`TestSuite`, genesis setup, live msg delivery helper) | In-memory MemDB app |
| x/*/keeper/keeper_test.go | Per-module keeper tests | In-memory |

### Scripts

| Script | Purpose |
|---|---|
| scripts/init-chain.sh | Idempotent genesis init with wevibe_epoch config |
| scripts/smoke-test.sh | RPC health + block production verification |
| scripts/protocgen.sh | Legacy local proto codegen helper script (`buf generate` + copy generated tree) |
| scripts/dev-mnemonics.env | Public test-only dev mnemonic seed(s) for local chain submitter wiring |

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
├── api/
│   ├── extract/route.ts
│   ├── identity/
│   │   └── adopt-local/route.ts
│   ├── lmstudio-models/route.ts
│   ├── ollama-models/route.ts
│   ├── openrouter-embedding-models/route.ts
│   ├── openrouter-models/route.ts
│   ├── org-setup/
│   │   ├── route.ts
│   │   └── finalize/route.ts
│   ├── provision-recall/route.ts
│   ├── sessions/
│   │   ├── route.ts
│   │   └── [id]/messages/route.ts
│   └── settings/
│       ├── route.ts
│       ├── embedding-readiness/route.ts
│       ├── readiness/route.ts
│       └── risk-appetite/route.ts
├── (auth)/
│   ├── connect-wevibe/page.tsx   # Contributor connect (identity adoption) flow
│   └── login/page.tsx            # Login page
├── (dashboard)/
│   ├── layout.tsx                 # Dashboard layout with sidebar/topbar + OrgProvider
│   ├── activity/page.tsx
│   ├── billing/page.tsx
│   ├── buy-org/page.tsx
│   ├── create-org/page.tsx
│   ├── discover/page.tsx
│   ├── discover/[orgId]/page.tsx
│   ├── epoch/page.tsx
│   ├── faucet/page.tsx
│   ├── health/page.tsx
│   ├── join-requests/page.tsx
│   ├── members/page.tsx
│   ├── moderation/
│   │   ├── new/page.tsx
│   │   ├── reported/page.tsx
│   │   └── history/page.tsx
│   ├── my-org/page.tsx
│   ├── my-submissions/page.tsx
│   ├── notifications/page.tsx
│   ├── profile/page.tsx
│   ├── recall-inspect/page.tsx    # Recall Health
│   ├── recovery/page.tsx
│   ├── sessions/
│   │   ├── page.tsx
│   │   └── extracted/page.tsx
│   └── settings/page.tsx
└── u/[wallet]/page.tsx            # Public profile — standalone, no sidebar
```

### `components/`

```
components/
├── layout/
│   ├── mcp-connection-guard.tsx
│   ├── notification-bell.tsx
│   ├── org-switcher.tsx
│   ├── sidebar.tsx
│   ├── tab-nav.tsx
│   └── topbar.tsx
├── memory/
│   └── preference-score-card.tsx
├── moderation/
│   ├── leader-pipeline-panel.tsx
│   └── moderator-review-panel.tsx
├── onboarding/
│   └── identity-onboarding.tsx
├── pairing/
│   └── pair-plugin.tsx
├── sessions/
│   └── memory-review.tsx
├── wallet-connect-button.tsx
└── ui/
    ├── client-time.tsx
    ├── modal.tsx
    ├── searchable-model-combobox.tsx
    ├── spinner.tsx
    ├── states.tsx
    ├── tooltip.tsx
    ├── badge.tsx
    ├── button.tsx
    └── card.tsx
```

### `lib/`

```
lib/
├── types.ts
├── config.ts
├── errors.ts
├── format.ts
├── toast.ts
├── settings.ts
├── settings-defaults.ts
├── nav-config.ts
├── deployment.ts
├── use-dashboard-state.ts
├── mcp-errors.ts
├── hub-error.ts
├── error-remediation.ts
├── mcp-client.ts
├── hub-client.ts
├── social-graph-client.ts
├── org-bridge.ts
├── org-context.tsx
├── org-pricing.ts
├── org-role.ts
├── role-colors.tsx
├── chain-client.ts
├── verify-queue.ts
├── extraction-queue.ts
├── draft-store.ts
├── session-types.ts
├── keyword-weights.ts
├── preference-score.ts
├── provider-readiness.ts
├── canonical-body.ts
├── merkle.ts
├── passkey.ts
├── wallet-connect.ts
├── wevibe-auth.ts
├── wevibe-signing.ts
├── wevibe-crypto.ts
├── wevibe-submit.ts
├── identity-context.tsx
├── sim-benchmark.json
└── __tests__/chain-client.test.ts + merkle.test.ts
```

### `e2e/` — Playwright tests

```
e2e/
├── billing.spec.ts
├── connection.spec.ts
├── leader-member-management.spec.ts
├── leader-org-management.spec.ts
├── leader-settings.spec.ts
├── moderation.spec.ts
├── navigation.spec.ts
├── reports.spec.ts
├── sessions.spec.ts
├── mcp-tools.test.ts
├── global-setup.ts
├── fixtures.ts
├── helpers/
│   ├── mock-hub.ts
│   └── test-data.ts
└── page-objects/
    └── index.ts
```

### `package.json` — Key dependencies
- `next` (framework)
- `react`, `react-dom`
- `@cosmjs/stargate`, `@cosmjs/proto-signing`, `@cosmjs/amino`, `@cosmjs/crypto`, `@cosmjs/encoding`, `cosmjs-types` (Cosmos chain signing + direct broadcast from dashboard)
- `@noble/curves`, `@noble/ed25519`, `@noble/hashes` (crypto)
- `better-sqlite3`, `sonner`, `wevibe-sdk-wasm`
- **devDependencies:** `@playwright/test`, `tailwindcss`

---

## WeVibe/wevibe-mcp — TypeScript MCP Client + HTTP API Server

**Language:** TypeScript
**Purpose:** MCP client for AI agents to interact with WeVibe Network. Also serves an HTTP API on `127.0.0.1:4450` for the OpenCode plugin (CO-244).
**Status (built + committed, `6ceac5d..264a29f`):** plugin onboarding + identity-sidecar/pairing/installer path landed (`D-SIDECAR-PLUGIN-OWNS-STATE`, `D-PLUGIN-ONBOARDING-HOOK`).

### `src/` — Main source

```
src/
├── server.ts
├── config.ts
├── types.ts
├── types/               # Directory (`index.ts`) for shared MCP type exports
├── session.ts
├── crypto.ts
├── crypto-utils.ts
├── pair-crypto.ts
├── pairing-export.ts
├── auth.ts
├── biometric.ts
├── identity-sidecar.ts
├── identity-runtime.ts
├── contribution.ts
├── extraction.ts
├── extraction-presets.ts
├── attestation.ts
├── guard.ts
├── blacklist.ts
├── llm.ts
├── llm-ollama.ts
├── llm-openai-compat.ts
├── llm-sampling.ts
├── embedding.ts
├── embedding-config.ts
├── embed-card.ts
├── retrieval-card.ts
├── moderation.ts
├── umbral.ts
├── vault.ts
├── pending-vault.ts
├── key-store.ts
├── keychain.ts
├── org-client.ts
├── hub-fetch.ts
├── hub-resolver.ts
├── artifact-policy.ts
├── artifact-extract.ts
├── artifact-transform.ts
├── deserialize.ts
├── recovery.ts
├── canonical.ts
├── denial-queue.ts
├── trust-panel.ts
├── manifest.ts
├── retrieve-cli.ts
├── http-server.ts
├── session-token.ts
├── risk-appetite.ts
├── serve-signing.ts
└── admin.ts             # Admin CLI (identity/setup/org moderation/recovery ops)
```

- **Separate plugin repo (NOT under `wevibe-mcp/src/`):**
  - `wevibe-opencode-plugin/plugins/wevibe-plugin.ts` (server/plugin runtime hook)
  - `wevibe-opencode-plugin/tui/wevibe.tsx` (TUI onboarding UI)
  - Installed via `wevibe-opencode-plugin/bin/install-opencode.ts` (`npm run install-opencode`) into `~/.config/opencode/tui/wevibe.tsx` with `tui.json`/`opencode.json` merge.

- **Built path runtime note:** no biometric prompt at process boot (LAZY boot), and PRE membership sync/registration are first-use deferred (`D-SIDECAR-PLUGIN-OWNS-STATE`, `D-PLUGIN-ONBOARDING-HOOK`).

### `tests/`

```
tests/
├── integration/
│   ├── capstone.test.ts
│   ├── e2e-flow.test.ts
│   ├── http-auth.test.ts
│   ├── http-reports.test.ts
│   └── http-serves.test.ts
├── security/
│   ├── attack-scenarios.test.ts
│   └── recall-pipeline.test.ts
├── production/
│   ├── hub-resilience.test.ts
│   ├── sampling-provider.test.ts
│   └── embedding-quality.test.ts
├── embedding.test.ts
├── embedding-config.test.ts
├── embedding-prefix.test.ts
├── server-tools.test.ts
├── sidecar.test.ts
├── contribution.test.ts
├── moderation.test.ts
├── moderation-approval.test.ts
├── retrieval-card.test.ts
├── retrieve-cli-harvest.test.ts
├── extraction.test.ts
├── extract-defaults.test.ts
├── extraction-presets.test.ts
├── artifact-extract.test.ts
├── artifact-policy.test.ts
├── artifact-transform.test.ts
├── deserialize.test.ts
├── llm.test.ts
├── guard.test.ts
├── risk-appetite.test.ts
├── canonical.test.ts
├── manifest.test.ts
├── org-client.test.ts
├── hub-fetch.test.ts
├── hub-resolver.test.ts
├── session-token.test.ts
├── session.test.ts
├── serve-signing-parity.test.ts
├── blacklist.test.ts
├── key-store.test.ts
├── vault.test.ts
├── pending-vault.test.ts
├── pair-crypto.test.ts
├── pair-crypto-reverse.test.ts
├── recovery.test.ts
├── recovery-status.test.ts
├── rotation.test.ts
├── threshold-recovery.test.ts
├── egress-policy.test.ts
├── steg-scan.test.ts
└── wasm-crypto.test.ts
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
├── rules/
│   └── injection.yar # YARA rule source
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
├── fixture_compliance.rs
├── fixtures_good.json
├── fixtures_bad.json
└── fixtures_redteam.json
```

---

## WeVibe/wevibe-sdk — Rust Crypto SDK

**Languages:** Rust  
**Purpose:** Crypto primitives + WASM bindings

### `crates/wevibe-sdk-core/` — Core Rust library

```
crates/wevibe-sdk-core/src/
├── lib.rs           # Library entry
├── crypto.rs        # Cryptographic operations
├── identity.rs      # Identity management
├── secp256k1.rs     # secp256k1 key/signature helpers
├── types.rs         # Type definitions
└── errors.rs        # Error types
```

### `crates/wevibe-sdk-wasm/` — WASM bindings

```
crates/wevibe-sdk-wasm/src/
└── lib.rs           # WASM bindings
```

---

## wevibe-faucet — Dev/Test VIBE Faucet

**Language:** Go  
**Purpose:** Development-mode token faucet for local/testnet VIBE distribution  
**Default port:** 4470 (container), not exposed to host by default (dev-only)

### Entry Point

#### `main.go`
**Role:** Bootstraps config + Cosmos broadcaster, starts HTTP server, and serves the dev/test faucet API.

```
GET  /v1/health — returns {"status":"ok","faucet_address":...,"chain_id":...}
POST /v1/fund   — JSON body {"address":...,"amount":...}; returns {"tx_hash":...,"funded":...}
```

- **Rate limiting:** per-address sliding-window limiter in `internal/faucet/faucet.go` (default **1 request / 60s**), configurable via `RATE_LIMIT_WINDOW_SECONDS` + `RATE_LIMIT_MAX`.
- **Request sizing:** per-request amount is caller-provided (`amount` in JSON body) and must be `> 0`.
- **Rate-limit response:** HTTP `429` with JSON `{ "error": "rate limit exceeded for address ..." }`.

### `internal/`

```
internal/
├── config/
│   └── config.go          # Env loading/defaults
├── faucet/
│   ├── faucet.go          # Cosmos tx build+broadcast + sliding-window limiter
│   └── faucet_test.go     # Idempotency, rate-limit, sequence-retry tests
└── server/
    └── server.go          # HTTP route handlers (/v1/health, /v1/fund)
```

### Config (`internal/config/config.go`)

- `CHAIN_RPC` (default `tcp://wevibed:26657`)
- `CHAIN_ID` (optional; if empty, chain ID is fetched from RPC status)
- `FAUCET_MNEMONIC` (**required**)
- `LISTEN_ADDR` (default `:4470`)
- `RATE_LIMIT_WINDOW_SECONDS` (default `60`)
- `RATE_LIMIT_MAX` (default `1`)

**Auth:** Dev-only endpoint (the hub gates access via `WEVIBE_DEV_ENDPOINTS=1` before forwarding); faucet itself has no auth in dev mode.
**Known issues:** Not a production service; never ships to mainnet.

---

## wevibe-umbral — Umbral PRE Sidecar

**Language:** Rust (edition 2021)  
**License:** GPL-3.0 (kept as an isolated sidecar process for GPL/Apache boundary separation)  
**Crate layout:** single crate with lib `wevibe_umbral` (`src/lib.rs`) + bin `wevibe-umbral` (`src/main.rs`)  
**Default gRPC endpoint:** `127.0.0.1:4460` (`wevibe-umbral serve --addr`; Docker runs `0.0.0.0:4460`)

### `src/`

```
src/
├── lib.rs           # Library exports (service, store, crypto, cli, generated)
├── main.rs          # CLI entry + gRPC server boot/subcommands
├── service.rs       # UmbralSidecarService gRPC implementation
├── crypto.rs        # Umbral serialize/deserialize helpers
├── store.rs         # KFragStore (DashMap-backed, disk persistence)
├── cli.rs           # CLI crypto commands (encrypt/reencrypt/decrypt/keypair/kfrags)
└── generated.rs     # Checked-in prost/tonic generated bindings
```

**KFrag store persistence:** `store.rs` persists the in-memory DashMap to disk at `WEVIBE_UMBRAL_KFRAG_STORE` (default `/data/kfrags.json`).

### gRPC contract

- **Proto:** `proto/umbral/v1/sidecar.proto`
- **Service RPCs:** `StoreKFrag`, `ReEncrypt`, `DeleteKFrags`, `DeleteOrgKFrags`, `Health`
- **CLI-only crypto commands (not RPCs):** `derive-epoch-keypair`, `generate-kfrags`, `encrypt`, `reencrypt`, `decrypt-reencrypted`

### `tests/`

```
tests/
├── epoch_kfrag.rs
├── integration.rs
└── roundtrip.rs
```

### Key dependencies (`Cargo.toml`)

- `umbral-pre`
- `tonic`, `tonic-prost`, `prost`
- `dashmap`
- `clap`

### Docker

- `Dockerfile` builds release binary and exposes port `4460`
- Container command: `wevibe-umbral serve --addr 0.0.0.0:4460`

**Role in WeVibe:** PRE boundary service. The hub owns kfrag lifecycle (store/delete keyed by org/epoch/member) and calls `ReEncrypt` during retrieval.

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

**Recall engine (`plugins/wevibe-plugin.ts`) — recall-mode + session-tie (D-RECALL-MODE-FLAG, 2026-06-21):** the recall/inject engine reads **`WEVIBE_RECALL_MODE`** from `process.env` (`prod` default | `test`); the same flag is read independently by the MCP (`http-server.ts`/`retrieve-cli.ts`) and the hub (`config.go` → `handlers.SetRecallMode` → `recallModeIsTest()` in `retrieval.go`, wired into `docker-compose.yml` as `WEVIBE_RECALL_MODE: ${WEVIBE_RECALL_MODE:-prod}`). Mode selects prod/test governor defaults (see DECISIONS D-RECALL-MODE-FLAG) with hub throttles behavior wired accordingly (trial-EXPIRY still enforced). The plugin no longer mints a process-global random-hex session id (see D-SESSION-SERVE-DEDUP): it captures OpenCode's real `sessionID` from the `chat.message` and `experimental.chat.system.transform` hook inputs, threads it to `/v1/recall` + `/v1/serves`, and gates injection through a per-session `sessionInjectedCids` set so each memory is injected **once per session** (not every turn). Every injection is logged (`[inject] <ISO> sid=… injected N: …`). In `test` only, persisted Earned-Trust auto-accept is disabled so every recalled candidate re-enters the review gauntlet and is re-counted.

---

## wevibe-social-graph — Public Profile Service

**Language:** Go  
**Purpose:** Serves public profile CRUD + contributor stats APIs. Persists profiles in a local SQLite store and enriches contributor stats from chain REST.

**HTTP routes (`internal/server/server.go`):**
- `GET /v1/health`
- `POST /v1/profiles`
- `GET /v1/profiles/batch?wallets=...`
- `GET /v1/profiles/{wallet}`
- `PATCH /v1/profiles/{wallet}` (requires Cosmos `signArbitrary` proof verified in `internal/server/signature.go`)
- `GET /v1/stats/contributor/{pubkey}`

### Structure

```
cmd/server/
└── main.go               # HTTP server entry point; wires SQLite store + chain client
internal/
├── server/
│   ├── server.go         # Route registration + handlers
│   ├── signature.go      # Cosmos arbitrary-signature verification for PATCH auth
│   └── signature_test.go
├── store/
│   └── store.go          # SQLite-backed profile persistence (`profiles` table): Create/Get/Update/ListBatch
└── chain/
    └── client.go         # Chain REST client: GetContributorStats + GetContributorReward
```

**Default port:** 4470 inside Docker (`wevibe-social-graph:4470`); hub config `SocialGraphURL` points here.
**Storage:** SQLite file path from `SOCIAL_GRAPH_DB_PATH` (default `/data/social-graph.db`).
**CORS/auth:** CORS middleware wraps all routes; reads are public, while profile PATCH requires wallet ownership proof.

---

## WeVibe/protocol — Protocol Definitions

```
wevibe-protocol/
├── openapi.yaml           # OpenAPI specification
├── README.md
├── contract_test.sh
├── test-vectors/          # Protocol test vectors
├── codegen/               # Proto/codegen helpers
├── js/                    # Generated JS protocol bindings
├── docs/                  # Protocol docs
├── buf.gen.yaml           # Buf generation config
└── package.json
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

## wevibe-sim/recall-sim/ — Recall-Alignment Simulation Suite

**Language:** Node.js (ES modules)  
**Purpose:** Offline, decision-grade validation of recall/extraction quality (DECISIONS.md `D-RECALL-ALIGNMENT`). Lives under `wevibe-sim/` (NOT a git repo). Mirrors the REAL extract→keyword→embed→rank pipeline and reads the REAL shipping extraction prompts from `wevibe-mcp/prompts/`. The ranker (`pipeline/rank.mjs`) is an exact mirror of the hub `retrieval.go` scoring (cosine + γ keyword boost + keyword gate + power-law sampling + new-memory boost + denial decay); `pipeline/retrieve-c3.mjs` replicates the C3 (full-proposal) cell. Embeddings use local `nomic-embed-text` (768-d), identical to production.

### Structure

```
recall-sim/
├── config.mjs            # config root: models, scale, watchdog caps, ablation CELLS
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

## Dependencies Summary

| Package | Language | Direct Deps |
|---|---|---|
| wevibe-hub | Go | chi/v5, google/uuid, pgx/v5, qdrant/go-client |
| wevibe-dashboard | TypeScript | next, react, tailwindcss, @radix-ui/*, playwright, @cosmjs/stargate, @cosmjs/proto-signing, cosmjs-types |
| wevibe-mcp | TypeScript | (many npm packages) |
| wevibe-guard | Rust | yara-x |
| wevibe-sdk | Rust | (core crypto) |

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
- See DECISIONS D-ECON-CANON.

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
- Leader revenue path: org demand leg (not yet built). See DECISIONS D-ECON-CANON / D-ECON-STORAGE-MARKET.
- See DECISIONS D-ECON-CANON, D-S32-TOKENOMICS-LOCKED.

### Layer 5 — Future Pluggable Attestation (post-mainnet roadmap)

- Separate components plug into the chain to validate claims cryptographically
  or via API session claims (for example: "user X, model Y, N turns, problem
  Z").
- This is the planned evolution of whitepaper §3.10 Session Attestation +
  §3.11 Difficulty Scoring.
- Enhancement target for the social/economic layers remains TBD.
- Infrastructure is not yet present; this is a major roadmap item.
- **Proposed generalization (`D-ATTEST-PROOF-TIER`, PENDING-SPIKE):** the
  socket generalizes the whitepaper Tier-0/1/2 grades into a single typed
  proof artifact (`proof_type`: tee_receipt / zktls_proof / zkml_proof),
  each verified off-chain before on-chain anchoring. The CO-282 spike proved
  the zkTLS path real for closed frontier models with privacy intact, but a
  ~6 KB asymmetric MPC sent-cap defers it (defer-and-keep-warm); see
  `wevibe-meta/workspace/spikes/zktls-attestation/RESULTS.md` and
  `DECISIONS.md` `D-ATTEST-PROOF-TIER`.
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
4. Epoch poller (`SyncEpochData`) runs on a ticker, loads orgs with committed memories from PostgreSQL (`pending_submissions.status='committed'`), compares Qdrant payload vs chain `GetMemoriesBatch`, and updates changed lifecycle state values.
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

**Denial Receipt Flow (CO-225; consumer loop finalized 2026-05-25 per D-2026-05-25-A):**
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
       │    org_id, epoch_id, memory_content_hash, serve_key_pubkey, serve_sig, serve_fingerprint, nonce, reason).
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
Dashboard denial-batch panel (on /chain-submit) — **DEAD/REMOVED UI stage**
       │
       │ `app/(dashboard)/chain-submit/page.tsx` is gone, and there is no
       │ in-dashboard MsgSubmitDenialBatch submit path.
       │
       │ Hub + chain pieces remain live:
       │   GET /v1/orgs/{orgID}/denials/pending (hub)
       │   MsgSubmitDenialBatch handler (chain)
       │
       │ Settlement currently requires a non-dashboard submitter path.
       ▼
wevibe-chain x/serve MsgSubmitDenialBatch handler
       │
       │ Per accepted denial entry:
       │   StoredDenialReceipt persisted (keyed org_id / epoch_id / memory_hash)
│   Calls memoryKeeper.ApplyEarnedTrustDecay (D-4.2): updates per-keyword
        │     weight using denial_rate-scaled decay; transitions to
        │     MEMORY_STATE_ARCHIVED if all weights ≤ retrievalThreshold (see DECISIONS D-4.2).
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
- `x/memory`: Earned Trust decay (see DECISIONS D-4.2 for params/rationale); archives a memory when all keyword weights ≤ retrievalThreshold.
- `x/serve`: `MsgSubmitDenialBatch`, `StoredDenialReceipt` (keyed org/epoch/memory-hash)
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

# RECALL → INJECTION PIPELINE — COMPLETE MAP & DEFECT LEDGER

> Charted against live code; every claim is `file:line`-cited — treat citations as load-bearing.
>
> Line citations approximate as of 2026-06-21; structure verified, exact lines drift with edits.
>
> **Layer ownership:** Stage 1 = `wevibe-hub` (Go) · Stage 2 = `wevibe-mcp` (TS) · Stage 3 = `wevibe-opencode-plugin` (TS) · Stage 4 = `opencode` runtime (vendored binary).
>
> **✅ Resolved 2026-06-21 (Phase 2 prune):** workspace re-check confirmed `UpdateMemoryKeywords` is live (`internal/api/handlers/keywords.go:306,400`) and dead `ScrollApprovedMemories` was removed.

## Pipeline at a glance (4 stages, root → leaf)

```
USER PROMPT (opencode session)
   │  plugin hook chat.message  (wevibe-plugin.ts:959)
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

**Files:** `internal/retrieval/{retrieval.go (1768L), ranking_core.go (246L), querylog.go, stats.go}`, `internal/api/handlers/{retrieval.go (614L), pool.go}`, `internal/config/config.go`, `cmd/wevibe-hub/main.go`, `internal/protocol/types.go`.

**Call chain (verbatim hops):**
1. `handlers.QueryMemories` (`handlers/retrieval.go:27`) — parse `QueryRequest`; require orgID/agentPubkey/prePubkey; default limit uses prod/test governor defaults (see DECISIONS D-RECALL-MODE-FLAG) (`:64-68`); membership (`:76`); trial+daily gate (`:83-122`); rate limit from `org_recall_rate_limits` (`:124-152`); resolve epochs (`:154-164`).
2. `retrieval.QueryByKeywords` (`retrieval.go:990`) — pure passthrough to `client.QueryPoints(...)`.
3. `QdrantClient.QueryPoints` (`retrieval.go:419`) — build Qdrant filter (org match, `must_not` ARCHIVED, optional DORMANT); `POST {restURL}/collections/{collection}/points/search` (`:478`); search limit = `recallDepth` (5000); dedup by CID; load pending-denial counts from PG `serve_events` (`:545`); build `[]RankCandidate` filtered to authorized epochs (`:578-640`).
4. `ScoreAndRank` (`ranking_core.go:162`) — the pure scoring engine (math below).
5. **Relevance floor + surface budget** (`retrieval.go:701-718`) — filter `weightedScore >= relevanceFloor`, then `cap = min(limit, surfaceBudget)`.
6. **D-9.4 power-law sampler** `probabilisticRank` (`retrieval.go:125-215`) — position 1 = strict argmax; positions 2+ sampled w/o replacement, weight `(score/maxScore)^(1/temp)`, `temp` (see DECISIONS D-9.4).
7. Contested flag (`retrieval.go:767`, `contestedThreshold` (see DECISIONS D-9.4)).
8. Back in handler (`handlers/retrieval.go:183-416`) — async query-log persist; **session dedup: drop CIDs served in last 24h for same `session_id`** (`:185-208`); chain attest `GetMemoriesBatch` (`:241`); Umbral `ReEncryptForMember` from `pending_submissions` (`:257-346`); banned filter (`:348-370`); contributor stats (`:372-393`); receipt (`:395-405`); emit `QueryResponse` (`:409-416`).

**Scoring math (`ranking_core.go`):** `keywordScore = Σ(queryWeight[kw]·memoryWeight[kw])` (`:102-136`); `gammaBoost = keywordScore·keywordBoostFactor` (`:187`, `keywordBoostFactor` at `retrieval.go:450-451`, value per DECISIONS D-9.4); `cappedBoost = min(gammaBoost, keywordBoostDelta·vectorScore)` (`:188-194`, `keywordBoostDelta` at `retrieval.go:450-451`, value per DECISIONS D-9.4); `final = vectorScore + cappedBoost` (`:195`); pending denial `final = max(0, final − denials·DenialDecayBPS/10000)` (`:197-199`, `DenialDecayBPS` value per DECISIONS D-4.2); new-mem boost `final ·= 1 + ServeBoostBPS/10000·max(0, 1−age/(grace+window))` (`:201-208`, `ServeBoostBPS` value per DECISIONS D-4.2); sort by `final` desc. Constants: `keywordBoostFactor` (`retrieval.go:450-451`, value per DECISIONS D-9.4), `keywordBoostDelta` (`retrieval.go:450-451`, value per DECISIONS D-9.4), `EMBED_DIM=768`, `recallDepth` (value per DECISIONS D-RECALL-MODE-FLAG), `DenialDecayBPS` (value per DECISIONS D-4.2), `ServeBoostBPS` (value per DECISIONS D-4.2).

**Structs (`internal/protocol/types.go`):** `QueryRequest` (`:268`: org_id, agent_pubkey, pre_pubkey, keyword_weights, vector, embedding_model_id, limit, session_id, include_dormant, relevance_floor, surface_budget, agent_sig); `MemoryResult` (`:284`); `ScoringBreakdown` (`:226`: keyword_score, vector_score, gamma, delta, capped_boost, combined_score, keyword_matches, unmatched_query_keywords); `QueryResponse` (`:313`: results, contested, receipt_id, requires_reencryption).

**Recall mode:** `config.go:48` reads `WEVIBE_RECALL_MODE` (default prod); `main.go:75` `SetRecallMode`; `pool.go:33` `recallModeIsTest()`. **Prod vs test differs ONLY in throttles** (prod/test governor defaults, see DECISIONS D-RECALL-MODE-FLAG; trial+rate-limit bypassed `handlers/retrieval.go:64-124`). Scoring/floor/budget/sampler are mode-independent.

**Qdrant layer:** pure HTTP REST client (no gRPC/SDK), `QdrantClient` (`retrieval.go:242`); **new `http.Client` per request, 10s timeout** (no keep-alive/pooling).

**Stage-1 dead/cruft — ✅ PRUNED 2026-06-21 (Phase 2a, retrieval.go 1767→1526, −241L; hub `go build ./...` + `go test ./...` green):**
- **REMOVED (re-verified truly dead, zero workspace callers):** `computeKeywordScore`, `applyPendingDenialDecay`, `applyNewMemoryBoost` (method), `CountPoints`, `ScrollApprovedMemories` — plus dead collateral `ErrInvalidOffset`, unused imports (`encoding/base64`, `errors`), dead const `MaxServesPerEpoch`, and the dead-only test `retrieval_optimistic_test.go` + dead `applyNewMemoryBoost` cases in `ranking_test.go`.
- **REMOVED dead `MemoryResult` fields:** `ConfidenceBps`, `RetrievalCount`, `WrappedDekEnc` (`internal/protocol/types.go`).
- **KEPT — gather was WRONG (these have live callers, NOT dead):** `GetKeywordWeights`, `ApplyServeBoostLocal`, `ApplyDenialDecayLocal` (← `internal/chain/watcher_serve.go`); `ScrollOrgMemoryPayloads`, `UpdateMemoryState` (← `internal/chain/sync.go`); `UpdateMemoryKeywords` (← `internal/api/handlers/keywords.go`).
- Still present (live, unchanged): `Gate: false` hardcoded (`retrieval.go:643`); Gaussian noise `sigma=0.0` no-op + index-time only; `QueryRequest.EmbeddingSchemaVersion` unused (request field).

---

## Stage 2 — MCP recall MIDDLE (`wevibe-mcp`, TS)

**Endpoint:** `POST /v1/recall` (`http-server.ts:953`) → `handleRecall` (`http-server.ts:231`); Bearer-token auth (`authorize()` `:105`).

**Call chain:**
1. `handleRecall` (`http-server.ts:231`) — authorize; `flushDenials()` fire-and-forget (`:236`); parse body, apply governor defaults for limit/relevance_floor/surface_budget (`:239-256`); require `query` (`:263`); call `retrieve(input)` (`:281`).
2. `retrieve()` (`retrieve-cli.ts:262`) — `initCrypto` → `ensureIdentity` (lazy biometric, registers PRE pubkey) → `loadMemberships` (`org-client.ts:240`) → select org → `getActiveHubUrlForOrg` → `buildQueryHarvest` (`:188`) → `buildNeedCard` (`retrieval-card.ts:84`) → `dissect_to_keywords` → `computeLocalEmbedding` (`embedding.ts`) → **`queryOrgMemories`** (`org-client.ts:121`, POSTs `/v1/orgs/{orgId}/query`) → `deserializeMemoryResult` (`deserialize.ts:56`).
3. **Per-memory decrypt loop** (`retrieve-cli.ts:386-483`) — fetch ciphertext via `hubFetchVerified` `GET /v1/orgs/{orgId}/memories/{cid}` (`:392`); `decryptMemoryBlob` (`org-client.ts:403`): `getOrCreatePreIdentity` → `getEpochUmbralPk` → **`umbralDecryptReencrypted`** (`umbral.ts`) → `decryptSymmetric(ciphertext, dek)` (AES); then `extractArtifacts` / `checkArtifactPolicy` / `transformMemoryContent` / `formatTrustPanel`; build `MemoryOutput`.
4. Back in `handleRecall` — **`runWeVibeGuard`** per memory (`http-server.ts:302`); provider-policy check (`:321-328`); emit `{status:'ok', memories:[…], reason_code?}` (`:354`).

**Decrypt + guard mechanics:**
- **Umbral = in-process WASM call** (`umbral.ts`), loaded from `vendor/umbral-wasm` relative to the package — **no environment variable, no binary, no subprocess**. Compiled from `wevibe-umbral/crates/core`, the same source as the native binary, so ciphertext stays byte-compatible in both directions. PRE secret key stored in OS keychain via `keytar` (`auth.ts:47`).
  - Superseded the `execFile` sidecar in 0.3.0. The old path required `WEVIBE_UMBRAL_SIDECAR_BIN` at every launch site and broke three times (2026-07-05, 07-13, 08-14) when a launch script dropped it. It also passed secrets as argv, where `ps` could read them.
- **wevibe-guard = `spawnSync`** (`guard.ts:43`); binary from `WEVIBE_GUARD_BIN` or relative fallback (`guard.ts:19-29`); JSON stdin → `{passed, detections, flags}`.
- **Guard does NOT block** — failing memories are still returned with `guard.passed=false` attached (`http-server.ts:314-318`); blocking is delegated to the plugin.
- **Decrypt failure silently skips the memory** (`retrieve-cli.ts:477-482` `continue`); only if ALL fail does `reason_code:'decrypt_failed'` surface. Partial loss is invisible to the caller.

**Types:** `RetrieveInput` (`retrieve-cli.ts:19`), `MemoryOutput` (`retrieve-cli.ts:41`), `MemoryWithGuard` (`http-server.ts:115`, adds `guard{passed,detections,flags}`), `ScoringBreakdown` (`types.ts:35`), deserialized `MemoryResult` (`types.ts:46`). **No `blocked` and no `source` field exists** anywhere in MCP types. `MemoryType = 'memory'` single value (`types.ts:99`, D-5.1).

**Recall mode:** `getRecallMode()` (`retrieve-cli.ts:93`); `RECALL_MODE_GOVERNORS` (`:80`) uses prod/test governor defaults (see DECISIONS D-RECALL-MODE-FLAG); used as request defaults in `handleRecall` (`http-server.ts:239`).

**Stage-2 dead/cruft:**
- `agentSig` — **✅ REPLACED 2026-06-21 with real request-body signing.** The dead `agent_sig` body field is gone; the MCP now signs the exact serialized request body with the agent Ed25519 key and sends it in header `X-Agent-Signature` (`org-client.ts` queryOrgMemories); the hub reads raw body bytes, `ed25519.Verify` against the middleware-authenticated pubkey, **401 on missing/invalid**, then unmarshals (`handlers/retrieval.go`), and stores the verified sig in `usage_receipts.agent_signature` (now meaningful, no DB migration). Hub `go build/test` + MCP tsc green. **⚠ WIRE-CONTRACT CHANGE: hub + MCP must be redeployed together** (old MCP → new hub = 401).
- **✅ PRUNED 2026-06-21 (Phase 2b, server.ts 663→388, −275L; tsc green):** removed the dead old-MCP-tool recall graveyard — `recallTimeScan`, `gateMemories`, `rerankByRelevance`, `disambiguateMemories`, `buildElicitationPreview`, `formatMemoryPresentation`, `FormattedMemory` type, `ALLOW_UNREVIEWED` — plus now-unused imports (`runWeVibeGuard`, `MemoryResult`, `getLlmProvider`) and dead-only test cases in `tests/security/recall-pipeline.test.ts` + `tests/server-tools.test.ts`. (Pre-existing unrelated failures remain in `tests/sidecar.test.ts`: "Invalid SecretKey" — NOT caused by the prune, verified.)
- **Risk appetite (consumer filter):** LIVE via dashboard settings page + TUI `/wevibe-risk` → `~/.wevibe/plugin-config.json` → plugin `getRiskAppetite()` filters at injection. **✅ 2026-06-21: removed the vestigial MCP path only** (`wevibe_set_risk_appetite` tool + MCP `getRiskAppetite/setRiskAppetite`); kept `getProviderPolicy/setProviderPolicy` (live) and the dashboard/TUI/plugin path (the real one).
- **✅ 2026-06-21: `loadMemberships` response now verified** — added `hubFetchVerifiedWithKey` (shared verify logic) + cached hub-level `response_pubkey` from `GET /v1/hub/serving-address`; `loadMemberships` no longer uses raw `fetch`. Caveat: the hub-level key is self-reported (not chain-anchored like org keys) — acceptable for the membership list; stronger anchoring is future work.
- Guard scan passes empty keywords+metadata (`http-server.ts:302`). *(still open)*

---

## Stage 3 — Plugin recall + inject LEAF (`wevibe-opencode-plugin/plugins/wevibe-plugin.ts`, ~1147L + `tui/wevibe.tsx`)

**Hooks returned:** `chat.message` (`:959`), `experimental.chat.system.transform` (`:983`), `experimental.session.compacting` (`:1122`).

**Recall trigger chain:**
- **Prewarm IIFE** (`:930-946`) at plugin load — `getRecallMode`, `ensureWeVibeMcpRunning`, `loadMemories(queryToUse)` where `queryToUse` is project-derived (`:898-929`, fallback `"project coding standards conventions best practices"`). **`activeSessionId` is null here → `currentSessionId()` returns `"prewarm"`** (`:290`) → recall sent with `session_id:"prewarm"`.
- `chat.message` (`:982`) → `triggerRecall` (`:891`) → `loadMemories` (`:667`). **Single-flight: if a recall is in-flight, the new one is silently dropped** (`:894`).
- `loadMemories` (`:667`) — cache check (5min TTL); `getRecallGovernorConfig()`; `POST 127.0.0.1:4450/v1/recall` with `{query, limit, session_id, relevance_floor, surface_budget}` (`:698`); clear+rebuild `cachedMemories`/`memoryIndex` (`:754`); per memory build `CachedMemory`, auto-deny if guard-blocked, **test-mode AUTO-APPROVE → `approvedCids.add(cid)`** (`:816-819`, *Phase 1 2026-06-21 — was a delete that forced re-popup*), enqueue undecided candidates for prod popup (`:856`, comment "Hub governs… no client-side re-governing" `:854`).

**State model (`:276-281`):** `approvedCids`, `deniedCids`, `reportedCids`, `pendingCids`, `sessionInjectedCids: Map<sid,Set>`. **Init gate (`:311`): `if (getRecallMode() !== "test") load accepted` — test mode starts with empty approvals.** Files in `.opencode/`: `wevibe-plugin-status.json` (accepted/denied/reported, written by `recordStatusSnapshot` `:370`), `…-queue.json`, `…-decisions.json`, `wevibe-tui-active.json` (heartbeat). Plus `~/.wevibe/blacklist.json` (`seedDeniedFromLocalBlacklist` `:292`, called at init AND every transform `:1027`).

**Injection mechanism — `experimental.chat.system.transform` (`:1006-1143`, Phase 1 2026-06-21):** (1) await in-flight recall ≤15s (`:1009`); (2) `drainDecisions` + reseed blacklist (`:1026`); (3) compute pending-undecided (`:1029`); (4) **TUI gate wait loop ONLY `if (isTuiLive())`** ≤5min, 250ms poll (`:1044-1061`); (5) **eligible filter requires `approvedCids.has(cid)`** (`:1070-1074`); early-return only if `eligible.length===0` (`:1077-1083`); (6) **EVERY-TURN PUSH: build `memoryBlock` from ALL `eligible` and `output.system.push` it every turn** (`:1088-1103`) — the SOLE injection point, fixes the once-per-session DOA; (7) header is **mode-aware** — test = honest ("you may acknowledge these team memories…"), prod = covert ("Do not mention WeVibe Network…") (`:1092-1094`); (8) **toast** in test mode when `newlyServed.length>0` via `client.tui.showToast` (`:1112-1121`); (9) **serve receipt once per session**: `newlyServed = eligible.filter(!injectedSet.has(cid))`, fire `/v1/serves` + `injectedSet.add` ONLY for those (`:1123-1143`).

**Popup gate:** `isTuiLive()` heartbeat <30s (`:353`); TUI writes heartbeat /10s, polls queue /5s (`wevibe.tsx:1004/1019`); `recordDecision` appends to decisions file (`wevibe.tsx:416`); `drainDecisions` (`:379`) maps accept→approved / deny→denied / block→denied+blacklist+hub denial / report→reported+hub report.

**Guard-failed memory handling (✅ 2026-06-21):** guard-FAILED memories are no longer silently auto-denied — they are surfaced in the approval popup with **Report as the default-selected action** and **Accept moved to the end** (deliberate navigation), Deny/Block unchanged (TUI `wevibe.tsx` builds the action order conditionally on the guard-flagged check; `a/d/b/r` shortcuts resolve by label). Plugin: guard-failed memories enqueue (not auto-deny) and count as pending-undecided; an explicit **Accept overrides the guard block** (the inject filter now gates on approval, not `blocked`); **guard-failed never auto-approves even in test mode**.

**Headless vs TUI behavior (load-bearing; Phase 1 2026-06-21):**

| Scenario | Prod | Test |
|---|---|---|
| **TUI live** | gate waits for accepts → approved memories inject, re-pushed every turn | auto-approved on recall → inject + toast, re-pushed every turn |
| **No TUI (headless `opencode run`)** | only memories approved in a PRIOR session (loaded from status file) inject; NEW memories never inject (popup-gated, by design) | **auto-approve → injects with no popup** (`:816` add), toast confirms; fixes the prior "nothing ever injects" headless/test failure |

**Stage-3 dead/cruft:** **✅ PRUNED 2026-06-21 (Phase 2c, tsc green):** removed `contextPaths` (populated-never-read) + the now-empty `tool.execute.before` hook that only fed it, and `memoryIndex` (populated-never-read). Still present: compaction filter omits `deniedCids`+appetite, inconsistent with inject filter (`:1138`); `"prewarm"` `sessionInjectedCids` entry never cleaned; 3 TUI render harnesses duplicate `parseRetrievalCard`/theme (`tui/_render_*.tsx`). **CORRECTION (2026-06-21): the client-side governor (`PROD/TEST_RECALL_GOVERNOR_DEFAULTS` + `getRecallGovernorConfig`) is NOT vestigial** — it is the SOURCE of `relevance_floor`/`surface_budget`/`limit` sent to MCP→hub, intentionally duplicated across plugin/MCP/hub per D-RECALL-MODE-FLAG. The `:854` "hub governs" comment means "don't RE-filter client-side after the hub returns," NOT that the governor params are dead. Likewise plugin `getRiskAppetite()` (`:124`) is LIVE (filters at injection); the dashboard settings page + TUI dialog are its real consumer UIs (write `~/.wevibe/plugin-config.json`). Only the **MCP** `wevibe_set_risk_appetite` tool + MCP `getRiskAppetite/setRiskAppetite` are vestigial (a 3rd redundant path; D-RISK-APPETITE-UI: tool "stays registered but is NOT the runtime path").

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

## OVERHAUL DIRECTION

Defect-ledger pointers (code-map only):
1. Injection persistence / serve-dedup split is mapped at `wevibe-plugin.ts:1088-1143`; see D-SESSION-SERVE-DEDUP.
2. Test-mode auto-approve + non-blocking toast path is mapped at `wevibe-plugin.ts:816-819,1088-1121` and `tui/wevibe.tsx:251-257,1093`.
   - **Toast surface (charted 2026-06-21):** `client.tui.showToast({ body: { title, message, variant: "info"|"success"|"warning"|"error", duration } })` (SDK `@opencode-ai/sdk/dist/gen/sdk.gen.d.ts:328-402`, types `types.gen.d.ts:3264-3286`, HTTP `POST /tui/show-toast`). Plugin usage is mapped at `wevibe-plugin.ts:1088-1096`.
3. Injected-block framing is mode-aware at `wevibe-plugin.ts:1092-1094`.

Design rationale lives in DECISIONS / session reports.

## DOA ROOT-CAUSE — "[inject] logged but the model saw nothing"

Defect ledger (code-map pointers):
1. Per-session gating path around `sessionInjectedCids` and serve loop (`wevibe-plugin.ts:1086-1143`).
2. Injected-block frame text behavior (`wevibe-plugin.ts:1092-1094`).
3. Headless/TUI gating path (`wevibe-plugin.ts:311,816-819,1044-1074`; `tui/wevibe.tsx:1004,1019`).
4. Silent partial drops upstream (`retrieve-cli.ts:477-482`; `http-server.ts:314-318`).
5. Prewarm/single-flight race path (`wevibe-plugin.ts:894,930-946`).

Design rationale lives in DECISIONS / session reports.

## OPERATIONAL NOTE — kfrag store persistence (FIXED 2026-06-21)

`KFragStore` is now **disk-backed** (not ephemeral-only): on `new()` it loads persisted entries and on `insert`/`delete`/`delete_org` it persists atomically (`temp → chmod 0600 → write → fsync → rename → chmod 0600`). Store path is `WEVIBE_UMBRAL_KFRAG_STORE` (default `/data/kfrags.json`). Docker wiring includes `WEVIBE_UMBRAL_KFRAG_STORE=/data/kfrags.json` plus volume `wevibe_umbral_kfrags:/data`.

**Operational implication:** normal sidecar restarts no longer wipe kfrags. The failure mode now is an empty store on first boot/new volume/corrupt file (or explicit `docker compose down -v`), which still causes re-encryption misses:
```
[recall] umbral ReEncrypt FAILED … member=… : kfrag not found in sidecar
[recall] umbral re-encryption complete reencrypted=0 requiresReencryption=N total=N
```

**Non-destructive recovery (re-mint + StoreKFrag, preserves corpus + chain):**
```
TOKEN=$(cat ~/.wevibe/mcp-session-token)
curl -s -X POST http://127.0.0.1:4450/v1/provision-recall \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"org_id":"wevibe-org-0"}'   # → {"status":"ok"}
```
`provisionRecall` (`wevibe-mcp/src/org-client.ts:644`) derives epoch keys from leader material and re-stores the member kfrag via hub StoreKFrag. Also exposed as `wevibe-admin provision-recall --org <id>` and the dashboard members-page button. `make dogfood` (`docker compose down -v`) still intentionally wipes volume-backed kfrags/corpus/chain state.
