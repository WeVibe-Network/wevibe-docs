# Recovery Runbook

This document covers recovery procedures for all key loss scenarios in WeVibe Network.
For background on the key hierarchy, see [KEY_MANAGEMENT.md](KEY_MANAGEMENT.md).

## Before you begin

All recovery procedures assume:
- An wevibe-chain endpoint is reachable at your configured `WEVIBE_CHAIN_RPC` / `WEVIBE_CHAIN_GRPC`
- The recovering agent has `wevibe-mcp` installed and configured
- You know the org ID (`echo_orgs` will list it once your identity is loaded)

---

## Scenario 1: Leader loses device — has recovery phrase

**Impact:** Loss of access to K_master and all locally cached keys.

**What is recoverable:** Everything. `K_master` is encoded in the recovery phrase.
`SK_mod` is recoverable from the leader’s sealed envelope (managed via anchor).

**Steps:**

1. On the new device, set up an identity:
   ```
   > Call echo_setup_identity
   ```
   This generates a new Ed25519 + X25519 keypair in the OS keychain.

   **Note:** This is a NEW identity. Your old identity (pubkey) remains the org leader
   on wevibe-chain — the new one does not. You cannot recover the original identity
   keypair without a separate backup, so you cannot authenticate as the original leader
   unless governance (or a co-leader) updates the on-chain record.

   **Implication:** If you have lost your device AND need to perform leader operations
   (invite members, rotate epoch, etc.), coordinate a leadership transfer via
   wevibe-chain governance or your chosen analytics surface before revoking the old key.

2. Restore K_master from the recovery phrase:
   ```
   > Call echo_recover_org with your 24-word recovery phrase
   ```
   This reconstructs K_master and re-derives all epoch keys into local memory.

3. Verify epoch keys are accessible:
   ```
   > Call echo_orgs
   ```
   Your org should appear with non-zero `encKeys` and `searchKeys` counts.

4. `SK_mod` is recovered automatically by `loadMemberships` (called internally by
   `echo_orgs`) from your sealed leader envelope — no manual step required.

5. Verify moderation access:
   ```
   > Call echo_moderate_queue for your org
   ```
   If items are returned and decrypt successfully, recovery is complete.

---

## Scenario 2: Leader loses device — no recovery phrase

**Impact:** K_master is unrecoverable. Epoch key re-derivation is not possible.

**What is recoverable:** `SK_mod` only (from the leader’s sealed envelope, after leadership is reassigned on-chain).

**What is NOT recoverable:** K_master, K_enc(e), K_search(e), K_audit(e).
Existing encrypted memories remain on wevibe-chain (and any analytics replicas) but cannot be decrypted without `K_enc(e)`.
New epoch key derivation is not possible.

**Steps:**

1. Coordinate with network governance or a co-leader. A governance transaction (or prior co-leader delegation) must update the leader pubkey on-chain. There is no self-serve path once `K_master` is lost.

2. Once the on-chain leader record (or your analytics surface) reflects the new pubkey, set up a new identity on the new device:
   ```
   > Call echo_setup_identity
   ```

3. Re-import your sealed leader envelope (created by anchor). Once the blockchain recognizes the new leader pubkey, `echo_orgs` will hydrate `SK_mod` automatically.

4. For members: existing envelopes are sealed to old member pubkeys. Re-invite affected members so fresh envelopes can be delivered.

**Prevention:** Record your recovery phrase at org creation. See [KEY_MANAGEMENT.md](KEY_MANAGEMENT.md).

---

## Scenario 3: Leader loses local mod key only (identity intact)

**Impact:** The local keychain entry `org-{id}-mod-privkey` is missing or corrupted.
The leader's identity keypair is intact.

**What is recoverable:** SK_mod, automatically.

**Steps:**

No manual steps required. `loadMemberships` (called by `echo_orgs` and moderation tools)
re-imports `SK_mod` from the leader’s sealed envelope on every call. The local keychain
entry is used by the anchor tooling when sealing `SK_mod` to new moderators, but is not
required for the leader’s own moderation operations.

If you need to re-seal SK_mod to a new moderator and the local entry is missing:

1. Recover K_master from the recovery phrase:
   ```
   > Call echo_recover_org with your 24-word recovery phrase
   ```

2. After recovery, re-invite the moderator:
   ```
   > Call echo_invite_member with the moderator's pubkey and role: moderator
   ```

---

## Scenario 4: Moderator loses device

**Impact:** Loss of the moderator's identity keypair and locally cached mod keys.

**What is recoverable:** Everything. The leader re-issues the invitation.

**Steps:**

1. Moderator sets up a new identity on the new device:
   ```
   > Call echo_setup_identity
   ```
   This generates a new Ed25519 + X25519 keypair. The new pubkey is different from
   the old one.

2. Moderator shares their new pubkey with the org leader:
   ```
   > Call echo_orgs
   ```
   The pubkey is shown in the identity info output.

3. If your analytics/dashboard surface supports membership removal, use it; otherwise
   simply re-invite the moderator under the new pubkey:
   ```
   > Call echo_invite_member with the moderator's new pubkey and role: moderator
   ```

4. On the new device, moderator calls `echo_orgs` to load their membership.
   SK_mod will be included in the sealed invitation.

---

## Scenario 5: Member loses device

**Impact:** Loss of the member's identity keypair and epoch keys.

**Steps:**

1. Member sets up a new identity:
   ```
   > Call echo_setup_identity
   ```

2. Member shares new pubkey with org leader.

3. Leader re-invites:
   ```
   > Call echo_invite_member with the member's new pubkey and role: member
   ```

4. Member calls `echo_orgs` to load the new membership and epoch keys.

---

## Verification checklist

After any recovery procedure, verify:

- [ ] `echo_orgs` returns the expected org with correct role
- [ ] `encKeys` and `searchKeys` counts are non-zero
- [ ] For leaders/moderators: `echo_moderate_queue` returns items and decrypts without error
- [ ] For members: `echo_contribute` completes without error on a test submission
