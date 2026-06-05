# WeVibe Network Validator Setup Runbook

Last reviewed: Sprint 32 (2026-06)

## Overview

WeVibe validators are standard CometBFT/Cosmos SDK validators. The chain currently wires these custom modules:

- `x/org`
- `x/memory`
- `x/serve`
- `x/emissions`
- `x/bandwidth`
- `x/reputation`
- `x/attestation` (wired, currently disabled at message execution)

There is **no** `x/operator`, `x/serving`, `x/challenge`, or `x/receipt` module. There is no `MsgRegisterOperator` / `register-operator` validator onboarding transaction.

## Validator identity and key management

- **Passkey-first identity applies to users**, not validator operators.
- Validators use standard Cosmos keyring keys (`wevibed keys ...`) and the standard staking validator lifecycle.

Example key commands:

```bash
wevibed keys add validator --keyring-backend os --home ~/.wevibed
wevibed keys list --keyring-backend os --home ~/.wevibed
```

## Prerequisites

- **Go**: 1.25.9 (matches `wevibe-chain/go.mod`)
- **Docker** + **Docker Compose v2** (for local containerized validator)
- **jq** (required by chain scripts)
- Persistent storage sized for your environment (80GB+ recommended for long-running nodes)

## Quick start — Docker Compose

```bash
cd wevibe-chain

# Build and start the local validator container
make localnet-start

# Follow logs
make localnet-logs

# Check sync info
curl -s http://localhost:26657/status | jq '.result.sync_info'

# Optional smoke test
./scripts/smoke-test.sh

# Stop or reset
make localnet-stop
make localnet-reset
```

## Quick start — Native binary

`scripts/init-chain.sh` defaults `WEVIBED_HOME` to `/root/.wevibed`, so set it explicitly for local native runs.

```bash
cd wevibe-chain
mkdir -p build
go build -o ./build/wevibed ./cmd/wevibed

CHAIN_ID=wevibe-local-1 \
MONIKER=my-validator \
WEVIBED_BINARY=./build/wevibed \
WEVIBED_HOME="$HOME/.wevibed" \
./scripts/init-chain.sh start
```

In another terminal:

```bash
curl -s http://localhost:26657/status | jq '.result.sync_info'
```

## Genesis configuration (`scripts/init-chain.sh`)

Chain ID: `wevibe-local-1`  
Denom: `uvibe`

### Genesis accounts seeded by script

| Account key | Amount | Purpose |
|---|---|---|
| `validator` | `10000000000000uvibe` | Initial validator account |
| `hub-submitter` | `1000000000uvibe` | Hub submitter account (local dev mnemonic) |
| `foundation` | `100000000000000uvibe` | Foundation treasury allocation |
| `faucet` | `1000000000000uvibe` | Local faucet account |

Validator gentx self-delegation: `1000000000000uvibe`.

### Initialization flow performed by script

1. `wevibed init <moniker> --chain-id ... --home ...`
2. Opens RPC/API/gRPC listeners in `config.toml` and `app.toml` (`26657`, `1317`, `9090`).
3. Creates validator key and seeds genesis accounts.
4. Generates and collects validator gentx.
5. Adds `wevibe_epoch` to genesis (default `60s`, override with `WEVIBE_EPOCH_DURATION_SECONDS`).
6. Sets local governance params (`min_deposit=10000000uvibe`, `max_deposit_period=172800s`, `voting_period=172800s`).
7. Disables `x/mint` inflation (supply handled by `x/emissions`).
8. Seeds `app_state.emissions` and `app_state.reputation`.
9. Validates genesis.

## Joining the validator set (standard staking flow)

There is no WeVibe-specific operator registration transaction. Use standard staking commands.

### New chain genesis validator

Use `wevibed genesis gentx ...` and `wevibed genesis collect-gentxs ...` (already handled by `scripts/init-chain.sh` for local setup).

### Existing running network validator

1. Create an operator key:

```bash
wevibed keys add validator --keyring-backend os --home ~/.wevibed
```

2. Get validator consensus pubkey:

```bash
wevibed comet show-validator --home ~/.wevibed
```

3. Create `validator.json` (format expected by `tx staking create-validator`):

```json
{
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"<base64-consensus-pubkey>"},
  "amount": "1000000uvibe",
  "moniker": "my-validator",
  "identity": "",
  "website": "",
  "security": "",
  "details": "",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
```

4. Submit create-validator tx:

```bash
wevibed tx staking create-validator ./validator.json \
  --from validator \
  --chain-id <network-chain-id> \
  --keyring-backend os \
  --home ~/.wevibed
```

5. Verify validator appears:

```bash
VALOPER=$(wevibed keys show validator --bech val --address --keyring-backend os --home ~/.wevibed)
wevibed query staking validator "$VALOPER" --home ~/.wevibed
```

