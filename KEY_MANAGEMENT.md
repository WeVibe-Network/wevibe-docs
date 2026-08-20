# Key Management

*Last reviewed: 2026-08-18. Supersedes the Sprint-32 (2026-06) edition.*

**Scope.** This document is the complete operational key-surface reference: who holds what, storage locations, derivation, the DEK/wrap lifecycle, loss scenarios, rotation, and known defects.

> **SECURITY — contents of this document.** This document describes DERIVATION, STORAGE, SHARING, and PROVENANCE only. It contains NO key material: no seeds, mnemonics, DEKs, KEKs, ciphertext, or plaintext values. Names, sizes, fingerprints, and paths only.

## Runbook & doc dispositions

Each sibling security doc resolves to exactly one authoritative home; nothing stays duplicated across the family (per the WO-KEYMAP-DOCFAMILY chart):

| Doc | Disposition | Note |
|---|---|---|
| `RECOVERY_RUNBOOK.md` | **RETIRED** | Its key-loss scenarios are absorbed here §18. |
| `RUNBOOK-EPOCH-SK-COMPROMISE.md` | **FLAG-REWRITE** | Stale: claims 2-of-3 shareholders reconstruct an epoch SK — they reconstruct K_master (Shamir 2-of-3), from which the epoch SK is then derived; the epoch SK itself is never split. The on-chain `compromised` status it references is unimplemented. |
| `RUNBOOK-PRE-RECOVERY.md` | **FLAG-STALE** | "wevibe-recover CLI not implemented" is now false — the capability is live as `wevibe-admin recover-org/setup-threshold/recover-threshold/recovery-status`. |
| `SECURITY-MODEL.md` | **Partial absorb** | The threat model and revocation scope stay there; its key-mechanics and wire-format sections are absorbed here. |
| `SECURITY.md` | **CROSS-REFERENCE** | Vulnerability-disclosure only — no key content. |

## Cross-reference (do NOT duplicate)

Canonical sources are linked by name from this document, never copied:

- `WHITEPAPER.md` Appendix A — derivation formulas.
- `SECURITY-MODEL.md` — threat model (and revocation scope).
- `SECURITY.md` — vulnerability disclosure.
- `LEADER_QUICKSTART.md` · `VALIDATOR_QUICKSTART.md` · `HUB-REBUILD.md` — operations.
- `MASTER.md` · `DECISIONS.md` — canon decisions.

## Key Inventory

Every secret across all components, one row per key. Sizes, paths, mode bits, algorithms, env-var names, DB columns, and fingerprints only — values are never recorded. Provenance uses the seven origins: **(a)** Keplr/Leap software wallet · **(b)** platform passkey/biometric authenticator · **(c)** OS keychain · **(d)** BIP39 mnemonic · **(e)** seed file (bench) · **(f)** in-process random/HKDF · **(g)** env var/config.

