# Security Model

Last reviewed: Sprint 32 (2026-06)

## Threat model

WeVibe Network is designed so that compromised validators and public chain observers cannot decrypt memory content without endpoint key compromise.

**What wevibe-chain validators see:**
- Ciphertext blobs (opaque)
- Wrapped DEKs (pending-review wraps to `PK_mod`; approved-memory wraps to `K_enc(e)`)
- Plaintext keyword terms and weights (public discovery metadata)
- Retrieval confidence values, lifecycle state, and timestamps
- Serve receipts (content hash, org id, serve key, nullifier)

**What validators never see:**
- Decrypted memory content
- Org secret material (`K_master`, `K_enc(e)`, `K_audit(e)`, `SK_mod`)
- Contributor private keys
- Moderator comments or guard findings

**Custody and signing boundaries:**
- Identity is passkey-first and self-custodial (`D-IDENTITY-PROGRESSIVE-CUSTODY`). The protocol identity keypair (Ed25519 + X25519/PRE) is client-generated, not wallet-derived.
- Contributors are wallet-free and unbonded in normal operation. Reputation is the soft stake and is removed when a contributor is removed from an org.
- The leader is the sole on-chain signer for published content (`D-LEADER-SOLE-SIGNER`) and bears sole bond/accountability for that publication.
- Moderator review is org-local labor. Moderators do not co-sign chain transactions.

## Key hierarchy

```
Org master key (leader-held)
    │
    └── Epoch keys (derived per epoch via HKDF-SHA256)
            ├── K_enc(e)   — approved-memory DEK wrapping key
            └── K_audit(e) — audit log encryption key
```

`PK_mod`/`SK_mod` is a separate Umbral moderation keypair used only to decrypt pending submissions for review. It is distinct from the on-chain signing path.

Each epoch has its own key set. Members who leave cannot access future epochs.
Members who join cannot access epochs before `history_access_from_epoch`.

## Revocation Scope

Revocation in WeVibe is cryptographic denial of future re-encryption; it does not erase endpoint-resident plaintext.

| Content Category | Revocation Effective? |
|-----------------|----------------------|
| Un-retrieved memories | YES (cryptographically) |
| In-flight memories | YES if session terminated |
| Already-decrypted memories in agent plaintext | NO |
| Derivative artifacts | NO |

Operationally, revocation removes PRE re-encryption capability by deleting the serving key fragment path. It cannot retroactively erase plaintext already present in agent memory or downstream artifacts generated from that plaintext.

## Submission flow

```
1. Contributor (passkey identity; wallet optional later) writes raw notes.
2. wevibe-guard scans for credentials and prompt injection locally.
3. If clean:
   a. Contributor generates a fresh DEK.
   b. Notes encrypted: AES-256-GCM(DEK, notes).
   c. DEK sealed to the moderation public key (`PK_mod`).
   d. Submission hash: SHA-256(ciphertext || wrapped_dek).
   e. Contributor signs the submission canonical body with their Ed25519 identity key.
   f. Submission enters the leader-gated pending review path (no contributor auto-approve/certification path).
4. Memory content is committed on-chain only after leader signature.
5. wevibe-chain stores ciphertext — validators cannot decrypt it.
```

## Moderation flow

```
1. Moderator pulls the pending submission using wevibe-sdk.
2. Moderator decrypts with `SK_mod` and reviews plaintext content.
3. If approved:
   a. Moderator re-wraps the DEK under `K_enc(e)` for retrieval and records the review decision.
   b. Leader (not moderator) signs and submits the publish/approval transaction.
4. wevibe-chain records the approval, initializes retrieval confidence, and updates lifecycle state.
5. wevibe-clients sync the approval and rebuild local indexes.
```

## Wire format

All cryptographic wire formats are defined in `protocol/test_vectors/` and tested
against the Rust implementation in `wevibe-sdk`.

Key formats:
- `seal_to_pubkey` output: `ephemeral_pubkey(32) || nonce(12) || ciphertext+tag`
- `encrypt_symmetric` output: `nonce(12) || ciphertext+tag`
- Keyword metadata on-chain: plaintext `keyword` + `weight` pairs (public discovery labels)
