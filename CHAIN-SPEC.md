# WeVibe Chain — Complete Surface & Integration Specification

**Version:** 1.0 (as-built, code-verified 2026-07-18)
**Status:** Definitive map of the chain: everything it has, everything any service can consume, the economic system, the execution model, and the plug-in contract for auxiliary and future services.

---

## §0 How to read this spec

Every claim in this document carries one of three provenance tiers, marked inline:

| Tier | Meaning | Marker |
|---|---|---|
| **AS-BUILT** | Verified directly in source code today (protos, keepers, app wiring). Authoritative. | *(built)* |
| **CANON** | Locked in `DECISIONS.md` / `WP-DESIGN-SPEC.md` / `ROADMAP.md` but not yet in code. The design contract the code is growing toward. | *(canon)* |
| **ROADMAP** | Designed-and-deferred; not solidified until mainnet. | *(roadmap)* |

**Primary sources (as-built):** `wevibe-chain/proto/wevibe/**` (wire surface), `wevibe-chain/x/**` (keepers — behavioral truth), `wevibe-chain/app/app.go` (wiring), `wevibe-chain/cmd/wevibed/cmd/root.go` (runtime config).
**Canon sources:** `wevibe-docs/WP-DESIGN-SPEC.md` (whitepaper copy), `wevibe-docs/ROADMAP.md` (token-economic model), `wevibe-docs/DECISIONS.md` (locked decisions, D-* references throughout).

> ⚠ **The in-repo chain docs are partially stale.** `wevibe-chain/docs/{WHITEPAPER,MODULES,ARCHITECTURE,PARAMETERS,API}.md` describe an older design (confidence-bps lifecycle, contest mechanics, org treasuries, tier payouts) that does **not** match the shipped code. This spec supersedes them. §9 is the full divergence register.

---

## §1 Platform identity & runtime

### 1.1 Stack *(built)*

| Property | Value |
|---|---|
| Framework | Cosmos SDK **v0.53.5** + CometBFT **v0.38.20** (locked: D-S29-SDK-V053 — do NOT bump to v0.54.x, store/v2 breaks x/upgrade) |
| Language / toolchain | Go 1.26 (Docker `golang:1.26-bookworm`) |
| Binary | `wevibed` (app name `wevibed`, home `~/.wevibed`) |
| Chain ID (local/dev) | `wevibe-local-1` |
| Account prefix | `wevibe` |
| Validator prefix | `wevibevaloper` |
| Consensus prefix | `wevibevalcons` |
| Bond/fee denom | `uvibe` (1 VIBE = 1,000,000 uvibe; `sdk.DefaultBondDenom = uvibe`) |
| Unordered transactions | Enabled (`authkeeper.WithUnorderedTransactions(true)`) |

### 1.2 Network endpoints *(built)*

| Endpoint | Port | Consumer |
|---|---|---|
| CometBFT RPC (blocks, tx gossip, WS event subscriptions) | `26657` | hub watcher, indexers |
| P2P | `26656` | validators/full nodes |
| gRPC (query services + tx service) | `9090` | hub gRPC client, social-graph |
| REST / gRPC-gateway (+ Swagger) | `1317` | dashboard, social-graph, any HTTP client |

### 1.3 Module inventory *(built)*

**13 stock SDK modules:** `auth`, `bank`, `staking`, `slashing`, `mint`, `distribution`, `consensus`, `epochs`, `gov`, `feegrant`, `authz`, `upgrade`, `genutil`.

**8 custom WeVibe modules:**

| Module | One-line role |
|---|---|
| `x/org` | Org slots, registration economics, membership, leader/serving-key authority, org config, feegrants |
| `x/memory` | Encrypted memory commitments, approval, Earned-Trust decay, reports, merkle roots |
| `x/serve` | Serve & denial receipt batches, signature verification, dedup, per-epoch stats |
| `x/bandwidth` | Per-org per-epoch flat rate limits (testnet anti-spam guard) |
| `x/reputation` | Contributor XP, serve stats, contributor/moderator/leader accountability profiles |
| `x/emissions` | 32-year fixed emission schedule, contributor reward accrual + claims |
| `x/identity` | Passkey (ed25519) ↔ wallet (secp256k1) migration aliases |
| `x/attestation` | Session-attestation storage socket — **wired but tx-disabled** (D-ATTEST-ROADMAP) |

### 1.4 Module accounts & permissions *(built, `app/app.go:118-127`)*

| Account | Permissions | Purpose |
|---|---|---|
| `fee_collector` | — | fee intake |
| `distribution` | — | staking rewards |
| `mint` | Minter | SDK mint module |
| `emissions` | **Minter** | mints contributor emission payouts |
| staking bonded / not-bonded pools | Burner, Staking | standard |
| `gov` | Burner | standard |
| `org` | **Burner** | burns 50% of every org-slot acquisition price |
| per-org accounts | (derived, not module accounts) | `authtypes.NewModuleAddress("orgacct/" + orgID)` — the org's operating account (gas/feegrant funding, 50% acquisition retain) |

All module account addresses are **blocked** for direct sends (`BlockedAddresses()`).

### 1.5 The epoch system *(built)*

- The chain runs **one custom epoch**: identifier `wevibe_epoch`, via stock `x/epochs`.
- Duration is set in genesis (`scripts/init-chain.sh`): **`WEVIBE_EPOCH_DURATION_SECONDS` env, default 60s local-dev** (production target 86400s = 1 day). `wevibed` validates the same env at startup (`cmd/wevibed/cmd/root.go:48`).
- **`x/mint` inflation is disabled at genesis** — `x/emissions` is the sole supply schedule.
- **`x/epochs` runs first in BeginBlockers**; its `AfterEpochEnd` hooks fire in registration order:
  1. **`EmissionsKeeper.AfterEpochEnd`** — mints the epoch emission (§2.6, §4.3).
  2. **`MemoryKeeper.AfterEpochEnd`** — current-epoch marker → validity-expiry archival → Earned-Trust decay on the **settled** epoch `N−5` → per-org merkle roots (§2.2, §5.2).
- Both hooks are **fail-resilient**: they log-and-continue on every recoverable error and return nil, because the SDK epoch dispatcher discards the whole cached-write batch on a non-nil error (the CO-039 zero-decay incident; `x/memory/keeper/epoch_hooks.go:34-42`).

### 1.6 Upgrade machinery *(built)*

- `x/upgrade` wired with a registered no-op handler **`v2`** (`app/upgrades/v2`) that proves the governance → halt → swap → restart flow.
- Custom `InitChainer` persists the module version map (D-S29-INITCHAINER-VERSION-MAP) and writes a `0xFFFFFFFF` sentinel key into every mounted KV store so no IAVL tree is empty at genesis (D-S29-CHAIN-RESTART-FOUNDATION).
- Store loader reads `upgrade-info.json` on restart (D-S29-UPGRADE-STORE-LOADER).

### 1.7 Governance & authority *(built)*

- Every module's `MsgUpdateParams` requires `authority == gov module address` — parameter changes go through **gov proposals only**.
- There is no custom admin key. The only non-gov authorities in the system are the per-org **leader wallet** and **serving key** (§2.1.4, §3.4).

---

## §2 The eight custom modules — full specification

### 2.1 `x/org` — organizations, slots, membership, authority

**Purpose.** The org is the container everything else keys on: memories, serves, bandwidth, reputation, emissions all reference `org_id`. This module owns the scarce slot registry, registration economics, the membership roster, per-org runtime config, and the two-key authority model.

#### 2.1.1 State *(built, `proto/wevibe/org/v1/state.proto`)*

| KV prefix | Type | Contents |
|---|---|---|
| `org/{org_id}` | `StoredOrg` | org_id, leader (ed25519 member pubkey), created_at, renewal_height, storage_quota, retrieval_budget, status (ACTIVE=0 / DORMANT=1 / SUSPENDED=2 / CLOSED=3), domain, hub_serving_address, leader_wallet_address, slot, account_address, hub_endpoints[], hub_response_pubkey, description, tech_stack, focus_areas, name + aggregates {total_committed_memories, total_active_members, total_epoch_rotations, total_upheld_reports, last_activity_epoch} |
| `slotreg/` | `StoredSlotRegistry` | `next_slot` — monotonic slot counter |
| `member/{org_id}/{pubkey}` | `StoredMemberRecord` | role (`leader`\|`member`), x25519_pubkey, can_contribute, can_moderate |
| `orgconfig/{org_id}` | `StoredOrgConfig` | serve_receipt_required, contest_stake_vibe, vocab_hash, embedding_model_id, min_contributions_per_epoch |
| `params` | `Params` | see 2.1.6 |

- **Org ID format:** `wevibe-org-{slot}` — slot-derived, permanent, leader-independent (`x/org/types/slot.go`).
- **Org account address:** `authtypes.NewModuleAddress("orgacct/" + orgID)` — a derived module-style address that holds the org's operating funds (acquisition retain half, feegrant funding). It is NOT a treasury with payout semantics (see §9 divergence #2).

#### 2.1.2 Registration economics *(built, `x/org/keeper/keeper.go:187-293,903-934`)*

