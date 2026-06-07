# Self-Hosting WeVibe Network

Last reviewed: Sprint 32 (2026-06)

## Overview

This guide covers the current self-hosting setup for WeVibe using the Docker stack in `wevibe-server/docker-compose.yml` and operational targets in `wevibe-meta/Makefile`.

The stack runs 9 services:

1. `wevibe-postgres`
2. `wevibe-qdrant`
3. `wevibe-chain`
4. `wevibe-umbral`
5. `wevibe-social-graph`
6. `wevibe-faucet`
7. `wevibe-hub`
8. `wevibe-mcp`
9. `wevibe-dashboard`

Operational model:

- Identity is passkey-first; wallet connection is optional until a flow needs chain signing.
- The hub records and serves data. Leader chain transactions are signed in the dashboard (CosmJS), not by the hub.

## Prerequisites

- Git
- Docker Engine + Docker Compose v2
- GNU Make
- Ollama running on the host (`http://localhost:11434`)
- Node.js 20+ on the leader machine (for the leader-local moderation MCP server)

## Repos to clone

`docker-compose.yml` uses sibling paths (for example `../wevibe-chain` from inside `wevibe-server`). Clone repos into one shared parent directory.

```bash
mkdir wevibe-workspace
cd wevibe-workspace

git clone https://github.com/WeVibe-Network/wevibe-meta.git
git clone https://github.com/WeVibe-Network/wevibe-server.git
git clone https://github.com/WeVibe-Network/wevibe-chain.git
git clone https://github.com/WeVibe-Network/wevibe-social-graph.git
git clone https://github.com/WeVibe-Network/wevibe-umbral.git
git clone https://github.com/WeVibe-Network/wevibe-mcp.git
git clone https://github.com/WeVibe-Network/wevibe-sdk.git
git clone https://github.com/WeVibe-Network/wevibe-faucet.git
```

The faucet service is built from `./wevibe-faucet` (compose build context: `../wevibe-faucet`).

## Bring up the stack

From `wevibe-meta`:

```bash
make docker-up
```

`make docker-up` runs `docker compose up -d --build` in `wevibe-server` and waits for the stack health script to complete.

Published ports from `wevibe-server/docker-compose.yml`:

| Service | Host port(s) | Notes |
|---|---:|---|
| `wevibe-postgres` | `5433` | PostgreSQL (`5432` in-container) |
| `wevibe-qdrant` | `6333` | Qdrant API |
| `wevibe-chain` | `26656`, `26657`, `1317`, `9090` | P2P, RPC, REST, gRPC |
| `wevibe-umbral` | none published | Internal sidecar on `4460` |
| `wevibe-social-graph` | `4470` | Health: `/v1/health` |
| `wevibe-faucet` | none published | Internal-only listener `:4470` |
| `wevibe-hub` | `4440` | Health: `/health` |
| `wevibe-mcp` | `4450` | Container MCP gateway |
| `wevibe-dashboard` | `3000` | Browser UI |

Chain initialization is automatic on first boot: the `wevibe-chain` container entrypoint runs `wevibe-chain/scripts/init-chain.sh` and starts the node.

## Caddy / reverse-proxy note

`wevibe-server/wevibe-infra/Caddyfile` reverse-proxies to `wevibe-hub:4440`:

- site address: `{$WEVIBE_DOMAIN:localhost}`
- upstream: `wevibe-hub:4440`

Use this as the hub reverse-proxy layer when exposing hub over a domain. Caddy is not one of the 9 services in `docker-compose.yml`.
When `WEVIBE_DOMAIN` is set, Caddy can provision HTTPS automatically.

## VPS / split-host configuration (`.env`)

`wevibe-server/.env.example` defines browser-facing dashboard URLs. For VPS or split-host deploys, create `wevibe-server/.env` and set public endpoints:

- `WEVIBE_HUB_URL`
- `WEVIBE_CHAIN_RPC`
- `WEVIBE_CHAIN_REST`
- `WEVIBE_SOCIAL_GRAPH_URL`
- chain display/config fields (`WEVIBE_CHAIN_ID`, `WEVIBE_BECH32_PREFIX`, `WEVIBE_COIN_DENOM`, `WEVIBE_COIN_MIN_DENOM`)

Compose uses `${VAR:-default}` fallbacks, so unset values use local defaults.

## Leader-local moderation MCP server (required caveat)

`WEVIBE_MCP_URL` must stay `http://localhost:4451`. It points to the leader-local moderation MCP server, which holds the leader's Umbral moderation key and must not be moved into Docker/VPS hosting.

Run it on the leader machine from the `wevibe-mcp` repo:

```bash
cd wevibe-mcp
npm run dashboard
```

Default dashboard MCP port is `4451`.

## Health checks

From `wevibe-meta`:

```bash
make health
```

This checks hub (`4440`), dashboard (`3000`), qdrant (`6333`), chain RPC (`26657`), social graph (`4470`), container MCP (`4450`, with token), and host Ollama (`11434`).

## Reset / teardown

From `wevibe-meta`:

```bash
make docker-down
```

Full reset (wipe volumes, then rebuild/restart):

```bash
make docker-down && make docker-up
```
