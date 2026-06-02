# WeVibe Network — Product Development Plan
**Version:** 2.20
**Date:** 2026-05-05
**Sprint:** 22 (complete)

> **[STALE SNAPSHOT — 2026-06-02]** This is a Sprint-22 historical snapshot and is NOT current. Notably, the chain module list below is wrong: the live chain has **7 modules** — `x/attestation`, `x/bandwidth`, `x/emissions`, `x/memory`, `x/org`, `x/reputation`, `x/serve`. The modules `x/serving`, `x/challenge`, `x/receipt`, and `x/operator` **do not exist**. For current state see `SESSIONCONTINUANCE.md`, `MASTER.md`, and `DECISIONS.md`. This file needs a full rewrite or retirement.

---

## Current State Summary

**Product:** End-to-end encrypted organizational memory system for AI coding agents. Two-tier memory delivery: universal MCP baseline + invisible platform plugin injection.

**Architecture:** Leader-custodied E2EE. Go hub (opaque relay), TypeScript MCP client, Rust crypto SDK + content scanner. PostgreSQL, Qdrant, Ollama (admin CLI only).

**Two-Tier Memory Delivery (locked):**
- **Tier 1 — MCP server (universal baseline):** Works on any coding agent with MCP support. Agent calls `recall` tool explicitly. Less reliable (agent may not call it). No plugin required.
- **Tier 2 — Platform plugin (invisible injection):** System prompt injection, sub-second, agent never decides. Memories pre-fetched at session start, cached, filtered per-prompt, injected into system prompt before LLM sees anything. Different plugin per platform, same shared layer (retrieve-cli + wevibe-guard) underneath.

## Sprint 24 Key Changes

- Moderator approvals now hinge on hub-managed quorum: `required_approvals` is stored in org config, approval votes persist in PostgreSQL, and leaders retain an override path.
- Dashboard shipped a new Reports queue plus a settings control for `required_approvals`, aligning moderator UX with the off-chain quorum flow.
- The OpenCode plugin gained Accept / Deny / Report actions; reported memories never inject and instead feed the hub report API.
- `MsgApproveMemory` authorizes moderators after quorum, and integration tests cover `MsgGrantTrialAllowance` to guarantee fee grants back moderator approvals.

**Six packages (public):**
- wevibe-hub (Go) — opaque relay, stores ciphertext
- wevibe-mcp (TypeScript) — retrieval backend, contribution pipeline, MCP server (Tier 1 read + write path)
- wevibe-sdk (Rust/WASM) — crypto primitives, identity, key derivation
- wevibe-guard (Rust/YARA-X) — content scanner, detector-not-judge
- Platform plugins (TypeScript) — Tier 2 invisible injection per coding agent

**Two packages (private, chain):**
- wevibe-chain (Go/Cosmos SDK) — sovereign L1 appchain modules (8 modules, all production-hardened)
- wevibe-commitllm-bridge (Rust) — CommitLLM ZK receipt verification bridge

**Retrieval:** Vector-first staged scoring (ADR-025) with selective LLM re-ranking for contested queries.

**Scoring formula (live):**
```
final = vector_score + min(gamma * keyword_boost, delta * vector_score)
gamma = 0.1, delta = 0.15
If top-2 within epsilon=0.20 -> MCP-side LLM re-ranking via MCP sampling
```

**Security:** 8-layer defense-in-depth pipeline. Submit-time: wevibe-guard -> OCR sanitize -> encrypt -> human review. Recall-time: decrypt -> wevibe-guard -> OCR sanitize -> artifact extract -> egress policy -> selective transform -> privilege tag -> present. Tier 2 plugins run wevibe-guard on every cached memory at session start; blocked memories never inject.

**MCP surface (Tier 1):** 3 agent-facing tools (`recall`, `contribute`, `reject`) + ambient memory resource. 16-command `wevibe-admin` CLI for all admin operations.

**Plugin surface (Tier 2, OpenCode):** System prompt injection via `experimental.chat.system.transform`. Session-start caching via retrieve-cli. Per-prompt keyword filtering. Context accumulation via `tool.execute.before`. Compaction preservation via `experimental.session.compacting`. Zero custom tools. Zero permission prompts.

