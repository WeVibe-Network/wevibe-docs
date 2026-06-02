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

## Step 4: Run recall + contribution flow

**Session start + recall query:**
> Once the plugin is installed and you have joined an org, recall runs on the client side without a manual recall tool call. The plugin harvests session context, builds the retrieval query, and sends it to the hub.

**Memory injection request (default gate):**
> When a relevant memory is found, the plugin shows a Memory Injection Request with the memory plus details (score, matched keywords, wevibe-guard result, contributor stats, and memory stats). Default behavior is **Gated approval**: your agent pauses and waits for your approval before injection.

**Contribution:**
> Contribution is manual and dashboard-driven. Open the dashboard, go to **Sessions**, select a coding session, click **Extract Memories**, review the extracted candidates, choose the destination org for each memory, then click **Submit**. Nothing leaves your machine until you submit.

**Consumer settings (2×2):**
> Content: **Implementations + DNDs** or **DNDs only**. Gate: **Gated approval** or **No gated approval**. Default is **Implementations + DNDs + Gated approval**.

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
