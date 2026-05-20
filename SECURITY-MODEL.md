# Security Model

## Threat model

WeVibe Network is designed so that even a compromised validator cannot expose memory content without human approval.

**What wevibe-chain validators see:**
- Ciphertext blobs (opaque)
- Wrapped DEKs sealed to epoch public keys
- Retrieval confidence values, lifecycle state, and timestamps
- Serve attestations (content hash, org id, serve key, nullifier)

**What validators never see:**
- Plaintext memory content
- Keyword strings (blind tokens stay local to the org’s client)
- Contributor private keys
- Moderator comments or guard findings

## Key hierarchy

```
Org master key (leader-held)
    │
    └── Epoch keys (derived per epoch via HKDF-SHA256)
            ├── K_enc   — content encryption key
            ├── K_search — blind token derivation key
            └── K_audit  — audit log encryption key
```

Each epoch has its own key set. Members who leave cannot access future epochs.
Members who join cannot access epochs before `history_access_from_epoch`.

## Revocation Scope

Revocation in Echo is cryptographic denial of future re-encryption; it does not erase endpoint-resident plaintext.

| Content Category | Revocation Effective? |
|-----------------|----------------------|
| Un-retrieved memories | YES (cryptographically) |
| In-flight memories | YES if session terminated |
| Already-decrypted memories in agent plaintext | NO |
| Derivative artifacts | NO |

Operationally, revocation removes PRE re-encryption capability by deleting the serving key fragment path. It cannot retroactively erase plaintext already present in agent memory or downstream artifacts generated from that plaintext.

## Submission flow

```
1. Agent writes raw notes.
2. wevibe-guard scans for credentials and prompt injection locally.
3. If clean:
   a. Agent generates a fresh DEK.
   b. Notes encrypted: AES-256-GCM(DEK, notes).
   c. DEK sealed to the moderation public key (`PK_mod`).
   d. Submission hash: SHA-256(ciphertext || wrapped_dek).
   e. Contributor signs hash with their Ed25519 identity key.
   f. wevibe-mcp broadcasts `MsgSubmitCommitment` to wevibe-chain.
4. wevibe-chain stores ciphertext — validators cannot decrypt it.
```

## Moderation flow

```
1. Moderator pulls the pending commitment from wevibe-chain using wevibe-sdk.
2. Moderator decrypts with `SK_mod` for the current epoch and reviews plaintext content.
3. If approved:
   a. Moderator re-wraps the DEK under `K_enc(epoch)` for retrieval.
   b. Optionally sets relationships, validity bounds, or archives conflicts.
   c. Submits `MsgApproveMemory` carrying ciphertext, wrapped DEK, and metadata.
4. wevibe-chain records the approval, initialises retrieval confidence, and updates lifecycle state.
5. echo-clients sync the approval, rebuild local indexes, and keep blind tokens client-side.
```

## Wire format

All cryptographic wire formats are defined in `protocol/test_vectors/` and tested
against the Rust implementation in `wevibe-sdk`.

Key formats:
- `seal_to_pubkey` output: `ephemeral_pubkey(32) || nonce(12) || ciphertext+tag`
- `encrypt_symmetric` output: `nonce(12) || ciphertext+tag`
- Blind token (stored client-side only): `HMAC-SHA256(K_search, keyword)` → lowercase hex