1. Signer sends `MsgRegisterOrg` with leader pubkey, leader wallet, hub serving key, quotas, and org profile text.
2. Keeper reads `next_slot`, enforces `slot_cap` (default **32**).
3. **One org per leader pubkey** — any existing org with the same `leader` → `ErrLeaderAlreadyOwnsOrg`.
4. **Slot price (as-built):** `price(slot) = base_burn_price × (1 + burn_price_increase_percent/100 × slot)` — **linear** in the slot index. Defaults: base = **10,000,000 uvibe (10 VIBE)**, increase = **20%/slot**. So slot 0 = 10 VIBE, slot 10 ≈ 30 VIBE, slot 31 = 72 VIBE. (Canon WP-DESIGN-SPEC describes an ascending curve; the stale in-repo whitepaper's compounding `1.10^n` formula does not match code — §9 #3.)
5. **Payment split 50/50:** half **burned** via the `org` module account (Burner), half **transferred to the org's own account** (`orgacct/…`) — capitalizing the org's operating account. This is the network-level anti-spam + the funding source for org feegrants.
6. Leader auto-inserted as member with role `leader`; `total_active_members=1`.
7. **Feegrants issued from the org account** (`x/org/keeper/serving_feegrant.go`):
   - **Serving grant** → hub serving key: `AllowedMsgAllowance{BasicAllowance (unlimited), msgs: [MsgSubmitServeBatch, MsgSubmitDenialBatch]}`.
   - **Leader grant** → leader wallet: `AllowedMsgAllowance` over all 13 org/memory msg type URLs.
   - Net effect: **org operations are gas-sponsored by the org account** — leaders and hubs pay no personal gas for in-protocol actions.
8. `next_slot++`; event `org_registered`.

#### 2.1.3 Messages *(built, `proto/wevibe/org/v1/tx.proto`)*

| Message | Signer / authorization | Effect |
|---|---|---|
| `MsgRegisterOrg` | any funded account | slot allocation + burn + org/member/feegrant creation; returns `org_id` |
| `MsgAddMember` | **leader wallet** (signer == leader_wallet_address), org ACTIVE | adds member (role must be `member`; `leader` rejected) with x25519 key + capability bools; event `member_added` |
| `MsgRemoveMember` | leader wallet | removes member (leader cannot be removed); decrements active count; event `member_removed` |
| `MsgSetMemberCapabilities` | leader wallet, org ACTIVE | sets `can_contribute` / `can_moderate`; event `member_capabilities_set` |
| `MsgSetOrgConfig` | leader wallet (min_contributions_per_epoch ≤ 100) | sets serve_receipt_required, contest_stake_vibe, vocab_hash, embedding_model_id, min_contributions_per_epoch |
| `MsgSetServingKey` | leader wallet | rotates/revokes hub serving key; re-issues serving feegrant to the new key (old feegrant lingers but is inert — x/serve auth re-checks the registered address) |
| `MsgSetServingInfo` | leader wallet | sets `hub_endpoints` (ordered 1–3 transport URLs, D-CHAIN-RESOLVED-HUB-ENDPOINT) + `hub_response_pubkey` (ed25519 response-signing key, D-HUB-RESPONSE-SIGNED) |
| `MsgRotateEpoch` | leader wallet, org ACTIVE | increments `total_epoch_rotations` and returns the new epoch number — the org-level epoch counter the leader advances (drives the Umbral epoch-key schedule off-chain) |
| `MsgTransferLeadership` | leader wallet; new leader must already be a member; new wallet bech32-valid | flips roles (old leader → member, new → leader), updates org leader + leader_wallet_address, re-issues leader feegrant to the new wallet; event `leadership_transferred` |
| `MsgCloseOrg` | leader wallet, org ACTIVE | status → CLOSED (terminal); event `org_closed` |
| `MsgGrantTrialAllowance` | leader wallet | creates a `feegrant.PeriodicAllowance` for a grantee: `2000 uvibe × daily_submissions` per 24h period for `trial_days` days (gas for trial contributors) |
| `MsgUpdateParams` | gov authority | parameter update |

#### 2.1.4 The two-key authority model (D-S32-CO044-KEY-SEPARATION) *(built)*

Every org has **three distinct keys**:

| Key | Curve | Role | Blast radius if stolen |
|---|---|---|---|
| **Leader wallet** (`leader_wallet_address`) | secp256k1 | Sole authority for all org-decision txs: membership, config, serving-key rotation, memory approve/report, epoch rotation, leadership transfer, close | Full org control (rotatable via… nothing — protect it; it is the root) |
| **Hub serving key** (`hub_serving_address`) | secp256k1 | ONLY signer accepted by `x/serve` for serve/denial batches (enforced by `requireServingKeySigner`) | Can submit serve/denial batches + drain its own gas. Cannot touch org decisions. Leader-revocable via `MsgSetServingKey` |
| **Leader member pubkey** (`leader`) | ed25519 | The in-org identity (member record role `leader`); contributor-level identity | Identity-level, not a chain-tx signer |

Enforcement lives in `x/memory/keeper/msg_server.go:33-42` (`requireLeaderWallet`) and `x/serve/keeper/msg_server.go:33-42` (`requireServingKeySigner`) — both compare the **authenticated tx signer** against chain-registered addresses, never a self-declared field.

#### 2.1.5 Queries *(built, `proto/wevibe/org/v1/query.proto`)*

| gRPC | REST path | Returns |
|---|---|---|
| `GetOrg` | `/wevibe/org/v1/org/{org_id}` | org record (incl. serving address, endpoints, response pubkey) |
| `GetMembers` | `/wevibe/org/v1/members/{org_id}` | member list with roles + capability bools |
| `IsMember` | `/wevibe/org/v1/is_member/{org_id}/{pubkey}` | bool |
| `GetOrgConfig` | `/wevibe/org/v1/config/{org_id}` | serve_receipt_required, min_contributions_per_epoch, contest_stake_vibe |
| `GetOrgProfile` | `/wevibe/org/v1/profile/{org_id}?epoch=` | rich aggregate: member/moderator counts, approved memory count, total/unique/self serves, bandwidth used+cap (memory & serve), model_breakdown, quotas |
| `GetOrgAccount` | `/wevibe/org/v1/account/{org_id}` | org account address + uvibe balance |
| `Params` | `/wevibe/org/v1/params` | params |

#### 2.1.6 Params *(built, defaults)*

`min_registration_fee` 10,000,000 · `annual_renewal_fee` 5,000,000 · `default_storage_quota` 1,073,741,824 (1 GiB) · `default_retrieval_budget` 10,000 · `grace_period_epochs` 30 · `burn_price_decay_epochs` 10 *(present, unused by the current pricing path)* · `base_burn_price` 10,000,000 · `burn_price_increase_percent` 20 · `slot_cap` 32.

#### 2.1.7 Events *(built)*

`org_registered` (org_id, leader) · `member_added` (org_id, member_pubkey, role, member_x25519_pubkey) · `member_removed` (org_id, member_pubkey, removed_by, block_height) · `member_capabilities_set` · `leadership_transferred` (org_id, old_leader, new_leader) · `org_closed` · (defined, unemitted: `org_renewed`, `org_dormant`, `org_config_set`, `org_burn_paid`).

#### 2.1.8 Keeper APIs consumed by other modules *(built)*

`HasOrg`, `GetOrg`, `GetMember`, `IsMember`, `IsLeader`, `IsModerator`, `GetServingAddress`, `GetLeaderWallet`, `GetOrgConfig`, `GetAllOrgs`, `GetAllMembers`, `IncrementOrgCommittedMemories`, `IncrementOrgUpheldReports`, `IncrementOrgEpochRotations`, `SetOrgLastActivityEpoch`, `UpdateOrgStatus`, `RecomputeOrgActiveMembers`, `GetOrInitBandwidthState` (via bandwidth keeper).

---

### 2.2 `x/memory` — encrypted memories, Earned-Trust decay, reports

**Purpose.** The heart of the chain: stores encrypted memory commitments with full provenance anchors, runs the FROZEN D-4.2 Earned-Trust decay at epoch end, anchors moderation reports with evidence bundles, and computes per-org per-epoch merkle roots for data-availability proofs.

#### 2.2.1 State *(built, `proto/wevibe/memory/v1/state.proto`)*

| KV prefix | Type | Contents |
|---|---|---|
| `pending/{org}/{content_hash_hex}` | `StoredPendingCommitment` | content_hash, keywords[], contributor_id (ed25519 hex), epoch, submitted_at_height, memory_type, contributor_address, mc_version |
| `approved/{org}/{content_hash_hex}` | `StoredMemoryCommitment` | the full committed record (below) |
| `report/{org}/{content_hash_hex}/{reporter}` | `StoredMemoryReport` | moderation report + evidence bundle |
| `merkle/{org}/{epoch}` | `StoredEpochMerkleRoot` | merkle_root, memory_count |
| `count/{org}` | uint64 | approved memory count |
| `relationship/{org}:{src}:{tgt}` | `StoredMemoryRelationship` | CONTRADICTS/REPLACES/DEPRECATES/SUPERSEDES edge, proposer, approved, epoch |
| `validity/{org}:{cid}` | `StoredValidityMetadata` | valid_after_epoch, valid_until_epoch, scope_tags_bz |
| `current_epoch` | uint64 | last epoch seen by the hook |

**`StoredMemoryCommitment`** (the canonical committed memory): org_id, content_hash (32B), encrypted_blob (≤1 MiB), keywords[], contributor_pubkey, contributor_address, epoch (commit epoch), committed_at_height, committing_leader_pubkey, **state**, last_active_epoch, wrapped_dek_enc, memory_type, approved_at_epoch, **plaintext_hash, salt, ciphertext_hash, wrapped_dek_hash, contributor_sig** (the §7.4-canon anchor fields), serve_count_total, denial_count_total, archived_epoch, **mc_version** (Memory Contract schema version).

**`KeywordWeight`**: keyword, weight (decimal string, exact `big.Rat`), serve_count, denial_count — per-keyword earned-trust substrate.

**`MemoryState` enum (as-built):** `PENDING`(1), `PENDING_KEYWORD`(2), `PENDING_CHAIN`(3), `COMMITTED`(4), `DENIED`(5), `ARCHIVED`(6), `REPORTED_DELETED`(7). Chain-committed memories live at `COMMITTED`; the pre-chain states (2,3) mirror the hub-side moderation pipeline (canon WP §7.1) and are not written by chain msgs.

**`MemoryType` enum:** `MEMORY_TYPE_MEMORY`(1) only (extensible).

#### 2.2.2 Messages *(built, `proto/wevibe/memory/v1/tx.proto` + msg_server.go)*

| Message | Signer / authorization | Validation & effect |
|---|---|---|
| `MsgSubmitCommitment` | **leader wallet** | content_hash 32B; contributor must be an org member who is leader or `can_contribute`; valid memory_type; org exists; no duplicate pending; pending count < `max_pending_per_org` (1000). Stores pending; event `commitment_submitted` |
| `MsgApproveMemory` | **leader wallet** + committing_leader must be org leader member | Full hash-binding verification (below). Promotes pending → approved at state `COMMITTED`; increments org count/aggregates, contributor profile, leader profile |
| `MsgReportMemory` | **leader wallet** | Stores an upheld-report evidence bundle (below); updates org/moderator/leader accountability counters |
| `MsgUpdateParams` | gov authority | param update |

**ApproveMemory hash-binding (the contributor-signed anchor, canon WP §7.4, as-built):**
1. `submissionHash = sha256(encrypted_blob ‖ wrapped_dek_enc)` must equal `content_hash` — binds the committed bytes to the pending commitment.
2. `ciphertext_hash = sha256(encrypted_blob)` must match the supplied field.
3. `wrapped_dek_hash = sha256(wrapped_dek_enc)` derived.
4. Canonical body `wevibe.submit_memory.v1` (10 newline-joined lines: version, ciphertext_hash, contributor_pubkey, epoch_id, memory_type, org_id, plaintext_hash, salt, submission_hash, wrapped_dek_hash) must verify against `contributor_sig` under the contributor's ed25519 pubkey (from the pending record — never from the msg).
5. On success: `approved_at_epoch = current epoch`, state `COMMITTED`, org aggregates (`total_committed_memories++`, `last_activity_epoch`), reputation `IncrementContribution(contributor)`, `IncrementLeaderChainCommit(leader)`.

**ReportMemory evidence bundle (as-built):** plaintext ≤ 4096 B, ciphertext ≤ 8192 B, capsule (~200 B) required — OR `plaintext_oversized=true` with all three empty; `plaintext_hash` always required; one report per (org, memory, reporter). Effects: `StoredMemoryReport` persisted; `IncrementOrgUpheldReports`; contributor profile `IncrementContribution` (note: same counter as approvals — see §9 #10); each approving moderator `IncrementModeratorUpheld`; committing leader `IncrementLeaderUpheldReport`.

#### 2.2.3 The Earned-Trust decay model — D-4.2, FROZEN *(built, `x/memory/keeper/lifecycle.go`; canon DECISIONS.md D-4.2 + DMO-006/DMO-007; R-DECAY-FROZEN: never change without an explicit Walter-approved order)*

**Data model.** Each committed memory carries per-keyword weights in `[0,1]` (exact rational strings). Serves and denials land as event-time **counters**; weight movement happens **once per epoch** at epoch end.

**Event-time handlers (counters only):**
- `ApplyServeBoost(org, hash, epoch)` — serve_count_total++, per-matched-keyword serve_count++ (matched set from the serve receipt).
- `ApplyDenialDecay(org, hash, epoch)` — denial_count_total++, per-matched-keyword denial_count++.

**Epoch-time (`ApplyEpochDecay(settledEpoch)`)** — for every `COMMITTED` memory, per keyword `kw`:

```
trust        = 1 − denialRate,          denialRate = denial_total / (serve_total + denial_total)
trustEarned  = serve_total ≥ trust_min_serves(1)  AND  denialRate < trust_max_rate(0.30)

if serves>0 and kw matched:   weight += serveD(0.022) × serves × (serveFloor(0.40) + (1−serveFloor)·trust²)
if denials>0 and kw matched:  weight −= denialD(0.090) × denials × (denialFloor(0.30) + (1−denialFloor)·denialRate)
if kw unmatched or no events (and idle not suppressed):
                              weight −= idleD(0.060) × idleMult
                              idleMult = idleProtect(0.05)                if trustEarned
                                       = idleUntrusted(1.00) × idleScale  otherwise

clamp weight to [0,1]
```

**Traffic-adaptive idle (Goldilocks) scaling:** `idleScale = clamp( (orgEvents/activeMemories) / idleTrafficRef(0.22), idleTrafficFloor(1.0), 1.0 )`… with `idleTrafficFloor` default 10000 bps = 1.0, so scale ∈ [1.0, 1.0] unless params change; if the org had **zero** serve+denial events in the assessed epoch, idle decay is **suppressed** for that epoch (zero-signal guard).

**Archival rule:** after decay, if **every** keyword weight ≤ `retrieval_threshold` (1500 bps = 0.15), the memory transitions to `ARCHIVED` with `archived_epoch` set. Archived memories are excluded from further decay and from serve boost/denial.

**Grace period:** memories younger than `grace_epochs` (20) skip decay entirely.

**Settlement lag:** the epoch hook assesses epoch `N − IdleDecaySettleEpochs(5)`, because serve/denial traffic is relayed hub→chain asynchronously and lands 1–3 epochs late; assessing the just-ended epoch would read zero traffic and fire the zero-signal guard every epoch (the CO-042 zero-decay root cause). *(built, `x/memory/keeper/epoch_hooks.go:13-25`)*

**Sim parity:** the canonical sim (`wevibe-sim/ranking-fix.js` `applyDecay`/`ET_BASE`) implements the identical constants and arithmetic — parity is a locked invariant (R-DECAY-FROZEN; verified R-29 2026-06-28).

#### 2.2.4 Validity windows & relationships *(built — keeper-only)*

- **Validity metadata** (`validity/{org}:{cid}`): `IsValidInEpoch` gates serve acceptance (a serve outside the validity window is rejected); `CheckEpochExpiry` at epoch end auto-archives memories past `valid_until_epoch` and deletes the metadata. **No message writes this state today** — genesis-seeded only.
- **Relationships** (`relationship/…`): store + query edges (CONTRADICTS/REPLACES/DEPRECATES/SUPERSEDES). `applyConfidencePenalty` is a **no-op stub** — edge effects are NOT applied on-chain. (Supersession semantics are canon D-SUPERSESSION-DEMOTE, design-locked, retrieval-side.) Genesis-seeded only today.

#### 2.2.5 Queries *(built)*

| gRPC | REST path | Returns |
|---|---|---|
| `GetMemory` | `/wevibe/memory/v1/memory/{org_id}/{content_hash}` | full StoredMemoryCommitment |
| `GetPendingCommitments` | `/wevibe/memory/v1/pending/{org_id}` | pending list |
| `GetMemoryCount` | `/wevibe/memory/v1/count/{org_id}` | approved count |
| `GetEpochMerkleRoot` | `/wevibe/memory/v1/merkle_root/{org_id}/{epoch}` | root + memory_count |
| `ListRelationships` | `/wevibe/memory/v1/relationships/{org_id}/{cid}` | edges touching cid |
| `GetValidity` | `/wevibe/memory/v1/validity/{org_id}/{cid}` | metadata + found |
| `GetMemoriesBatch` | `/wevibe/memory/v1/memories_batch/{org_id}` | batch fetch + not_found list (hub bulk path) |
| `Params` | `/wevibe/memory/v1/params` | params |

#### 2.2.6 Params *(built, defaults)*

`max_pending_per_org` 1000 · `pending_retention_epochs` 7 *(present; no purge path active)* · `max_blob_size_bytes` 1,048,576 · `max_keywords_per_memory` 20 · `retrieval_threshold_bps` 1500 · `initial_confidence_bps` 0 *(vestigial from the retired confidence model)* · `contest_window_epochs` 10 *(vestigial — no contest flow built)* · `grace_epochs` 20 · `serve_d_bps` 220 · `denial_d_bps` 900 · `idle_d_bps` 600 · `serve_floor_bps` 4000 · `denial_floor_bps` 3000 · `idle_protect_bps` 500 · `idle_untrusted_bps` 10000 · `trust_min_serves` 1 · `trust_max_rate_bps` 3000 · `idle_traffic_ref_bps_per_mem` 2200 · `idle_traffic_floor_bps` 10000. Validation: all bps ≤ 10000; grace ≥ 1; trust_min_serves ≥ 1; traffic ref ≥ 1.

#### 2.2.7 Events & epoch hook

Events: `commitment_submitted` (org_id, contributor_id, block_height) · (defined, unemitted: `memory_rejected`, `expired_purged`).
Epoch hook (`AfterEpochEnd` on `wevibe_epoch`): `setCurrentEpoch` → `CheckEpochExpiry` → `ApplyEpochDecay(epoch−5)` → `ComputeAndStoreEpochMerkleRoot` per org with approved memories (merkle over that epoch's committed content hashes).

#### 2.2.8 Keeper APIs consumed by other modules *(built)*

`GetApprovedMemory`, `GetPendingCommitment`, `GetAllPendingForOrg`, `GetApprovedCount`, `GetApprovedCountByContributor`, `GetActiveMemoryCountByOrg`, `GetContributorsWithApprovalsInEpoch` (emissions qualifying set), `IsValidInEpoch`, `ApplyServeBoost`, `ApplyDenialDecay`, `GetEpochMerkleRoot`, `IterateUpheldReports`, `GetUpheldReport`, `GetMemoryReports`, `GetRelationship`, `ListRelationshipsForMemory`.

---

### 2.3 `x/serve` — serve & denial receipts

**Purpose.** The high-traffic ingestion path: batched, signature-verified serve and denial receipts from org hubs. Produces the event stream that feeds decay (matched keywords + counters), epoch stats, contributor stats, and reputation XP. All dedup is **chain-computed** — the chain trusts nothing client-supplied.

#### 2.3.1 State *(built, `proto/wevibe/serve/v1/state.proto` + keeper prefixes)*

| KV prefix | Type | Contents |
|---|---|---|
| `fingerprint/{hex}` | → receipt key | serve dedup marker (points at the receipt's store key) |
| `receipt/{org}/{epoch}/{fingerprint}` | `StoredServeReceipt` | memory hash, contributor_id, epoch, is_self_serve, model_id, turn_count, matched_keywords[], serve_key_pubkey, fingerprint |
| `denial/{org}/{epoch}/{fingerprint}` | `StoredDenialReceipt` | memory hash, deny_key, reason, epoch, serve_fingerprint, serve_key_pubkey |
| `denyfingerprint/{hex}` | marker | denial dedup |
| `stats/{org}/{epoch}` | `StoredEpochServeStats` | total_serves, unique_memories_served, unique_serve_keys, self_serves, model_breakdown map, total_denials |
| `contributor/{id}/{epoch}` | `StoredContributorEpochServes` | serve_count, self_serve_count, org_ids[], total_turns |
| `memcount/{org}/{hash}/{epoch}` | uint64 | per-memory serves this epoch (decay input) |
| `denycount/{org}/{hash}/{epoch}` | uint64 | per-memory denials this epoch (decay input) |
| `memfirst/…`, `keyfirst/…` | markers | uniqueness tracking for stats |
| `matchedkw/{org}/{cid}/{epoch}/{keyword}` | marker | matched-keyword set per memory per epoch (decay input) |

#### 2.3.2 Messages *(built, `proto/wevibe/serve/v1/tx.proto`)*

| Message | Signer / authorization | Effect |
|---|---|---|
| `MsgSubmitServeBatch` | **org serving key** (signer == org.hub_serving_address) | batch of ≤ 500 `ServeEntry`; returns {accepted, rejected_duplicate, rejected_invalid} |
| `MsgSubmitDenialBatch` | **org serving key** | batch of ≤ 500 `DenialEntry`; returns {accepted, rejected} |
| `MsgUpdateParams` | gov authority | params |

**ServeEntry fields:** memory_content_hash, contributor_id, model_id, turn_count, contributor_wallet, **matched_keywords[] (required, non-empty — DMO-007)**, serve_key_pubkey (ed25519), serve_sig, nonce.

**Per-entry acceptance pipeline (`ProcessServeBatch`):**
1. Batch ≤ `max_serves_per_batch`; whole batch consumes serve bandwidth (`ConsumeServeBandwidth(org, epoch, len)`).
2. Entry signature check: ed25519 `serve_sig` over the canonical body `wevibe-serve-v1` (7 lines: version, org_id, hex(memory_hash), epoch, hex(serve_key_pubkey), sorted matched keywords comma-joined, hex(nonce)).
3. Dedup: `serveFingerprint = sha256(memory_hash ‖ serve_key_pubkey ‖ epoch_be64)`; seen → reject-duplicate.
4. Memory must exist (`GetApprovedMemory`) and pass `IsValidInEpoch`.
5. Per-memory cap: `memcount < max_serves_per_memory_per_epoch` (100).
6. `is_self_serve = (hex(serve_key_pubkey) == contributor_id)`.
7. Persist receipt + fingerprint marker + matched-keyword markers; increment memcount; update epoch stats (unique memories via `memfirst`, unique keys via `keyfirst`, totals, self-serves, model_breakdown); update contributor epoch serves.
8. **Cross-module cascades (non-fatal, log-and-continue):** `memoryKeeper.ApplyServeBoost` (counters); `reputationKeeper.RecordServe` — keyed to the **authoritative contributor from the stored memory record**, never the serve payload (CO-041 Task F, R-ONE-PATH).

**DenialEntry acceptance pipeline:**
1. Signature over canonical body `wevibe-denial-v1` (7 lines: version, org_id, hex(memory_hash), epoch, hex(serve_key_pubkey), hex(serve_fingerprint), hex(nonce)).
2. Dedup: `denialFingerprint = sha256(canonicalDenialBody(nonce=nil))`.
3. **Must reference a real prior serve**: the `serve_fingerprint` must resolve to a stored receipt with non-empty matched keywords, same memory hash, same serve key — i.e., **you can only deny a serve you actually received** (this is the Block-path integrity rule).
4. Memory must still exist.
5. Persist denial receipt + fingerprint; increment `denycount` + epoch denial stats; store the originating matched keywords; `ApplyDenialDecay` counters (non-fatal).
Event `denial_batch_submitted` with accepted/rejected counts + full rejection-reason breakdown logged.

#### 2.3.3 Queries & params *(built)*

Queries: `GetEpochServeStats` (`/wevibe/serve/v1/stats/{org}/{epoch}`) · `GetContributorServes` (`/wevibe/serve/v1/contributor/{id}/{epoch}`) · `GetMemoryServeCount` (`/wevibe/serve/v1/memory/{org}/{hash}/{epoch}`) · `Params`.

Params (defaults): `max_serves_per_batch` 500 · `self_serve_discount_percent` 50 *(present; the operative self-serve discount lives in reputation XP, §2.5)* · `max_serves_per_memory_per_epoch` 100 · `min_org_age_epochs` 1 *(not enforced in the serve path)* · `diminishing_returns_threshold` 10 *(not enforced)*.

Events: `denial_batch_submitted` (emitted) · `serve_batch_submitted`, `serve_recorded` (defined).

---

### 2.4 `x/bandwidth` — per-org rate limits

**Purpose.** Testnet/alpha anti-DDoS: flat per-org per-epoch caps on memory submissions and serve receipts. Canon intent: removed at mainnet once per-memory storage deposits activate (ROADMAP).

**State:** `state/{org}/{epoch}` → {memory_used, memory_cap, serve_used, serve_cap} · `override/{org}` → custom caps · `params`.

**Message:** `MsgSetBandwidthOverride` · `MsgUpdateParams` (gov). ⚠ **As-built defect:** the override handler authorizes via `IsLeader(orgID, msg.Signer)`, which looks up a **member record** (keyed by ed25519 pubkey hex) using the tx signer's **bech32 wallet** — the lookup always misses, so `MsgSetBandwidthOverride` currently always returns `ErrUnauthorized`. Every other org-admin message gates on `signer == leader_wallet_address`. Until fixed, caps are effectively params-defaults only (§9 #12).

**Behavior:** state lazily initialized per (org, epoch) from override-or-params; `ConsumeServeBandwidth` (+n per batch, `ErrServeBandwidthExhausted` if over) is called from `x/serve`. `ConsumeMemoryBandwidth` exists but has **no production caller** — memory submissions are not bandwidth-charged on-chain today; the memory cap is display-only (via `GetBandwidthState`/`GetOrgProfile`). New epoch ⇒ fresh counters (no reset message).

**Queries:** `GetBandwidthState` · `GetBandwidthOverride` · `GetRemainingBandwidth` · `Params` (`/wevibe/bandwidth/v1/...`).

**Params (defaults):** `default_memory_cap_per_epoch` **10,000** · `default_serve_cap_per_epoch` **50,000** (the stale in-repo docs say 1,000/10,000 — code is authoritative).

---

### 2.5 `x/reputation` — contributor XP & accountability profiles

**Purpose.** Presentation-layer aggregates over signed chain events: contributor XP + serve stats + cross-org breadth, plus per-role accountability profiles (contributor / moderator / leader). Canon: reputation is provenance made visible (WP §8.1); serve attribution is social, **never** payout-coupled (WP §8.2).

#### 2.5.1 State *(built)*

| Prefix | Type | Contents |
|---|---|---|
| `active/` | flag | module activation |
| `stats/{developer}` | `StoredReputationStats` | memory_count, difficulty_bucket[11], domain_tags map, provenance_breakdown map, xp, serve_count, self_serve_count, org_breadth, first_seen_epoch, serve_xp |
| `memory/{developer}/{cid}` | `StoredAttestedMemory` | difficulty, quality, domain_tags, provenance |
| `orgset/{developer}` | `StoredContributorOrgSet` | org_ids[] |
| `profile/{contributor}/{org}` | `StoredContributorProfile` | total_approved_memories, total_serves_received, total_denials_received, total_reports_filed/upheld_against, memberships[], first/last contribution epoch |
| `modprofile/{pubkey}/{org}` | `StoredModeratorProfile` | total_approvals, approvals_later_upheld_count, approved_memory_hashes (ring ≤1000), first/last approval epoch |
| `leaderprofile/{pubkey}/{org}` | `StoredLeaderProfile` | total_chain_commits_signed, total_upheld_reports_committed, epoch_rotations[], first_leadership_epoch, current_leader |

#### 2.5.2 Messages *(built)*

| Message | Signer | Effect |
|---|---|---|
| `MsgUpdateReputation` | any signer | records an attested memory (difficulty/quality ≤ 10, domain_tags, provenance) for a developer; returns XP. Gated on `params.active` |
| `MsgIncrementContribution` / `MsgIncrementServe` / `MsgRecordBan` | **gov authority only** | external profile increments — effectively disabled for regular use (the hub must NOT bundle these; x/memory and x/serve invoke the keeper internally instead — CO-035) |
| `MsgUpdateParams` | gov | params |

**Internal invocation (the real path):** `RecordServe(contributor, org, epoch, isSelfServe)` from x/serve → serve_count++, self_serve_count++, serve_xp += `serve_xp_per_serve`(5) or `self_serve_xp_per_serve`(2), first_seen_epoch, org breadth + orgset. `IncrementContribution` / `IncrementLeaderChainCommit` / `IncrementModeratorUpheld` / `IncrementLeaderUpheldReport` from x/memory.

#### 2.5.3 Queries *(built — the richest query surface on the chain)*

`GetReputation` · `GetXP` · `IsActive` · `Params` · `GetServeStats` · `GetContributorOrgSet` · `GetCrossOrgProfile` · `GetContributorProfile` (stats + histogram + top domains + provenance + epoch serves/turns) · `ModeratorProfile` · `LeaderProfile` · `UpheldReportsByContributor|Moderator|Leader` (paginated) · **`VerifyUpheldReport`** — returns the full evidence bundle for a reported memory: plaintext, ciphertext, capsule, plaintext_hash, salt, ciphertext_hash, wrapped_dek_hash, contributor_sig + contributor_pubkey, encrypted_blob, wrapped_dek_enc, content_hash, epoch, memory_type, and the **canonical_body** itself — everything an external party needs to independently re-verify a moderation action. (All under `/wevibe/reputation/v1/...`.)

**Params (defaults):** `active` true · `max_difficulty` 10 · `max_quality` 10 · `serve_xp_per_serve` **5** · `self_serve_xp_per_serve` **2**.

---

### 2.6 `x/emissions` — the 32-year schedule & contributor rewards

**Purpose.** Executes the locked VIBE emission schedule at epoch end and manages the contributor reward ledger + claim path.

#### 2.6.1 State *(built)*

| Prefix | Type | Contents |
|---|---|---|
| `pool/` | `StoredEmissionPool` | total_supply, daily_mint, operator_share, validator_share, epoch, **validator_pool_remaining_uvibe, contributor_pool_remaining_uvibe, contributor_rollover_uvibe**, start_epoch, total_epochs_elapsed |
| `emission/{epoch}` | `StoredDailyEmission` | total_emitted, validator_share |
| `contribreward/{passkey_pubkey}` | uint64 string | pending claimable reward |
| `lifetimereward/{passkey_pubkey}` | uint64 string | all-time earnings |
| `gate/{operator}/{org}/{epoch}` | `StoredAsymmetricGate` | storage_passed, retrieval_allowed (legacy/roadmap gating) |
| `bootstrap/{operator}` + `bootstrappool/` + `bootstrapExpiry` | credits | early-adopter credit ledger |

#### 2.6.2 Epoch mint (`MintDailyEmission`, runs from the epoch hook) *(built)*

```
remaining_epochs = max(1, schedule_duration_days(11680) − total_epochs_elapsed)
validator_emission  = validator_pool_remaining / remaining_epochs      (pool accounting)
contributor_budget  = min( contributor_pool_remaining / remaining_epochs,
                           contributor_annual_cap(1e13 uvibe) / 365 )

qualifying = contributors with ≥ contributor_qualify_threshold(1) memories
             APPROVED (state COMMITTED) in this epoch, network-wide
             (from x/memory GetContributorsWithApprovalsInEpoch)

if none qualify:        budget → contributor_rollover
else:                   total = budget + rollover
                        per_contributor = total / len(qualifying)
                        remainder → rollover (no VIBE lost to rounding)
                        credit contribreward/{passkey_pubkey} per contributor
                        MintCoins(per_contributor × n) into the emissions module account
```

- Rewards are keyed to the contributor's **passkey pubkey** (the ed25519 identity), not a wallet — wallet-free earning is structural.
- `total_supply` and per-epoch `StoredDailyEmission` update; `total_epochs_elapsed++`.
- The **validator** emission line currently debits the pool and records the share in `StoredDailyEmission` — coin movement to validators is roadmap (validators earn standard SDK staking rewards meanwhile; chain ROADMAP "present but not fully wired").

#### 2.6.3 Messages *(built)*

| Message | Signer | Effect |
|---|---|---|
| `MsgMintDailyEmission` | gov authority | manual/authority trigger of the same mint path |
| `MsgClaimContributorReward` | wallet signer | **Identity-migration-gated claim**: `x/identity.ResolveIdentity(passkey_pubkey)` must return `is_migrated=true` and `wallet == signer`; pays the full pending balance from the emissions module account to the wallet; zeroes pending; event `emissions.contributor_reward_claimed` |
| `MsgUpdateParams` | gov | params |

#### 2.6.4 Queries & params *(built)*

Queries: `GetEmissionPool` · `Params` · `ContributorReward` (`/wevibe/emissions/v1/contributor-reward/{pubkey}` → pending_withdrawal + all_time_earnings).

Params (defaults): `daily_mint_amount` 1e9 *(legacy)* · `operator_share_percent` 80 · `validator_share_percent` 20 · `storage_weight_percent` 30 · `retrieval_weight_percent` 70 · `rarity_multiplier_cap` "3.0" · `bootstrap_duration_epochs` 365 · **`total_supply_uvibe` 1,000,000,000,000,000 (1e15 = 1B VIBE)** · **`validator_emission_pool_uvibe` 570,000,000,000,000 (570M VIBE)** · **`contributor_annual_cap_uvibe` 10,000,000,000,000 (10M VIBE/yr)** · **`schedule_duration_days` 11,680 (32 × 365)** · **`contributor_qualify_threshold` 1**. Validation: operator+validator = 100; storage+retrieval = 100. Constant: `EpochsPerYear = 365`.

---

### 2.7 `x/identity` — passkey ↔ wallet migration

**Purpose.** Holds the one-way alias from a contributor's passkey ed25519 identity to a Cosmos wallet, which gates reward withdrawal (canon: link ≠ migrate; ROADMAP §1.4).

**State:** `alias/{passkey_pubkey}` → {wallet_address, is_migrated, migrated_at_epoch} · `params` (active: true).

**Message:** `MsgMigrateIdentity` — any wallet signer; requires an **ed25519 signature from the passkey** over the canonical body `wevibe.migrate_identity.v1\npasskey_pubkey:{hex}\nwallet:{bech32}\nnonce:{N}`; one-way (already-migrated → error). Event `identity.migrated`.

**Query:** `ResolveIdentity` (`/wevibe/identity/v1/resolve/{passkey_pubkey}`) → wallet, is_migrated, found.

**Consumers:** `x/emissions` (claim gating); any external service that needs to resolve passkey → wallet.

---

### 2.8 `x/attestation` — session-attestation socket (disabled)

**Purpose (canon):** the reserved anchor for session-provenance proofs — "user X using model Y took N turns to solve problem Z" — and, later, GSTV receipt commitments (WP §8.11–8.12, ROADMAP §2).

**State schema (built, fully defined):** `StoredSessionAttestation`{org_id, session_hash (32B), model_id, turn_count, token_count, provider_type (LOCAL=1 CommitLLM-class | CLOUD=2 provider-signed), commitllm_receipt_hash, provider_signature_hash, contributor_id, epoch, submitted_at_height}; indexed by `attestation/{org}/{hash}` and `session_epoch/{org}/{epoch}/{hash}`.

**Status (built):** `MsgSubmitSessionAttestation` **returns `ErrAttestationDisabled` unconditionally** — the socket is fully wired (state, queries, genesis, params, AutoCLI) but the tx path is deliberately off until verification infrastructure exists (D-ATTEST-ROADMAP). `VerifyCommitLLMReceipt` / `VerifyCloudProviderSignature` are stubs returning "unverified: …".

**Queries (live):** `GetSessionAttestation` · `ListSessionAttestations` · `Params` (`max_attestations_per_epoch` 10,000, `require_attestation_for_serve` false).

**Why it matters for the future:** this is the chain's **designed pluggable socket for AI attestation systems** — see §8.

---

## §3 The consumable surface — everything a service can read or write

### 3.1 Transports *(built)*

| Transport | Port | What it carries |
|---|---|---|
| gRPC | `9090` | All module Query services, the tx service (broadcast + simulate), Tendermint service (blocks), node service |
| REST (gRPC-gateway + Swagger) | `1317` | Every query above over HTTP/JSON |
| CometBFT RPC + WebSocket | `26657` | Blocks, txs, `tm.event` subscriptions (NewBlock, Tx events by attribute) |
| P2P | `26656` | Validator gossip |

### 3.2 Write surface — transaction classes *(built)*

| Class | Messages | Signer | Gas paid by |
|---|---|---|---|
| Org administration | `MsgAddMember`, `MsgRemoveMember`, `MsgSetMemberCapabilities`, `MsgSetOrgConfig`, `MsgSetServingKey`, `MsgSetServingInfo`, `MsgRotateEpoch`, `MsgTransferLeadership`, `MsgCloseOrg`, `MsgGrantTrialAllowance` | org **leader wallet** | org account via leader feegrant |
| Memory lifecycle | `MsgSubmitCommitment`, `MsgApproveMemory`, `MsgReportMemory` | org **leader wallet** (+ contributor ed25519 sig inside Approve) | org account via leader feegrant |
| Serve/denial relay | `MsgSubmitServeBatch`, `MsgSubmitDenialBatch` | org **serving key** | org account via serving feegrant |
| Org acquisition | `MsgRegisterOrg` | any funded wallet | the signer (must hold ≥ slot price + fees) |
| Trial onboarding | (any msg) on behalf of a grantee | grantee | leader-funded `PeriodicAllowance` from `MsgGrantTrialAllowance` |
| Identity & rewards | `MsgMigrateIdentity` (passkey-signed), `MsgClaimContributorReward` | the claimant's wallet | the signer |
| Reputation annotation | `MsgUpdateReputation` | any signer | the signer |
| Governance | `MsgUpdateParams` (every module), software upgrades | gov module account | proposal process |
| Attestation | `MsgSubmitSessionAttestation` | — | **disabled** |
| Bandwidth admin | `MsgSetBandwidthOverride` | — (broken authz, §2.4) | — |

### 3.3 Canonical signing bodies (the off-chain signature contract) *(built)*

Four canonical byte formats are signed outside the chain tx envelope and verified inside keepers. Any client generating these MUST match byte-for-byte:

1. **`wevibe-serve-v1`** (serve entry, ed25519 serve key): 7 lines — version · org_id · hex(memory_hash) · epoch · hex(serve_key_pubkey) · sorted matched keywords joined by `,` · hex(nonce). No trailing newline.
2. **`wevibe-denial-v1`** (denial entry, same key): version · org_id · hex(memory_hash) · epoch · hex(serve_key_pubkey) · hex(serve_fingerprint) · hex(nonce).
3. **`wevibe.submit_memory.v1`** (memory approval anchor, contributor ed25519): version · `ciphertext_hash:` · `contributor_pubkey:` · `epoch_id:` · `memory_type:` · `org_id:` · `plaintext_hash:` · `salt:` · `submission_hash:` · `wrapped_dek_hash:` (key:value lines).
4. **`wevibe.migrate_identity.v1`** (identity migration, passkey ed25519): version · `passkey_pubkey:{hex}` · `wallet:{bech32}` · `nonce:{N}`.

**Fingerprints (chain-computed, never client-supplied):** serve = `sha256(memory_hash ‖ serve_key_pubkey ‖ be64(epoch))`; denial = `sha256(canonicalDenialBody(nonce=nil))`.

### 3.4 Read surface index *(built)*

Full query inventory in §2 per module. The highest-value reads for external services:

- **Org directory:** `GetOrg` (serving address, hub endpoints, response pubkey), `GetOrgProfile` (aggregates), `GetOrgAccount`.
- **Memory content:** `GetMemory` / `GetMemoriesBatch` (encrypted blob + wrapped DEK + anchor fields + keyword weights + state), `GetEpochMerkleRoot`.
- **Reputation/social:** `GetContributorProfile`, `GetCrossOrgProfile`, `LeaderProfile`, `ModeratorProfile`, `UpheldReportsBy*`, `VerifyUpheldReport`.
- **Economics:** `GetEmissionPool`, `ContributorReward`, `GetEpochServeStats`, bandwidth state/remaining.
- **Identity:** `ResolveIdentity`.

### 3.5 Event stream (CometBFT WS subscriptions) *(built)*

Subscribe `tm.event='Tx'` with attribute filters. Emitted today: `org_registered`, `member_added`, `member_removed`, `member_capabilities_set`, `leadership_transferred`, `org_closed`, `commitment_submitted`, `denial_batch_submitted`, `emissions.contributor_reward_claimed`, `identity.migrated`. Defined-but-unemitted constants exist for serve batches, bandwidth, attestations — check before relying (§9). The hub's `ChainWatcher` (`wevibe-hub/internal/chain/watcher*.go`) is the reference consumer.

### 3.6 Generated clients & contracts *(built)*

- **Go:** `wevibe-chain/x/{module}/types/*.pb.go` — regen only via Docker `make proto-gen` (`ghcr.io/cosmos/proto-builder:0.18.1`; R-19 — never hand-edit).
- **TypeScript:** `wevibe-protocol/js/wevibe/{org,memory,serve,bandwidth,reputation,emissions,identity,attestation}/v1/*` (ts-proto) + `openapi.yaml` + `test-vectors/` + `contract_test.sh` — the cross-language byte-parity harness.
- **CLI:** `wevibed query|tx` with AutoCLI for every module (`x/*/module/autocli.go`), plus standard SDK commands. Epoch duration env: `WEVIBE_EPOCH_DURATION_SECONDS`.

---

## §4 The economic system — design spec

Two layers, cleanly separated: **VIBE flows** (token movement) and **signal flows** (reputation/decay — social, never token-coupled; canon WP §8.2).

### 4.1 VIBE supply & allocation *(canon-locked, ROADMAP §1.1; pool params built)*

| Allocation | Amount (VIBE) | uvibe | Status |
|---|---|---|---|
| Total supply | 1,000,000,000 | 1e15 | param |
| Foundation (genesis, unlocked) | 100,000,000 (10%) | 1e14 | canon |
| Validator genesis self-delegation | 10,000,000 (1%) | 1e13 | canon |
| Validator 32-yr emission pool | 570,000,000 (57%) | 5.7e14 | param (built) |
| Contributor 32-yr emission pool | 320,000,000 (32%) | 3.2e14 | genesis-seeded (built) |

**Genesis seeding (local dev, built — `scripts/init-chain.sh`):** foundation account 1e14 uvibe (100M VIBE) · faucet account 1e12 (1M VIBE) · validator gentx self-delegation 1e12 (1M VIBE) · emission pool carries validator_pool_remaining 5.7e14 + contributor_pool_remaining 3.2e14 + rollover 0. (Canon's "Validator genesis 1% = 10M VIBE" is the mainnet allocation intent; local dev seeds 1M.)

Schedule: **11,680 daily epochs (32 × 365)**, linear drawdown of each pool with the contributor line capped at **10M VIBE/year** (1e13 uvibe). Qualification: ≥1 memory committed (approved to `COMMITTED`) in the epoch, network-wide. Split: equal among qualifiers; no-qualifier epochs and integer remainders roll forward — nothing is burned by rounding.

### 4.2 Org slots & acquisition *(built; canon §ROADMAP-1)*

- **Scarcity:** hard slot cap — 32 (alpha default in params) / 320 testnet / 3200 mainnet (canon governance trajectory).
- **Price:** ascending per-slot — as-built linear `base × (1 + 0.20 × slot)` from 10 VIBE; canon describes an ascending curve to the same end.
- **Split:** 50% **burned** (deflation) + 50% to the **org account** (operating float: gas, feegrants, future storage deposits).
- **Permanence:** `org_id` is slot-derived and leader-independent; one org per leader pubkey.
- **Roadmap:** Harberger self-assessed rent with forced-sale window; Dutch resale of freed slots; dormancy detection.

### 4.3 Contributor emissions & the claim path *(built)*

Contribution-only payout: per **approved memory** (qualify at ≥1/epoch), **never** per serve or retrieval. Accrues to the passkey identity; withdraws to a wallet only after `MsgMigrateIdentity` (dual-signed, one-way) + `MsgClaimContributorReward`. Leaders earn **no** emissions; there is no per-serve royalty and no protocol-enforced moderator split (canon ROADMAP §1.3).

### 4.4 The Earned-Trust decay engine (D-4.2, FROZEN) *(built; §2.2.3)*

The chain's quality-pressure system: per-keyword weights boosted by matched serves (trust²-scaled), cut by matched denials (denial-rate-scaled), eroded by idle (trust-gated, traffic-adaptive, zero-signal-suppressed); auto-archive when every keyword falls below 0.15; 20-epoch grace for new memories; 5-epoch settlement lag for async relay. **This is the mechanism that makes the memory corpus self-cleaning** — knowledge that stops being used sinks; knowledge that gets denied sinks faster; proven knowledge idles cheap.

### 4.5 Serve & reputation economics *(built)*

- Serve receipts are **attribution**, not payment. They produce: per-keyword decay counters, epoch stats, model_breakdown, contributor serve counts/turns, reputation XP (5/serve, 2/self-serve), org breadth.
- Anti-gaming (built): chain-computed fingerprint dedup · per-memory epoch serve cap (100) · batch cap (500) · per-entry ed25519 signature verification · self-serve XP discount · denial-requires-serve binding · serving-key-only submission · bandwidth caps.
- Reputation is cross-org and public (org breadth, difficulty histogram, provenance breakdown, accountability counters for moderators/leaders via upheld reports).

### 4.6 Bandwidth & spam economics *(built + roadmap)*

Today: flat per-org per-epoch caps (10k memory / 50k serve, leader-overridable) + slot-price burn + gas fees (validator-local min gas 0.025 uvibe observed; hub pays a 2,000 uvibe floor, D-13.12). Roadmap: per-memory **storage deposits** (decaying rent, keeper-claimable deletion bounty — liveness pricing, not participation cost) replace bandwidth at mainnet; network-governed fee floor via exactly one of `x/globalfee` (static) or `x/feemarket` (dynamic) — decision-gated, not wired (ROADMAP §9).

### 4.7 Demand-leg (roadmap, canon ROADMAP §1.3)

Org recall-access payments settle through an on-chain **router** that burns `max(n%, floor)` per payment and routes the remainder in-transaction to the leader wallet; the network holds no revenue account; hub `membership_active`/`org_credits` mirror chain payment events. Not built; the chain-side hook surface is org account + watcher events.

### 4.8 What the economy deliberately does NOT do *(canon)*

No per-serve payout · no platform tribunal (the chain publishes; it does not judge) · no revocation/DRM on served plaintext (D-PLAINTEXT-IRREVOCABLE) · no global leaderboard (per-org badge scoping) · no token-coupled attestation (attestation is opt-in provenance, never a CONTRIBUTION/RECEIPT gate — but as of 2026-07-23 provenance ADMISSIBILITY is a distinct SERVING gate: inadmissible provenance means a committed memory cannot be served; D-PROVENANCE-ADMISSIBILITY-2026-07-23 §22).

---

## §5 Executables within the chain — means & motivation

"Executable" surface on a Cosmos chain = message handlers + scheduled hooks + block lifecycle + governance. What runs, when, and why:

### 5.1 Message servers (the interactive executables)

Every state transition is an explicit, signed, authorized tx. The motivation for the message-per-intent design: **every consequential action is an individually signed, attributable, replay-protected on-chain event** (the unforgeable audit log, canon WP §10.7). Full inventory in §3.2.

### 5.2 Epoch hooks (the settlement engine)

| Hook | Runs | Why it exists (motivation) |
|---|---|---|
| `EmissionsKeeper.AfterEpochEnd` | every `wevibe_epoch` end | Mints the epoch emission + settles contributor rewards. Epoch-driven minting makes the 32-year schedule deterministic and amortizes distribution to one pass per day |
| `MemoryKeeper.AfterEpochEnd` | every `wevibe_epoch` end | (a) stamp current epoch → (b) archive validity-expired memories → (c) run Earned-Trust decay for the **settled** epoch `N−5` → (d) compute per-org merkle roots. Motivation: decay is a *settlement*, not an event reaction — event-time handlers only count (cheap, O(1) per receipt), the epoch hook does the O(memories) weight pass once, atomically, with fully-landed relay data |

**Why the event-time/epoch-time split matters:** serve/denial batches arrive asynchronously from hubs and must never block on expensive re-ranking. Counters land instantly (receipt acceptance is latency-critical); weights move at epoch end (correctness-critical, order-deterministic). This is also what makes the chain tolerant of hub relay lag — the 5-epoch settle window guarantees completeness before scoring (CO-042).

**Resilience contract:** hooks return nil always; every sub-step logs-and-continues (CO-039). A failed decay step must never roll back the batch.

### 5.3 Block lifecycle

- **BeginBlockers:** `epochs` (first — drives the hooks above), then mint/distribution/slashing/staking/auth/authz/consensus.
- **EndBlockers:** staking, bank, gov, feegrant.
- **PreBlocker:** upgrade, auth.
- **AnteHandler:** standard SDK (sig verification, fee deduction incl. **feegrant** — the mechanism that lets org accounts sponsor leader/serving/trial traffic), plus unordered-tx support.

### 5.4 Governance executables

- `MsgUpdateParams` per module (gov-only) — every economic and decay constant is governance-tunable in principle; **decay constants are additionally frozen by project rule** (R-DECAY-FROZEN — Walter-approved order required even though gov could technically change them).
- `x/upgrade` software-upgrade proposals (v2 no-op handler registered as the proven path).

### 5.5 Genesis (the cold-start executable)

Full `InitGenesis`/`ExportGenesis` per module (org registry + slot counter, memories + reports + relationships + validity + merkle roots, serve receipts + stats, bandwidth state/overrides, reputation stats/profiles, emission pool + credits + gates, identity aliases, attestation records). Genesis is a complete state round-trip — a network can be exported and re-launched faithfully. The emission pool (validator/contributor remaining, rollover, start epoch) is **genesis-seeded**.

---

## §6 Auxiliary services plugged into the chain today

The chain's rule for aux services (canon WP §2.3): **the chain is the only durable authority; every other component is disposable and rebuildable from chain state + member-held keys.** Current plug-ins, and exactly how each touches the chain:

| Service | Lang | Chain touches | Trust role |
|---|---|---|---|
| **wevibe-hub** (`wevibe-server/wevibe-hub`) | Go | **Reads:** gRPC :9090 query clients for org/memory/serve/bandwidth/emissions/attestation/reputation (org state, configs, merkle roots, stats, profiles) + CometBFT WS block/tx watcher (`internal/chain/watcher*.go`: memory/serve/report/org watchers) + reconcile loop vs its Postgres mirror. **Writes:** relayed `MsgSubmitServeBatch`/`MsgSubmitDenialBatch` signed by each org's **serving key** (funded by faucet, gas sponsored by org feegrant) — `internal/chain/{submit,broadcast,faucet}.go` | Serving infra: retrieval (Qdrant), REST for dashboard/mcp, Umbral sidecar host. **Never holds epoch_sk, never decrypts** (D-MISSION-INVARIANT). Disposable: rebuildable from chain (D-HUB-REBUILDABLE) |
| **wevibe-dashboard** (`wevibe-server/wevibe-dashboard`) | Next.js/TS | **Reads:** REST :1317 (e.g. `GetOrgAccount`) + hub REST. **Writes:** builds and submits org/memory txs directly — hand-rolled protobuf encoders for all `MsgRegisterOrg/AddMember/…/SubmitCommitment/ApproveMemory/ReportMemory` in `lib/chain-client.ts`, signed by the leader wallet | Leader/member UI. No privileged chain role |
| **wevibe-mcp** | TS | **No direct chain tx.** Produces the contributor-side crypto: computes the 4 hash fields and **signs `wevibe.submit_memory.v1`** with the contributor's ed25519 identity (`src/contribution.ts` → `canonical.js`); delivers payloads to dashboard/hub paths | Local contribution/recall edge: extraction, encryption, signing, recall client |
| **wevibe-protocol** | proto + TS | Not a runtime service — the **wire contract repo**: buf/ts-proto generated JS clients for all 8 modules, `openapi.yaml`, cross-language test vectors + `contract_test.sh` | Single source for generated clients (consumers: dashboard, tests, future services) |
| **wevibe-sdk** | Rust + WASM | No chain txs — client-side crypto library (Umbral seal/open, canonical hashing) consumed by mcp/plugin | Browser/host crypto portability |
| **wevibe-umbral** | Rust | No chain txs — PRE sidecar (kfrag mint leader-side; capsule re-encryption hub-side gRPC); epoch key schedule follows the org's chain-visible epoch | The confidentiality engine; hub's only crypto op is secret-less re-encryption |
| **wevibe-guard** | Rust (YARA-X) | No chain interaction — sanitization scans at submission and recall (local) | Consumer-side safety edge |
| **wevibe-social-graph** | Go | **Reads:** REST :1317 — reputation contributor profiles (`/wevibe/reputation/v1/contributor/{id}`) + emissions contributor rewards (`/wevibe/emissions/v1/contributor-reward/{pubkey}`). Display-only over RPC | Reference social-graph client: profiles, badges, org discovery (forkable; chain = SoT) |
| **wevibe-faucet** | Go | **Writes:** bank sends from a faucet mnemonic via its own Cosmos broadcaster (rate-limited). Used by the hub to keep org serving keys funded | Dev/testnet token dispenser |
| **wevibe-opencode-plugin** | TS | No direct chain tx. Consumer serve/denial **ed25519 entries signed locally** (per-org serve key) flow plugin → mcp → hub → chain batches | The human injection gate; recall/contribute UX in the coding agent |
| **wevibe-sim** (non-git) | JS | Parity harness: re-implements Earned-Trust decay (`ranking-fix.js`) for offline scoring validation; Go↔JS parity fixtures pin the arithmetic | Decay-model test oracle |
| **wevibe-bench** | various | Exercises the chain via the hub for the org benchmark (org-0 pool) | Evaluation infra |

**Deliberately absent from the chain path:** the dashboard server-side never signs org txs for users (leader wallet signs); the hub never signs org-decision txs (leader wallet only); no service holds another service's keys.

---

## §7 The integration contract — how to plug a new service into the chain

### 7.1 Read integration

1. **Point queries** — gRPC `9090` or REST `1317` (§2, §3.4). Cheapest: REST for occasional reads; gRPC streams/clients for hot paths (hub model).
2. **Event subscriptions** — CometBFT WS `:26657`, filter tx events by attribute (§3.5). This is how the hub maintains its mirror with zero polling.
3. **Rebuild pattern (canon D-HUB-REBUILDABLE)** — any derived index must be reconstructable from chain state (+ org keys for content). `vocab_hash` + `embedding_model_id` are chain-anchored per org so downstream indices can prove they match the canonical pipeline.
4. **Proof artifacts** — per-org per-epoch `GetEpochMerkleRoot` for data-availability proofs; `VerifyUpheldReport` for independent moderation-evidence verification.

### 7.2 Write integration — pick your signer class

| Your service needs to… | You must hold/be | Gas | Notes |
|---|---|---|---|
| Submit serve/denial batches for an org | the org's registered **serving key** (secp256k1) | org feegrant | per-entry ed25519 consumer sigs still required inside the batch |
| Perform org admin / memory lifecycle | the org's **leader wallet** | org feegrant | sole authority; cannot be delegated (by design) |
| Contribute a memory | the contributor's **passkey ed25519** to sign the canonical body; a leader wallet commits it | org feegrant | contributor never needs a wallet or gas |
| Serve/deny as a consumer | a per-org **serve key** (ed25519, offline-generated) | none (hub relays) | signature + fingerprint binding only |
| Claim emissions | a **migrated wallet** (x/identity) | own | one-way migration; link ≠ migrate |
| Register an org | any wallet with ≥ slot price | own | one org per leader pubkey; slot cap |
| Change params | a **gov proposal** | deposit | all module params |
| Record sessions | — | — | attestation tx **disabled**; use state schema when activated |

### 7.3 The identity/key map (who is who)

```
Passkey ed25519 (global identity) ──x/identity alias──► Cosmos wallet secp256k1 (rewards, org registration)
        │                                                    
        ├─ member record per org (+ x25519 for key envelopes)  
        ├─ per-org serve key ed25519 (pseudonymous serve/denial signing)
        └─ contributor_sig on every memory (canonical anchor)
Org: leader wallet secp256k1 (root authority) · serving key secp256k1 (serve relay only) · hub response key ed25519 (response integrity) · Umbral epoch keys (secp256k1 scalars, leader-derived, never on-chain)
```

### 7.4 Hard rules for integrators

- Never trust client-supplied dedup/identity fields — the chain recomputes fingerprints and reads attribution from stored records; mirror that discipline.
- Fees are uvibe-denominated; base denom everywhere; display VIBE = 1e6 uvibe.
- Keywords/matched-keyword sets are **public plaintext metadata** by design (WP §4.6); memory bodies are always ciphertext on-chain — if your service sees plaintext, it is outside the chain trust boundary and must never send it back up (D-MISSION-INVARIANT: the hub receives no plaintext, no epoch_sk).
- Regenerate clients from proto (`make proto-gen` Docker, R-19); never hand-edit `.pb.go`.

---

## §8 Pluggable sockets for future AI/LLM systems

The chain was designed to absorb future AI capability as **plug-ins, not forks**. The existing sockets, mapped to the research axes:

### 8.1 For small specialized local LLMs

- **`x/attestation` provider model:** `ProviderType LOCAL` + `commitllm_receipt_hash` — the CommitLLM commit-and-audit receipt lane for open-weight/self-hosted inference (Ollama/vLLM/llama.cpp class). Schema is already on-chain; the verifier stub is the single replacement point.
- **`model_id` + `model_breakdown` on every serve receipt and epoch stat (built):** per-model serve analytics are already chain-consumable — a local-model cohort's real usage is measurable without any chain change.
- **Reputation `provenance` + difficulty/quality fields (built):** the two-layer difficulty scoring input slots (roadmap §3) are already stored.

### 8.2 For cryptographically certified run systems (attestation provenance)

- **`x/attestation` is the socket (wired, tx-disabled):** `StoredSessionAttestation` already carries session_hash / model_id / turn_count / token_count / provider_type / receipt hashes. Activation = enable the msg + replace two verifier stubs. Roadmap generalizes to typed proofs `{proof_type: tee_receipt | zktls_proof | zkml_proof}` — TEE confidential-inference receipts (hardware-attested workload ⇒ weights measurement), attested-gateway receipts for closed models, zkTLS when the cost ceiling clears (ROADMAP §2).
- **Verification tiers T0–T4 (canon WP §8.12):** T3 = attested-runtime receipts anchored through this socket. Grades ride as **labels**, never gates.
- **Folded extraction attestation (canon):** a second receipt over the extraction step itself binds memory candidates to an attested extractor + pinned prompt — the "certified memory" ladder (`pipeline-attested → session-attested → attested-gateway → self-declared`).

### 8.3 For goal-trajectory learning mechanisms

- **GSTV (Goal-Sealed Trajectory Verification, canon WP §8.12):** goal seals {goal, executable predicate, state₀ hash, contributor sig} + working-tree hash chains + receipts {predicate, ablation, negative}. Today these are plugin-local artifacts; the seal schema carries a **`chain_anchor` hook (unset in MVP)** and `x/attestation` is the reserved anchor point (WP §11.3 Table 20).
- **Chain-side substrate already live:** `mc_version` (Memory Contract schema versioning on submit/approve), per-memory anchor fields, epoch merkle roots — a trajectory anchor can land as an attestation record referencing memory content hashes without touching x/memory.

### 8.4 For communal trajectory verification

- **T4 — stranger convergence (canon):** k independent contributors, k independent seals, matching context fingerprint, convergent fix. The chain already has the raw material: cross-org serve receipts with `model_id`/`turn_count`, contributor org-breadth (`orgset`), epoch stats, and the public query surface to detect convergence. What is missing is the convergence detector itself — an aux service over §3.4 reads, no chain change required to start.
- **Reputation org-breadth + cross-org profiles (built):** "verified across N orgs" is already computed.

### 8.5 Chain-pluggability checklist for any new AI service

1. **Read-only analytics/verification service** → consume §3.4 queries + §3.5 events; zero chain change. (social-graph is the template.)
2. **New receipt/proof type** → land it through `x/attestation` state (or a sibling records module); keep verification off-chain, anchor hashes on-chain (ROADMAP §2's one-rule: verified off-chain before anchoring).
3. **New per-memory metadata** → extend the Memory Contract (`mc_version` bump) — chain schema already versioned for this.
4. **New economic leg** → governance: params + (if needed) a messages module; never bolt token flows onto serve/reputation signals (canon §4.8 separations).
5. **Never**: plaintext to the hub, epoch_sk anywhere off the leader machine, token-coupled attestation, or a contribution gate on any receipt tier.

---

## §9 Divergence register — in-repo docs vs shipped code

`wevibe-chain/docs/{WHITEPAPER,MODULES,ARCHITECTURE,PARAMETERS,API,CLI}.md` (v2.0 "Production") drifted from the code. Authoritative answers live in THIS spec. Known divergences:

1. **Memory lifecycle:** docs describe PENDING/APPROVED/STABLE/DEGRADED/DORMANT/CONTESTED/ARCHIVED/REJECTED with confidence-bps thresholds. **Code:** PENDING(_KEYWORD/_CHAIN)/COMMITTED/DENIED/ARCHIVED/REPORTED_DELETED with Earned-Trust keyword weights (D-4.2). The confidence-bps params (`initial_confidence_bps`, stable/degraded/dormant thresholds) are vestigial or reserved-removed.
2. **Treasury & tier payouts:** docs describe `MsgFundTreasury`/`MsgWithdrawTreasury`/`MsgSetRepTiers` and epoch payouts from org treasuries. **Code:** none of these messages exist; org accounts are operating floats funded by the 50% acquisition retain; contributor payout is the network-wide emissions split.
3. **Slot pricing:** in-repo whitepaper says compounding `1.10ⁿ` from 1 VIBE. **Code:** linear `10 VIBE × (1 + 0.20×slot)`.
4. **Contest mechanics:** docs describe `MsgContestMemory`/`MsgResolveContest` stake-escrow disputes. **Code:** not built — accountability is the report path (`MsgReportMemory` + upheld-report profiles + `VerifyUpheldReport`). `contest_window_epochs` param is vestigial. (Consumer-filed reports + expose gate + `clear_report` are canon-near-term per WP §7.5 + chain ROADMAP; as-built reports are leader-mediated with reporter attribution.)
5. **Relationship/validity messages:** docs describe `MsgRelateMemories`/`MsgApproveRelationship`/`MsgSetValidityBounds`/`MsgArchiveMemory`/`MsgRejectMemory`/`MsgPurgeExpired`. **Code:** state + queries + genesis only; no msgs; relationship effects are a no-op stub; validity consumed by serve gating + epoch expiry.
6. **Bandwidth defaults:** docs 1,000/10,000 per epoch. **Code:** 10,000/50,000.
7. **Reputation XP:** docs 1 normal / 0 self-serve. **Code:** 5 / 2.
8. **Attestation:** docs imply live submission with stub verification. **Code:** msg fully disabled (`ErrAttestationDisabled`, D-ATTEST-ROADMAP).
9. **Events:** several event constants are defined but not emitted (serve batch, bandwidth, org burn/config/dormant, memory rejected/purged). Emitted set in §3.5.
10. **`IncrementContribution` reuse:** `x/memory.ReportMemory` increments the contributor's *contribution* counter (same counter as approvals) — naming quirk to be aware of when reading profiles; upheld-report counts live on the org/leader/moderator profiles.
11. **Emissions docs §9.2:** tier-matched per-org payout from treasuries — not built; §4.3 here is the real flow. `daily_mint_amount`/`operator_share` params are legacy; the pool-remaining fields drive the schedule.
12. **Bandwidth override authz:** `MsgSetBandwidthOverride` gates on `IsLeader(member-pubkey)` with the signer's bech32 wallet → always `ErrUnauthorized`; and `ConsumeMemoryBandwidth` has no production caller (memory submissions uncharged). Serve bandwidth (50k/epoch) is the only live cap.

---

## Appendix A — Source-of-truth index

| Domain | File(s) |
|---|---|
| Wire surface | `wevibe-chain/proto/wevibe/{org,memory,serve,bandwidth,reputation,emissions,identity,attestation}/v1/*.proto` |
| App wiring, module accounts, hook order, genesis marker | `wevibe-chain/app/app.go` |
| Epoch runtime | `wevibe-chain/cmd/wevibed/cmd/root.go` (`WEVIBE_EPOCH_DURATION_SECONDS`) |
| Decay (D-4.2) | `wevibe-chain/x/memory/keeper/lifecycle.go` + `epoch_hooks.go` |
| Emission schedule | `wevibe-chain/x/emissions/keeper/keeper.go` (`MintDailyEmission`) |
| Org economics/feegrants | `wevibe-chain/x/org/keeper/{keeper,serving_feegrant}.go` |
| Serve/denial pipeline | `wevibe-chain/x/serve/keeper/{keeper,msg_server}.go`, `x/serve/types/canonical.go` |
| Canon economics | `wevibe-docs/ROADMAP.md` §1 (token model), `wevibe-docs/WP-DESIGN-SPEC.md` §2–§11 |
| Locked decisions | `wevibe-docs/DECISIONS.md` (D-4.2, D-S32-CO044, D-ATTEST-*, D-HUB-REBUILDABLE, D-PLAINTEXT-IRREVOCABLE, D-13.12…) |
| Hub chain integration | `wevibe-server/wevibe-hub/internal/chain/{grpc_client,query,submit,broadcast,faucet,watcher*}.go` |
| Dashboard chain integration | `wevibe-server/wevibe-dashboard/lib/chain-client.ts` |
| Generated TS clients | `wevibe-protocol/js/wevibe/**` (+ `openapi.yaml`, `test-vectors/`) |

*End of spec. As-built claims verified against `wevibe-chain` working tree on 2026-07-18; canon claims cite the doc and section inline.*