**Chain modules (private, Cosmos SDK) — all production-hardened:**
- x/attestation: Merkle root submission, batched roots per epoch (12 tests passing)
- x/org: registration, dynamic pricing, storage quota accounting (19 tests passing)
- x/serving: operator registration, per-org serving sets, sticky rotation (29 tests passing)
- x/challenge: VRF-seeded storage challenges, 15-leaf sampling, deadline enforcement, slashing (26 tests passing)
- x/emissions: fixed daily pool, operator/validator split, work-score formula, asymmetric gating (21 tests passing)
- x/receipt: retrieval receipt proofs, signature verification, budget consumption (22 tests passing)
- x/reputation: per-developer aggregates, dormant by default (28 tests passing)

**Chain status:** Cosmos SDK app wiring landed. `app/app.go` registers all standard SDK keepers alongside WeVibe modules, provides TxConfig via `x/auth/tx`, mounts KV stores, exposes gRPC/RPC services. `wevibed` CLI implements init/start/keys/genesis/query/tx flows; `wevibed init → gentx → collect-gentxs → start` produces a single-node CometBFT network (block height 1 confirmed). Docker compose validated; smoke test passes. Integration tests (CO-162) prove full tx pipeline via MsgServiceRouter; gRPC query services (CO-163) provide 17 RPCs across 7 modules with REST gateway annotations. All keeper tests green. Hub integration pending.

**Dependencies (MCP server):** Node.js, ImageMagick, Tesseract, wevibe-guard binary. No Ollama.
**Dependencies (admin CLI):** Above + Ollama (for moderation keyword extraction).

---

## Metrics

| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| Adversarial Gold@1 | 100% | >=85% | MET |
| Adversarial Gold@3 | 100% | >=95% | MET (ship threshold) |
| Adversarial MRR | 1.000 | -- | -- |
| 7-model avg Delta | +3.4 | >0 | MET |
| 7-model recall rate | 100% (7/7) | 100% | MET |
| wevibe-mcp tests | 336 | -- | 333 pass, 3 skip, 0 fail |
| wevibe-guard tests | 27 | -- | 27 pass, 0 fail |
| wevibe-hub tests | all | -- | all pass (cached) |
| wevibe-dashboard | -- | -- | type check clean |
| Theater tests | 0 | 0 | MET |
| Ollama deps (MCP server) | 0 | 0 | MET |
| wevibe-chain keeper tests | 157 | -- | all pass (CO-155) |
| wevibe-commitllm-bridge tests | 10 | -- | 10 pass |
| wevibed smoke test | init→gentx→collect→start (single validator) | ≥1 per release | MET (CO-159) |
| app/app.go integration tests | 2 (keeper wiring + export) | maintain green | MET |
| wevibe-chain integration tests | 20 (9 tx + 11 query tests) | maintain green | MET (CO-162, CO-163) |
| gRPC query services | 17 RPCs across 7 modules + x/operator | complete | MET (CO-163, CO-169) |

---

## Sprint Ledger

### Sprint 12 (CO-059 through CO-077) -- all DONE
Pipeline fixes, keyword overhaul (blind tokens -> plaintext), weighted scoring, scoring observability, adversarial benchmark creation.

### Sprint 13 (CO-078 through CO-090a) -- all DONE

| CO | Package | Status | Topic |
|---|---|---|---|
| CO-078-083 | multi | DONE | Early-phase extraction tuning |
| CO-084 | wevibe-mcp + wevibe-hub | REVERTED | Vocabulary-constrained query extraction (regression) |
| CO-085 | benchmark | DONE | Parallelized benchmarks |
| CO-086 | wevibe-mcp + wevibe-hub | DONE | Revert CO-084 |
| CO-087 | wevibe-hub + wevibe-mcp | DONE | Vector-first retrieval + calibrated keyword boost (ADR-025) |
| CO-088 | WeVibe-Retriever | DONE | Phase 3 training (FAILED -- representation collapse, nomic confirmed) |
| CO-089 | wevibe-hub | DONE | EnsureCollection before UpsertPoint |
| CO-090/090a | benchmark | DONE | Fix benchmark scripts |