| Key | Component | Provenance | Curve | Created | Stored | Shared |
|---|---|---|---|---|---|---|
| **wevibe-mcp — org hierarchy + agent identity (the crypto heart)** | | | | | | |
| K_master | wevibe-mcp (leader device) | (f) | Symmetric 32B org root (no EC curve) | Org creation; 32B OsRng | `~/.wevibe/keys/keys.json` envelope account `org-{id}-master` (AES-256-GCM under machine-seed KEK); vault branch is production-dead | Never shared raw — recovery = BIP39 24-word ORG phrase (sole encoding) + Shamir 2-of-3 split |
| encKey(e) | wevibe-mcp | (f) HKDF child of K_master | Symmetric 32B | Per epoch via `deriveEpochKeys`: HKDF-SHA256, salt=32×0x00, info `wevibe-enc-` ‖ epoch 4-byte BE | Sealed per member into hub `key_envelopes.enc_envelope` (opaque) | Envelope to every active member — **DEAD-SHIPPED**: zero production consumers, never wraps a DEK |
| searchKey(e) | wevibe-mcp | (f) HKDF child of K_master | Symmetric 32B | Per epoch: HKDF-SHA256, salt=32×0x00, info `wevibe-search-` ‖ epoch 4-byte BE | Sealed per member into `key_envelopes.search_envelope` (opaque) | **DEAD-SHIPPED**: opened into `membership.searchKeys`, zero readers; blind-token consumer `compute_blind_token` has zero production callers |
| auditKey(e) | wevibe-mcp | (f) HKDF child of K_master | Symmetric 32B | Per epoch: HKDF-SHA256, salt=32×0x00, info `wevibe-audit-` ‖ epoch 4-byte BE | Never sealed anywhere | **DEAD-DERIVED**: never consumed; hub `audit_log.encrypted_entry` has zero writers and zero readers — no audit-log encryption exists |
| epoch_sk / umbral_pk | wevibe-mcp (leader device) | (f) HKDF child of K_master | secp256k1 — 32B canonical scalar / 33B compressed pubkey | On demand per epoch: HKDF-SHA256(K_master, salt=∅, info `wevibe-umbral-epoch-`+decimal epoch), output used VERBATIM as the scalar (canonical, reject-not-adjust) | Never persisted — leader process memory only | `epoch_sk` never leaves the leader device (**CONSUMED**: kfrag mint + recall decrypt); hub stores only `umbral_pk` (`epoch_manifests.umbral_pk`); `rotateEpoch` omits it → empty e+1 manifest (live defect) |
| SK_mod / PK_mod | wevibe-mcp | (f) fresh in-process X25519 | X25519 (`StaticSecret` 32B / 32B pub) | Org creation; fresh pair minted on every `rotateEpoch` | keys.json envelope account `org-{id}-mod-privkey` | `mod_envelope` sealed to leader + moderators' identity X25519; PK_mod public in epoch manifest |
| identity-seed-v1 | wevibe-mcp (every agent) | (f) `randomBytes(32)` first run | 32B seed — root of identity Ed25519 (seed verbatim) + X25519 (HKDF `wevibe-x25519-v1`) | First run; write/use gated by fail-open biometric prompt (keychain backend only) | OS keychain service `wevibe-network` account `identity-seed-v1`; file backend keys.json (bench/container default) | Raw seed never leaves device; carries its OWN 24-word IDENTITY BIP39 phrase (SEPARATE from the org phrase); wrappers: passkey PRF KEK, wallet KEK, pairing blob |
| pre-identity-key (PRE) | wevibe-mcp (every agent) | (f) in-process random | secp256k1 — 32B canonical scalar / 33B compressed pubkey (invalid-scalar retry) | First run | keys.json (AES-256-GCM under machine-seed KEK) — ALWAYS file-backed | `pre_pubkey` registered with the hub as the kfrag target; secret used for Umbral recall decrypt |
| X-Agent-Signature key | wevibe-mcp (sole producer; the plugin signs nothing) | (f) = identity Ed25519 | Ed25519 — privkey is the 32B identity seed ITSELF (verbatim); 64B sig | On demand in `loadIdentity` (biometric gate) | Via identity-seed-v1 (same store) | Signs raw JSON request bodies; hub `ed25519.Verify` — 401 on mismatch; membership keyed by ed pubkey hex |
| org serve/event key | wevibe-mcp | (f) HKDF child of identity seed | Ed25519 (derived) | On demand per org: HKDF-SHA256(seed, salt `wevibe-org-serve-key-v1-salt`, info `wevibe-org-serve-key-v1:{orgId}`) | Not persisted — derived on demand | Signs canonical serve/event/denial bodies; pubkey on-chain as `serve_key_pubkey` (x/serve receipt state) |
| per-memory DEK | wevibe-mcp contributor; dashboard submit path | (f) in-process random | Symmetric 32B AES-256-GCM content key | OsRng 32B per memory | Never at rest in plaintext: `wrapped_dek_mod` = `seal_to_pubkey(DEK, PK_mod)`; Umbral capsule+ct in hub `pending_submissions`; chain gets opaque ciphertext + content-free events only | DEK → seal_to_pubkey(DEK, PK_mod) → Umbral capsule → ReEncrypt(capsule,kfrag) → cfrag → decrypt_reencrypted → DEK → AES-256-GCM — **NO K_enc hop** (`wrapped_dek_enc` is a legacy label for the same mod-wrapped blob) |
| machine-seed.bin (KEK input) | wevibe-mcp | (f) `randomBytes(32)` first run | Symmetric KEK input — KEK = SHA-256(`wevibe-keystore-v1` ‖ seed): single unsalted SHA-256, no KDF (known construction gap) | First run | `~/.wevibe/keys/machine-seed.bin` (0600, plaintext beside keys.json) | Never shared; AES-256-GCM-wraps keys.json |
| device-key-v1 | wevibe-mcp | (f) in-process random | Symmetric 32B DEK | First run | keys.json envelope (file backend) | Never shared; at-rest key of the pending vault — DEKs written, never read by production code (dead end) |
| vault passphrase / KEK | wevibe-mcp | (c) OS keychain | Symmetric — KEK = Argon2id(passphrase; t:3, m:65536, p:4; salt 32B; dkLen 32) → AES-256-GCM | Via keychain item service `wevibe-vault` (macOS `security` tool / Linux secret-tool) | Passphrase in OS keychain (`wevibe-vault`); vault at `~/.wevibe/vault.enc` | Unlocks vault.enc only — production-dead: `createVault` has zero production callers, file absent on disk |
| Shamir shares (3) | wevibe-mcp (split) + wevibe-hub (custody) | (f) in-process split of K_master | Symmetric GF(256) 2-of-3 — 33B each | Leader `setup-threshold` | Hub `recovery_shares.sealed_share` (sealed to holder X25519, opaque; keyed org_id+share_index) | share1 → leader, share2/3 → holders' X25519; out via `GET /v1/orgs/{id}/recovery/shares/{pubkey}`; splits K_master ONLY (never epoch_sk — epoch recovery = BIP39(K_master)) |
| mcp-session-token | wevibe-mcp host | (f) in-process random | Symmetric 32B → 64-hex string | First MCP start; read-from-disk-first | `~/.wevibe/mcp-session-token` (0600) | Bearer on local API :4450 (plugin + dashboard server routes); stable across restarts (does NOT rotate); timingSafeEqual |
| **wevibe-hub — operator keys only: env vars are the ONLY raw secrets; DB has no SECRET column (all key columns PUBLIC or OPAQUE-CIPHERTEXT); hub NEVER holds epoch_sk / K_master / DEK / plaintext.** | | | | | | |
| HUB_NODE_PRIVKEY | wevibe-hub | (g) env var | Ed25519 (32B seed or 64B raw) | Operator-supplied; default = all-zeros 64-hex (docker-compose.yml:140 + code fallback receipts.go:70-71) — flagged CONFIRMED LIVE in AUDIT.md | Env → process var; never persisted | Signs usage-receipt commitments → `usage_receipts.node_signature`; pubkey NOT on-chain |
| WEVIBE_CHAIN_SUBMITTER_MNEMONIC (submitter/serving key) | wevibe-hub | (d) BIP39 mnemonic via env (required — FATAL if unset; dev default = public 12-word test mnemonic) | secp256k1 — HD `m/44'/118'/0'/0/0` in an in-memory keyring | Hub boot | Env var only; keyring `BackendMemory` — never persisted | ONE global key is every org's default serving address; signs serve/deny/event batches; address = on-chain `StoredOrg.hub_serving_address` (field 14); leader-whitelisted, revocable via `MsgSetServingKey` |
| WEVIBE_HUB_RESPONSE_SEED (hubsign response key) | wevibe-hub | (g) env var | Ed25519 (32B seed, lowercase hex) | Operator-supplied; if unset EPHEMERAL — pubkey churns every boot | Env var only | Signs every hub response `X-Hub-Signature` (sha256 body); pubkey served at `/v1/hub/serving-address` → on-chain `StoredOrg.hub_response_pubkey` (field 19) |
| FAUCET_MNEMONIC | wevibe-hub (DEV-ONLY) | (d) BIP39 mnemonic via env | secp256k1 | Dev setup | Env var only | Dev/dogfood faucet funding |
| STRIPE_SECRET_KEY | wevibe-hub | (g) env var | N/A — third-party API key (not a WeVibe crypto key) | Operator-supplied | Env var only | Billing (Stripe) |
| WEVIBE_QDRANT_API_KEY | wevibe-hub | (g) env var | N/A — third-party API key | Dev default = well-known dev string | Env (compose forwards `QDRANT_API_KEY`) | Qdrant embedding-store auth |
| **wevibe-chain — node-operator scope: the chain NEVER holds/derives/sees any private key — on-chain state is pubkeys + content-free events only.** | | | | | | |
| validator consensus key | wevibe-chain node | (d) BIP39 mnemonic via `wevibed init` (`InitializeNodeValidatorFilesFromMnemonic`; lazy at `start`) | Ed25519 CometBFT FilePV (64B Go privkey = 32B seed + 32B pub) | Node init | `~/.wevibed/config/priv_validator_key.json` (+ `data/priv_validator_state.json`) | Never leaves the node; signs consensus votes/proposals ONLY — never org content |
| node p2p key | wevibe-chain node | (d) same init/start generator | Ed25519 | Node init | `~/.wevibed/config/node_key.json` | Never leaves the node; p2p transport auth |
| keyring account/wallet keys | wevibe-chain node operator | (d) BIP39 mnemonic via `wevibed keys add/import/import-hex` | secp256k1 32B — BIP44 `m/44'/118'/0'/0/0`, HRP `wevibe` | Operator command | Keyring backend, default `os` → macOS Keychain service `cosmos` (file/test/pass/kwallet alternatives) | Account-layer txs only (create-validator, gentx, bank/gov/distr/slashing) — never org content; the node process itself holds NO keyring account |
| **wevibe-dashboard (browser) — holds the identity seed (wrapped, plus a tab-scoped raw cache) and transient DEKs; the wallet and the authenticator hold their own keys, never extractable.** | | | | | | |
| chain account signing key (wallet-held) | wevibe-dashboard → Keplr/Leap | (a) software wallet | secp256k1 | Wallet setup (outside the dashboard) | NEVER in the browser — stays in the extension keystore; browser sees pubkey + bech32 address only | Signs chain txs via CosmJS `directBroadcast`; the hub never signs |
| passkey credential | wevibe-dashboard | (b) platform authenticator | WebAuthn ES256/RS256 resident credential (UV-required, PRF extension; RP-ID = hostname) | Passkey create | In the authenticator (iCloud/Google sync) | Privkey never leaves the authenticator; `credentialIdB64` → hub blob key |
| passkey PRF KEK | wevibe-dashboard | (b) PRF output of the passkey credential | Symmetric 32B — HKDF-SHA256(PRF_output, salt=random 32B per-wrap, info `wevibe-seed-kek-v1`) → AES-256-GCM | Per wrap | Never persisted (re-derived on demand; salt travels in the envelope) | Nothing shared; wraps the 32B identity seed; blob → IndexedDB + hub identity blob; missing PRF support = hard throw (drift vs D-IDENTITY-CARRIER cl.7) |
| wallet-derived KEK | wevibe-dashboard | (a) Keplr/Leap `signArbitrary` | Symmetric 32B — sig over fixed message `wevibe-identity-seed-wrap-v1` → SHA-256 → HKDF-SHA256(salt=random 32B, info `wevibe-seed-kek-wallet-v1`) → AES-256-GCM | Per wrap | Never persisted; zeroized after use | Wraps the same identity seed (kind `wallet`, credentialIdB64 `wallet:`+addr); hardware wallets (Ledger/Keystone) give non-deterministic sigs — declared, NOT enforced |
| X25519 identity keypair | wevibe-dashboard + wevibe-mcp (same derivation from the same seed) | (f) in-process HKDF of identity seed | X25519 — HKDF-SHA256(seed, salt=∅, info `wevibe-x25519-v1`), 32B scalar / 32B pub | On demand per use | Memory only, zeroized | xPub only; the ECDH side of envelope seal/open |
| pairing secret (16-byte) | wevibe-dashboard | (f) in-process random (`crypto.getRandomValues(16)`) | Symmetric 16B → 26-char base32 code; KEK = HKDF(secret, salt, info `wevibe-pair-v1`) → AES-256-GCM | Per pairing code | Transient; hub `pairing_blobs` keyed by SHA-256(secret) — single-use, 15-min expiry | One-time identity-seed handoff across devices; the seed never touches the Next.js server |
| **wevibe-bench — commissioned benchmark identity on :4550 (never :4450, the operator's real keychain MCP).** | | | | | | |
| bench leader seed | wevibe-bench (bench MCP :4550) | (e) seed file (bench) | Ed25519 — privkey = seed VERBATIM (32B); fp = sha256(pubkey)[:8]; plus a one-way secp256k1 HD child `m/44'/118'/0'/0/0` (bench wallet signer) | Commissioned from `~/.wevibe/bench/leader-seed.txt` (65B, 0600); env override `WEVIBE_BENCH_MCP_SEED` takes priority | Bench keystore `~/.wevibe/bench/leader-keystore/keys.json` (FILE backend via `WEVIBE_SEED_BACKEND=file`); public sidecar `identity.json`; `leader-mnemonic.txt` is a legacy artifact with no readers (mnemonic now derived in-memory from the seed for `import-identity`) | Bench identity seam: canonical fp `aa2aa706` = fp(ed_pubkey), `0e93b599` = fp(seed). **STALE, never re-adopt:** fp literal `f7733d6e` (pre-reseed leader; survives only as a stale comment at lconfig.py:27 (no live hardcoded literal)) and sidecar fp `90aab0a8` (bench identity.json, pre-rotation) |

**Not a key / not stored** — components with no key material of their own:

| Component | Verdict | Note |
|---|---|---|
| wevibe-sdk (library) | Holds NO keys | Pure library: all key material enters via parameters (`from_secret_bytes`, `from_bytes`, `sign`, `seal_to_pubkey`/`open_envelope`); `LocalIdentity` is ZeroizeOnDrop; zero keychain/fs writes. secp256k1 API = k256 `SecretKey::from_bytes` (big-endian, canonical-rejecting — semantically the Umbral `try_from_be_bytes`) |
| wevibe-guard | Holds NO keys | YARA-X content scanner; advisory fail-open; no crypto surface |
| wevibe-protocol | Holds NO live keys | Shared protobuf/contracts; `test_vectors/` fixtures are public deterministic test data — no live secrets |
| wevibe-social-graph | Holds NO private keys | Verify-only: custom sha256 → stdlib `ecdsa.Verify` over r‖s (64B) → `btcec/v2` 33B-compressed pubkey → bech32 HRP `wevibe` |

(Also key-less: `wevibe-opencode-plugin` — stateless, Bearer-token + public sidecar only, signs nothing; `wevibe-biometric` — fail-open user-presence ceremony that creates, holds, and derives nothing.)

**Invariants carried by this table (exact):**

1. **Epoch keys — 4 from K_master via HKDF-SHA256.** encKey/searchKey/auditKey: salt = 32×0x00, info `wevibe-enc-` / `wevibe-search-` / `wevibe-audit-` + epoch 4-byte BE, 32B output. Umbral epoch key: salt = ∅, info `wevibe-umbral-epoch-` + decimal epoch, output used VERBATIM as a secp256k1 scalar. Status: only `epoch_sk` is CONSUMED; enc/search are DEAD-SHIPPED; audit is DEAD-DERIVED.
2. **Three DISTINCT curves.** SK_mod/PK_mod = X25519 · Umbral epoch key + member pre-identity-key = secp256k1 (32B scalar / 33B compressed pub) · identity = Ed25519 (seed verbatim) + X25519 (HKDF `wevibe-x25519-v1`). No crossover in code.
3. **DEK wrap has NO K_enc hop:** DEK → seal_to_pubkey(DEK, PK_mod) → Umbral capsule → ReEncrypt(capsule,kfrag) → cfrag → decrypt_reencrypted → DEK → AES-256-GCM. (`wrapped_dek_enc` is a legacy label for the mod-wrapped blob.)
4. **Two secrets, one encoding.** K_master (32B) → BIP39 24-word ORG phrase + Shamir 2-of-3. The identity seed (32B) is SEPARATE and carries its OWN 24-word IDENTITY phrase. Same encoder, unrelated material — never conflate them.
5. **Boundary:** the hub NEVER holds epoch_sk / K_master / DEK / plaintext; the chain holds pubkeys + content-free events only.

## Derivation Hierarchy

WeVibe holds exactly TWO unrelated secret roots, and every key in the system traces back to one of them: the per-org master key `K_master` (ORG tree) and the per-device `identity-seed-v1` (IDENTITY tree). They never derive from each other, never wrap each other, and share no material. All HKDF usage is SHA-256 with 32-byte outputs; the two trees deliberately use different salt schemes so no derived key can collide across domains.

**Trust boundary (derivation view):** the HUB never holds `epoch_sk`, `K_master`, any DEK, or any plaintext — it receives only public keys (`umbral_pk`), Umbral capsules, ciphertext, and opaque sealed envelopes. The CHAIN holds pubkeys + content-free events only. Every derived secret below is produced and consumed at the edge (leader/member MCP); nothing derived from `K_master` or the identity seed is ever transmitted.

### 1. ORG tree (root: `K_master`)

`K_master` is a 32-byte `OsRng` random secret, generated ONCE per org by the LEADER at org creation. It never leaves the leader machine except as (a) the BIP39 phrase, (b) Shamir shares, or (c) sealed per-member epoch-key envelopes.

```
K_master  (32B, OsRng random, per-org, leader-generated at org creation)
 ├─ (a) BIP39(K_master) ──────────────────► 24-word ORG recovery phrase
 │        (bip39::Mnemonic::from_entropy; shown once)
 │
 ├─ (b) Shamir 2-of-3 split OF K_master ──► 3 sealed shares (GF(256), 33B each)
 │        (splits K_master itself — never any epoch key)
 │
 └─ per epoch e — two DISTINCT derivation schemes, both on the leader machine:
      │
      ├─ (c) SCHEME A — deriveEpochKeys (SDK core; TS wrapper is a WASM passthrough)
      │        salt = 32×0x00; epoch = 4-byte BIG-ENDIAN appended; 32B outputs
      │        K_enc(e)    = HKDF-SHA256(K_master, salt=32×0x00, info="wevibe-enc-"    ‖ epoch_BE4)
      │        K_search(e) = HKDF-SHA256(K_master, salt=32×0x00, info="wevibe-search-" ‖ epoch_BE4)
      │        K_audit(e)  = HKDF-SHA256(K_master, salt=32×0x00, info="wevibe-audit-"  ‖ epoch_BE4)
      │
      └─ (d) SCHEME B — epochUmbralSeed (MCP)
               salt = ∅ (empty); epoch = DECIMAL ASCII interpolated into the info string; 32B output
               epoch_sk(e)  = HKDF-SHA256(K_master, salt=∅, info="wevibe-umbral-epoch-<e>")
               epoch_sk is used VERBATIM as the secp256k1 scalar — NO second KDF,
               canonical-rejecting (non-canonical output → error, never clamped/adjusted)
                    └─ umbral_pk(e) = epoch_sk(e) · G   (secp256k1, 33B compressed, 0x02/0x03 prefix)
```

Schemes A and B differ in salt (32×0x00 vs ∅), epoch encoding (4-byte BE vs decimal ASCII), and epoch placement (appended bytes vs string-interpolated). Only the Scheme B output is a curve scalar; Scheme A outputs are symmetric keys.

**Derived-vs-consumed status (the net unambiguous statement).** Code derives 4 epoch keys from `K_master`; exactly 1 is actively consumed:

| Derived key | Scheme | Status | Note |
|---|---|---|---|
| `epoch_sk` / `umbral_pk` | B | **CONSUMED** | `epoch_sk` mints Umbral kfrags (delegating key) and decrypts re-encrypted capsules; `umbral_pk` is published to the hub epoch manifest and encrypts DEKs at approval |
| `K_enc(e)` | A | **DEAD-SHIPPED** | sealed into `enc_envelope` and distributed to every active member, but the unsealed map has ZERO production readers and `K_enc` never wraps a DEK |
| `K_search(e)` | A | **DEAD-SHIPPED** | sealed into `search_envelope` and distributed, but has ZERO readers; its natural blind-token consumer has zero production callers |
| `K_audit(e)` | A | **DEAD-DERIVED** | derived, then discarded — never sealed, never consumed; no audit-log encryption exists anywhere in the tree |

### 2. IDENTITY tree (root: `identity-seed-v1`)

`identity-seed-v1` is a 32-byte random secret, generated ONCE per DEVICE on first run, entirely SEPARATE from any org's `K_master`. It lives in the OS keychain (or the file-backed backend where that is the default, e.g. bench/container).

```
identity-seed-v1  (32B random, per-device, separate from K_master)
 ├─ Ed25519 privkey = seed VERBATIM            (SigningKey::from_bytes; no KDF)
 │      └─ signs X-Agent-Signature raw request bodies / membership identity
 ├─ X25519 privkey  = HKDF-SHA256(seed, salt=∅, info="wevibe-x25519-v1")
 │      └─ ECDH side of the identity: unseals this member's enc/search/mod key envelopes
 ├─ per-org serve/event key = HKDF-SHA256(identitySeed, info="wevibe-org-serve-key-v1:{orgId}")
 │      └─ Ed25519 key that signs canonical serve/event/denial bodies for that org
 └─ BIP39(identity-seed-v1) ──► its OWN 24-word IDENTITY recovery phrase
        (distinct from the org phrase; separate export/import path)
```

Note the member `pre-identity-key` (the Umbral PRE receiving key, secp256k1) is NOT part of this tree — it is an independently generated random scalar, not derived from the identity seed.

### 3. The "two secrets" clarification

Two UNRELATED 32-byte secrets — the org `K_master` and the device identity seed — share one BIP39 encoding path: both pass through the same function `bip39::Mnemonic::from_entropy` applied to two different 32-byte inputs. The identical encoding (24 words either way) is the single most common conflation to avoid: a 24-word phrase is just an alphabet for 32 bytes of entropy. The ORG recovery phrase encodes `K_master` (recovering every org epoch key, including the Umbral chain); the IDENTITY recovery phrase encodes the device identity seed (recovering only that device's signing/ECDH identity). Neither phrase can yield the other secret, and neither is related to any wallet key.

### 4. Curves — three DISTINCT, non-interchangeable

| Key | Curve | Secret encoding | Public encoding |
|---|---|---|---|
| Identity signing key | **Ed25519** | 32B seed VERBATIM | 32B |
| Identity X25519 key AND `SK_mod`/`PK_mod` (moderation envelope sealing) | **X25519** | 32B raw scalar | 32B |
| Umbral epoch key (`epoch_sk`/`umbral_pk`) AND member `pre-identity-key` | **secp256k1** | 32B canonical scalar | **33B compressed** (0x02/0x03) |

There is no crossover: Ed25519 only signs, X25519 only does ECDH (envelope sealing/unsealing), secp256k1 only serves the Umbral PRE path (delegate + receiver sides).

Key envelopes (the mechanism that carries `K_enc`/`K_search`/`SK_mod` to members) seal with an ephemeral X25519 ECDH → HKDF-SHA256(salt=32×0x00, info="wevibe-envelope-v1") → AES-256-GCM; wire blob = `eph_pub(32) ‖ nonce(12) ‖ ct+tag`.

### 5. DEK wrap chain — NO `K_enc` hop

A per-memory DEK (32B `OsRng` random) travels ONE chain; there is no symmetric re-wrap under `K_enc(e)` at any hop:

```
DEK ──► seal_to_pubkey(DEK, PK_mod)                 [X25519 envelope → wrapped_dek_mod]
    ──► Umbral capsule = umbral_encrypt(epoch umbral_pk, DEK)   [+ umbral_ciphertext]
    ──► ReEncrypt(capsule, kfrag)                   [→ cfrag]
    ──► decrypt_reencrypted(member_pre_sk, umbral_pk, capsule, cfrag)  → DEK
    ──► AES-256-GCM decrypt
```

The legacy field name `wrapped_dek_enc` is a LABEL ONLY for the same mod-wrapped blob — no re-wrapping occurs anywhere along write → hub → chain. `K_enc` survives solely as a (currently inert) key-distribution envelope key; the Umbral capsule is the confidentiality core.

### 6. Key provenance — the seven origins

Every secret in the system originates from exactly one of these seven mechanisms:

1. **Wallet** — Keplr/Leap software wallet (signing origin; any wallet-derived key is a one-way SLIP-10 HD child that cannot yield either tree's seed).
2. **Platform passkey** — WebAuthn passkey; its PRF output derives the identity-seed wrap KEK (info `wevibe-seed-kek-v1`) — a genuine seed origin (b). (Distinct from the MCP's native biometric gate, which is a user-presence ceremony, NOT a credential/origin.)
3. **OS keychain** — stores `identity-seed-v1` (service `wevibe-network`) and the vault passphrase.
4. **BIP39 mnemonic** — the two 24-word phrases encoding `K_master` and the identity seed; reconstruction path only.
5. **Seed file** — bench/commissioned deployments (e.g. leader seed file under the bench config dir).
6. **In-process random / HKDF** — `OsRng` generation (`K_master`, DEKs, `SK_mod`, identity seed, pre-identity-key) and every HKDF child above.
7. **Env var / config** — service-side secrets supplied via environment or config (hub node privkey, hub response seed, response/signing seeds).

## Curve Table

Every secret in WeVibe belongs to one curve family with one role. The table covers the full key
surface — memory-hierarchy keys, chain/validator keys, and the authenticator credential — with the
curve, on-wire encoding, and purpose of each.

| Key | Curve (library) | Encoding | Purpose |
|---|---|---|---|
| Identity signing key | Ed25519 (`ed25519-dalek`) | 32B seed (private key = seed **verbatim**) / 32B pub / 64B sig | Signs `X-Agent-Signature` (raw request body), the `WeVibe-Signed` header, and membership operations; the member identity asserted to hub and chain. |
| Identity X25519 key | X25519 (`x25519-dalek`) | 32B scalar / 32B pub | Derived from the identity seed via HKDF-SHA256 (salt=∅, info=`wevibe-x25519-v1`). ECDH for envelope sealing/unsealing — each member's enc/search/mod envelopes. |
| `SK_mod` / `PK_mod` | X25519 (`x25519-dalek`, `StaticSecret`) | 32B scalar / 32B pub | Moderation envelope sealing: contributors wrap the DEK with `seal_to_pubkey(DEK, PK_mod)`; the moderator/leader opens with `SK_mod`. A fresh X25519 pair is minted on every epoch rotation. |
| Umbral epoch key `epoch_sk` / `umbral_pk` | secp256k1 (`k256`, via `umbral-pre`) | 32B canonical scalar / 33B compressed pub (0x02/0x03 prefix) | Proxy re-encryption (PRE): `epoch_sk` is the delegating key that mints kfrags; `umbral_pk` encrypts approved DEKs into an Umbral capsule; `umbral_pk` is stored in the hub epoch manifest. |
| Member pre-identity-key `pre_sk` / `pre_pubkey` | secp256k1 (`@noble/secp256k1` TS / `k256` Rust) | 32B canonical scalar / 33B compressed pub (0x02/0x03 prefix) | PRE receiving key: the kfrag target; recall runs `decrypt_reencrypted(pre_sk, umbral_pk, capsule, cfrag, …)` to recover the DEK. |
| Chain keyring keys (`validator`, `hub-submitter`, `foundation`, `faucet`) | secp256k1 (Cosmos SDK keyring) | BIP39 mnemonic → BIP44 `m/44'/118'/0'/0/0` (Cosmos standard) | Transaction signing and submission — `hub-submitter` posts content-free events. The chain holds pubkeys + content-free events only; it never touches DEKs or plaintext. |
| Validator consensus key + p2p node key | Ed25519 (CometBFT) | CometBFT keyfiles: consensus key in `priv_validator_key.json`, node key in `node_key.json` | Consensus infrastructure only: block proposal/commit signatures and p2p peer identity. Not part of the memory-encryption hierarchy. |
| Passkey credential | ES256 / RS256 (WebAuthn) | Authenticator-held credential — not a WeVibe curve | Dashboard login and seed-wrapping. The credential private key is held by the platform authenticator and never exported to WeVibe. |

### Three distinct curves

Exactly **three distinct curves**, non-interchangeable, with no crossover in code:

- **SK_mod/PK_mod = X25519** · **Umbral epoch key + member pre-identity-key = secp256k1** (32B scalar / 33B compressed pub) · **identity = Ed25519 (seed verbatim) + X25519 (HKDF `wevibe-x25519-v1`)**.
- Ed25519 signs; X25519 does ECDH envelope sealing; secp256k1 carries PRE (Umbral + member receiving) and Cosmos transaction signatures — nothing else. In particular, the claim "every secret is a secp256k1 scalar" is false: the identity signing and sealing keys are not.
- **DEK wrap (no K_enc hop):** DEK → `seal_to_pubkey(DEK, PK_mod)` → Umbral capsule → `ReEncrypt(capsule, kfrag)` → cfrag → `decrypt_reencrypted` → DEK → AES-256-GCM. `K_enc` never wraps a DEK anywhere on this chain.
- **Epoch-key derivation and status.** The code derives **four** epoch keys from `K_master` via HKDF-SHA256. `enc`/`search`/`audit`: salt=32×0x00, info `wevibe-enc-` / `wevibe-search-` / `wevibe-audit-` + epoch as 4-byte big-endian, 32B output. `umbral`: salt=∅, info `wevibe-umbral-epoch-` + decimal epoch, output used as a **verbatim secp256k1 scalar** (canonical-rejecting; no second KDF). STATUS: only `epoch_sk` is CONSUMED (kfrag mint + recall decrypt); enc/search are DEAD-SHIPPED (sealed and distributed but zero readers); audit is DEAD-DERIVED (no consumer anywhere; no audit-log encryption exists).

### Encoding distinction

- X25519 and Ed25519 are **32B raw everywhere** — 32B seed/scalar private, 32B public.
- secp256k1 is asymmetric in encoding: **32B scalar private + 33B compressed public** (0x02/0x03 prefix). TS (`@noble/secp256k1` compressed mode) and Rust (`k256`) are byte-compatible on both sides.
- The X25519 envelope wire blob is `eph_pub(32) ‖ nonce(12) ‖ ciphertext+tag` (AES-256-GCM after an ECDH → HKDF-SHA256(salt=32×0x00, info=`wevibe-envelope-v1`) step).

### Two secrets, two recovery phrases

`K_master` (32B) → BIP39 24-word **ORG** phrase + Shamir 2-of-3. The identity seed (32B) is
**separate**, with its **own** 24-word **IDENTITY** phrase. The same BIP39 encoder renders two
unrelated 32-byte secrets — never treat the two phrases as one.

### Wallet-signature verification (verification note)

WeVibe's verifier-side wallet-signature scheme (wevibe-social-graph and its wevibe-hub twin) is a
**custom scheme, not ADR-036/signArbitrary** (no EIP-712): custom sha256 digest → stdlib
`ecdsa.Verify` over `r||s` (64B) → `btcec/v2` parse of the 33B compressed secp256k1 pubkey →
bech32 address binding with HRP `wevibe`. Verifiers hold no private keys. Known deviation carried
from the key-map audit: the address-hash preimage uses the 32B X-only slice (`pubkeyBytes[1:]`)
rather than the full 33B compressed pubkey, so it can never match a real chain-derived wallet
address — internally consistent, flagged pre-existing.

### Provenance and trust boundaries

The keys above originate from seven provenance classes: **(a)** Keplr/Leap software wallet;
**(b)** platform passkey/biometric authenticator; **(c)** OS keychain; **(d)** BIP39 mnemonic;
**(e)** seed file (bench); **(f)** in-process random/HKDF; **(g)** env var/config. Mapping: identity
seed ← (c) (file-backed on bench); `K_master`, `SK_mod`, DEKs ← (f); pre-identity-key ← (f);
epoch keys ← HKDF of `K_master` (f); chain keyring keys ← (d)/(g); passkey credential ← (b);
operator wallet ← (a). Trust boundaries: the **hub never holds `epoch_sk` / `K_master` / DEK /
plaintext** — it stores only pubkeys, sealed envelopes, capsules, and cfrags; the **chain holds
pubkeys + content-free events only**.

## DEK / KEK / Wrap Lifecycle

One random content key — the DEK — carries every memory payload write → hub → chain → recall. It is wrapped at exactly four boundaries, in exactly the order below (verified on-touch: WO-KEYMAP-FLOWS-CANON §2, WO-KEYMAP-P2-DERIVATION §3).

### Wrap order (exact)

```
WRITE — contributor                                           contribution.ts:78-86
  DEK             = random(32, OsRng)                          [crypto.rs:260-264]
  ciphertext      = AES-256-GCM(DEK, nonce, plaintext)         [crypto.rs:234-245]
  wrapped_dek_mod = seal_to_pubkey(DEK, PK_mod)                [contribution.ts:86]
      seal_to_pubkey := ephemeral X25519 ECDH
                       → HKDF-SHA256(salt=32×0x00, info="wevibe-envelope-v1")
                       → AES-256-GCM                           [crypto.rs:177-203]
      blob layout   = eph_pub(32) ‖ nonce(12) ‖ ct+tag

APPROVAL — moderator/leader                                   moderation.ts:237-245
  DEK = open_envelope(wrapped_dek_mod, SK_mod)                 [moderation.ts:237-240]
  {capsule, umbral_ciphertext} =
      umbral_encrypt(epoch umbral_pk, DEK)                     [moderation.ts:242-245;
                                                                umbral ops.rs:23-38]
  → hub pending_submissions.umbral_capsule / umbral_ciphertext [keyword_extraction.go:400-412]

PROVISION — leader                                            org-client.ts:810-824
  kfrag = generate_kfrags(delegating_sk = epoch_sk,
                          receiving_pk  = member pre_pubkey,
                          threshold     = (1,1))               [umbral ops.rs:68]
  → hub StoreKFrag (umbral sidecar kfrag store, keyed by (org, epoch, member_pk))

RECALL — member                                               org-client.ts:506-565
  cfrag = ReEncrypt(capsule, kfrag)                            [sidecar service.rs:103]
  DEK   = decrypt_reencrypted(receiving_sk  = member pre_sk,
                              delegating_pk = umbral_pk,
                              capsule, cfrag, umbral_ciphertext) [umbral ops.rs:91-139]
  plaintext = AES-256-GCM-decrypt(ciphertext, DEK)             [crypto.rs:247-258]
  (hub-side retrieval entry: retrieval.go:388)
```

Crypto.ts/SDK anchors above are `wevibe-sdk` core (`crypto.rs`) and `wevibe-mcp` TS (`contribution.ts`, `moderation.ts`, `org-client.ts`); umbral anchors are `wevibe-umbral` core (`ops.rs`, `service.rs`); hub anchors are `wevibe-hub` (`keyword_extraction.go`, `retrieval.go`).

**Curves at each hop — three DISTINCT, never interchangeable:** SK_mod/PK_mod = X25519 (moderation envelope seal/open) · Umbral epoch key (epoch_sk/umbral_pk) and member pre-identity-key (pre_sk/pre_pubkey) = secp256k1 (32B scalar / 33B compressed pub) · identity = Ed25519 (seed verbatim) + X25519 (HKDF "wevibe-x25519-v1"). All member key envelopes (enc/search/mod) seal to the member's identity X25519 pubkey.

**NO K_enc re-wrap hop.** `wrapped_dek_enc` IS `wrapped_dek_mod` — the same blob with zero re-wrapping: `WrappedDekEnc: wrappedDekMod` at moderation.go:1148 (`wevibe-hub/internal/api/handlers/`). The "enc" label travels write → hub → chain (chain-client.ts:200, tx.pb.go:193, redrive.go:149, retrieval.go:566) with zero re-wrapping at any hop. encKey (K_enc) is sealed into member envelopes (org-client.ts:642/941/1097 → `membership.encKeys` :366-373) but never wraps a DEK — dead-shipped with zero production readers.

### D-2.4 vs WHITEPAPER CODE-3

Code does D-2.4: the DEK is re-encrypted under the epoch public key via an Umbral capsule at approval (moderation.ts:242-245; DECISIONS.md:253-261, reinforced by D-LEADER-SIDE-UMBRAL-MINT :275-303, which explicitly marks the umbral info string "distinct from the K_enc/K_audit derivation" — K_enc survives only as a key-distribution envelope, never a DEK wrap). WHITEPAPER Appendix A (WHITEPAPER.md:1090,1102-1108) and §4.3 (:341) — `wrapped_dek_enc = AES-256-GCM(K_enc(e), nonce, DEK)` — is UNIMPLEMENTED; the whitepaper's own §4.5 (:355-383) already names the Umbral capsule as the confidentiality core, internally contradicting Appendix A. CODE-3's K_enc-wrap wording awaits a canon ruling to retire it (flagged open, P2 report §7).

### Epoch keys in this flow (consistency)

4 epoch keys derive from K_master via HKDF-SHA256, on the leader machine:

- K_enc(e) / K_search(e) / K_audit(e) = HKDF-SHA256(K_master, salt=32×0x00, info="wevibe-enc-" | "wevibe-search-" | "wevibe-audit-" + epoch as 4-byte big-endian), 32B output each [SDK crypto.rs:273-293].
- Umbral epoch seed = HKDF-SHA256(K_master, salt=∅, info="wevibe-umbral-epoch-" + epoch as decimal ASCII), used VERBATIM as the secp256k1 scalar — no second KDF; canonical-rejecting, never clamped [org-client.ts:53-63; umbral crypto.rs:50-61].

STATUS: only epoch_sk is CONSUMED (kfrag mint org-client.ts:810-824; recall decrypt :529-535). K_enc/K_search are DEAD-SHIPPED (sealed and distributed but inert); K_audit is DEAD-DERIVED (no consumer anywhere — no audit-log encryption exists in code).

### Root secrets (consistency)

TWO unrelated 32-byte secrets share one BIP39 encoding — never conflate them: K_master (32B) → BIP39 24-word ORG phrase + Shamir 2-of-3 (Shamir splits K_master, NOT the epoch SK) [recovery.ts:34-45; crypto.rs:304-308]. Identity seed (32B) = SEPARATE secret with its OWN 24-word IDENTITY phrase [export: admin.ts:269-284]. Same `bip39::Mnemonic::from_entropy` encoder applied to two different inputs.

### Who holds what along this chain (consistency)

HUB never holds epoch_sk / K_master / DEK / plaintext — hub stores umbral_pk (public), capsule, umbral_ciphertext, ciphertext, the wrapped_dek_mod blob, the embedding shadow, and opaque sealed envelopes only. CHAIN holds pubkeys + content-free events only. Scoping note (FLOWS-CANON §3/A5): the content-free guarantee governs the x/serve event log; the chain memory directory (`StoredMemoryCommitment`) holds ciphertext + wrapped-DEK blob by design, plus the bounded report plaintext/capsule path (`StoredMemoryReport`).

### KEKs (at-rest wrappers, distinct from the DEK wrap chain)

- **Envelope KEK** — ephemeral X25519 ECDH → HKDF-SHA256(salt=32×0x00, info="wevibe-envelope-v1") → AES-256-GCM; carries enc/search/mod/SK_mod envelopes in transit and to rest [crypto.rs:177-203].
- **Passkey PRF KEK** — HKDF info "wevibe-seed-kek-v1"; dashboard-side passkey/biometric-derived wrapper for the identity seed at rest. (Listed per dispatch spec; not anchored in the two source reports.)
- **Wallet KEK** — HKDF-SHA256(sig, info="wevibe-seed-kek-wallet-v1") + AES-256-GCM in the dashboard (`wallet-seed-wrap.ts`); wraps a fresh browser ed25519 seed; no wallet KEK exists in MCP/server.
- **Machine-seed KEK** — plain SHA-256("wevibe-keystore-v1" ‖ machine-seed.bin), NO KDF — FLAGGED: single unsalted SHA-256 with no iteration/memory-hardness [key-store.ts:71-85]; it AES-256-GCM-wraps `~/.wevibe/keys/keys.json` (12B nonce, :87-101), while `machine-seed.bin` (32B random, mode 0600) sits plaintext in the SAME directory, so any reader of the keys dir trivially recomputes the KEK. Exposure is a construction gap (no domain-separated KDF), not entropy; every other wrapper path uses domain-separated HKDF.
- **Vault key** — argon2id (t=3, m=65536, p=4) protecting the leader's `~/.wevibe/vault.enc` (PRODUCTION-DEAD — `createVault` has zero production callers; `vault.enc` absent on disk). (Params per dispatch spec; not anchored in the two source reports.)

Provenance of every secret/KEK in this chain resolves to exactly one of the 7 origins: (a) Keplr/Leap software wallet; (b) platform passkey/biometric authenticator; (c) OS keychain; (d) BIP39 mnemonic; (e) seed file (bench); (f) in-process random/HKDF; (g) env var/config. Mapping: wallet KEK ← (a) · passkey PRF KEK ← (b) · identity seed ← (c) keychain account 'wevibe-network' or keys.json · K_master and identity seed phrases ← (d) · bench leader seed ← (e) `~/.wevibe/bench/leader-seed.txt` · DEK / SK_mod / machine-seed.bin ← (f) · hub signer seeds ← (g) env names `HUB_NODE_PRIVKEY` / `WEVIBE_HUB_RESPONSE_SEED`.

## Provenance Matrix

Every WeVibe secret traces back to exactly ONE of seven origins. Origins answer "where did this key come from"; storage is a separate question (e.g. the identity seed is GENERATED under origin (f), stored AT REST under origin (c), and WRAPPED under origins (a), (b), (d)). The hub never holds epoch_sk, K_master, DEKs, or plaintext; the chain holds pubkeys and content-free events only — neither is an origin.

**The 7 origins:** (a) Keplr/Leap software wallet · (b) platform passkey/biometric authenticator · (c) OS keychain · (d) BIP39 mnemonic · (e) seed file (bench) · (f) in-process random/HKDF · (g) env var/config.

**Three DISTINCT curves — never conflate:**

| Curve | Carries |
|---|---|
| X25519 | moderation keypair SK_mod/PK_mod (moderation envelopes) |
| secp256k1 | Umbral epoch key (epoch_sk) + member pre-identity-key (`pre_pubkey` = Umbral PRE recipient) — 32B scalar / 33B compressed pubkey |
| Ed25519 + X25519 | identity: Ed25519 (seed verbatim) + X25519 (HKDF-SHA256, info "wevibe-x25519-v1") |

### (a) Keplr/Leap software wallet

The wallet signs; it never holds the identity seed. Keplr/Leap is dashboard-browser-only — `wevibe-mcp` has zero Keplr/Leap references.

| Key produced | Construction |
|---|---|
| Chain account signing key | user's own secp256k1 keypair, held INSIDE the Keplr/Leap extension; the browser sees pubkey + address only. Chain tx signing via CosmJS `directBroadcast`; the hub never signs (`wallet-connect.ts`, `chain-client.ts`). |
| Wallet-derived KEK | KEK = HKDF-SHA256(SHA-256(wallet signature over fixed message "wevibe-identity-seed-wrap-v1"), salt, info "wevibe-seed-kek-wallet-v1") → 32 B AES-256-GCM key wrapping the identity seed; zeroized after use. Wrapped ciphertext persisted in indexedDB `wevibe-dashboard/keys`, record `dashboard-identity`, kind `wallet` (`wallet-seed-wrap.ts`). |
| Bench leader secp256k1 HD child | one-way SLIP-10 `m/44'/118'/0'/0/0` from the leader seed (BIP39→PBKDF2→SLIP-10, `wallet.ts`) — a derivation that cannot yield the seed back. |

D-WALLET-AS-IDENTITY-SIGNIN re-roots sign-in in the wallet for wallet-holders, but remains LINK-not-DERIVE: the identity seed is still wrapped with a wallet-derived KEK; one identity per human. The pre-wallet PRE-key design (D-1.4) is amended: the PRE encryption key is generated client-side (passkey-protected), NOT wallet-BIP32-derived.

### (b) Platform passkey/biometric authenticator

Browser-dashboard only.

| Key produced | Construction |
|---|---|
| WebAuthn resident credential | ES256/RS256, resident + user-verification-required + PRF extension (`passkey.ts`); the privkey never leaves the authenticator (syncable via iCloud/Google). |
| Passkey PRF KEK | PRF output (fixed eval salt "wevibe-prf-eval-v1") → HKDF-SHA256(per-user salt, info "wevibe-seed-kek-v1") → 32 B AES-256-GCM KEK wrapping the identity seed; ciphertext + salt + iv uploaded to the hub. |

NOTE: the MCP's native biometric gate (`wevibe-biometric` napi-rs addon — Touch ID / Windows Hello) is a user-presence ceremony, NOT a credential — it derives nothing, creates nothing, holds nothing (fail-open; the OS keychain at rest is the real security floor, the prompt is defense-in-depth). Passkey/PRF lives in the browser dashboard only.

### (c) OS keychain

| Item | Details |
|---|---|
| `identity-seed-v1` | service "wevibe-network", account `identity-seed-v1`, via `@napi-rs/keyring` (macOS Keychain / Windows Hello credential store) — the default seed backend; base64 of the 32 B identity seed at rest; use/export gated by the biometric prompt (gate sites exist on the keychain backend only; none on file/test backends). |
| Vault passphrase | macOS Keychain service "wevibe-vault" (`security add-generic-password -s wevibe-vault`) / Linux secret-tool (`vault.ts`); unlocks the argon2id vault `~/.wevibe/vault.enc` (which would hold K_master — PRODUCTION-DEAD: `createVault` and `storePassphraseInKeychain` have zero production callers; `vault.enc` absent on disk). |

NOTE: the org key envelopes — accounts `org-{orgId}-master` and `org-{orgId}-mod-privkey` under the same service name — are FILE-backed (`~/.wevibe/keys/keys.json`), NOT OS keychain: `getStore()` returns the AES-256-GCM file store or the in-memory test store, never the OS keychain (`key-store.ts` `getStore`/`storeKeyEnvelope`). `device-key-v1` is file-backed the same way.

### (d) BIP39 mnemonic

TWO SECRETS share one encoding — two distinct 32-byte secrets, each with its OWN 24-word BIP39 phrase:

| Secret | Phrase | Handling |
|---|---|---|
| K_master (32 B) | the ORG recovery phrase (24 words) + Shamir 2-of-3 | stored in the file-backed envelope `org-{id}-master` in `~/.wevibe/keys/keys.json` (AES-256-GCM, machine-seed KEK); the ORG phrase is the display-once recovery path (the `~/.wevibe/vault.enc` argon2id vault is PRODUCTION-DEAD — `createVault` has zero production callers; file absent on disk). |
| Identity seed (32 B) | its OWN, separate 24-word IDENTITY phrase (entropy-verbatim, lossless) | break-glass export/import via MCP admin; never persisted (`admin.ts`). |

"Recovery phrase" is ambiguous without qualification: org phrase = K_master; identity phrase = identity seed. They are never interchangeable.

### (e) Seed file (bench)

| Item | Details |
|---|---|
| Seed file | `~/.wevibe/bench/leader-seed.txt` — one line, 64-hex = 32 B seed, mode 0600. Env override `WEVIBE_BENCH_MCP_SEED` takes priority (`lib.sh` `load_leader_seed`, which enforces 0600 and validates 64-hex). |
| Keys derived | Ed25519 identity (privkey = seed verbatim → pubkey) + SLIP-10 secp256k1 wallet key (one-way). |
| Fingerprints | fp = sha256(input)[:8]; a fingerprint comparison must name WHICH input was hashed: fp(seed_bytes) = `0e93b599`, fp(ed_pubkey_bytes) = `aa2aa706`. Same identity, different hashed inputs — an unfamiliar fingerprint alone is not evidence of a different identity. |
| Consumption | `bench-mcp.sh` derives the mnemonic in-memory from the seed → `import-identity --phrase` → MCP FILE backend (`WEVIBE_SEED_BACKEND=file`, keystore scoped by `WEVIBE_BENCH_LEADER_KEYSTORE`; headless, no Touch ID). The identity seam asserts the served fp equals the expected `aa2aa706` (verify-clean check 11, `mcp-fresh-4550`); an unreachable seam is a hard failure, not a skip. Bench runs against `:4550` only — never `:4450` (the operator's keychain identity). |
| Legacy artifact | `~/.wevibe/bench/leader-mnemonic.txt` (148 B, 0600) is a manual artifact; no code reads it — the mnemonic is derived in-memory from the seed since the script rewrite. |

### (f) In-process random/HKDF

**Freshly generated (randomBytes(32) / OsRng, 32 B unless noted):**

| Key | Notes |
|---|---|
| Identity seed | `randomBytes(32)` at creation (`key-store.ts` `generateIdentitySeed`); afterwards stored/wrapped per origins (a), (b), (c), (d). |
| machine-seed.bin | 0600; feeds the file-store DEK = sha256("wevibe-keystore-v1" ‖ machine-seed.bin) for `keys.json` AES-256-GCM. |
| device-key-v1 | 32 B DEK at rest for the pending vault; file-backed via `getStore()`. |
| pre-identity-key | 32 B secp256k1 scalar — the member pre-identity-key registered with the org. |
| Per-memory DEK | OsRng 32 B per memory. **NO K_enc hop** — recall path: DEK → `seal_to_pubkey`(DEK, PK_mod) → Umbral capsule → `ReEncrypt`(capsule, kfrag) → cfrag → `decrypt_reencrypted` → DEK → AES-256-GCM decrypt. |

**HKDF-SHA256 derivations:**

| Key | Construction |
|---|---|
| Epoch keys | FOUR from K_master via HKDF-SHA256. enc/search/audit: salt = 32×0x00, info = "wevibe-enc-" / "wevibe-search-" / "wevibe-audit-" + epoch 4-byte big-endian, 32 B output each. Umbral: salt = ∅, info = "wevibe-umbral-epoch-" + decimal epoch, output taken VERBATIM as the secp256k1 epoch scalar (`crypto.rs`, `org-client.ts`). STATUS: only epoch_sk is CONSUMED; enc/search are DEAD-SHIPPED; audit is DEAD-DERIVED. |
| Identity keypair | Ed25519 privkey = seed verbatim; X25519 = HKDF-SHA256(seed, salt = ∅, info "wevibe-x25519-v1") (`crypto.rs`). |
| Org serve key | HKDF-SHA256(identitySeed, salt "wevibe-org-serve-key-v1-salt", info "wevibe-org-serve-key-v1:{orgId}") → Ed25519 (`serve-signing.ts`). |
| Envelope KEK | X25519 ECDH → HKDF info "wevibe-envelope-v1" (`crypto.rs`). |
| Pairing KEK | HKDF info "wevibe-pair-v1" from the pairing secret (`pair-crypto.ts`). |
| Vault key | argon2id(passphrase; t=3, m=65536 KiB, p=4) — the only non-HKDF KDF in this origin (`vault.ts`). |

### (g) Env var/config

**Secret-bearing env vars (values never committed; e.g. gitignored `wevibe-bench/config/cloud.env`, 0600):**

| Env var | Secret |
|---|---|
| `HUB_NODE_PRIVKEY` | hub node ed25519 signing key. |
| `WEVIBE_CHAIN_SUBMITTER_MNEMONIC` | BIP39 mnemonic of the chain submitter account. |
| `WEVIBE_HUB_RESPONSE_SEED` | seed for the hub response-signing key (ed25519; ephemeral key when unset, `hubsign.go`). |
| `FAUCET_MNEMONIC` | BIP39 mnemonic of the faucet account. |
| `STRIPE_SECRET_KEY` | payment-provider API secret. |
| `WEVIBE_QDRANT_API_KEY` | vector-store API key. |
| `WEVIBE_BENCH_MCP_SEED` | bench leader seed (64-hex = 32 B); bench only; overrides the (e) seed file. |

**Selectors (non-secret, but they steer where secrets live):** `WEVIBE_SEED_BACKEND` (`keychain` default | `file`, required for bench) · `WEVIBE_KEYSTORE_PATH` · `WEVIBE_HOME`.

Across every origin the boundary holds: the hub never holds epoch_sk, K_master, DEKs, or plaintext, and the chain holds pubkeys and content-free events only.

## Who Sees What

The trust model is a strict split of knowledge: every secret has exactly one home, and every party that touches ciphertext does so without the key that opens it. The matrix below reflects the BUILT system as charted in the WO-KEYMAP reports (hub, chain, MCP/SDK, Umbral source; independent negative sweeps found zero hits for epoch_sk/K_master anywhere in the hub or chain repos).

| Party | Holds / sees | Never sees |
|---|---|---|
| Leader | K_master (32B, OsRng; stored in the file-backed envelope `org-{id}-master` in `~/.wevibe/keys/keys.json` (AES-256-GCM, machine-seed KEK)) and its BIP39 24-word ORG phrase + Shamir 2-of-3 shares of K_master; identity seed (32B — a SEPARATE secret with its OWN 24-word IDENTITY phrase); all four per-epoch keys derived on the leader machine only (see Invariants) — with epoch_sk the only one actually consumed; epoch_sk + umbral_pk; SK_mod/PK_mod (X25519, fresh random per epoch); minted member pre_keypairs (secp256k1) until provisioned; kfrag minting (delegating_sk = epoch_sk, receiving_pk = member pre_pubkey); the sealed envelopes it ships to the hub | Member plaintext without the member's pre_sk; hub/validator operator secrets (env keys, consensus key) |
| Member | pre_keypair minted by the leader's MCP (secp256k1: 32B scalar / 33B compressed pubkey); identity seed + IDENTITY phrase; its own sealed envelopes (enc/search/mod) sealed to its X25519 identity pubkey; the DEK transiently, only after `decrypt_reencrypted` succeeds for an authorized memory | K_master, the ORG phrase, epoch_sk, other members' DEKs/plaintext |
| Hub | umbral_pk (public); Umbral capsule + umbral_ciphertext; member ciphertext (`ciphertext_hex`); the `wrapped_dek_mod` blob; embedding vectors (the disclosed semantic shadow); opaque sealed `key_envelopes` (enc/search/mod per member) and sealed `recovery_shares`; serve_key_pubkey + serve_sig; content hashes (plaintext/ciphertext/wrapped-dek) + salt | epoch_sk, K_master, any DEK, any plaintext, member pre_sk — the hub is a carrier, never an attester; it holds zero org secrets in any DB column (every key-bearing column is PUBLIC or OPAQUE-CIPHERTEXT) |
| Umbral sidecar | Finished kfrags ONLY, keyed (org_id, epoch_id, member_pk), persisted to `/data/kfrags.json` (mode 0600, atomic rename); capsules/ciphertext transiently, solely as ReEncrypt inputs | epoch_sk, the epoch HKDF seed, K_master, any DEK, any plaintext, any member private key — its surface is 5 RPCs (StoreKFrag / ReEncrypt / DeleteKFrags / DeleteOrgKFrags / Health); no keygen, no decrypt |
| Chain | Pubkeys and addresses only (leader ed25519 pubkey; member ed25519/x25519 pubkeys; `hub_serving_address`; `leader_wallet_address`; `hub_response_pubkey`; `hub_endpoints`; consumer serve_key_pubkey; passkey_pubkey ↔ wallet alias) + the content-free, consumer-signed x/serve event log + policy anchors; the memory directory (`StoredMemoryCommitment`) holding ciphertext + wrapped DEK BY DESIGN; the bounded report path (`StoredMemoryReport`) carrying ≤4KB plaintext + capsule | Any private key, seed, or mnemonic; DEK; epoch_sk; K_master (full negative sweep: 0 proto fields, 0 commits ever touching a private key) |
| Browser (dashboard) | A browser-local ed25519 identity seed, wrapped under a wallet-derived KEK (HKDF-SHA256 over the wallet signature, info `wevibe-seed-kek-wallet-v1`, AES-256-GCM); the KEK exists only transiently in browser memory; the hub stores the resulting `dashboard_keys.pubkey` plus the wrapped identity-seed and pairing blobs as opaque ciphertext (`identity_blobs.ciphertext`, `pairing_blobs.ciphertext`) it cannot read; plaintext of memories its identity is authorized to recall | K_master, epoch_sk, hub/chain/operator secrets, other members' DEKs/plaintext — never held in usable form; the ORG phrase surfaces only transient, display-once (React state, never persisted); the wrapped seed blob is mirrored to the hub `/v1/identity/blob` but stays opaque (sealed, non-custodial) |
| Operator (validator / hub operator) | Hub env-var secrets — `HUB_NODE_PRIVKEY` (Ed25519 receipt key; default is an all-zeros value when unset), `WEVIBE_CHAIN_SUBMITTER_MNEMONIC` (required; secp256k1 HD m/44'/118'/0'/0/0; in-memory keyring only), `WEVIBE_HUB_RESPONSE_SEED` (Ed25519; ephemeral per boot if unset), plus dev-only/third-party env keys; validator consensus key (`priv_validator_key.json`, ed25519), p2p node key, and keyring account keys (default backend `os` → OS keychain) | K_master, epoch_sk, any DEK, member pre_sk, plaintext — operator keys sign ONLY consensus votes/proposals, receipt commitments, and gas-carrying tx envelopes; they never sign or open org content |
| Software wallet (Keplr/Leap) | Its own user-managed key material; the operator's signature over the sign-in challenge (inputs to the dashboard KEK derivation above); its on-chain address as a linked alias | K_master, the ORG phrase, epoch_sk, DEKs, plaintext — the relation is LINK-not-DERIVE: the wallet never derives any WeVibe org or identity key |
| Authenticator (passkey) | The passkey private key sealed inside the platform authenticator — it never leaves; signature outputs only; the chain stores just the ed25519 `passkey_pubkey`, and each `passkey_signature` is verified then discarded | Everything else — no WeVibe secret, ciphertext, or plaintext ever crosses the authenticator boundary |

### Boundary notes

- The hub previously held epoch_sk; it was removed in commit d2f1617. The invariant below is about the current tree: the hub now receives only umbral_pk and finished kfrags.
- Known defect tracked in the source reports (not a design boundary): `rotateEpoch` does not publish the new epoch's umbral_pk, so the hub stores an empty value and rotated-epoch recall/moderation break until that is fixed; kfrags are likewise NOT re-provisioned by rotation — `provisionRecall` must re-run per epoch.

### Invariants behind the split (stated exactly)

**Epoch keys.** Exactly 4 epoch keys derive from K_master via HKDF-SHA256, all derived on the leader machine. K_enc / K_search / K_audit: salt = 32×0x00 (32 zero bytes), info = `"wevibe-enc-"` | `"wevibe-search-"` | `"wevibe-audit-"` ‖ epoch as 4-byte big-endian, 32B output each. epoch_sk (Umbral): salt = ∅, info = `"wevibe-umbral-epoch-"` ‖ epoch as decimal string, output used VERBATIM as a secp256k1 scalar (no second KDF, no clamping). STATUS: only epoch_sk is CONSUMED (capsule + kfrag path); K_enc/K_search are DEAD-SHIPPED (derived and sealed into member envelopes but have zero DEK-wrap consumers in code); K_audit is DEAD-DERIVED (derived, never shipped or consumed).

**Curves — three DISTINCT ones.** (1) SK_mod/PK_mod = X25519. (2) Umbral epoch key + member pre-identity-key = secp256k1 (32B scalar / 33B compressed pubkey). (3) Identity = Ed25519 (device seed used verbatim) + X25519 (HKDF-SHA256 of the same seed, salt = ∅, info `"wevibe-x25519-v1"`). All key envelopes seal to members' X25519 identity pubkeys.

**DEK wrap — NO K_enc hop.** The one real path: DEK → `seal_to_pubkey(DEK, PK_mod)` → Umbral capsule → `ReEncrypt(capsule, kfrag)` → cfrag → `decrypt_reencrypted` → DEK → AES-256-GCM. Concretely: write = `wrapped_dek_mod = seal_to_pubkey(DEK, PK_mod)` (ephemeral X25519 ECDH → HKDF-SHA256, salt = 32 zero bytes, info `wevibe-envelope-v1` → AES-256-GCM); approval = DEK sealed into a capsule under the epoch umbral_pk; provision = `kfrag = umbral_generate_kfrag(epoch_sk, member_pre_pubkey)` at threshold (1,1); recall = hub-side `ReEncrypt(capsule, kfrag)` → member-side `decrypt_reencrypted(pre_sk, epoch_umbral_pk, capsule, cfrag)` → DEK → AES-256-GCM decrypt. There is NO epoch-K_enc AES re-wrap hop: `wrapped_dek_enc` is literally the same blob as `wrapped_dek_mod`.

**Two secrets.** K_master (32B) → BIP39 24-word ORG phrase + Shamir 2-of-3 over K_master. The identity seed (32B) is SEPARATE and has its OWN 24-word IDENTITY phrase. Epoch keys are never Shamir-split — they re-derive deterministically from K_master.

**Hub/chain negative invariant.** The HUB NEVER holds epoch_sk / K_master / DEK / plaintext. The CHAIN holds pubkeys + content-free events only — with the scoping note below for the memory directory.

**Provenance — 7 origins.** Every secret in this matrix traces to exactly one of: (a) Keplr/Leap software wallet; (b) platform passkey/biometric authenticator; (c) OS keychain; (d) BIP39 mnemonic; (e) seed file (bench); (f) in-process random/HKDF; (g) env var/config. K_master and every DEK are origin (f); org/identity phrases are (d); wallet-linked dashboard keys are (a); passkeys are (b); validator keyring accounts default to (c); hub operator keys are (g).

### Four Exit Guarantees

The doctrine: no party can read, withhold, rewrite, or kill an organization's knowledge — the chain is the sole durable authority over an otherwise disposable, rebuildable infrastructure, and the split above is what makes each guarantee hold. READ: the hub — the only party that touches every ciphertext in flight — never holds epoch_sk, K_master, a DEK, or plaintext (zero repo hits, both sweeps), so it cannot decrypt anything it carries; the chain holds pubkeys and content-free events plus ciphertext for which it stores no key; the sidecar holds kfrags that can re-encrypt but never decrypt; decryption terminates exclusively in a member's pre_sk, minted under the leader's K_master. WITHHOLD: ciphertext and the wrapped DEK are committed on-chain in the memory directory (`StoredMemoryCommitment`), so an authorized member can pull the encrypted object and its delegation capsule from the chain itself even if the hub disappears — the hub forwards, it does not escrow. REWRITE: every commitment carries content hashes and consumer signatures (`contributor_sig`, `serve_sig`) verified against on-chain pubkeys, with policy anchors versioned on-chain, so standing is recomputed at the edge from immutable events rather than from anything a hub or operator asserts. KILL: for the same reason — the encrypted knowledge lives on-chain, under keys no infrastructure party holds, so retiring, wiping, or losing the hub destroys only disposable state; K_master (recoverable via the ORG phrase or any 2-of-3 Shamir shares) is the only durable decryption root, and it never leaves the leader.

One precision note on "content-free": that guarantee scopes to the x/serve event log ONLY. The chain's memory directory (`StoredMemoryCommitment`) holds ciphertext + wrapped DEK by design, and the bounded report path (`StoredMemoryReport`) carries ≤4KB plaintext + capsule — both are committed, size-bounded, explicitly specified exceptions, not discretionary reads: no party gains the ability to open arbitrary content, and the four guarantees hold because the stored bytes remain under keys that stay on the leader's side of the boundary.

## Component: wevibe-mcp

`wevibe-mcp` is the crypto heart: the org LEADER's MCP generates and holds the org root secret, derives every epoch key, seals/opens DEKs, runs Umbral PRE provisioning and decrypt, and signs outbound requests. Key material lives under `~/.wevibe/` — `keys/keys.json` (AES-256-GCM file-backed envelopes under the `machine-seed.bin` KEK), `machine-seed.bin` itself, and the session-token file — plus the OS keychain for the identity seed. One correction over older docs that must be remembered: the org account names `org-{id}-master` and `org-{id}-mod-privkey` are FILE-BACKED ENVELOPES inside `keys.json` (service namespace `wevibe-network`) — NOT OS keychain items.

Provenance codes used below (7 origins): (a) Keplr/Leap software wallet; (b) platform passkey/biometric authenticator; (c) OS keychain; (d) BIP39 mnemonic; (e) seed file (bench); (f) in-process random/HKDF; (g) env var/config.

### Key inventory

| Key | Provenance | Created | Stored | Shared |
|---|---|---|---|---|
| `K_master` — 32B org master | (f) in-process random; leader, OsRng at org creation | org creation (`org-client.ts:627`) | file-backed envelope, account `org-{id}-master` in `~/.wevibe/keys/keys.json` (AES-256-GCM, machine-seed KEK) — NOT OS keychain | never shared; recovery is ONLY the BIP39 24-word ORG phrase + Shamir 2-of-3 shares |
| `encKey(e)` / `searchKey(e)` / `auditKey(e)` | (f) HKDF-SHA256 children of K_master (Scheme A) | epoch setup and every `rotateEpoch` | encKey sealed into `enc_envelope`, searchKey into `search_envelope` (per-member, to the member's identity X25519 pubkey); auditKey is never sealed anywhere | envelopes distributed to all active members at create/invite/rotate; status: enc/search DEAD-SHIPPED, audit DEAD-DERIVED (below) |
| `epoch_sk` / `umbral_pk` | (f) HKDF-SHA256 child of K_master (Scheme B) | derived on demand per epoch | nowhere — never persisted; only `umbral_pk` leaves the machine | `umbral_pk` → hub epoch manifest; `epoch_sk` stays leader-side as the kfrag delegating key |
| `SK_mod` / `PK_mod` (X25519) | (f) in-process random keypair | org creation (`org-client.ts:639`); FRESH pair minted at every `rotateEpoch` (`:1067`) | file-backed envelope, account `org-{id}-mod-privkey` in keys.json — NOT OS keychain | `PK_mod` is public (org manifest); `SK_mod` sealed into `mod_envelope` to leader + moderator identity X25519 keys only |
| `identity-seed-v1` — 32B | (f) in-process random, first run per agent | first run | OS keychain (service `wevibe-network`, `@napi-rs/keyring`), biometric-gated via `requireBiometric` (fail-open; keychain backend only) — OR file backend keys.json when `WEVIBE_SEED_BACKEND=file` (g) (bench/container default) | never; root of identity Ed25519 (seed verbatim) + X25519 (HKDF info `wevibe-x25519-v1`); has its OWN 24-word IDENTITY phrase |
| `pre-identity-key` (pre_sk, secp256k1) | (f) in-process random 32B scalar (invalid-scalar retry) | first run | keys.json under machine-seed KEK — ALWAYS file-backed, never keychain (`auth.ts:17`) | pubkey (33B compressed) registered with hub as kfrag target; secret is the member's receiving_sk for recall decrypt |
| org serve/event key | (f) HKDF-SHA256 child of the identity seed | derived on demand per org | never persisted | pubkey embedded in canonical serve/event/denial preimages; signs serve/event bodies via @noble/ed25519 (`serve-signing.ts:95-128`) |
| DEK — 32B per memory | (f) in-process random (OsRng) | per-memory write by contributor | never raw: `wrapped_dek_mod` = seal_to_pubkey(DEK, PK_mod); after approval, Umbral capsule + ciphertext at hub | see wrap chain below; MCP never holds other members' DEKs |
| `machine-seed.bin` | (f) in-process random | first keystore use | `~/.wevibe/keys/machine-seed.bin`, 32B, mode 0600 — plaintext on disk in the SAME dir as keys.json | never; KEK = SHA-256("wevibe-keystore-v1" ‖ seed), a single unsalted SHA-256 with NO KDF; wraps keys.json via AES-256-GCM (12B nonce) |
| `device-key-v1` — 32B | (f) in-process random | first `getDeviceKey()` call (`key-store.ts:195-204`) | service `wevibe-network`, account `device-key-v1`, via the production FILE store (keys.json) — NOT OS keychain | never; wraps pending-vault DEK entries (`pending-vault.ts` — dead in prod, below) |
| vault (`~/.wevibe/vault.enc`) | (c) OS keychain passphrase (service `wevibe-vault` via `security find-generic-password`) | would be Argon2id {t:3, m:65536 KiB, p:4}, 32B salt, 32B dkLen → AES-256-GCM (`vault.ts:12-15`) | vault.enc if it existed; org `k_master_hex` inside | PRODUCTION-DEAD: `createVault` has zero production callers → vault.enc uncreatable in prod; file absent on disk; vault branch of `persistOrgKeys` silently no-ops |
| `mcp-session-token` | (f) in-process random (32B → 64-hex) | first use; read-from-disk-first thereafter | `~/.wevibe/mcp-session-token`, mode 0600 | Bearer auth on the local MCP API :4450; stable across boots — does NOT rotate |
| Shamir shares — 2-of-3 of K_master, 33B each | (f) GF(256) split of K_master (`recovery.ts:34-45`); splits K_master — never an epoch SK, never the identity seed | leader `setup-threshold` admin op (`admin.ts:723-763`) | hub `recovery_shares` table: share1 sealed to leader identity X25519, shares 2/3 to holder identity X25519 pubkeys | recovery quorum (any 2 shares reconstruct K_master); fetched via `GET /v1/orgs/{id}/recovery/shares/{pubkey}` (hub handler `GetRecoveryShare`) |

### Epoch keys — the full set of FOUR, exact derivations, status

`K_master` feeds exactly FOUR HKDF-SHA256 epoch-key derivations, in two deliberately distinct schemes.

**Scheme A — `deriveEpochKeys`** (true derivation site is the SDK Rust core `wevibe-sdk-core/src/crypto.rs:273-293`; MCP `crypto.ts:32-39` is a WASM passthrough):

- `encKey(e)` = HKDF-SHA256(K_master, salt = 32 zero bytes, info = "wevibe-enc-" ‖ epoch as 4-byte big-endian) → 32B
- `searchKey(e)` = HKDF-SHA256(K_master, salt = 32 zero bytes, info = "wevibe-search-" ‖ epoch as 4-byte big-endian) → 32B
- `auditKey(e)` = HKDF-SHA256(K_master, salt = 32 zero bytes, info = "wevibe-audit-" ‖ epoch as 4-byte big-endian) → 32B

**Scheme B — `epochUmbralSeed`** (`org-client.ts:53-63`) — distinct from Scheme A in every parameter:

- `epoch_seed(e)` = HKDF-SHA256(K_master, salt = ∅ empty, info = "wevibe-umbral-epoch-" + epoch as ASCII DECIMAL, 32B output)
- The 32B output is used VERBATIM as the secp256k1 scalar — `SecretKey::try_from_be_bytes` (wevibe-umbral core `crypto.rs:50-61`) is canonical-REJECTING (k256): a non-canonical output errors, never silently clamped or adjusted. No second KDF.

**STATUS — only ONE of the four is consumed:**

| Key | Status |
|---|---|
| `epoch_sk` / `umbral_pk` (Scheme B) | CONSUMED — kfrag mint (delegating_sk) + recall decrypt; only `umbral_pk` goes to the hub |
| `encKey` | DEAD-SHIPPED — sealed into `enc_envelope`, opened into `membership.encKeys` by `loadMemberships`, zero production readers; never wraps a DEK |
| `searchKey` | DEAD-SHIPPED — sealed into `search_envelope`, opened into `membership.searchKeys`, zero readers (the blind-token consumer `compute_blind_token` has zero production callers) |
| `auditKey` | DEAD-DERIVED — never sealed, never consumed; no audit-log encryption exists anywhere |

**Known defect (live):** `rotateEpoch` (`org-client.ts:1040-1196`) re-derives enc/search for e+1, mints a fresh X25519 SK_mod, and re-seals the enc/search/mod envelopes — but derives NO Umbral epoch keypair and its payload carries no `umbral_pk`. The hub stores an empty `umbral_pk`, and `getEpochUmbralPk` throws for every post-rotation epoch, breaking recall and moderation embed. Epoch 0 is unaffected (org setup sends `umbral_pk`).

### Curves — THREE distinct, no crossover

- `SK_mod`/`PK_mod` = **X25519** (moderation envelope sealing). `SECURITY-MODEL.md`'s "Umbral moderation keypair" wording for this key is stale/wrong.
- Umbral epoch key (`epoch_sk`/`umbral_pk`) AND member `pre-identity-key` = **secp256k1** (32B canonical scalar / **33B compressed** pub, 0x02/0x03 prefix).
- Identity = **Ed25519** (privkey = the identity seed VERBATIM, `SigningKey::from_bytes`) + **X25519** (HKDF-SHA256(seed, salt=∅, info="wevibe-x25519-v1")).
- X25519 and Ed25519 keys are 32B raw everywhere. Envelope sealing (the ECDH that carries enc/search/mod keys and SK_mod around): ephemeral X25519 ECDH → HKDF-SHA256(salt = 32 zero bytes, info = "wevibe-envelope-v1") → AES-256-GCM; wire blob = eph_pub(32) ‖ nonce(12) ‖ ct+tag.

### DEK wrap lifecycle — one chain, NO K_enc hop

DEK → seal_to_pubkey(DEK, PK_mod) → Umbral capsule → ReEncrypt(capsule, kfrag) → cfrag → decrypt_reencrypted → DEK → AES-256-GCM

1. WRITE (contributor): DEK = OsRng 32B; ciphertext = AES-256-GCM(DEK, plaintext); `wrapped_dek_mod` = seal_to_pubkey(DEK, PK_mod) — X25519 envelope (`contribution.ts:78-86`).
2. APPROVAL (moderator/leader): open_envelope(wrapped_dek_mod, SK_mod) → DEK; then {capsule, umbral_ciphertext} = umbral_encrypt(epoch umbral_pk, DEK) → stored at hub (`moderation.ts:237-245`).
3. PROVISION (leader): kfrag = generate_kfrags(delegating_sk = epoch_sk, receiving_pk = member pre_pubkey, (1,1)) → hub kfrag store (`org-client.ts:810-824`).
4. RECALL (member): cfrag = ReEncrypt(capsule, kfrag); DEK = decrypt_reencrypted(receiving_sk = member pre_sk, delegating_pk = umbral_pk, capsule, cfrag, umbral_ciphertext); plaintext = AES-256-GCM decrypt (`org-client.ts:506-565`).

There is NO K_enc re-wrap hop. `wrapped_dek_enc` is a legacy LABEL for the same mod-wrapped blob (`WrappedDekEnc: wrappedDekMod`, hub `moderation.go:1148`); the "enc" name travels write → hub → chain with zero re-wrapping at any hop. WHITEPAPER CODE-3's `wrapped_dek_enc = AES-256-GCM(K_enc, DEK)` is unimplemented; DECISIONS D-2.4 (Umbral capsule) is what code does.

### The two secrets — and the identity tree

Two UNRELATED 32-byte secrets share one encoding; conflating them is the classic error:

- **K_master (32B)** → BIP39 24-word ORG phrase (`recovery.ts:17-22`, `org-client.ts:688`) + the Shamir 2-of-3 split above.
- **identity-seed-v1 (32B)** = a SEPARATE secret with its OWN 24-word IDENTITY phrase (export: `admin.ts:269-284` `export-identity`).

Both encode via the SAME function (`bip39::Mnemonic::from_entropy`, SDK `crypto.rs:304-308`) applied to two different inputs.

```
identity-seed-v1 (32B)
  ├─ Ed25519 signing key = seed VERBATIM   → X-Agent-Signature / WeVibe-Signed / membership identity
  └─ X25519 = HKDF-SHA256(seed, salt=∅, info="wevibe-x25519-v1") → ECDH to unseal this member's envelopes

org serve/event key = HKDF-SHA256(identity seed, salt="wevibe-org-serve-key-v1-salt",
                      info="wevibe-org-serve-key-v1:{orgId}") → 32B Ed25519 seed (@noble/ed25519)
```

### What MCP holds vs never holds

- MCP as org LEADER: K_master (keys.json envelope), the derived epoch keys (enc/search/audit), epoch_sk (on demand), SK_mod, plus its own identity seed, pre-identity-key, and envelopes. Mints kfrags; re-seals envelopes on invite/rotate.
- MCP as MEMBER: its own identity seed + pre-identity-key + envelopes addressed to it (enc/search; mod_envelope only if moderator). Does NOT hold K_master or epoch_sk.
- MCP NEVER holds other members' DEKs: a member's DEK travels as an Umbral capsule encrypted to that member's pre_pubkey; plaintext materializes only inside the intended member's own MCP via decrypt_reencrypted with its own pre_sk.
- HUB NEVER holds epoch_sk / K_master / DEK / plaintext — it stores and forwards umbral_pk, wrapped blobs, capsules, cfrags, and sealed envelopes it cannot open. CHAIN holds pubkeys + content-free events only.

### X-Agent-Signature

The `X-Agent-Signature` key is the identity Ed25519 key (the identity seed itself — NOT the pre-identity-key). Signer: `org-client.ts:149,165` (signs the raw JSON request body); hub verifies with ed25519.Verify (`retrieval.go:46-90`) and 401s on mismatch. The org serve/event key above is a different HKDF child of the same seed and does NOT sign X-Agent-Signature.

### Dead code / cold ends on the key surface

- `auditKey` — derived at every `deriveEpochKeys`, never consumed.
- `membership.encKeys` / `membership.searchKeys` — populated by `loadMemberships`, zero production readers.
- epoch-sk / epoch-pk envelope re-store branch after `rotateEpoch` (`org-client.ts:1167,1174`) — written, never read back (dead defensive branch).
- `deleteKeychainItem` — zero call sites; no code path ever deletes a stored identity seed.
- `createVault` / `storePassphraseInKeychain` — production-dead; vault.enc uncreatable in prod (see table).
- `verifySessionToken` — unreachable (with `TOKEN_FILE_PATH`, `__resetTokenForTests`).
- Adjacent: pending-vault (`loadPendingDek`/`listPending`) writes DEKs under `device-key-v1` never read by prod code; `crypto.ts:77-83` splitSecret/reconstructSecret wrappers have zero callers (recovery.ts calls the WASM directly).

## Component: wevibe-hub

The hub (`wevibe-server/wevibe-hub`) is a store-and-forward relay plus chain submitter. Its key-custody boundary has two halves: a small set of operator secrets supplied as env vars — the hub's ONLY raw secrets — and per-org key material it stores exclusively in PUBLIC or sealed form. The hub derives no org keys and decrypts no org content.

Provenance codes used below (7 origins): (a) Keplr/Leap software wallet; (b) platform passkey/biometric authenticator; (c) OS keychain; (d) BIP39 mnemonic; (e) seed file (bench); (f) in-process random/HKDF; (g) env var/config.

### Operator keys — the hub's ONLY raw secrets

Every raw secret the hub holds is an env var read at boot; none is persisted by the hub.

| Env var | Key | Provenance | Stored | Role |
|---|---|---|---|---|
| `HUB_NODE_PRIVKEY` | Ed25519 receipt-signing key (32-byte seed or 64-byte raw, hex-encoded) | (g) env var/config | process env only | signs usage-receipt commitments → `usage_receipts.node_signature`; pubkey NOT on-chain |
| `WEVIBE_CHAIN_SUBMITTER_MNEMONIC` | secp256k1 submitter/serving key, HD `m/44'/118'/0'/0/0` | (d) BIP39 mnemonic delivered via (g) env | in-memory Cosmos keyring only, never persisted | signs serve/deny/event batches; address served at `/v1/hub/serving-address`; on-chain `StoredOrg.hub_serving_address` (field 14) |
| `WEVIBE_HUB_RESPONSE_SEED` | Ed25519 response-signing seed (32 bytes, lowercase hex) | (g) env; ephemeral (f) in-process random when unset | process env only | signs every response via `SigningMiddleware` → `X-Hub-Signature`; pubkey served at `/v1/hub/serving-address`, on-chain `StoredOrg.hub_response_pubkey` (field 19) |
| `FAUCET_MNEMONIC` | dev-only faucet funder key | (g) env | env only | funds dev/dogfood accounts |
| `STRIPE_SECRET_KEY` | vendor API key — NOT a WeVibe key | (g) env | env only | billing |
| `WEVIBE_QDRANT_API_KEY` | vector-DB API key | (g) env | env only | embedding-store auth |

Per-key notes:

- **`HUB_NODE_PRIVKEY` — LIVE RISK.** The shipped default is an all-zero 64-hex value, present at three sites: the compose default, the code fallback in `internal/receipts/receipts.go`, and the (empty) `.env.example`. Flagged CONFIRMED LIVE in `AUDIT.md`. Any deployment that does not set this variable signs usage-receipt commitments with a key known to everyone.
- **`WEVIBE_CHAIN_SUBMITTER_MNEMONIC`** is REQUIRED — the hub exits FATAL at boot if unset. It signs `MsgSubmitServeBatch` / `MsgSubmitDenialBatch` / event batches. The keyring is `BackendMemory` only; the mnemonic never touches disk via the hub.
- **`WEVIBE_HUB_RESPONSE_SEED`** is loaded by `hubsign.NewFromEnv()`. If unset the hub mints an ephemeral key in-process at boot, so its pubkey CHURNS EVERY BOOT and any previously registered `hub_response_pubkey` stops verifying. Set it in production.

### Per-org serving key model — ONE global key

The design-era per-org serving key (HD-derived per org from a hub master mnemonic at `m/44'/118'/0'/0/{account_index}`) was DELETED from the code (`bb883a8`, per D-ECON-STORAGE-MARKET amendments a11 + a13). The current model:

- ONE global secp256k1 key — the `WEVIBE_CHAIN_SUBMITTER_MNEMONIC` key — serves as every org's default serving address.
- The org leader whitelists it at registration and can revoke/rotate it via `MsgSetServingKey` (leader-revocable).
- The hub relays leader-authorized, hub-signed batches; it does not attest content.
- **No "hub master mnemonic" exists.** `WEVIBE_CHAIN_SUBMITTER_MNEMONIC` is the only mnemonic root in the hub. (`wevibe-hub/docs/TOPOLOGY.md:819-832` still documents the deleted two-key model and is stale.)

### DB-stored key material (`db/schema.sql`)

Classification rule: **NO SECRET column exists anywhere in the schema** — no column holds a raw seed, private key, DEK, or plaintext. Every key-bearing column is PUBLIC (verifiable data or a public component) or OPAQUE-CIPHERTEXT (sealed against the hub; the hub can store and forward it but cannot open it).

| Table.Column | Type | Class | Content |
|---|---|---|---|
| `epoch_manifests.umbral_pk` | BYTEA | PUBLIC | leader-supplied epoch Umbral pubkey (secp256k1, 33B compressed) |
| `epoch_manifests.pk_mod` | TEXT | PUBLIC | epoch modulation pubkey, X25519 public component |
| `pending_submissions.umbral_capsule` | BYTEA | OPAQUE-CIPHERTEXT | Umbral PRE capsule over the DEK |
| `pending_submissions.umbral_ciphertext` | BYTEA | OPAQUE-CIPHERTEXT | Umbral ciphertext half |
| `pending_submissions.ciphertext_hex` | TEXT | OPAQUE-CIPHERTEXT | AES-256-GCM ciphertext of memory content |
| `pending_submissions.wrapped_dek_mod` | TEXT | OPAQUE-CIPHERTEXT | DEK sealed to `PK_mod` (X25519 seal) |
| `rotation_buffer.ciphertext_hex`, `rotation_buffer.wrapped_dek_mod` | TEXT | OPAQUE-CIPHERTEXT | same classes held across epoch rotation |
| `usage_receipts` commitments + signatures | TEXT | PUBLIC | hash commitments + PUBLIC signatures (agent + node) |
| `audit_log.encrypted_entry` | TEXT | OPAQUE-CIPHERTEXT | zero writers/readers in current code |
| `key_envelopes.enc_envelope` / `search_envelope` / `mod_envelope` | TEXT | OPAQUE-CIPHERTEXT | sealed per member `(org_id, pubkey)`; hub code is pure pass-through with zero crypto |
| `identity_blobs.ciphertext` | TEXT | OPAQUE-CIPHERTEXT | sealed identity blob |
| `pairing_blobs.ciphertext` | TEXT | OPAQUE-CIPHERTEXT | sealed pairing blob |
| `recovery_shares.sealed_share` | TEXT | OPAQUE-CIPHERTEXT | sealed Shamir share; leader-signed write only |
| `recovery_shares.holder_pubkey` | TEXT | PUBLIC | share holder's identity pubkey |

Why the DEK columns are opaque to the hub — the DEK wrap path has NO `K_enc` hop:

> DEK → `seal_to_pubkey(DEK, PK_mod)` → Umbral capsule → `ReEncrypt(capsule, kfrag)` → cfrag → `decrypt_reencrypted` → DEK → AES-256-GCM.

The DEK is sealed directly to the epoch X25519 `PK_mod`; re-encryption runs in the Umbral sidecar; decryption happens on the member's device. The hub only stores and forwards `wrapped_dek_mod`, `umbral_capsule`, `umbral_ciphertext`, and `ciphertext_hex`.

What the envelopes seal — epoch keys from `K_master`. Four epoch keys are derived per epoch from `K_master` via HKDF-SHA256:

- enc / search / audit: salt = 32×`0x00`, info = `"wevibe-enc-"` | `"wevibe-search-"` | `"wevibe-audit-"` + epoch 4-byte big-endian, 32B output each.
- umbral: salt = ∅, info = `"wevibe-umbral-epoch-"` + epoch in decimal, output used VERBATIM as the secp256k1 scalar.
- STATUS: **only epoch_sk CONSUMED; enc/search DEAD-SHIPPED; audit DEAD-DERIVED.**

`key_envelopes` stores sealed per-member copies of these keys for delivery; the hub cannot open them. Recovery shares seal the org-side secrets: exactly TWO secrets exist — `K_master` (32B), rendered as the BIP39 24-word ORG phrase and split Shamir 2-of-3 (the shares in `recovery_shares.sealed_share` are those shares, sealed to each holder's pubkey) — and the identity seed (32B), a SEPARATE secret with its OWN 24-word IDENTITY phrase.

The columns above span three DISTINCT curve families:

1. `SK_mod` / `PK_mod` = X25519.
2. Umbral epoch key + member pre-identity key = secp256k1 (32B scalar / 33B compressed pub).
3. Identity = Ed25519 (seed verbatim) + X25519 (HKDF info `"wevibe-x25519-v1"`).

### What the hub does NOT hold

The hub previously held `epoch_sk`; it was **removed in `d2f1617`** (hub-side epoch keypair generation, kfrag minting, and `epoch_sk` response fields all deleted) — state "removed in d2f1617", not "never held". The current boundary:

- HUB NEVER holds `epoch_sk` / `K_master` / DEK / plaintext. What it does hold: `umbral_pk` + Umbral capsule + ciphertext + sealed key envelopes + sealed Shamir shares + the embedding shadow (vectors + metadata).
- CHAIN holds pubkeys + content-free events only.

### Umbral sidecar (`wevibe-umbral`)

- `StoreKFrag` stores FINISHED KFRAGS only — never `epoch_sk`. In-memory `DashMap` keyed `(org_id, epoch_id, member_pk)`, persisted to `/data/kfrags.json` (override: env `WEVIBE_UMBRAL_KFRAG_STORE`), file mode 0600, atomic rename.
- The hub-side code under `internal/umbral/` is the gRPC client only; no crypto state lives there.

## Component: wevibe-chain + wevibe-umbral

Trust boundary in one line: the chain STORES only pubkeys, addresses, signatures and
ciphertext, and VERIFIES signatures against caller-supplied or on-chain pubkeys — it never
holds, derives, or sees a private key. The umbral sidecar STORES only PRE re-encryption
fragments (kfrags) and can re-encrypt but not decrypt. The single secret key in this
boundary — the umbral `epoch_sk` — exists only in-process on the leader device.

### Node-operator scope (validator node)

Keys held by whoever runs a `wevibed` validator node. Provenance legend (7 origins):
(a) Keplr/Leap software wallet; (b) platform passkey/biometric authenticator; (c) OS
keychain; (d) BIP39 mnemonic; (e) seed file (bench); (f) in-process random/HKDF;
(g) env var/config.

| key | provenance | created by | stored at | curve / size | signs |
|---|---|---|---|---|---|
| Validator consensus key | (d) | `wevibed init` → `InitializeNodeValidatorFilesFromMnemonic` (lazily at `wevibed start`) | `~/.wevibed/config/priv_validator_key.json` (+ `data/priv_validator_state.json` signing state) | Ed25519 FilePV (64B Go privkey = 32B seed + 32B pubkey) | CometBFT votes/proposals ONLY |
| Node p2p key | (d) | same init/start path | `~/.wevibed/config/node_key.json` | Ed25519 | p2p transport auth (peer identity) |
| Keyring account/wallet keys | (d) / (a) / (g) | `wevibed keys add\|import\|import-hex` | keyring backends `os\|file\|test\|pass\|kwallet\|memory`, default `os` → macOS Keychain service "cosmos" (backend `file` → `keyring-file/`) | secp256k1 (32B privkey), BIP44 `m/44'/118'/0'/0/0` (coin type 118 default — `SetCoinType` never called), HRP `wevibe` | account-layer txs ONLY (bank/gov/distr/slashing, gentx, create-validator) |

Ops note: a dev machine that never ran a validator legitimately has no
`~/.wevibed/config/priv_validator_key.json` / `node_key.json`; these files exist only on
actual validator hosts.

**Stated boundary:** `wevibed start` loads FilePV + node key only — the node process holds
NO keyring account key. The consensus key NEVER signs org content: serve/org/memory content
is signed off-chain (consumer serve keys, leader wallet) and only *verified* on-chain.

### On-chain state (all PUBLIC)

Everything the chain persists is public material:

| field | module | type / size | what it is |
|---|---|---|---|
| `StoredOrg.leader` | org | Ed25519 pubkey (hex) | leader's member identity key |
| `StoredOrg.hub_serving_address` | org | bech32 address | org/hub serving key's auth address (secp256k1 account) |
| `StoredOrg.leader_wallet_address` | org | bech32 | leader wallet |
| `StoredOrg.account_address` | org | bech32 (slot-derived) | org's own account |
| `StoredOrg.hub_endpoints` | org | 1–3 http(s) URLs | hub endpoints (not a key) |
| `StoredOrg.hub_response_pubkey` | org | Ed25519, 32B lowercase hex | hub INTEGRITY key — hub signs responses, clients/chain verify |
| `StoredMemberRecord.pubkey` / `.x25519_pubkey` | org | Ed25519 pubkey / X25519 pubkey | member identity + key-exchange pubkeys |
| `serve_key_pubkey` (serve receipts + denials) | serve | 32B Ed25519 pubkey | consumer per-org serve key (below) |
| `StoredIdentityAlias.passkey_pubkey` / `.wallet_address` | identity | Ed25519 pubkey hex / bech32 | passkey ↔ wallet alias (below) |
| `wrapped_dek_enc`, `encrypted_blob` | memory | ciphertext | wrapped DEK + encrypted memory (opaque to the chain) |
| `capsule` | memory | Umbral PRE capsule | PUBLIC, the re-encryption input |
| `contributor_sig`, `serve_sig`, `passkey_signature` | memory/serve/identity | Ed25519 signatures | verified against stored or caller-supplied pubkeys |

> **Note:** `serve_key` and `umbral_pk` are NOT fields on `StoredOrg`. The serve key pubkey
> lives in the serve receipt state as `serve_key_pubkey`; `umbral_pk` is hub-side state
> (hub DB) and appears in no chain proto at all.

### Serve-key boundary — two keys sharing a name

|  | consumer per-org serve key | org/hub serving key |
|---|---|---|
| curve | Ed25519 | secp256k1 chain account |
| signs | event CONTENT (keeper verifies against `serve_key_pubkey`) | the gas-carrying tx ENVELOPE |
| on-chain record | `serve_key_pubkey` (serve receipt state) | `StoredOrg.hub_serving_address` |
| gate | signature verification | `requireServingKeySigner` ante check — envelope signer must equal `hub_serving_address` |
| fees | n/a | org feegrant pays gas |
| rotation | by the consumer | leader-only `MsgSetServingKey` |

### x/identity — passkey ↔ wallet alias

`StoredIdentityAlias` = `passkey_pubkey` (Ed25519, 32B hex — the store key) +
`wallet_address` (bech32), plus `is_migrated`/`migrated_at_epoch` bookkeeping. All public;
no secret is persisted — the accompanying `passkey_signature` is verified and then
discarded.

### The chain NEVER holds a private key

Verified by full negative sweep: 0 secret-shaped fields (`priv|secret|mnemonic|sk|seed|master`
across the wevibe module protos) and 0 commits introducing one (`git log -S private_key`).
The closest-to-secret data on-chain are ciphertext (`wrapped_dek_enc`, `encrypted_blob`),
the PUBLIC PRE `capsule`, and public signatures. Stated canonically: **CHAIN holds pubkeys
+ content-free events only.** And because the kfrag flow transits the hub: **HUB never
holds epoch_sk / K_master / DEK / plaintext** — it holds `umbral_pk` + kfrags only.

### wevibe-umbral — PRE re-encryption at the edge

**Epoch keys — 4 from `K_master` via HKDF-SHA256.** `K_master` is one of the TWO root
secrets: `K_master` (32B) → BIP39 24-word ORG phrase + Shamir 2-of-3; the identity seed
(32B) is SEPARATE with its OWN 24-word IDENTITY phrase. Canonical parameters per epoch e:

| key | salt | info | output | STATUS |
|---|---|---|---|---|
| enc | 32 zero bytes | `"wevibe-enc-"` + epoch 4-byte BE | 32B | DEAD-SHIPPED |
| search | 32 zero bytes | `"wevibe-search-"` + epoch 4-byte BE | 32B | DEAD-SHIPPED |
| audit | 32 zero bytes | `"wevibe-audit-"` + epoch 4-byte BE | 32B | DEAD-DERIVED |
| **umbral** | ∅ (empty) | `"wevibe-umbral-epoch-"` + epoch decimal | 32B, consumed VERBATIM as the secp256k1 scalar | **CONSUMED** |

Only `epoch_sk` is CONSUMED; enc/search are DEAD-SHIPPED; audit is DEAD-DERIVED.

**Epoch keypair + kfrags (minted leader-side):**

- `epoch_sk = SecretKey::try_from_be_bytes(epoch seed)` — canonical secp256k1 scalar
  (k256 0.13.4, transitive via umbral-pre 0.11.0), derived in-process on the leader device
  (MCP WASM). The `derive-epoch-keypair --seed <hex>` CLI consumes an already-derived
  32-byte seed; no HKDF or `K_master` lives in the umbral repo.
- `umbral_pk = epoch_sk·G` — 33B compressed secp256k1, sent to the hub.
- kfrag = PRE re-encryption fragment minted leader-side with `delegating = epoch_sk`,
  `receiving = member pre_pubkey` (the member's secp256k1 PRE pubkey). A kfrag can only
  `reencrypt(capsule, kfrag) → cfrag`; it can never decrypt. It is not itself a secret key.
- Dual-live: hub-side native sidecar (`wevibe-umbral:4460`, ReEncrypt-only) + MCP
  in-process WASM running byte-identical ops.
- The CLI/WASM emits the secret key in-process only — by design (the MCP needs it locally
  to mint kfrags); MCP-local, never persisted, never networked.

**Sidecar persistence — kfrags ONLY.** DashMap keyed `(org_id, epoch_id, member_pk)`,
flushed atomically to `/data/kfrags.json` (mode 0600); the persisted schema has no
secret-key field. RPC surface is exactly 5: `StoreKFrag`, `ReEncrypt`, `DeleteKFrags`,
`DeleteOrgKFrags`, `Health` — no keygen, no decrypt, no plaintext. The sidecar never holds
`epoch_sk`, the epoch seed, `K_master`, any member private key, or plaintext.

**DEK wrap path (NO K_enc hop):** `DEK → seal_to_pubkey(DEK, PK_mod) → Umbral capsule →
ReEncrypt(capsule, kfrag) → cfrag → decrypt_reencrypted → DEK → AES-256-GCM`.

### Canonical references for this component

- **Curves — three DISTINCT:** `SK_mod`/`PK_mod` = X25519 · Umbral epoch key + member
  pre-identity key = secp256k1 (32B scalar / 33B compressed pub) · identity = Ed25519
  (seed verbatim) + X25519 (HKDF info `"wevibe-x25519-v1"`).
- **Provenance origins (7):** (a) Keplr/Leap software wallet; (b) platform
  passkey/biometric authenticator; (c) OS keychain; (d) BIP39 mnemonic; (e) seed file
  (bench); (f) in-process random/HKDF; (g) env var/config.

## Component: wevibe-dashboard (browser)

The dashboard is the Next.js browser surface. Two theses govern its key inventory:

1. **One seed, many wrappers.** The 32-byte Ed25519 identity seed is the only true
   client-side key. Every at-rest wrapper (wallet-derived KEK, passkey PRF KEK) is
   re-derived on demand, never persisted, and zeroized after use.
2. **The browser delegates all org crypto.** The browser WASM exposes no Umbral
   functions, so K_master, epoch keys, SK_mod and the Umbral keypairs are all minted
   and held by the MCP; the dashboard only relays sealed envelopes it cannot open.

### Provenance legend (7 origins)

Every table row below tags its origin with one of the seven canonical provenances:
(a) Keplr/Leap software wallet · (b) platform passkey/biometric authenticator ·
(c) OS keychain · (d) BIP39 mnemonic · (e) seed file (bench) · (f) in-process
random/HKDF · (g) env var/config.

### Key inventory (browser surface)

| Key | Provenance | Created | Stored (browser) | Shared |
|---|---|---|---|---|
| Chain account signing key (secp256k1) | (a) Keplr/Leap software wallet | Wallet setup, outside the dashboard | **NEVER in browser** — extension-held; the browser sees pubkey + bech32 address only | Never; signing happens in-extension (`getOfflineSigner`, direct raw-RPC broadcast) |
| Identity ed-seed (32-byte Ed25519, seed verbatim) | (f) in-browser WASM `generateIdentity()`, or adopted from another device via pairing code | First unlock surface (second-mint escape hatch exists — see Flags) | (a) AES-256-GCM wrapped blob in IndexedDB `keys/dashboard-identity`; (b) **RAW base64 in sessionStorage `wevibe.session.seed.v1`** — accepted XSS tradeoff, tab-scoped, cleared on lock (see XSS note below); (c) memory `unlockedSeed` | Wrapped blob mirrored to hub `/v1/identity/blob` (opaque to the hub) |
| Wallet-derived KEK (AES-256-GCM wrap key) | (a) Keplr/Leap `signArbitrary` signature over the fixed message `wevibe-identity-seed-wrap-v1` | Per wrap | Never — re-derived on demand via HKDF-SHA256(signature → SHA-256, random 32B salt, info `wevibe-seed-kek-wallet-v1`), zeroized after use | Nothing; the random salt travels in the wrap envelope |
| Passkey PRF KEK (AES-256-GCM wrap key) | (b) platform passkey via WebAuthn PRF output | Per wrap | Never — re-derived via HKDF-SHA256(PRF output, salt, info `wevibe-seed-kek-v1`), zeroized | Nothing; stored discriminator is `kind === 'wallet'` vs absent (`kind:'passkey'` is never written) |
| Passkey credential (WebAuthn) | (b) platform authenticator (Touch ID / Windows Hello / hardware key) | Passkey creation | Not in browser — lives in the authenticator (iCloud/Google sync); RP-ID = `window.location.hostname` | `credentialIdB64` only (hub blob key) |
| X25519 identity keypair (xPriv/xPub) | (f) HKDF-SHA256 from the ed-seed, info `wevibe-x25519-v1` | Per use, on demand | Memory only; zeroized | xPub only |
| Pairing secret (16-byte) | (f) `crypto.getRandomValues` | Per pairing code | Transient only | 26-char base32 pairing code shown to the user; the seed is AES-256-GCM encrypted under HKDF(secret, salt, info `wevibe-pair-v1`) and uploaded to hub `/v1/pair` keyed by SHA-256(secret); single-use, 15-minute expiry |
| Per-memory content DEK | (f) in-browser random | Per memory submission | Transient; immediately sealed to `pk_mod` (X25519) | `wrappedDekMod` (Umbral capsule) → hub; full path below |
| K_master (32-byte org master DEK) | (f) in-process random, minted **inside the MCP** (`generateDek`) | MCP org-setup | **NOT browser** — MCP key envelope + OS keychain vault | The BIP39 24-word mnemonic surfaces in the browser **transient, display-once** (React state, never persisted); the browser never holds an org private key in usable form |

**XSS tradeoff (flagged):** the raw identity seed in sessionStorage
`wevibe.session.seed.v1` means any script-injection into the dashboard origin can read
the unlocked identity seed for the lifetime of the tab. This is a deliberate, accepted
tradeoff — tab-scoped, cleared on lock — but it is the one place raw key material is
DOM-readable, and it is the definitive answer to "does the browser store raw key
material": yes, tab-scoped.

### Curves present on this surface (three DISTINCT curves)

SK_mod/PK_mod = X25519 · Umbral epoch key + member pre-identity-key = secp256k1
(32B scalar / 33B compressed pub) · identity = Ed25519 (seed verbatim) + X25519
(HKDF `wevibe-x25519-v1`). On the browser surface that means: the wallet/chain
signing key and any Umbral epoch pubkey are secp256k1; the identity seed is Ed25519
with its derived X25519 sibling; and `pk_mod` (the DEK wrap target) is X25519.

### Browser HOLDS vs delegates

**Browser HOLDS:**

- the identity ed-seed (unlocked in memory, raw in sessionStorage, wrapped in IndexedDB)
- the X25519 identity keypair derived from it (memory only)
- per-memory content DEKs (transient, sealed to `pk_mod` immediately)
- the wrapped-seed ciphertext blob (safe — ciphertext)
- the K_master BIP39 recovery mnemonic, transient and display-once only

**Browser NEVER holds:**

- the wallet private key (stays in Keplr/Leap; the browser sees pubkey + address only)
- the KEKs (wallet-derived and passkey PRF) — re-derived on demand, never persisted,
  zeroized after use
- K_master / epoch_sk / Umbral keys / sk_mod — all MCP-side; the browser only relays
  sealed envelopes it cannot open

**Epoch keys (MCP-derived; stated here because the browser relays their envelopes).**
4 from K_master via HKDF-SHA256. enc/search/audit: salt=32×0x00, info
"wevibe-enc-"|"wevibe-search-"|"wevibe-audit-" + epoch 4-byte BE, 32B. umbral:
salt=∅, info "wevibe-umbral-epoch-"+decimal, VERBATIM secp256k1 scalar. STATUS: only
epoch_sk CONSUMED; enc/search DEAD-SHIPPED; audit DEAD-DERIVED.

**Two secrets, two recovery phrases.** K_master (32B) → BIP39 24-word ORG phrase +
Shamir 2-of-3. Identity seed (32B) = SEPARATE, OWN 24-word IDENTITY phrase. These are
two independent secrets with independent recovery paths — the org phrase never recovers
the identity seed and vice versa.

**Delegation of custody:** the wallet holds the chain signing key; the platform
authenticator holds the passkey credential + PRF secret; the MCP holds all org crypto
(sealed into envelopes, OS keychain); the hub holds only opaque wrapped-seed and
pairing blobs (non-custodial — unreadable).

**Non-custody invariants:** the HUB NEVER holds epoch_sk/K_master/DEK/plaintext.
The CHAIN holds pubkeys + content-free events only.

### Per-memory DEK wrap path (NO K_enc hop)

On submission the browser generates a fresh DEK and seals it directly to the modulus
pubkey; on recall the Umbral re-encryption path returns it. Exactly:
DEK → seal_to_pubkey(DEK, PK_mod) → Umbral capsule → ReEncrypt(capsule,kfrag) →
cfrag → decrypt_reencrypted → DEK → AES-256-GCM. There is no K_enc hop in this path —
the DEK is sealed to PK_mod, never to an epoch encryption key.

### Org-setup delegation (browser cannot run Umbral)

Org creation is delegated wholesale to the MCP: the dashboard's server-side route
calls `POST /v1/org-setup` (+ `/finalize`) at `WEVIBE_MCP_HTTP_URL` (default
`http://127.0.0.1:4450`) because the browser WASM has no Umbral functions. The
response carries the org pubkeys + MCP signature + sealed envelopes +
`recovery_phrase` (the K_master BIP39 24-word ORG mnemonic), which the dashboard
displays once from React state and never persists. The MCP Bearer credential is
`~/.wevibe/mcp-session-token`, read server-side by the dashboard route — it never
reaches the browser context.

### Flags and open drift

1. **HW-wallet KEK non-reproducibility NOT enforced at runtime.** Ledger/Keystone
   produce non-deterministic `signArbitrary` signatures, so a wallet-derived KEK is
   not reproducible across sessions on hardware wallets; but `isNanoLedger` /
   `isKeystone` are declared and never read — there is no runtime guard, and a
   hardware wallet merely fails later with `WalletUnlockMismatchError`. Alpha =
   software wallets only, documented but unenforced.
2. **PRF hard-throw un-corrected vs D-IDENTITY-CARRIER cl.7.** Passkey unlock
   hard-throws when PRF output is unavailable instead of implementing cl.7's
   "PRF enhancement-with-fallback"; BIP39/transfer exist only as separate manual
   entry points. Open drift item.
3. **"Create a separate identity" second-mint escape hatch.** An unguarded
   `createGuestIdentity` lets a user mint a second identity in-dashboard — a
   cl.5 violation surface, dashboard-only.
4. **KEK salt = random per-wrap, not "perUserSalt".** The passkey PRF KEK salt is a
   fresh random 32 bytes per wrap, not the "perUserSalt" named in the decision text;
   it still honors "never a static global salt" while deviating from the decision's
   phrasing. The wallet KEK likewise uses a random per-wrap salt carried in the
   envelope.
5. **Cross-doc TOPOLOGY contradiction on wallet-signature verification.**
   `wevibe-hub/docs/TOPOLOGY.md:1102,1117` labels the PATCH/wallet-proof path as
   Cosmos `signArbitrary`; code and `wevibe-docs/TOPOLOGY.md:1558/1567/1591` verify a
   **custom** scheme — sha256 digest → stdlib `ecdsa.Verify` over `r||s` (64B) →
   `btcec/v2` 33B-compressed pubkey → bech32 `wevibe` binding — and a real Keplr
   `signArbitrary` signature would NOT verify (no ADR-036/Amino envelope). Code = the
   latter; the hub's own TOPOLOGY is the stale side.
## Component: wevibe-biometric

### What it IS

`wevibe-biometric` is a **Rust napi-rs v3 native addon** (a single `src/lib.rs`, ~262 lines) that exports **exactly two functions**:

- `is_biometric_available()` — a platform probe; **never prompts**.
- `require_biometric(reason)` — an async OS prompt returning `{proceed, outcome, detail}`.

Platform coverage:

| Platform | Prompt backend | Availability probe |
|---|---|---|
| macOS | Touch ID via `LAContext` (biometrics-only policy) | `objc2-local-authentication` |
| Windows | Windows Hello (face / fingerprint / PIN; biometrics+password policy) | `UserConsentVerifier` on a throwaway MTA thread |
| Linux | compiled **WITHOUT robius** (`Cargo.toml:25-26`) | returns `unavailable` → **fail-open no-op** (`lib.rs:163-171`) |

The only data crossing the FFI boundary is `reason: String` in and the `BiometricResult` object out. The addon **holds, creates, derives, and stores NO key material** — there is no crypto crate in its dependency set and no seed/DEK/private-key type anywhere in the file. It is purely a **user-presence ceremony**, not a cryptographic gate.

The TS boundary (`wevibe-mcp/src/biometric.ts`) preserves this contract: `requireBiometric(reason)` returns a boolean and **never throws**; on addon-load failure it logs `fail-open-proceed` and returns `true` (`biometric.ts:108-129`). Test/CI override via the `WEVIBE_BIOMETRIC_FORCE` env var (`biometric.ts:46-64`).

### Fail-open mapping (the decisive logic)

`lib.rs:237-261` maps every OS outcome to a proceed decision:

| OS outcome | Result | Meaning |
|---|---|---|
| unavailable / not-enrolled / not-configured / disabled-by-policy / disconnected / no-passcode | `proceed: true` | **FAIL-OPEN** |
| any other subsystem or prompt error | `proceed: true` | **FAIL-OPEN** |
| explicit user-cancel / app-cancel / system-cancel | `proceed: false` | retryable block |
| auth-failure / attempts-exhausted | `proceed: false` | retryable block |

**Invariant** (stated in the file header, `lib.rs:13-22`): absent, unenrolled, or erroring biometric hardware must **NEVER** lock a user out of their own identity. The **OS keychain-at-rest is the real security floor**; the biometric prompt is defense-in-depth layered on top of it. Only a live human cancel or a live prompt whose check actively failed blocks — and both are retryable.

### The three gate sites (all in `key-store.ts`, keychain backend only)

| Site | Prompt reason | Operation gated | Ordering vs keyring access |
|---|---|---|---|
| `:262` | "Create your WeVibe identity" | `storeIdentitySeed` (first write) | gate **BEFORE** write (`:262` vs `:280`) |
| `:290` | "Export your WeVibe identity recovery phrase" | `loadIdentitySeed` (phrase export) | read at `:285`, gate **BEFORE** returning the seed |
| `:336` | "Unlock your WeVibe identity" | `loadIdentity` (every use) | read at `:330` **FIRST**, gate **BEFORE** derivation (`:343`) |

**Ordering nuance.** On the load path the keychain READ (`loadIdentitySeedB64`, `:330`) completes *before* the prompt fires — the seed bytes are already in process memory when Touch ID appears. The gate therefore blocks seed **USE** (keypair derivation) and seed **EXPORT**, not the keychain read itself. `loadIdentity` memoizes the keypair per process (`:326-328`, `:345-347`), so the prompt fires at most once per process, and boot deliberately avoids it (`server.ts:330-338`; `identity-runtime.ts:6-18`).

**Fail-open implication.** An attacker who can read the OS keychain does **NOT** need Touch ID — the prompt adds nothing cryptographic to that attacker's path. The real at-rest protection is the OS keychain ACL (macOS Keychain / Windows Hello credential store) or, for the file backend, the 0600 `keys.json` + `machine-seed.bin` AES-256-GCM envelope (`key-store.ts:62-101`).

> **Note.** The biometric gate is the **keychain backend only**. The `file` backend (`~/.wevibe/keys/keys.json`) and the `test` backend (in-memory) have **NO gate at all** (`key-store.ts:259,289,335`).

### The gated secret in context: one seed, many wrappers (D-IDENTITY-CARRIER)

The gated secret is the **32-byte identity seed**, stored base64 at rest under account `identity-seed-v1` in keychain service `wevibe-network` via `@napi-rs/keyring` (`key-store.ts:10-11`; `keychain.ts:19-22`). Per decision `D-IDENTITY-CARRIER` (`DECISIONS.md:103-116`), there is ONE canonical seed; each surface wraps ITS OWN COPY under ITS OWN KEK — **five wrappers**:

| Wrapper | What it wraps / unlocks | Provenance origin¹ |
|---|---|---|
| **Passkey PRF KEK** (browser) | the 32 B identity seed; KEK = HKDF-SHA256(PRF_output, perUserSalt, info `wevibe-seed-kek-v1`), AES-256-GCM; ciphertext+salt+iv uploaded to hub (hub stores ciphertext only — never the seed) | (b) platform passkey/biometric authenticator |
| **Touch-ID keychain** (MCP/plugin) | the 32 B identity seed at rest in `wevibe-network/identity-seed-v1`, gated by this addon | (c) OS keychain |
| **BIP39 mnemonic floor** | the seed's **own 24-word identity phrase** (entropy-verbatim, lossless) — break-glass / no-dependency recovery | (d) BIP39 mnemonic |
| **Keplr wallet KEK** (browser, dashboard-only) | the same seed; KEK = HKDF-SHA256(SHA-256(wallet-sig over fixed msg `wevibe-identity-seed-wrap-v1`), salt, info `wevibe-seed-kek-wallet-v1`) | (a) Keplr/Leap software wallet |
| **Pairing-code handoff** (5th wrapper) | pairing secret → HKDF KEK (info `wevibe-pair-v1`) for one-time seed transfer | (f) in-process random/HKDF |

¹ Origins follow the 7-origin provenance matrix: (a) Keplr/Leap software wallet; (b) platform passkey/biometric authenticator; (c) OS keychain; (d) BIP39 mnemonic; (e) seed file (bench); (f) in-process random/HKDF; (g) env var/config.

Any **ONE** wrapper recovers the **same** seed — no surface mints a second identity. Every wrapper must yield the byte-identical 32-byte seed or the **pubkey-integrity check fails** on mismatch (`wevibe-auth.ts:350-352, 689-692, 745-748`).

Two precision points this component lives inside:

- **Two secrets share one encoding.** The BIP39 wrapper above is the seed's OWN 24-word **identity** phrase. It is NOT the org master secret: K_master (32 B) carries its own separate 24-word **org** phrase plus Shamir 2-of-3. Both are 32-byte secrets encoded as 24-word BIP39; they are distinct.
- **What the gated seed derives** (and note the MCP's native Touch ID / Windows Hello prompt is a user-presence ceremony that derives NOTHING — the PRF/KEK machinery belongs to the browser passkey wrapper alone): the identity Ed25519 private key is the seed **verbatim**, plus an X25519 key via HKDF-SHA256 with info `wevibe-x25519-v1`. Identity thus uses Ed25519 + X25519 — one of the three DISTINCT curve roles in the system (the others: X25519 for SK_mod/PK_mod; secp256k1 for the Umbral epoch key and the member pre-identity-key).

## Component: wevibe-opencode-plugin + wevibe-bench

Two components, one thesis: **no long-lived key lives where the agent loop runs.** The opencode
plugin holds no secret at all — it Bearer-authenticates to the host MCP and reads public/numeric
state files. The bench harness holds exactly one root secret (a 0600 seed file) that derives a
seed-based identity deliberately *other* than the operator's keychain identity. Neither component
signs with org-hierarchy material; the hub never holds epoch_sk/K_master/DEK/plaintext, and the
chain holds pubkeys + content-free events only.

### 1. Plugin secret boundary — the plugin holds NO long-lived secret

The plugin mints nothing and signs nothing. It reads four artifacts:

| # | Artifact | What it carries | Mode |
|---|---|---|---|
| a | `~/.wevibe/mcp-session-token` | Bearer token to the host MCP | 0600 |
| b | `~/.wevibe/identity.json` | **Public** identity sidecar — booleans/counts only | — |
| c | `~/.wevibe/plugin-config.json` | Numeric risk config only | — |
| d | `~/.wevibe/blacklist.json` | Code-wired but **never created** (no denies yet) | (would be 0644) |

- **(a) Session token.** 64-char hex of `randomBytes(32)` (provenance origin **(f)** in-process
  random), minted by `wevibe-mcp` (`dist/session-token.js:29`, chmod 0600 at write), server-verified
  with `timingSafeEqual`. The plugin reads it (`plugins/wevibe-plugin.ts:1156-1165`) and attaches it
  as `Authorization: Bearer` (`:1193`) to `WEVIBE_MCP_HTTP`, default `127.0.0.1:4450`. The plugin
  never writes this file.
- **(b) Identity sidecar.** Minted by `wevibe-mcp` admin, not the plugin. The plugin engine never
  reads it at runtime; the TUI consumes booleans/counts only (`ed25519PublicKey` presence,
  `adoptedAt`, `orgs` count — `tui/wevibe.tsx:1248-1255`). No private material.
- **(c) Risk config.** `recall_relevance_floor`, `recall_max_injected`, `inject_char_budget`
  (`wevibe-plugin.ts:412,417,424`) plus TUI `risk_appetite`. Numeric only.
- **(d) Blacklist.** Write/read paths are wired (`wevibe-plugin.ts:439-459` write, `:634-646` read,
  lazy-create on first deny) but the file does not exist anywhere — no pack has been denied since
  install. Shape = array of pack/cid id strings; no key material. (Hygiene: the lazy create uses
  default `writeFileSync`, i.e. 0644 — unhardened.)

**State files live under `~/.wevibe/unbound/<sha256(realpath)>/state/` — NOT `~/.opencode/`**
(`plugins/wevibe-paths.ts:34-56`): `wevibe-plugin-status.json` (accepted/denied/reported cid
arrays), `-queue.json`, `-decisions.json`, `funnel-snapshot.json` (counters),
`wevibe-tui-active.json` (heartbeat ts). All carry **cids/counters/timestamps only**. Hygiene note:
these scoped state files are 0644 while the cid-bearing `~/.wevibe/served-memories.json`
(cid→ts map) is 0600 — inconsistent hardening.

**X-Agent-Signature is produced by the MCP, not the plugin** (sole producer
`wevibe-mcp/src/org-client.ts:165`, signing at `:145-150`). The plugin→MCP hop is **Bearer-only** —
unsigned. The signing key is the host's Ed25519 identity key: ed privkey = the 32-byte identity
seed itself (identity keypair = **Ed25519**, seed verbatim, + **X25519** via HKDF-SHA256 info
string `"wevibe-x25519-v1"`). On the daily-driver host that seed lives in the OS keychain
(provenance origin **(c)**): service `wevibe-network`, account `identity-seed-v1`,
biometric-gated. Host ed fingerprint `05c4b8cb` is the raw ed25519 public-key hex prefix — it IS
the keychain identity.

The identity seed is one of the system's TWO root secrets and must not be conflated with the
other: **K_master (32B) → BIP39 24-word ORG phrase + Shamir 2-of-3; the identity seed (32B) is
SEPARATE, with its OWN 24-word IDENTITY phrase.** The plugin sees neither.

### 2. Bench identity seam — the `:4450` vs `:4550` boundary

Two ports, two identities:

| Port | What it is | Identity | Bench-touchable |
|---|---|---|---|
| `:4450` | Operator's REAL host `wevibe-mcp` (daily driver) | OS-keychain identity, ed fp `05c4b8cb`; **no seed support** | **NEVER** |
| `:4550` | The ONE commissioned bench MCP (managed service, `bench-mcp.sh start`) | Seed-derived: `fp(seed)=0e93b599`, ed-pubkey fp `aa2aa706` | yes |

Bench secret inventory under `~/.wevibe/bench/`:

- **`leader-seed.txt`** — the bench's sole root secret: 32-byte seed as 64-hex + newline (65 B),
  0600. Provenance: seed file (origin **(e)**), commissioned from the root of Walter's benchmark
  Keplr wallet (Keplr lineage = origin **(a)**). Loaded by `lib.sh:104-121 load_leader_seed()`:
  env override `WEVIBE_BENCH_MCP_SEED` (origin **(g)**) takes priority, else the file; 0600 mode is
  enforced and 64-hex shape validated. `bench-mcp.sh:372-381` derives fingerprint + mnemonic
  in-memory from it — no hardcoded seed anywhere in the scripts.
- **`leader-mnemonic.txt`** — the 24-word BIP39 of the SAME 32-byte seed (a pure function; it
  round-trips), 148 B, 0600. This is the **IDENTITY** phrase of the identity seed — not the
  K_master ORG phrase. **Legacy artifact**: since the script rewrite the mnemonic is derived
  **in-memory** from the seed (`bench-mcp.sh:376-379`) and passed to `import-identity --phrase`;
  no code reads the file.
- **What the seed/mnemonic derives:** (i) the **secp256k1** wallet key via SLIP-10
  `m/44'/118'/0'/0/0` (secp256k1 = 32B scalar priv / 33B compressed pub); (ii) the **Ed25519**
  identity (privkey = seed verbatim) + its X25519 (HKDF info `"wevibe-x25519-v1"`).
- **`leader-keystore/`** — `keys.json` (0600; shape `{version, nonce_b64, ciphertext_b64,
  tag_b64}` = **AES-256-GCM** ciphertext under KEK = `sha256("wevibe-keystore-v1" ||
  machine-seed.bin)`) + `machine-seed.bin` (32 B, 0600).
- **`identity.json`** — public `IdentitySidecar` (never holds seed/privkey). **STALE**: bench
  sidecar ed fp `90aab0a8` ≠ served `aa2aa706` (written 2026-07-30, pre-rotation;
  `import-identity` does not write the sidecar).

**Seed backend:** `WEVIBE_SEED_BACKEND=file` is REQUIRED for bench (headless — no Touch ID).
`seedBackend()` (`key-store.ts:26-30`) defaults to `keychain`; `file` only under that env var
(set at `bench-mcp.sh:256,398`, keystore scoped to the bench home). Routing is fail-closed: while
`file` is set the keychain branch is unreachable — no fallthrough onto the operator's personal
keychain identity. The env-seed backend (`WEVIBE_IDENTITY_SEED_HEX`) is fully retired (dead path
purged).

**The seam hazard.** `POST /v1/org-setup` stamps the **SERVING MCP's own** ed pubkey as
`leader_pubkey` (`org-client.ts:614-619,674`); the hub persists it as the org's leader membership
row (pubkey only — the hub never holds epoch_sk/K_master/DEK/plaintext, and the chain holds
pubkeys + content-free events only). Pointing any bench component at `:4450` therefore mints every
fresh org under the operator's keychain identity (`05c4b8cb`) instead of the bench identity
(`aa2aa706`); the harness then never finds its membership and fails ~30 s later with
`RuntimeError: leader membership did not include org_id=…`. An unexpected Touch ID prompt during a
bench run is this same defect — the keychain was reached when it must not be.

**Guards** (identity is asserted at the seam, never inferred):

- The bench run path pins `:4550` (`lconfig.py:33-36`) and has **no bring-up machinery** — it only
  connects; the bench MCP is a managed service commissioned from `leader-seed.txt` by
  `bench-mcp.sh start` / `make redeploy`.
- `bench_preflight.py:53` — `FORBIDDEN_PORT=4450`.
- `verify-clean.sh:564-574` — check 11 (`mcp-fresh-4550`): compares the bench MCP's served
  fingerprint against the one derived from the leader seed. Wrong or unverifiable identity =
  hard-fail; **unreachable is a hard failure, not a skip** — a run on an unverified seam is
  VOID-INSTRUMENT.
- Fingerprint discipline: `fp(seed_bytes)=0e93b599` and `fp(ed_pubkey_bytes)=aa2aa706` name the
  SAME identity over different hashed inputs — never compare fingerprints without naming which
  input was hashed.

### 3. Flags (open items)

- **OPEN-FOR-WALTER — `f7733d6e` survives only as a stale comment.** No bench file hardcodes the
  retired pre-reseed leader fp anymore (the three former literals at `bench-mcp.sh:89`,
  `verify-clean.sh:555`, `bench_preflight.py:44` were purged by STRIP-2b). The one remaining mention
  is a stale comment at `lconfig.py:27` (comment-only, and it misstates the fp as the bench leader
  anchor; the canonical anchor is `aa2aa706`).
- **OPEN-FOR-WALTER — `AGENTS.md §2.1` cites a deleted guard file.** The "fail-fast guard in
  `create_org` (`wevibe_bench/lifecycle/orchestrator.py`)" no longer exists — that file was deleted in
  the overhaul. The current equivalents are the `bench_preflight.py` identity assertion and
  `verify-clean.sh` check 11 (`mcp-fresh-4550`).
- **Latent trap:** `config.py:270` `mcp_recall_url` defaults to `:4450` — dead config today, but
  it would silently aim bench recall at the operator's identity if ever re-wired.

## Component: wevibe-sdk / wevibe-guard / wevibe-protocol / wevibe-social-graph

**Verdict: NONE of these four components holds a private key.** Each touches
key material only in transit — as call parameters, public keys, or public test
fixtures. Durable key custody lives exclusively in the wevibe-mcp host process
and the OS keychain. Consistent with the system-wide boundary: the hub never
holds epoch_sk / K_master / DEK / plaintext, and the chain holds pubkeys +
content-free events only.

| Component | Role | Private keys? | Key surface |
|---|---|---|---|
| wevibe-sdk | Rust + WASM crypto library | none | key material enters via parameters only |
| wevibe-guard | YARA-X content scanner | none | no key surface at all |
| wevibe-protocol | shared contract + test-vector fixtures | none | public test data only |
| wevibe-social-graph | wallet-signature verifier | none | verifies public keys/signatures only |

### wevibe-sdk — library, no persistent keys

wevibe-sdk is a library, not a service: it persists nothing, writes nothing to
keychain or filesystem, and all key material enters via parameters:

| API | Curve / purpose | Anchor |
|---|---|---|
| `from_secret_bytes` | secp256k1 scalar import | secp256k1.rs:34 |
| `from_bytes` | Ed25519 identity seed import | identity.rs:32 |
| `sign` | Ed25519 signing | crypto.rs:163 |
| `seal_to_pubkey` / `open_envelope` | X25519 ECDH + AES-256-GCM envelopes | crypto.rs:177,205 |

Properties:

- `LocalIdentity` (the Ed25519 identity holder) is `ZeroizeOnDrop` — key
  material is zeroed on drop (identity.rs:11-18).
- Zero keychain/fs writes anywhere in crate sources; persistence lives only in
  consumers (wevibe-mcp), outside the SDK. The only fixture-writing writes in
  the tree are test-fixture REGEN inside test files — not a runtime path.
- The secp256k1 module is scoped to PRE/Umbral secrets only (sole `k256`
  import in the crate). Identity signing is Ed25519; identity encryption is
  X25519 — NOT every secret is a secp256k1 scalar.
- Notes (verified, non-security): the Rust `PreIdentity` type is
  EXISTS-BUT-UNREACHABLE (no consumers, no WASM export; the MCP TypeScript
  `PreIdentity` is a separate type). TOPOLOGY.md's
  `PRE_DERIVATION_LABEL = "wevibe-pre-identity/v1"` is stale — code uses
  `"secp256k1-bip32-derive"` (a single HMAC-SHA512 round, no chain code, no
  index, no clamping). `secrecy` is declared but unused in Cargo.toml — dead
  dependency.

**DEK wrap path (through SDK primitives — NO K_enc hop):**
DEK → `seal_to_pubkey`(DEK, PK_mod) → Umbral capsule →
ReEncrypt(capsule, kfrag) → cfrag → `decrypt_reencrypted` → DEK →
AES-256-GCM content cipher. The wrap target PK_mod is X25519; the re-encryption
layer rides the secp256k1 Umbral epoch key.

**Epoch keys as seen by the SDK (exact spec).** Four keys derive from K_master
via HKDF-SHA256:

| Purpose | salt | info | Output | Status |
|---|---|---|---|---|
| enc | 32×0x00 | "wevibe-enc-" + epoch 4-byte BE | 32B | DEAD-SHIPPED |
| search | 32×0x00 | "wevibe-search-" + epoch 4-byte BE | 32B | DEAD-SHIPPED |
| audit | 32×0x00 | "wevibe-audit-" + epoch 4-byte BE | 32B | DEAD-DERIVED |
| umbral | ∅ (empty) | "wevibe-umbral-epoch-" + decimal epoch | VERBATIM secp256k1 scalar (epoch_sk) | CONSUMED |

Only epoch_sk is CONSUMED today — the sole secp256k1 scalar actually fed into
the Umbral PRE layer.

**Provenance.** The SDK itself contributes only in-process random/HKDF
material; every persistent secret it touches arrives from a consumer-side
origin. The seven canonical provenance origins across the system are:
(a) Keplr/Leap software wallet; (b) platform passkey/biometric authenticator;
(c) OS keychain; (d) BIP39 mnemonic; (e) seed file (bench); (f) in-process
random/HKDF; (g) env var/config.

### wevibe-guard — YARA-X content scanner, no key surface

wevibe-guard has NO key surface: no keychain, no key files, no signing, no
decryption.

- Advisory, fail-open: a scan error yields empty results and processing
  continues (scanner.rs:132-134, flags.rs:232-234) — guard never blocks the
  pipeline on scanner failure.
- The content-suppression gate is provider-policy: applied by the calling hub
  http-server, not by guard itself.
- Crypto crates in Cargo.lock are transitive via the yara-x dependency only;
  they serve scanner internals and touch no WeVibe key material.

### wevibe-protocol — shared contract + test-vector fixtures

No live keygen, storage, signing, or keychain calls anywhere in the tree or
its git history. Two fixture directories:

- `test-vectors/` (hyphen) — recall-ranking-parity.json; orphaned from docs.
- `test_vectors/` (underscore) — README + 7 fixtures.

Fixtures are public test data, with one caveat: 4 of the 5 key-related
fixtures contain deterministic TEST-ONLY key values
(epoch_key_derivation.json, mnemonic_roundtrip.json, seal_open_envelope.json,
shamir_roundtrip.json). They are referenced by file only and NEVER quoted in
this document; because they encode the derivation shapes, fixture files must
never be fed into a production path.

The fixtures exercise the system's TWO-SECRET model: K_master (32B) ↔ BIP39
24-word ORG phrase, additionally split via Shamir 2-of-3 — and the identity
seed (32B), a SEPARATE secret with its OWN 24-word IDENTITY phrase.

Housekeeping (verified): `test_vectors/README.md` is stale — TODO header, and
its checklist lists 3 fixtures that do not exist while naming none of the 7
that do.

### wevibe-social-graph — signature verifier, no private keys

wevibe-social-graph VERIFIES wallet signatures; it holds no private keys
(grep: zero production hits) and reads only public profiles + SQLite. It never
signs, mints, or stores secret material.

Verification pipeline (`internal/server/signature.go`):

1. custom sha256 digest over the signed payload (signature.go:41)
2. stdlib `ecdsa.Verify` over the 64-byte r||s signature (:48-50)
3. `btcec/v2 ParsePubKey` of the 33-byte compressed secp256k1 pubkey (:43)
4. bech32 address binding under HRP `"wevibe"` (:80)

This is neither signArbitrary/ADR-036 nor EIP-712. The PATCH verify path is
latent (EXISTS-BUT-UNREACHABLE): no workspace client currently posts
wallet_pubkey/wallet_signature; only unit tests exercise it.

### Canonical cross-language curve invariants

Three DISTINCT curves — never conflated:

| Use | Curve | Secret | Public |
|---|---|---|---|
| Moderation keypair (SK_mod / PK_mod) | X25519 | 32B | 32B |
| Umbral epoch key + member pre-identity-key | secp256k1 | 32B canonical scalar | 33B compressed |
| Identity signing | Ed25519 | 32B seed, held verbatim | 32B (64B signatures) |
| Identity encryption | X25519 | derived from identity seed via HKDF, info "wevibe-x25519-v1" | 32B |

secp256k1 byte-level invariants (the TS↔Rust interop backbone):

- Secret: a 32-byte big-endian canonical scalar. k256 `SecretKey::from_bytes`
  (SDK) ≡ Umbral `try_from_be_bytes`: both big-endian, both reject
  non-canonical scalars.
- Public: `pubkey = sk·G`, compressed 33-byte encoding with 0x02/0x03 prefix.
- Byte-compatible with `@noble/secp256k1 getPublicKey(sk, true)` — the TS side
  produces byte-identical compressed pubkeys (33-byte asserted in the MCP auth
  path).

Ed25519 is identity only (32B priv/pub, 64B signature); X25519 is encryption
only (32B). This document must never claim every secret is a secp256k1
scalar.

### Flags carried forward

1. **Address-hash preimage deviation.** The social-graph address derivation
   hashes `pubkeyBytes[1:]` — the 32-byte X-only slice of the compressed
   pubkey — instead of the full 33-byte compressed pubkey
   (signature.go:67). A derived address can therefore never match a real
   chain-derived wallet address. The wevibe-hub twin (wallet_sig.go:51) has
   the IDENTICAL deviation; pre-existing (report 1786917834).
2. **SDK secp256k1 API name.** The SDK constructor is `from_bytes` (k256),
   not `try_from_be_bytes` (Umbral-side naming). Same invariant, different
   API — do not document them as one function.
## Identity & Session Authentication

The identity layer is per-human and per-device, and is INDEPENDENT of any org key hierarchy. One 32-byte identity seed is the identity DEK ("one seed, many wrappers", D-IDENTITY-CARRIER): each surface wraps ITS OWN copy under ITS OWN KEK, and any single wrapper suffices to recover the identical seed. Session authentication spans two distinct seams: per-request Ed25519 signatures toward the hub, and a local bearer token guarding the MCP's own API.

### The identity seed (`identity-seed-v1`)

| Property | Value |
|---|---|
| Class | 32 B random (`randomBytes(32)`, `key-store.ts:249-251`), minted once per device on first run |
| Keychain backend (default) | OS keychain via `@napi-rs/keyring` — service `wevibe-network`, account `identity-seed-v1` (`keychain.ts:5,19-22`; `key-store.ts:10-11`); biometric-gated |
| File backend | `~/.wevibe/keys/keys.json` — AES-256-GCM envelope; KEK = SHA-256(`wevibe-keystore-v1` ‖ machine-seed), machine-seed = 32 B random at `~/.wevibe/keys/machine-seed.bin` (0600) (`key-store.ts:62-101`) |
| Backend selection | `WEVIBE_SEED_BACKEND=file` selects the file backend (bench/container default); otherwise the keychain backend |
| Ed25519 privkey | the seed VERBATIM (Rust/WASM `ed25519-dalek`, `crypto.rs:147-161`) |
| X25519 privkey | HKDF-SHA256(seed, salt=∅, info `wevibe-x25519-v1`) (`crypto.rs:147-161`) |
| Sharing | never shared; root of the identity Ed25519 + X25519 pair only |

The keypair is never stored — it is derived on demand and memoized per process (`key-store.ts:326-328,343-347`). The biometric gate (`wevibe-biometric` napi-rs addon; `requireBiometric` at `key-store.ts:262,290,336`) is a fail-open user-presence ceremony on the keychain backend only: it creates, holds, and derives nothing, and it gates seed USE/derivation/export — not the keychain read itself. The `file` backend has no gate; its 0600 AES-256-GCM envelope is the at-rest floor there. Provenance: (f) in-process random for a fresh mint; the same seed slot can instead be populated from (d) a BIP39 mnemonic import, (e) a seed file (bench), or (g) an env var.

### Request signing: `X-Agent-Signature`

- Signs the raw JSON request body with the identity Ed25519 key (`org-client.ts:149,165`) — NOT the secp256k1 PRE key and NOT any org-hierarchy key.
- Hub verifies with `ed25519.Verify` against the member's registered pubkey (`retrieval.go:46-90`) and 401s on mismatch.
- Producer is the MCP, NOT the plugin: the signature is minted inside the MCP org-client; the plugin/dashboard never produce it.

### Membership auth: `WeVibe-Signed` Authorization

- Signs the ISO timestamp (`auth.ts:105,111`) carried in the `WeVibe-Signed` Authorization header.
- Membership is keyed by edPubkey hex (`org-client.ts:619`; hub `middleware.go:55`): the identity Ed25519 public key IS the member identity across the hub.

### Local session token (:4450)

- Path `~/.wevibe/mcp-session-token`; 32 B random rendered as 64 hex chars; file mode 0600 (`session-token.ts:7,30,40-42`).
- Bearer auth on the MCP local API at :4450; consumers include the dashboard server-side routes that delegate org-setup to the MCP (`route.ts:136-148`).
- Read-from-disk-first: the token is STABLE across boots and does NOT rotate on MCP restart (`session-token.ts:85-89`).
- Verification uses `timingSafeEqual`.

### Seed wrapping — one seed, many wrappers

Each surface wraps its own copy of the SAME 32-byte identity seed under its own KEK; every wrapper must yield the identical seed or the pubkey-integrity check fails. KEKs are re-derived on demand, never persisted (WebCrypto `extractable:false`), and zeroized after use.

#### Passkey PRF KEK (browser)

- Credential: platform authenticator (Touch ID / Windows Hello / hardware key), created resident + UV-required + PRF extension (`passkey.ts:70-111`); the credential privkey and PRF secret never leave the authenticator.
- RP-ID = `window.location.hostname` (`passkey.ts:80`) — the wrapper is RP-scoped.
- KEK = HKDF-SHA256(PRF_output, salt=random 32 B per-wrap, info `wevibe-seed-kek-v1`) → AES-256-GCM wraps the identity seed (`passkey.ts:179-200`). Ciphertext + salt + IV persist in the wrapped blob (IndexedDB, mirrored to the hub); the blob is safe-at-rest.
- Provenance (b) platform passkey/biometric authenticator.

#### Wallet KEK — Option A (browser)

- The wallet signs the fixed message `wevibe-identity-seed-wrap-v1` via `signArbitrary` (`wallet-seed-wrap.ts:15`).
- KEK = HKDF-SHA256(SHA-256(wallet-sig), salt=random 32 B, info `wevibe-seed-kek-wallet-v1`) → AES-256-GCM wraps the seed (`wallet-seed-wrap.ts:53-80`).
- Wrapper record: `credentialIdB64 = "wallet:" + <bech32 address>`, kind `wallet` (`wevibe-auth.ts:459,465`).
- Hardware caveat: Ledger/Keystone signatures are non-deterministic, so the KEK is NOT reproducible there — software wallets only (Keplr/Leap). This is NOT enforced at runtime: `isNanoLedger`/`isKeystone` are declared but never read, and a hardware attempt simply yields `WalletUnlockMismatchError` (`wevibe-auth.ts:733-737`).
- Provenance (a) Keplr/Leap software wallet; the wallet private key never enters the browser.

#### Identity BIP39 phrase (break-glass)

The identity seed carries its OWN 24-word BIP39 phrase (entropy verbatim, lossless; `admin.ts:269-310`), held offline by the human. It is DISTINCT from the org recovery phrase — see the boundary note below. Browser note: while unlocked, the dashboard also caches the raw seed in sessionStorage `wevibe.session.seed.v1` (tab-scoped, cleared on lock) — an accepted XSS tradeoff (`wevibe-auth.ts:100-116`).

### Org serve/event signing key

- Derivation: HKDF-SHA256(identitySeed, salt `wevibe-org-serve-key-v1-salt`, info `wevibe-org-serve-key-v1:{orgId}`) → 32 B Ed25519 seed (`serve-signing.ts:95-128`).
- Scope: one key per (identity, org); derived on demand, never persisted.
- Use: signs the canonical serve/event/denial bodies (@noble/ed25519) reported toward hub/chain surfaces.

### Boundary with the org key hierarchy

Identity is NOT the org hierarchy — two separate key trees that meet only in membership attribution. Stated exactly:

- **TWO SECRETS, one encoding.** K_master (32 B) → BIP39 24-word ORG phrase + Shamir 2-of-3. Identity seed (32 B) = SEPARATE secret with its OWN 24-word IDENTITY phrase. Both are 32-byte secrets encoded as 24-word BIP39; never conflate them.
- **EPOCH KEYS (org side, for disambiguation).** 4 from K_master via HKDF-SHA256. enc/search/audit: salt=32×0x00, info `wevibe-enc-` | `wevibe-search-` | `wevibe-audit-` + epoch 4-byte BE, 32 B each. umbral: salt=∅, info `wevibe-umbral-epoch-` + decimal epoch, VERBATIM secp256k1 scalar. STATUS: only epoch_sk CONSUMED; enc/search DEAD-SHIPPED; audit DEAD-DERIVED. None of them derives from the identity seed.
- **CURVES (three DISTINCT).** (1) SK_mod/PK_mod = X25519 (org moderation). (2) Umbral epoch key + member pre-identity-key = secp256k1 (32 B scalar priv / 33 B compressed pub). (3) identity = Ed25519 (seed verbatim) + X25519 (HKDF `wevibe-x25519-v1`).
- **DEK WRAP (NO K_enc hop).** Per-memory DEK → seal_to_pubkey(DEK, PK_mod) → Umbral capsule → ReEncrypt(capsule, kfrag) → cfrag → decrypt_reencrypted → DEK → AES-256-GCM. The identity seed participates in none of these hops.
- **CUSTODY.** The HUB NEVER holds epoch_sk / K_master / DEK / plaintext — on this surface it stores pubkeys, membership attribution, and opaque wrapped-seed blobs only (non-custodial: unreadable without the corresponding wrapper). The CHAIN holds pubkeys + content-free events only.

### Provenance origins referenced by this section

| Origin | How it surfaces here |
|---|---|
| (a) Keplr/Leap software wallet | wallet-KEK wrapper (signArbitrary over the fixed wrap message); wallet privkey never enters the browser |
| (b) Platform passkey/biometric authenticator | PRF KEK wrapper; credential + PRF secret stay in the authenticator |
| (c) OS keychain | identity seed at rest, service `wevibe-network` / account `identity-seed-v1`, biometric-gated |
| (d) BIP39 mnemonic | the identity seed's own 24-word phrase (break-glass export/import) |
| (e) Seed file (bench) | `~/.wevibe/bench/leader-seed.txt` (32 B as 64 hex chars, 0600); env override `WEVIBE_BENCH_MCP_SEED` takes priority |
| (f) In-process random/HKDF | seed mint (`randomBytes(32)`); every derived child (Ed25519 verbatim, X25519 and serve-key via HKDF) |
| (g) Env var/config | `WEVIBE_SEED_BACKEND` selects the storage backend; bench seed env var supplies the seed |

## Recovery

Recovery acts on ROOTS, never on derived keys. Exactly **two secrets** frame the whole story: `K_master` (32 B) → BIP39 24-word **ORG** phrase + Shamir 2-of-3; the identity seed (32 B) is a **SEPARATE** secret with its **OWN** 24-word **IDENTITY** phrase. Neither phrase recovers the other's material. Custody invariants bound every path below: **the hub never holds epoch_sk / K_master / DEK / plaintext** (the recovery shares it stores are sealed ciphertext it cannot open), and **the chain holds pubkeys + content-free events only** — nothing recovery-sensitive is on-chain.

### Two phrases, two secrets (BIP39)

Two DISTINCT 24-word phrases exist. They encode two different 32-byte secrets and share ONLY the encoding — conflating them is the classic failure this section exists to prevent. Both are produced the same way: 32 bytes of OsRng entropy → 24 words via `bip39::Mnemonic::from_entropy` (`recovery.ts:17-26` → SDK `crypto.rs:304-308`).

**ORG phrase (K_master).** Shown to the leader **exactly once** at org creation, never persisted anywhere. It is the org's recovery root: restoring `K_master` from the phrase re-derives the complete epoch family, because all **4 epoch keys come from K_master via HKDF-SHA256** — K_enc/K_search/K_audit at salt = 32×0x00, info = `wevibe-enc-` / `wevibe-search-` / `wevibe-audit-` + epoch 4-byte BE (32 B each), and the Umbral epoch key at salt = ∅, info = `wevibe-umbral-epoch-` + epoch decimal, its 32 B used **VERBATIM as the secp256k1 scalar** `epoch_sk` (33 B compressed pubkey). STATUS: **only epoch_sk CONSUMED; enc/search DEAD-SHIPPED; audit DEAD-DERIVED** — recovery is untouched by those statuses, since `K_master` alone regenerates every epoch key for every epoch.

**IDENTITY phrase (identity seed).** The identity seed carries its **own** 24-word phrase, exported via `wevibe-admin export-identity` (biometric-gated at `key-store.ts:290` on the keychain backend; fail-open ceremony; the phrase is derived on demand and never persisted) and imported via `wevibe-admin import-identity` — the same path the bench uses to commission the leader MCP's identity. Because identity derivation is deterministic — **Ed25519 (seed VERBATIM) + X25519 (HKDF-SHA256, info `wevibe-x25519-v1`)** — the phrase alone restores membership identity, `X-Agent-Signature` request signing, and envelope decryption.

**Storage guidance (both phrases).** Write offline (paper or equivalent cold medium). Never paste into any agent, chat, or web form. Never store on the same device as the keystore/keychain the phrase backs up — a phrase and its live store on one machine collapses the recovery boundary.

### Shamir 2-of-3 over K_master (OPTIONAL)

`splitMasterKey` (`recovery.ts:34-45`) splits **K_master itself — never an epoch SK** — into 3 shares over GF(256) at a 2-of-3 threshold, each share 33 B. Each share is envelope-sealed — X25519 ECDH → HKDF-SHA256 (info `wevibe-envelope-v1`) → AES-256-GCM — to a holder X25519 pubkey:

| Share | Sealed to |
|---|---|
| 1 | leader's X25519 pubkey |
| 2 | holder #1's X25519 pubkey |
| 3 | holder #2's X25519 pubkey |

Sealed shares are uploaded to the hub `recovery_shares` table, which stores exactly `sealed_share` + `holder_pubkey`: the `sealed_share` column is **opaque ciphertext the hub can never open**; `holder_pubkey` is public routing metadata. Holders fetch their share via REST `GET /v1/orgs/{id}/recovery/shares/{pubkey}` (hub handler `GetRecoveryShare`, client at `admin.ts:774`). Any 2 of 3 shares reconstruct `K_master`; `K_master` alone then re-derives the epoch family.

**Status note.** Decision **D-10.1 made Shamir OPTIONAL** (the burned-epoch acceptance path); the older **D-10.2 wording — "recovery requires 2-of-3" — is STALE**. Where a share set exists, the quorum is nonetheless a canon-acknowledged governance surface: two shareholders reconstructing `K_master` can derive every epoch key — the K_master-quorum attack named in the decisions archive. That is why shares are *sealed to holder keys* rather than stored readably, and why the single ORG phrase remains the only non-quorum root.

### Encrypted leader vault (`vault.enc`)

Path: `~/.wevibe/vault.enc`. Key derivation: passphrase → **Argon2id (t=3, m=65536, p=4**, 32-byte salt, 32-byte output) → AES-256-GCM. The passphrase itself lives in the OS keychain under service **`wevibe-vault`** (macOS `security find-generic-password -s wevibe-vault`; Linux secret-tool). When unlocked, the vault can carry `k_master_hex` — the org master key — alongside org material.

> **FLAG — vault is DEAD in production.** `createVault` (and `storePassphraseInKeychain`) have **zero production callers**: the vault is uncreatable in prod, `vault.enc` is absent on disk, and the vault branch of `persistOrgKeys` silently no-ops. Treat the vault as not yet live; the BIP39 ORG phrase (+ optional Shamir set) is the operative `K_master` recovery path.

### `wevibe-admin` recovery tooling (LIVE)

The recovery CLI is implemented and shipped (`admin.ts:700-786`, binary registered in `package.json`). The claim in `RUNBOOK-PRE-RECOVERY.md` that the recover CLI is "not implemented" is **STALE** — the capability is live and the ceremony (Shamir 2-of-3 of `K_master`) is canon-consistent:

| Command | Role in the ceremony |
|---|---|
| `wevibe-admin setup-threshold` | Runs `splitMasterKey`: 2-of-3 split of `K_master`, seals each share to the leader + 2 holder X25519 pubkeys, registers them in hub `recovery_shares` (`admin.ts:723-763`). |
| `wevibe-admin recover-threshold` | Fetches sealed shares (REST `GET .../recovery/shares/{pubkey}`, `admin.ts:774`), unseals any 2 of 3, reconstructs `K_master`. |
| `wevibe-admin recover-org` | Restores org control from `K_master` — phrase-entered or threshold-reconstructed — and re-derives the epoch keys. |
| `wevibe-admin recovery-status` | Reports the registered recovery shares / threshold state for an org. |

Loss-class recoverability itself (which key classes a phrase or quorum can and cannot restore, e.g. `SK_mod` and per-memory DEKs) is tabulated in **Key Rotation → Key-loss recoverability per class**.

Provenance, by the seven key-origin classes: recovery phrases are origin **(d) BIP39 mnemonic**; the bench leader's identity arrives via origin **(e) seed file (bench)** — `~/.wevibe/bench/leader-seed.txt` (0600), consumed through `import-identity --phrase`, with the older `leader-mnemonic.txt` stash superseded. Neither bench path is a production recovery mechanism.

## Key Rotation

Rotation advances an org's epoch by one and redistributes all epoch-scoped key material. It is implemented as `rotateEpoch` (`wevibe-mcp/src/org-client.ts:1040-1196`) — the old doc's claim that `wevibe_rotate_epoch` is "not yet implemented" is false.

**Cadence: NONE.** There is no time-based epoch constant anywhere in the stack. Rotation happens only on explicit manual trigger — the CLI `rotate --org` command or the dashboard rotate button.

### What rotateEpoch does

1. `newEpoch = currentEpoch + 1` (`org-client.ts:1064`).
2. Re-derives the new epoch's keys from `K_master` (`org-client.ts:1065`). The audit key is derived then discarded — it has no consumer and is never sealed.
3. Mints a **FRESH random X25519 moderation keypair** (`SK_mod`/`PK_mod`) (`org-client.ts:1067`).
4. Re-seals the epoch material (`org-client.ts:1094-1117`):
   - `enc_envelope` + `search_envelope` (carrying the new epoch enc/search keys) → **all active members**, sealed to each member's identity X25519 pubkey;
   - `mod_envelope` (carrying the new `SK_mod`) → **leader and moderators only**.
5. `POST /epoch/rotate` to the hub: bumps the org's `current_epoch`, inserts a new `epoch_manifests` row, and batch-replaces the stored envelopes (`wevibe-hub/internal/orgs/orgs.go:245-280`).

**SK_mod IS epoch-scoped** — a fresh keypair is minted at every rotation and re-sealed to the moderation set. This corrects the old doc's "SK_mod not epoch-scoped".

The rotation boundary preserves the custody invariants: the hub stores only public or opaque sealed material (`new_pk_mod`, signatures, sealed envelopes, `umbral_pk` where present) — **the hub never holds epoch_sk, K_master, DEKs, or plaintext**; the **chain holds pubkeys + content-free events only**. The new `epoch_sk` is derived on the leader machine and never leaves it.

### Epoch keys rotation re-derives (canonical statement)

Rotation re-runs the standard derivation: **4 epoch keys from K_master via HKDF-SHA256**, in two schemes:

| Key | HKDF-SHA256(K_master, salt, info) → output | Status |
|---|---|---|
| K_enc(e) | salt = 32×0x00, info = `wevibe-enc-` + epoch 4-byte BE → 32 B | **DEAD-SHIPPED** |
| K_search(e) | salt = 32×0x00, info = `wevibe-search-` + epoch 4-byte BE → 32 B | **DEAD-SHIPPED** |
| K_audit(e) | salt = 32×0x00, info = `wevibe-audit-` + epoch 4-byte BE → 32 B | **DEAD-DERIVED** |
| epoch_sk(e) → umbral_pk | salt = ∅, info = `wevibe-umbral-epoch-` + epoch decimal → 32 B, used **VERBATIM as the secp256k1 scalar** | **CONSUMED** |

STATUS: **only epoch_sk CONSUMED; enc/search DEAD-SHIPPED; audit DEAD-DERIVED.** Rotation still re-seals enc/search envelopes for every active member; audit is derived and thrown away.

Curves touched by rotation are three DISTINCT families: **SK_mod/PK_mod = X25519** (minted fresh per rotation) · **Umbral epoch key (epoch_sk/umbral_pk) and member pre-identity-key = secp256k1** (32 B scalar / 33 B compressed pub) · **identity = Ed25519 (seed verbatim) + X25519 (HKDF-SHA256, info `wevibe-x25519-v1`)** — members receive envelopes sealed to the identity X25519 key.

### Known live defect: rotation publishes no Umbral pubkey

**rotateEpoch derives NO Umbral epoch keypair, and its payload carries exactly 4 fields — `new_pk_mod`, `signed_by`, `signature`, `envelopes` (`org-client.ts:1124-1134`). `umbral_pk` is ABSENT** (also absent from the signed canonical, `canonical.ts:69-84`).

Consequence chain:

1. Hub `RotateEpoch` validates only `new_pk_mod`/`signed_by`/`signature`/envelopes (`wevibe-hub/internal/api/handlers/orgs.go:291-299`) — there is **no UmbralPK guard**.
2. `hex.DecodeString("")` on the missing field yields empty bytes (`orgs.go:266`); the new `epoch_manifests` row stores an **empty `umbral_pk` BYTEA** (`orgs.go:271-274`), and the manifest JSON omits the field via `omitempty` (`orgs.go:308-310`, `types.go:96`).
3. Any client reading that manifest — `getEpochUmbralPk` (`org-client.ts:484-493`) — **throws `epoch manifest missing umbral_pk for epoch ${epochId}`** (`:492`).

This breaks both consumers of the epoch Umbral pubkey for **every post-rotation epoch**:

- **Recall** — the DEK wrap chain is DEK → `seal_to_pubkey(DEK, PK_mod)` → Umbral capsule → `ReEncrypt(capsule, kfrag)` → cfrag → `decrypt_reencrypted` → DEK → AES-256-GCM (there is NO K_enc hop). `decrypt_reencrypted` requires the epoch's `umbral_pk` as the delegating pubkey, so recall fails for every post-rotation epoch (`decryptMemoryBlob`, `org-client.ts:527` ← `retrieve-cli.ts:462`).
- **Moderation embed** — approval Umbral-encrypts the DEK under the epoch `umbral_pk`; the embed-retrieval-card path cannot complete without it (`moderation.ts:241` ← `POST /v1/mod/embed-retrieval-card`, `http-server.ts:3303`).

**Epoch 0 is unaffected** — org creation (`buildOrgCryptoSetup`, `org-client.ts:681`) does send `umbral_pk`.

Secondary dead end in the same function: the post-rotate `epoch-sk`/`epoch-pk` envelope re-store branch (`org-client.ts:1156-1176`) is a dead no-op — the hub returns only `{status, buffered_moved}`.

### Kfrags are NOT re-provisioned by rotation

Rotation never touches kfrags. `provisionRecall` (`org-client.ts:786-882`) is a **separate manual operation that must re-run per epoch** to mint kfrags under the new epoch's `epoch_sk` for each member's `pre_pubkey`. Even when it does re-run, the live defect above blocks the path first: an empty manifest `umbral_pk` makes `getEpochUmbralPk` throw before re-provisioned kfrags can be used.

### Key-loss recoverability per class

Two distinct root secrets frame this table (**two secrets**): `K_master` (32 B) → BIP39 24-word **ORG** phrase + Shamir 2-of-3; the identity seed (32 B) is a **SEPARATE** secret with its **OWN** 24-word **IDENTITY** phrase. Neither phrase recovers the other's material, and neither recovers SK_mod or DEKs. Provenance classes referenced below: (a) Keplr/Leap software wallet · (b) platform passkey/biometric authenticator · (c) OS keychain · (d) BIP39 mnemonic · (e) seed file (bench) · (f) in-process random/HKDF · (g) env var/config.

| Key class | Recoverable on loss? | Means |
|---|---|---|
| `K_master` | **Yes** | BIP39 24-word ORG recovery phrase (d) OR Shamir 2-of-3 reconstruction. Generated in-process at org creation (f). |
| enc / search / audit (any epoch) | **Yes** | Deterministic HKDF-SHA256 children of K_master — re-derivable for any epoch (f). |
| `epoch_sk` / `umbral_pk` (any epoch) | **Yes** | Deterministic from K_master (empty-salt HKDF, `wevibe-umbral-epoch-` + decimal) — re-derivable for any epoch (f). |
| `SK_mod` / `PK_mod` | **No — not from any phrase** | Random X25519 keypair minted per epoch (f). Recoverable only via a surviving sealed `mod_envelope` (leader/moderator) or the leader vault. **Loss orphans the old `wrapped_dek_mod` blobs for the moderation path; recall is unaffected** (recall consumes the Umbral capsule, not the mod envelope). |
| DEK (per-memory) | **No long-term backup** | Random per memory; pre-approval it exists only in the pending-vault under the device key; afterwards only as the mod-sealed blob + Umbral capsule (f). |
| Member `pre_key` (pre_sk/pre_pubkey) | **No** | Independent random secp256k1 keypair, not derived from any phrase (f). **Loss = re-invite + new kfrag.** |

## Key-Loss Scenarios

This section absorbs and **retires** `RECOVERY_RUNBOOK.md`. The runbook's five device-loss
scenarios and its identity-regeneration caveat were still accurate and are carried forward below in
corrected form; the runbook's six named MCP tools (`wevibe_setup_identity`, `wevibe_recover_org`,
`wevibe_orgs`, `wevibe_moderate_queue`, `wevibe_invite_member`, `wevibe_contribute`) are stale —
none exists under that name in the current surface, and recovery is now executed with
`wevibe-admin` subcommands.

**Why loss is bounded the way it is.** Nothing server-side can rescue a lost secret: the HUB never
holds `epoch_sk`/`K_master`/DEK/plaintext, and the CHAIN holds pubkeys + content-free events only.
All recovery material therefore lives in exactly four places: the two root secrets' phrase/share
encodings, the sealed envelopes distributed to members, the per-device wrappers over the identity
seed, and (for moderators/members) re-invitation by the leader.

Recoverability of any incident splits along the two secrets, which are independent:

1. **`K_master`** (32B, per-org) → BIP39 24-word **ORG phrase** + Shamir **2-of-3**. Losing it ends
   epoch-key re-derivation for that org.
2. **Identity seed** (32B, per-device) = **SEPARATE**, with its **OWN 24-word IDENTITY phrase**.
   Losing it ends that device's signing/ECDH identity, not its org's keys.

### Facts this section relies on (restated exactly)

- **Epoch keys:** 4 derived from `K_master` via HKDF-SHA256. `enc`/`search`/`audit`: salt = 32×0x00,
  info = `"wevibe-enc-"` | `"wevibe-search-"` | `"wevibe-audit-"` + epoch as 4-byte BE, 32B outputs.
  Umbral: salt = ∅, info = `"wevibe-umbral-epoch-"` + decimal epoch, output used **VERBATIM** as a
  secp256k1 scalar (`epoch_sk`). Status: only `epoch_sk` is CONSUMED; `enc`/`search` are
  DEAD-SHIPPED; `audit` is DEAD-DERIVED.
- **Three distinct curves:** `SK_mod`/`PK_mod` = X25519 · Umbral epoch key + member
  `pre-identity-key` = secp256k1 (32B scalar / 33B compressed pub) · identity = Ed25519 (seed
  verbatim) + X25519 (HKDF info `"wevibe-x25519-v1"`).
- **DEK wrap (NO `K_enc` hop):** DEK → `seal_to_pubkey(DEK, PK_mod)` → Umbral capsule →
  `ReEncrypt(capsule, kfrag)` → cfrag → `decrypt_reencrypted` → DEK → AES-256-GCM. Loss of a
  member's kfrag breaks only that member's re-encryption path; the ciphertext and capsule survive.
- **Key provenance — 7 origins:** (a) Keplr/Leap software wallet; (b) platform passkey/biometric
  authenticator; (c) OS keychain; (d) BIP39 mnemonic; (e) seed file (bench); (f) in-process
  random/HKDF; (g) env var/config. See *Derivation Hierarchy* for the full origin table.

### Current recovery surface (`wevibe-admin`)

| Command | Purpose |
|---|---|
| `wevibe-admin recover-org --org <org_id> --phrase "<ORG phrase>"` | Reconstruct `K_master` from the ORG phrase; re-derive all epoch keys |
| `wevibe-admin setup-threshold --org <org_id> --share2 <hex> --share3 <hex>` | Set up Shamir 2-of-3 backup of `K_master` |
| `wevibe-admin recover-threshold --org <org_id> --share <hex>` | Reconstruct `K_master` from threshold shares |
| `wevibe-admin recovery-status [--org <org_id>]` | Check recovery health |
| `wevibe-admin export-identity` | Show this identity's own 24-word IDENTITY phrase (biometric-gated) |
| `wevibe-admin import-identity --phrase [--force]` | Restore/pair an identity from its IDENTITY phrase |

### Scenario table

| Scenario | Recoverable? | How |
|---|---|---|
| **Leader loses device — HAS recovery phrase** | **Full** (org keys) | `wevibe-admin recover-org` with the **ORG** phrase → reconstructs `K_master` → re-derives all four epoch keys into local memory. Then re-import the sealed envelopes → `SK_mod` (X25519) rehydrates from the leader's personal envelope automatically on membership load. Verify with `wevibe-admin recovery-status`. Identity is the separate secret: if the identity seed was also lost, row 6 restores the *same* identity — a fresh generation instead triggers the caveat below. |
| **Leader loses device — NO recovery phrase** | **Partial** | `SK_mod` is recoverable from the leader's personal mod envelope (`org-{id}-mod-privkey`) once leadership is reassigned on-chain; `K_master` is **gone** — epoch keys cannot be re-derived, existing ciphertext is permanently unreadable, and the org must either derive a **new org** (fresh `K_master`) or **migrate** plaintext retained elsewhere. **Exception:** if Shamir 2-of-3 shares of `K_master` were set up beforehand (`setup-threshold`), any two shares reconstruct `K_master` via `recover-threshold`, converting this row into the one above. Prevention: capture the ORG phrase at org creation *and* set up threshold shares. |
| **Leader loses local mod key only, identity intact** | **Automatic** | Re-importing the personal envelope rehydrates `SK_mod` on membership load — no manual step is required for the leader's own moderation operations. The local keychain entry matters only when sealing `SK_mod` to a *new* moderator; if it is missing and a re-seal is needed: `recover-org` from the ORG phrase, then re-invite the moderator. |
| **Moderator loses device** | **Full** | Leader re-invites: moderator generates a new identity, shares the new ed pubkey, leader re-invites under it with role moderator; the sealed invitation delivers `SK_mod`. |
| **Member loses device** | **Full** | Leader re-invites **+ new kfrag**: member generates a new identity (new Ed25519 seed + new secp256k1 `pre-identity-key`), leader re-invites under the new pubkey with role member, issuing a kfrag for the new receiving key. The DEK wrap chain is unchanged — DEK → `seal_to_pubkey(DEK, PK_mod)` → capsule → `ReEncrypt(capsule, kfrag)` → cfrag → `decrypt_reencrypted` → DEK → AES-256-GCM — so the new kfrag alone restores DEK access to all existing memories; nothing is re-encrypted at rest. |
| **Identity seed lost** | **Full** | The identity phrase **OR any one wrapper** recovers the **same 32B seed**: the IDENTITY phrase (its own 24 words, distinct from the ORG phrase), the Keplr/Leap wallet, the platform passkey/biometric authenticator (Touch ID), or an OS-keychain pairing. Restore via `wevibe-admin import-identity`, or re-pair from the surviving authenticator. |

### Identity-regeneration caveat

Generating a new identity on a replacement device always produces a **NEW** identity — a NEW Ed25519
pubkey. Org membership, leadership, moderator grants, and kfrags are **keyed by the old ed pubkey**
(on the chain and the hub), so a regenerated identity holds no membership and cannot authenticate as
the original leader, moderator, or member. The ORG phrase never restores identity — it restores only
org keys; the two secrets are unrelated. To keep membership intact, **restore the same identity**
(IDENTITY phrase or a surviving wrapper, row 6) instead of regenerating. If regeneration is
unavoidable: members/moderators are restored by leader re-invitation under the new pubkey; a lost
*leader* identity requires the on-chain leader record to be updated through governance (or a
delegated co-leader) **before** the old key is revoked — coordinate the transfer first.

### Verification after any recovery

- `wevibe-admin recovery-status --org <org_id>` reports recovery health.
- Membership loads with the expected role and non-zero epoch-key counts.
- Leaders/moderators: a moderation queue item decrypts; members: a submission round-trips.

No recovery procedure ever writes key material into logs, reports, or tickets — sizes, fingerprints,
and key identities only.

## Known Defects, Risks & Canon Divergences

Every item below is **OPEN-FOR-WALTER** — recorded here so the rewritten canon names its own gaps honestly, and stays marked until Walter rules on each. Evidence anchors are file:line from the five KEYMAP charting passes (all re-verified on-touch). No key material appears anywhere in this document — algorithms, info strings, sizes, paths, mode bits, env-var names, DB columns, and 8-hex fingerprints only.

### 1. All-zeros `HUB_NODE_PRIVKEY` default — LIVE RISK — `OPEN-FOR-WALTER`

The hub's Ed25519 receipt-signing key defaults to an **all-zeros 64-hex value** at three sites:

- `wevibe-server/docker-compose.yml:140` (compose default; `AUDIT.md:176` cites `:134` — line drift),
- code fallback `wevibe-hub/internal/receipts/receipts.go:70-71`,
- `.env.example:45` (empty placeholder).

With the env var unset, usage-receipt commitments are signed with a publicly known key into `usage_receipts.node_signature`, so receipt attestation carries no trust. `AUDIT.md:22,176` flag this CONFIRMED LIVE. Needs: operator-generated key required at boot; defaults deleted, not documented.

### 2. `rotateEpoch` omits the new epoch's `umbral_pk` — LIVE defect — `OPEN-FOR-WALTER`

Context (epoch keys, stated exactly): **4 epoch keys derive from K_master via HKDF-SHA256.** enc/search/audit: salt=32×0x00, info `"wevibe-enc-"` | `"wevibe-search-"` | `"wevibe-audit-"` + epoch as 4-byte big-endian, 32-byte output. Umbral: salt=∅, info `"wevibe-umbral-epoch-"` + epoch decimal ASCII, output used **VERBATIM as a secp256k1 scalar** (reject-not-adjust). STATUS: only `epoch_sk` is CONSUMED; enc/search are DEAD-SHIPPED; audit is DEAD-DERIVED.

The defect: `rotateEpoch` (`wevibe-mcp/src/org-client.ts:1040-1196`) re-derives enc/search, mints a fresh X25519 SK_mod, and re-seals all envelopes — but derives **no Umbral keypair**. Its payload (`:1124-1134`) carries exactly four fields (`new_pk_mod`/`signed_by`/`signature`/`envelopes`); `umbral_pk` is ABSENT (also absent from the signed canonical). Hub `RotateEpoch` (`orgs.go:245-280`) has no `UmbralPK` guard: `hex.DecodeString("")` succeeds → an empty `umbral_pk` BYTEA is stored (`:266-274`), omitted from the manifest via `omitempty`.

Consequence: `getEpochUmbralPk` throws `"epoch manifest missing umbral_pk for epoch ${epochId}"` (`org-client.ts:492`) for every post-rotation epoch → **recall** (`decryptMemoryBlob` `:527`) AND **moderation embed** (`POST /v1/mod/embed-retrieval-card`) break. Epoch 0 is unaffected (`buildOrgCryptoSetup` sends `umbral_pk`). Kfrags are also NOT re-provisioned by rotation — `provisionRecall` must re-run per epoch separately. The post-rotate `epoch-sk`/`epoch-pk` re-store branch (`:1156-1176`) is a dead no-op.

### 3. `machine-seed.bin` KEK = plain SHA-256, NO KDF — construction gap — `OPEN-FOR-WALTER`

In the MCP file-backend keystore, the KEK over `~/.wevibe/keys/keys.json` is `SHA-256("wevibe-keystore-v1" ‖ machine-seed)` — a single unsalted SHA-256 with no iterations and no memory-hardness (`key-store.ts:71-85`), wrapping keys.json with AES-256-GCM (12-byte nonce). The weakness is a **construction gap, not entropy**: `machine-seed.bin` is 32 bytes of OS-random material (not brute-forceable), but it sits **mode 0600 in the same directory as the keys.json it protects** — anyone who can read the keys dir trivially recomputes the KEK. The codebase already uses domain-separated HKDF elsewhere (`"wevibe-x25519-v1"`, `"wevibe-envelope-v1"`); this path bypasses that pattern.

### 4. Stale `f7733d6e` comment reference vs canonical `aa2aa706` — `OPEN-FOR-WALTER`

No bench file hardcodes `f7733d6e` — the three former literals were purged by STRIP-2b, and the fp survives only as a stale comment at `lconfig.py:27` (comment-only). The canonical bench leader anchor is **`aa2aa706` = fp(ed_pubkey_bytes)**, where fp = first 8 bytes of SHA-256 over the Ed25519 pubkey, derived from the bench seed file (`~/.wevibe/bench/leader-seed.txt`, provenance origin **(e) seed file (bench)**; env override `WEVIBE_BENCH_MCP_SEED` takes priority). Note the input ambiguity that burned a prior investigation: for the SAME identity, `fp(seed_bytes)` is a different 8-hex value (`0e93b599`) — always name the hashed input. An identity seam that checks nothing against the current key is a void-instrument risk per the bench RUNBOOK.

### 5. `search_key` — dead-but-shipped; keep or retire? — `OPEN-FOR-WALTER`

`K_search(e)` (HKDF-SHA256 from K_master, salt=32×0x00, info `"wevibe-search-"` + epoch 4-byte BE) is **derived + sealed into `search_envelope`** (`org-client.ts:646/945/1101`) **+ unsealed by `loadMemberships` into `membership.searchKeys`** (`:381-390`) — yet has **zero production readers**. Its natural consumer, `compute_blind_token` (`wevibe-sdk` `lib.rs:92-97` / `crypto.rs:295-298`), has **zero production callers**. History: the 2026-06-05 commit `911774d` dropped K_search from KEY_MANAGEMENT.md (removing the ACCURACY FLAG banner) but never touched code — so this is a live doc/code divergence. Canon question for Walter: is the search-key/blind-token feature intended to return (keep derivation, re-add to canon), or should the search envelope + `compute_blind_token` be retired as dead code?

### 6. D-2.4 vs WHITEPAPER CODE-3 (DEK wrap) — needs a canon ruling — `OPEN-FOR-WALTER`

The live DEK wrap chain, stated exactly (NO K_enc hop): **DEK → seal_to_pubkey(DEK, PK_mod) → Umbral capsule → ReEncrypt(capsule, kfrag) → cfrag → decrypt_reencrypted → DEK → AES-256-GCM.** `wrapped_dek_enc` is a legacy LABEL for the same mod-wrapped blob (`moderation.go:1148` `WrappedDekEnc: wrappedDekMod`) — no re-wrap at any hop.

WHITEPAPER Appendix A (`WHITEPAPER.md:1090,1102-1108`) and §4.3 (`:341`) still carry `"wrapped_dek_enc = AES-256-GCM(K_enc, DEK)"` — **unimplemented**. D-2.4 (Umbral capsule at approval) is what code does, reinforced by D-LEADER-SIDE-UMBRAL-MINT; the whitepaper's own §4.5 (`:355-383`) already contradicts Appendix A. Needs a canon ruling to **retire the CODE-3 K_enc-wrap wording** (the dangling `CODE 1–4` labels at `:337/:341/:345` get fixed with it).

### 7. D-10.2 Shamir banner contradicts now-optional D-10.1 — needs amendment banner — `OPEN-FOR-WALTER`

Two-secrets context, stated exactly: **K_master (32B) → BIP39 24-word ORG phrase + Shamir 2-of-3** (Shamir splits K_master, never the epoch SK). **The identity seed (32B) is SEPARATE, with its OWN 24-word IDENTITY phrase.** `D-10.2` (`DECISIONS.md:1472`) says "Requires 2-of-3 Shamir", contradicting D-10.1's now-OPTIONAL Shamir (`:287`, amended in place at `:1435`). Needs an amendment banner or explicit reconciliation; D-10.2's separate-CLI and on-chain-logged clauses still hold.

### 8. Carry-forward defects and stale records — each `OPEN-FOR-WALTER`

1. **`WEVIBE_HUB_RESPONSE_SEED` ephemeral-if-unset** — the hubsign Ed25519 response key is freshly random per boot when the env var is unset (`hubsign.go:24-54`), so the `X-Hub-Signature` pubkey churns per boot and the on-chain `hub_response_pubkey` (`StoredOrg` field 19) goes stale; response signatures are unverifiable across restarts.
2. **`umbral_pk` signature-uncovered** — absent from both `CreateOrgMessage` (`orgs.go:148`) and `RotateEpochMessage` (`orgs.go:316`); the epoch pubkey the whole recall chain trusts on is not covered by the leader's signature.
3. **social-graph/hub wallet-address preimage uses `pubkeyBytes[1:]` X-only** — incompatible with the real chain wallet address derivation.
4. **Vault uncreatable in production** — `createVault` has zero production callers.
5. **`SECURITY-MODEL.md:38` "Umbral moderation keypair" is STALE/incorrect** — SK_mod/PK_mod = X25519. The curve truth, stated exactly — three DISTINCT curves: **SK_mod/PK_mod = X25519 · Umbral epoch key + member pre-identity-key = secp256k1 (32B scalar / 33B compressed pub) · identity = Ed25519 (seed verbatim) + X25519 (HKDF `"wevibe-x25519-v1"`)**. Code (`crypto.rs:134-145` `StaticSecret`), KEY_MANAGEMENT, and D-2.3 agree; only SECURITY-MODEL.md:38 is wrong.
6. **D-HUB-RESPONSE-SIGNED + D-CHAIN-RESOLVED-HUB-ENDPOINT recorded "not built"** in DECISIONS (`:3270`) but ARE implemented (`hubsign.go` + `X-Hub-Signature` on every response; `StoredOrg.hub_endpoints`/`hub_response_pubkey`/`hub_serving_address` + `GetOrg` query). Status lines need updating.

### Additionally flagged in the KEYMAP reports — each `OPEN-FOR-WALTER`

- **Per-org hub serving key deleted vs stale docs** — D-S32-CO044/CO047 (per-org HD serving key at `m/44'/118'/0'/0/{account_index}` from a hub master mnemonic) were superseded by D-ECON-STORAGE-MARKET amendments 11+13 (`DECISIONS.md:2815,2824`; commit `bb883a8`): ONE global submitter key (`WEVIBE_CHAIN_SUBMITTER_MNEMONIC`) now serves as every org's default serving address. `wevibe-hub/docs/TOPOLOGY.md:819-832` still documents the deleted two-key model — STALE. Related: the submitter env has a **public 12-word test-mnemonic dev default** (`docker-compose.yml:125`; required — FATAL if unset) — same dev-default risk class as item 1.
- **`RUNBOOK-EPOCH-SK-COMPROMISE.md`** — `:9` "2-of-3 shareholders reconstruct the epoch SK" is the wrong mechanism: the epoch SK is never split (Shamir splits K_master); shareholders would reconstruct K_master, then derive epoch_sk (the quorum-governance threat itself is canon-acknowledged, `DECISIONS.md:1446`). `:23,29` on-chain `compromised` status is unimplemented (zero hits in wevibe-chain/ and wevibe-server/).
- **`RUNBOOK-PRE-RECOVERY.md:51`** — "`wevibe-recover` CLI not implemented" is now false: the capability is live as `wevibe-admin recover-org/setup-threshold/recover-threshold/recovery-status` (`admin.ts:700-786`). Ceremony (Shamir 2-of-3 over K_master) remains canon-consistent.
- **`RECOVERY_RUNBOOK.md`** — all six named MCP tools absent from the current surface; omits the Umbral/kfrag + Shamir threshold path; S2 "no self-serve path" outdated. Retire after absorbing corrected scenarios.
- **Tracked `wevibe-docs/RECALL-PIVOT-SPEC.md` mirror stale** vs the operative root copy (missing §8.7 + recall-trigger rulings).
- **`DECISIONS.md:3894` PRF hard-throw anchors stale** — cites `wevibe-auth.ts:422,475`; actuals `:436, :477-478, :552-553`.
- **WHITEPAPER Appendix A derivation drift** — omits the search key entirely and shows no salt for the enc/audit keys (code: salt=32×0x00 for enc/search/audit; salt=∅ for the Umbral epoch seed).
- **Chain "content-free" scope must be stated precisely** — canon invariant: the HUB never holds epoch_sk/K_master/DEK/plaintext (zero repo hits, verified), and the CHAIN holds pubkeys + content-free events. The reports' verification scopes "content-free" to the x/serve event log: the chain memory directory (`StoredMemoryCommitment`) holds ciphertext + wrapped DEK by design, and `StoredMemoryReport` carries a bounded (≤4KB) report path. Docs must carry that scope exactly, not a blanket claim.
- **`MASTER.md:1502`** — one stale line (pre-decision hub-side keygen); note, don't inherit.
- **`HUB-REBUILD.md:143`** — "Umbral sidecar at 127.0.0.1:4460" is stale (container-only port; host 4460 is mandated SILENT per AGENTS.md §2.1).