## Monitoring

```bash
# Node health
curl -s http://localhost:26657/health

# Latest block height
curl -s http://localhost:26657/status | jq '.result.sync_info.latest_block_height'

# Chain ID from node status
curl -s http://localhost:26657/status | jq '.result.node_info.network'

# Peer count
curl -s http://localhost:26657/net_info | jq '.result.n_peers'

# Genesis chain ID
curl -s http://localhost:26657/genesis | jq '.result.genesis.chain_id'
```

Smoke test:

```bash
./scripts/smoke-test.sh
```

## Troubleshooting

### Chain does not start

```bash
# Validate genesis for local home
wevibed genesis validate --home ~/.wevibed

# Docker reset
make localnet-reset

# Native reset (destructive)
rm -rf ~/.wevibed
```

### Node stuck at height 0

```bash
# Container logs
make localnet-logs

# Verify RPC status payload
curl -s http://localhost:26657/status | jq '.result.sync_info'
```

### RPC not responding

```bash
curl -s http://localhost:26657/health

# If running in Docker
docker compose ps
```

## Reference

### Ports

| Port | Service | Protocol |
|---|---|---|
| 26656 | P2P | CometBFT |
| 26657 | RPC | CometBFT |
| 1317 | API | Cosmos SDK API |
| 9090 | gRPC | Cosmos SDK gRPC |

### Directory layout

```text
~/.wevibed/
├── config/
│   ├── app.toml
│   ├── config.toml
│   ├── genesis.json
│   ├── gentx/
│   ├── node_key.json
│   └── priv_validator_key.json
└── data/
    └── priv_validator_state.json
```

### Useful CLI commands

```bash
# Build daemon
go build -o ./build/wevibed ./cmd/wevibed

# Genesis operations
wevibed genesis add-genesis-account <addr> <amount> --home ~/.wevibed
wevibed genesis gentx validator 1000000000000uvibe --chain-id wevibe-local-1 --home ~/.wevibed
wevibed genesis collect-gentxs --home ~/.wevibed
wevibed genesis validate --home ~/.wevibed

# Query operations
wevibed query block 1
wevibed query tx <hash>
wevibed query wait-tx <hash>
wevibed query staking validators

# Start daemon
wevibed start --home ~/.wevibed
```

## Upgrading `wevibed`

1. Backup validator keys/state before replacing binaries.
2. Build the target binary from the checked-out release source:

```bash
cd wevibe-chain
go build -o ./build/wevibed ./cmd/wevibed
./build/wevibed version
```

3. Replace the running binary using your process manager (systemd/Cosmovisor/container image), then restart and verify RPC health.

Example (systemd-managed host):

```bash
sudo systemctl stop wevibed
sudo install -m 0755 ./build/wevibed /usr/local/bin/wevibed
sudo systemctl start wevibed
curl -s http://localhost:26657/health
```

## Secure key/state backup

Back up these files securely (offline/encrypted storage):

- `/var/lib/wevibed/config/priv_validator_key.json`
- `/var/lib/wevibed/data/priv_validator_state.json`
- `/var/lib/wevibed/config/node_key.json`
- `/var/lib/wevibed/config/genesis.json`

Example backup commands:

```bash
sudo systemctl stop wevibed

sudo tar -czf /backup/wevibed-config-$(date +%Y%m%d).tar.gz \
  -C /var/lib/wevibed/config \
  genesis.json priv_validator_key.json node_key.json

sudo cp /var/lib/wevibed/data/priv_validator_state.json \
  /backup/priv_validator_state-$(date +%Y%m%d).json

sudo systemctl start wevibed
```

Recovery (example):

```bash
sudo systemctl stop wevibed

sudo tar -xzf /backup/wevibed-config-YYYYMMDD.tar.gz -C /var/lib/wevibed/config
sudo cp /backup/priv_validator_state-YYYYMMDD.json /var/lib/wevibed/data/priv_validator_state.json

wevibed genesis validate /var/lib/wevibed/config/genesis.json

sudo systemctl start wevibed
```

## Governance participation

Use the standard Cosmos gov CLI surface exposed by `wevibed`:

```bash
# Submit proposal from JSON file
wevibed tx gov submit-proposal ./proposal.json --from <key> --chain-id <chain-id>

# Vote
wevibed tx gov vote <proposal-id> yes --from <key> --chain-id <chain-id>

# Deposit
wevibed tx gov deposit <proposal-id> <amount> --from <key> --chain-id <chain-id>

# Query proposals and votes
wevibed query gov proposals
wevibed query gov proposal <proposal-id>
wevibed query gov votes <proposal-id>
```

Use `wevibed tx gov --help` and `wevibed query gov --help` for full flag sets.