### Sprint 14 (CO-091) -- DONE

| CO | Package | Status | Topic |
|---|---|---|---|
| CO-091 | wevibe-mcp + wevibe-hub | DONE | Selective re-ranking (ADR-025 Phase 2). Gold@1 70%->100%. |

### Sprint 15 (CO-092 through CO-100a) -- all DONE

| CO | Package | Tests | Status | Topic |
|---|---|---|---|---|
| CO-092 | wevibe-mcp | 219 | DONE | Tool consolidation: 16+ -> 3 agent-facing. |
| CO-093 | wevibe-mcp | 226 | DONE | Auto-init: MCP resource ambient memories. |
| CO-094 | wevibe-mcp | 231 | DONE | Recall-time sanitization + privilege isolation. |
| CO-095 | wevibe-mcp | 294 | DONE | Artifact extraction + egress enforcement. |
| CO-096 | wevibe-mcp | 298 | DONE | MCP sampling: Replace all Ollama LLM calls. |
| CO-097 | wevibe-mcp | 298 | DONE | Bundled embedding: nomic-embed-text ONNX. |
| CO-098 | wevibe-mcp | 298 | DONE | Test audit: 15 theater tests -> 0. |
| CO-100 | wevibe-mcp | 332 | DONE | Pre-production security tests. |
| CO-100a | wevibe-mcp | 336 | DONE | Annotation leakage fix. |

### Sprint 16 (CO-107 through CO-108) -- DONE

| CO | Package | Status | Topic |
|---|---|---|---|
| CO-107 | wevibe-mcp | DONE | Compact memory format (~80 tokens/memory), MCP elicitation gate. |
| CO-108 | wevibe-mcp | DONE | Benchmark production readiness. |

### Sprint 17 (CO-113 through CO-115) -- DONE

| CO | Package | Tests | Status | Topic |
|---|---|---|---|---|
| CO-113 | wevibe-mcp | 326 | DONE | Contribution UX overhaul. |
| CO-114 | wevibe-mcp | 326 | DONE | Session buffer wiring. |
| CO-115 | wevibe-mcp | 326 | DONE | Rejection enforcement. |

### Sprint 18 (CO-133) -- DONE

| CO | Package | Status | Topic |
|---|---|---|---|
| CO-133 | wevibe-hub | DONE | Removed dead stub endpoints. |

### Sprint 19 (CO-134 through CO-138) -- CLOSED

| CO | Package | Tests | Status | Topic |
|---|---|---|---|---|
| CO-134 | wevibe-sdk + wevibe-hub | all pass | DONE | FeeModel canonical signing consistency. |
| CO-135 | wevibe-guard | 21/21 | DONE | wevibe-guard hardening: unicode YARA, URL_RE broadened. |
| CO-136 | wevibe-dashboard | clean | DONE | Fix approveSubmissionCanonical. |
| CO-137 | wevibe-hub | -- | DEFERRED | Error response standardization. |
| CO-138 | wevibe-mcp | 331/334 | DONE | Fix setTestStoreFromSnapshot TypeError. |

### Sprint 20 (CO-140 through CO-146) -- DONE

| CO | Package | Status | Topic |
|---|---|---|---|
| CO-140 | wevibe-mcp + .opencode | DONE | Shared retrieval CLI (retrieve-cli.ts) + initial plugin. |
| CO-141 | wevibe-mcp | DONE | Update all wevibe-guard callers to structured format. |
| CO-142 | wevibe-guard | DONE | Structured memory scanning (breaking: old format dead). |
| CO-143 | scripts + docs | DONE | Kill v2 collection name + fix clear/start scripts. |
| CO-144 | wevibe-mcp | DONE | Fix MCP tool naming (wevibe_ prefix) + extraction fallback. |
| CO-145 | wevibe-dashboard | DONE | Dynamic hub URL for LAN access. |
| CO-146 | .opencode plugin | DONE | Rebuild plugin: system prompt injection (Tier 2). |

