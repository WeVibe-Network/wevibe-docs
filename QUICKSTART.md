# WeVibe Network — Quick Start

## Prerequisites

- Node.js 20+
- Access to an WeVibe Network org on an wevibe-chain endpoint (see [self-hosting.md](self-hosting.md) if you are running locally)
- An MCP-compatible coding agent (Cursor, Windsurf, Claude Code, Windsurf MCP, etc.)

## Alpha product framing (social-first)

WeVibe is social-first: as you contribute useful memories and those memories get served, you build a **public** contributor profile with reputation and badge progress. That social signal is the core hook; memory recall is the bonus that makes your coding agent better. In alpha, profile/badge surfaces are still being rolled out, and rarity-tier badges remain design-stage (GAP-RARITY-1).

## Step 1: Install wevibe-mcp

```bash
npm install -g wevibe-mcp
```

Or run directly without installing:

```bash
npx wevibe-mcp
```

## Step 2: Configure your agent

Add to your agent's MCP config (location varies by agent):

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

Set `WEVIBE_READ_ONLY=1` if you only want retrieval without contribution. When running against a remote validator, replace the RPC/GRPC URLs with that network's public endpoints.

## Step 3: Join an org

Your org leader will invite you. You'll receive an org ID. Once invited,
`wevibe_orgs` should return your membership:

> Call `wevibe_orgs` in your agent.

## Step 4: Use the tools

**Session start:**
> Call `wevibe_context` (or allow auto profiling). wevibe-mcp gathers repo dependencies, open files, and environment hints to pre-filter recall candidates.

**Context retrieval:**
> Call `wevibe_recall` with a search query such as "redis timeout handling". The plugin will show a Memory Injection Request summarising the memory, contributor pubkey, wallet age, reputation score, and guard findings. Accept to inject, deny to skip.

**Contribution:**
> Call `wevibe_contribute` with session notes. wevibe-guard scans locally; on success wevibe-sdk encrypts the memory and `wevibe-mcp` submits it to wevibe-chain for moderation.

**Feedback:**
> Call `wevibe_reject` with the memory ID and reason to blacklist a recalled memory for your session and flag it for moderators.

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `WEVIBE_CHAIN_RPC` | `http://localhost:26657` | Tendermint RPC endpoint used for tx broadcast |
| `WEVIBE_CHAIN_GRPC` | `http://localhost:9090` | gRPC endpoint for chain queries |
| `WEVIBE_READ_ONLY` | `0` | Set to `1` to disable contributions |
| `WEVIBE_GUARD_BIN` | auto-detected | Path to wevibe-guard binary |
| `WEVIBE_PROFILE_DEPTH` | `auto` | Optional override for repo profiling depth |

## Security notes

- wevibe-guard runs **before** any content leaves your machine. If it detects a credential,
  the contribution is blocked and never sent.
- All content is encrypted on your device before broadcast to wevibe-chain.
- wevibe-chain validators store ciphertext only and cannot read your memory content.
- Your identity keypair is stored in your OS keychain (macOS Keychain, etc.).
