# Leader Quick Start

Last reviewed: Sprint 32 (2026-06)

This guide covers org creation and administration. For member setup, see
[quickstart.md](quickstart.md). For key management background, see
[KEY_MANAGEMENT.md](KEY_MANAGEMENT.md).

## Prerequisites

- Node.js 20+
- wevibe-mcp installed: `npm install -g wevibe-mcp` or run via `npx wevibe-mcp`
- Ollama running locally for auto-extraction and embeddings (optional but recommended)
- Access to a wevibe-chain endpoint (local or remote), with VIBE in your own wallet to acquire an org slot — org capacity is a scarce, capped set of slots acquired by auction, and you sign the registration from your own wallet (DECISIONS.md `D-ECON-STORAGE-MARKET`, `D-S32-CO044-REGISTERORG-FLOW`)
- A secure place to store your recovery phrase (paper, hardware vault, or password manager)

## Ollama Setup (Optional — Required for Auto-Extraction)

For automatic session extraction and semantic embeddings, install Ollama:

```bash
# Install Ollama (macOS)
brew install ollama

# Start Ollama
ollama serve

# Pull required models (one-time setup)
ollama pull qwen3:4b    # For session extraction
ollama pull nomic-embed-text:v1.5  # For semantic embeddings (PINNED — never :latest)
```

Configure environment variables:
- `WEVIBE_OLLAMA_URL` — Ollama endpoint (default: `http://localhost:11434`)
- `WEVIBE_EXTRACTION_MODEL` — Extraction model (default: `qwen3:4b`)
- `WEVIBE_EMBEDDING_MODEL` — Embedding model (default: `nomic-embed-text:v1.5`, pinned)
- `WEVIBE_AUTO_CONTRIBUTE` — Set to `0` to disable auto-extraction (default: `1`)

## Step 1: Configure your agent

Add to your agent's MCP config:

(identical to the MCP config in QUICKSTART.md)

```json
{
  "mcpServers": {
    "wevibe": {
      "command": "wevibe-mcp",
      "env": {
        "WEVIBE_CHAIN_RPC": "http://localhost:26657",
        "WEVIBE_CHAIN_GRPC": "http://localhost:9090",
        "WEVIBE_READ_ONLY": "0"
      }
    }
  }
}
```

## Step 2: Set up your identity

Identity is passkey-first (DECISIONS.md `D-IDENTITY-PROGRESSIVE-CUSTODY`): your signing and encryption keys are client-generated and stored in your OS keychain.

```
> Call wevibe_setup_identity
```

This generates your Ed25519 signing keypair and X25519 encryption keypair, stored in
your OS keychain. Your Ed25519 public key is your permanent leader identity for the org.

A wallet is still required, but only to acquire and bond the org slot during org creation under the storage-market economy.

## Step 3: Create your org

```
> Call wevibe_create_org with:
  - org_name: "Your Org Name"
  - domain: "React, Next.js, TypeScript"   # your domain of EXPERTISE / specialization — NOT a DNS host
```

Creation acquires an org **slot** at the current auction price (ascending while the slot cap fills, or a descending Dutch-resale price for a freed slot) and is signed from your own wallet. The resulting `org_id` is a permanent slot identifier, independent of your wallet — it survives a future leadership transfer or resale.

After creation, wevibe-mcp (via wevibe-sdk) will display a **24-word recovery phrase**.
This is the only copy. It encodes your org master key (`K_master`).

### Record the recovery phrase now

Write it down before dismissing the output. If you lose your device without this phrase,
epoch key re-derivation is not possible and existing memories cannot be decrypted.

- Store it offline (paper or hardware vault)
- Do not store it in plaintext on the same device as the running agent
- Do not paste it into any AI tool or chat interface

## Step 4: Invite members

For each team member who should contribute memories:

```
> Call wevibe_invite_member with:
  - org_id: [your org ID from wevibe_orgs]
  - invitee_pubkey: [member's Ed25519 pubkey hex]
  - invitee_x25519_pubkey: [member's X25519 pubkey hex]
  - role: "member"
```

Members retrieve their pubkeys by calling `wevibe_setup_identity` on their device
and sharing the output with you.

## Step 5: Invite moderators

Moderators review and approve submitted memories before they enter the knowledge base.
Invite trusted team members as moderators:

```
> Call wevibe_invite_member with:
  - org_id: [your org ID]
  - invitee_pubkey: [moderator's Ed25519 pubkey hex]
  - invitee_x25519_pubkey: [moderator's X25519 pubkey hex]
  - role: "moderator"
```

Moderators receive a sealed copy of `SK_mod` — the private key required to decrypt
pending submissions. Use the anchor CLI or dashboard to deliver envelopes securely. As
leader, you also hold `SK_mod` and can moderate directly.

## Step 6: Review the moderation queue

You and your designated moderators can review and act on pending submissions:

```
> Call wevibe_moderate_queue for org [org ID]
```

This returns pending items with decrypted plaintext (if your mod key is loaded).

To approve:
```
> Call wevibe_moderate_approve for submission [hash]
```

To deny:
```
> Call wevibe_moderate_deny for submission [hash] with reason: "..."
```

## Step 7: Verify org health

After setup, confirm everything is working:

```
> Call wevibe_orgs
```

Check:
- Your role is `leader`
- `encKeys` and `searchKeys` counts are non-zero (epoch keys loaded)
- `modPrivkey` is non-null (mod key loaded from your local envelope)

## Recovery

If you lose access to your device:

1. On the new device, call `wevibe_setup_identity` to generate a new identity.
2. Call `wevibe_recover_org` with your 24-word recovery phrase to restore `K_master`
   and re-derive epoch keys.
3. Re-import your personal sealed envelope (managed by anchor) to recover `SK_mod`.

See [RECOVERY_RUNBOOK.md](RECOVERY_RUNBOOK.md) for complete recovery procedures covering all scenarios.