### Sprint 21 (CO-154, CO-155, CO-160, CO-161, CO-162, CO-163) -- DONE

| CO | Package | Status | Topic |
|---|---|---|---|
| CO-154 | wevibe-chain + wevibe-commitllm-bridge | DONE | Chain foundation hardening: SDK store test panic, key corruption, Merkle proof stub, CI deps |
| CO-155 | wevibe-chain | DONE | Migrate all 7 module keepers from in-memory maps to KVStore (157 tests passing) |
| CO-160 | wevibe-chain + wevibe-commitllm-bridge | DONE | Repository split: extracted both repos from wevibe-server |
| CO-161 | wevibe-chain | DONE | Local validator stack: Dockerfile, docker-compose, init-chain.sh, smoke-test.sh |
| CO-162 | wevibe-chain | DONE | Chain integration tests: TestSuite with MsgServiceRouter dispatch, 9 tx tests, helpers.go |
| CO-163 | wevibe-chain | DONE | gRPC query services: 17 RPCs across 7 modules, queryServer pattern, 11 query tests |

---

## Architecture Decisions

### ADR-025 Phase 2: Selective Re-ranking (Sprint 14)
MCP-side LLM re-ranking when hub signals ambiguity. E2EE preserved. Gold@1: 70%->100%.

### Defense-in-Depth Pipeline (Sprint 15)
Eight-layer sanitization on every recalled memory. Known limitation: semantic payload encoding bypasses syntactic extraction.

### Ollama Elimination (Sprint 15)
MCP server has zero external service dependencies beyond the hub.

### Tool Consolidation (Sprint 15)
Agent-facing surface reduced from 16+ tools to 3 + 1 resource.

### Detector-Not-Judge (Sprint 19)
wevibe-guard flags everything suspicious. No false-positive suppression at the regex layer.

### Two-Tier Memory Delivery (Sprint 20, locked)

**Tier 1 -- MCP (universal baseline):**
- Works on every coding agent with MCP support
- Agent calls `recall` tool explicitly
- Agent decides when to recall -- unreliable but universal
- No plugin installation required
- Trust: WEVIBE_ALLOW_UNREVIEWED flag or MCP elicitation if client supports it

**Tier 2 -- Platform plugin (invisible injection):**
- System prompt injection, sub-second, agent never decides
- Memories pre-fetched at session start, cached in plugin memory
- Per-prompt keyword filtering against cached memories
- Context accumulates as agent reads files (tool.execute.before)
- Zero custom tools, zero permission prompts
- Trust established at plugin install time, not per-memory
- wevibe-guard blocks malicious content; flags are informational
- Each platform has its own plugin, all call same retrieve-cli + wevibe-guard

**Shared layer (both tiers use):**
```
retrieve-cli.ts  --  retrieval + decrypt + sanitize (JSON stdout)
wevibe-guard bin   --  scan + flag detections (JSON stdout)
```

**Platform plugin implementations:**
```
OpenCode:     experimental.chat.system.transform (proven by opencode-rules, open-mem)
Claude Code:  hooks / dynamic CLAUDE.md
Cursor:       dynamic .cursorrules (Claude Code compat)
Cline:        extension API / .clinerules
```

### Chain Architecture (Sprint 19-22, evolving)

Cosmos SDK + CometBFT sovereign L1 appchain. Single token (VIBE; base denom uvibe). Batched Merkle root attestation. Feeless stake-weighted bandwidth. Dynamic org pricing.

**Chain status (Sprint 21 complete):**
- All 7 module keepers migrated from in-memory `map[string]*Type` to `storetypes.KVStore` (CO-155)
- `StoreContext` pattern bridges `context.Context` API with SDK store access
- JSON serialization for all stored values
- Deterministic key format: `prefix/field1/field2` with `strconv.FormatUint`
- `app/app.go` + `cmd/wevibed` deliver runnable Cosmos SDK app (CO-159)
- `wevibed init → gentx → collect-gentxs → start` produces single-node CometBFT network (CO-161)
- Docker compose validated; smoke test passes
- Integration tests prove full tx pipeline via MsgServiceRouter (CO-162)
- gRPC query services: 17 RPCs across 7 modules with REST gateway annotations (CO-163)
- Next: hub integration + multi-validator orchestration

**Chain repo structure (private):**
```
wevibe-server/
├── wevibe-chain/              # Go/Cosmos SDK - chain modules
│   ├── x/attestation/       # Merkle root submission (keeper + module + types)
│   ├── x/org/               # Registration, dynamic pricing
│   ├── x/serving/           # Operator registration, serving sets
│   ├── x/challenge/         # VRF challenges, slashing
│   ├── x/emissions/         # Pool distribution, work scoring
│   ├── x/receipt/           # Receipt proofs
│   ├── x/reputation/        # Developer aggregates
│   └── x/operator/        # Real operator staking, bond escrow, slash (CO-167)
└── wevibe-commitllm-bridge/   # Rust - CommitLLM ZK bridge
    └── src/
        ├── verification.rs   # Receipt verification
        ├── commitllm_types.rs # Stub types (CI-safe)
        └── ...
```

---

## Roadmap

### Sprint 22: Chain Production Hardening

| Task | Owner | Status | Notes |
|------|-------|--------|-------|
| Wire wevibe-chain modules into Cosmos SDK app (`app/app.go`, `wevibed`) | Chain eng | ✅ | CO-159 delivered; TxConfig + keeper wiring complete |
| Publish Docker compose for local validator stack | Chain eng | ⏳ | Pending — requires hub container + RPC/gRPC exposure |
| Integration tests between wevibe-hub and wevibe-chain | Platform eng | ⏳ | Needs Docker compose + mocked receipts |
| Draft validator setup / gentx documentation | Chain eng | ⏳ | Base commands captured; convert to runbook |

### Sprint 23: Emissions + Operator Rewards
- `x/emissions` module: fixed daily pool, operator/validator split, work-score formula, asymmetric gating
- `x/reputation` module (dormant by default): per-developer aggregates, activates on attested roots
- Bootstrap credit pool handling

### Sprint 24: CommitLLM Bridge Integration (Tier 1)
- `wevibe-commitllm-bridge` crate: receipt verification, attestation metadata generation
- Attestation hook in wevibe-mcp Merkle batching pipeline
- Merkle leaf format with optional provenance field

### Sprint 25: Session Proxy Attestation (Tier 2) + Difficulty Scoring
- Standalone session proxy for closed-weight models (Claude, GPT-4o)
- Proxy attestation: content-addressed turns, WeVibe signing key, transparency log
- Two-layer difficulty scoring: structural signal (model coefficient × turn-range × failed-alternative multiplier) + LLM grading
- LLM grading service: non-obviousness, specificity, reasoning progression axes

### Sprint 26: LLM Grading Fairness + Anti-Gaming
- Grading prompt engineering with fairness safeguards
- Catch system prompt injectors who manipulate grading context
- Catch reward-farming via trivial memories (console.log workarounds)
- Catch inflation via artificially extended sessions (LLM grades catch this)
- Counterfactual difficulty validation (P2)
- Developer reputation profile in dashboard

### Sprints 27-30: Integration + Public Release
S27: End-to-end chain integration with wevibe-hub/wevibe-mcp
S28: GitHub publication, npm/crates.io publication
S29: Developer reputation dashboard page, landing page
S30: Mainnet launch prep, token launch, federation design

### What We're Skipping
- VPS provisioning and docker-compose productionization (do later)
- Claude Code Tier 2 plugin (Cursor compat for free later)
- Cline VS Code extension (do later)
- Landing page + benchmark numbers for pitch (do later)

---

## Key File Paths

| File | Description |
|------|-------------|
| /Users/jerrysmith/Desktop/wevibe-workspace/PDP.md | This document (v2.20) |
| /Users/jerrysmith/Desktop/wevibe-workspace/S20-SESSION-CONTINUANCE.md | Sprint 20 session continuance |
| wevibe-server/wevibe-hub/ | Private hub API repo (Go) |
| wevibe-server/wevibe-dashboard/ | Private dashboard repo (Next.js) |
| wevibe-server/wevibe-infra/ | Private infra repo (Terraform) |
| wevibe-mcp/src/server.ts | MCP server -- Tier 1 (3 tools + 1 resource) |
| wevibe-mcp/src/retrieve-cli.ts | Shared retrieval CLI (both tiers) |
| wevibe-mcp/src/admin.ts | Admin CLI -- 16 commands |
| wevibe-mcp/src/guard.ts | Shared wevibe-guard invocation |
| wevibe-mcp/src/embedding.ts | Bundled nomic-embed-text ONNX |
| wevibe-guard/src/main.rs | Scanner CLI entry point |
| wevibe-guard/src/scanner.rs | YARA + credential + exfil scanning |
| .opencode/plugins/wevibe-guard.ts | OpenCode Tier 2 plugin |

**Private Repos (WeVibe-Network org):**
- https://github.com/WeVibe-Network/wevibe-chain
- https://github.com/WeVibe-Network/wevibe-commitllm-bridge

---

## Environment

- Docker services: wevibe-hub (:4440), wevibe-postgres (:5433), wevibe-qdrant (:6333)
- Ollama: :11434 (admin CLI only)
- PostgreSQL: user=wevibe, db=wevibe_hub
- Qdrant collections: `org_{orgID}_memories` (768-dim, cosine, REST-only port 6333), created lazily per org
- wevibe-guard binary: wevibe-guard/target/release/wevibe-guard

---

## Open Items

### P0 -- Sprint 22
- [x] Wire wevibe-chain modules into Cosmos SDK app (app.go + cmd/wevibed)
- [ ] Docker compose for local chain testing
- [ ] Chain integration tests with wevibe-hub

### P1 -- Pre-Partner
- [ ] Re-run adversarial benchmark with current pipeline
- [ ] Re-run 7-model benchmark
- [ ] Re-seed memories and benchmark post-wipe

### P2 -- Post-Partner / Pre-Chain
- Sprint 25: Session proxy attestation + difficulty scoring
- Sprint 26: LLM grading fairness + anti-gaming
- CO-137: wevibe-hub error response standardization (deferred)

### Explicitly Killed
- WeVibe-Retriever custom embedding model (representation collapse)
- Vocabulary-constrained query extraction (CO-084, reverted)
- Blind token keyword confidentiality (replaced by plaintext keywords)
- MCP elicitation as primary gate (replaced by two-tier architecture)
- Plugin custom tool approach (replaced by system prompt injection)
- Per-memory user approval prompts (replaced by install-time trust + wevibe-guard)

---

## Standing Rules

R-ONE-PATH: One transport, one design, no fallbacks or dual-handling.
R-LONGEVITY: No shortcuts. Every solution is the long-term solution.
R-ABORT: Stop and report, never improvise.
R-NO-DB-HACKS: Wipe and recreate, no ALTER TABLE.
R-TEST-OUTPUT: Verbatim test output in all reports.
R-SEARCH: Use web search before guessing.
R-IMPL-REPORT: Every CO produces an implementation report at canonical path.
R-TWO-TIER: Tier 1 (MCP) = universal baseline. Tier 2 (plugin) = invisible injection. Both call same shared layer. Never re-debate.
R-DETECT-NOT-JUDGE: wevibe-guard flags everything suspicious. Plugin blocks malicious. Flags are informational.
R-PLATFORM-SPECIFIC: Each coding agent gets its own Tier 2 plugin. All plugins call the same wevibe-guard binary and retrieve-cli. The binary and CLI are the shared layer; the plugin is the platform shim.
R-INVISIBLE: Tier 2 memories inject into system prompt. Agent never calls a tool. Agent never knows. Sub-second.
R-TASK-TOOL: Use Task tool for parallel work within COs when fixes touch different files/packages.

---

*PDP v2.20 -- Sprint 22 complete. CO counter at CO-170. wevibe-chain ships with 8 modules (all 10 test packages pass), real operator staking, Merkle root attestations via gRPC, and ValidatorUpdates from EndBlocker.*
