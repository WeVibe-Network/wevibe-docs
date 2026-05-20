# WeVibe Network Validator Setup Runbook

## Overview

WeVibe Network is a Cosmos SDK-based blockchain with 8 modules: `x/attestation`, `x/org`, `x/serving`, `x/challenge`, `x/emissions`, `x/receipt`, `x/reputation`, and `x/operator`. Validators secure the network by participating in consensus and can register as operators via the `x/operator` module.

## Prerequisites

- **Go**: 1.26.1 (chain requires 1.25.9)
- **Docker**: 29.1.3+
- **Docker Compose**: v2+
- **jq**: For JSON processing in scripts
- **80GB+ storage**: For chain data

## Quick Start — Docker Compose

```bash
cd wevibe-chain

# Build and start the local validator
make localnet-start

# View logs
make localnet-logs

# Check chain status
curl -s http://localhost:26657/status | jq '.result.sync_info'

# Stop the chain
make localnet-stop

# Reset chain data
make localnet-reset
```

## Quick Start — Native Binary

```bash
# Build the binary
make build

# Initialize the chain (creates ~/.wevibed)
CHAIN_ID=echo-local-1 MONIKER=my-validator WEVIBED_BINARY=./build/wevibe-chain ./scripts/init-chain.sh start

# In another terminal, check status
curl -s http://localhost:26657/status
```

## Genesis Configuration

Chain ID: `echo-local-1` | Denom: `uecho`

### Accounts

| Account | Balance | Purpose |
|---------|---------|---------|
| `validator` | 1000000000uecho | Initial genesis account |
| `validator` (gentx) | 500000000uecho | Self-delegation for validator |

### Genesis Initialization Flow (`init-chain.sh`)

1. `wevibed init <moniker> --chain-id echo-local-1 --home ~/.wevibed`
2. Patch `config.toml`: change `laddr` from `127.0.0.1:26657` to `0.0.0.0:26657`
3. `wevibed keys add validator --keyring-backend test --home ~/.wevibed`
4. `wevibed genesis add-genesis-account <validator_addr> 1000000000uecho`
5. `wevibed genesis gentx validator 500000000uecho --chain-id echo-local-1`
6. Patch gentx if `delegator_address` is empty
7. `wevibed genesis collect-gentxs --home ~/.wevibed`
8. Configure `echo_epoch` (60s for local, 86400s for production)
9. Configure governance params (deposit, voting periods)
10. Disable `x/mint` inflation (emissions module handles supply)
11. Validate genesis

### Key Files

- Genesis: `~/.wevibed/config/genesis.json`
- Gentx: `~/.wevibed/config/gentx/*.json`
- Config: `~/.wevibed/config/config.toml`
- Priv validator: `~/.wevibed/config/priv_validator_key.json`

## Operator Registration

The `x/operator` module uses `MsgRegisterOperator` with these fields:

```
operator_address  - bech32 address
operator_id      - unique identifier
cons_pub_key     - consensus public key (tendermint pubkey)
bond_amount      - Coin in uecho
```

### Register via CLI

```bash
wevibed tx operator register-operator \
  --operator-address <address> \
  --operator-id <unique-id> \
  --cons-pub-key <tendermint-pubkey> \
  --bond-amount 1000000uecho \
  --keyring-backend test \
  --chain-id echo-local-1
```

### Query Operators

```bash
# Via gRPC gateway (port 9090)
curl -s http://localhost:9090/echo.operator.v1.Query/GetOperator

# Via REST (port 1317)
curl -s http://localhost:1317/echo/operator/v1/operators
```

## Monitoring

### Health Check

```bash
curl -s http://localhost:26657/health
# Returns {} when healthy
```

### Key Metrics

```bash
# Block height
curl -s http://localhost:26657/status | jq '.result.sync_info.latest_block_height'

# Chain ID
curl -s http://localhost:26657/status | jq '.result.node_info.network'

# Peers connected
curl -s http://localhost:26657/net_info | jq '.result.n_peers'

# Genesis chain ID
curl -s http://localhost:26657/genesis | jq '.result.genesis.chain_id'
```

### Smoke Test

```bash
./scripts/smoke-test.sh
# Exits 0 on success, 1 on failure
```

## Troubleshooting

### Chain won't start

```bash
# Check if data already exists
ls ~/.wevibed/config/genesis.json

# Reset if corrupted
make localnet-reset   # Docker
rm -rf ~/.wevibed       # Native
```

### Node stuck at block height 0

```bash
# Check logs for errors
make localnet-logs

# Verify genesis is valid
wevibed genesis validate-genesis --home ~/.wevibed
```

### RPC not responding

```bash
# Verify port is exposed
curl -s http://localhost:26657/status

# Check if container is running
docker ps | grep echo-validator
```

### pprof errors in binary

These are warnings from Cosmos SDK store registration — safe to ignore.

## Reference

### Ports

| Port | Service | Protocol |
|------|---------|----------|
| 26656 | P2P | CometBFT |
| 26657 | RPC | CometBFT |
| 1317 | REST API | Cosmos SDK |
| 9090 | gRPC | Proto |

### Directory Structure

```
~/.wevibed/
├── config/
│   ├── config.toml          # CometBFT config
│   ├── genesis.json        # Chain genesis
│   ├── gentx/              # Validator gentx files
│   ├── priv_validator_key.json
│   └── node_key.json
├── data/
│   └── (chain data)
└── wasm/                   # WASM contracts (if any)
```

### Useful Commands

```bash
# Build
make build

# Initialize (native)
wevibed init <moniker> --chain-id echo-local-1

# Keys
wevibed keys list --keyring-backend test
wevibed keys add validator --keyring-backend test

# Genesis
wevibed genesis add-genesis-account <addr> 1000000000uecho
wevibed genesis gentx validator 500000000uecho --chain-id echo-local-1
wevibed genesis collect-gentxs
wevibed genesis validate-genesis

# Queries
wevibed query block 1
wevibed query tx <hash>
wevibed query wait-tx <hash>

# Transactions
wevibed tx broadcast <tx.json>
wevibed tx sign <tx.json> --keyring-backend test

# Start
wevibed start --home ~/.wevibed
```

### HD Key Derivation

- **Path**: `m/44'/118'/0'/0/0`
- **Key name**: `submitter`
- **Denomination**: `uecho`

## Upgrading wevibed

### Preparations

1. **Monitor chain height** before upgrade window
2. **Backup state** before applying upgrade
3. **Test upgrade** on testnet first

### Manual Upgrade

```bash
# 1. Download new binary
wget https://github.com/wevibe-network/wevibe-chain/releases/v1.1.0/wevibed
chmod +x wevibed
sudo mv wevibed /usr/local/bin/

# 2. Verify version
wevibed version

# 3. Stop node
sudo systemctl stop wevibed

# 4. Apply upgrade (if usingCosmovisor)
export DAEMON_NAME=wevibed
export DAEMON_HOME=/var/lib/wevibed
cosmovisor run start --home $DAEMON_HOME

# Or manually restart
sudo systemctl start wevibed
```

### Cosmovisor Setup

```bash
# Install Cosmovisor
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@latest

# Setup directories
mkdir -p ~/.cosmovisor/genesis/bin
mkdir -p ~/.cosmovisor/upgrades/v1.1.0/bin

# Link current binary
ln -s /usr/local/bin/wevibed ~/.cosmovisor/genesis/bin/wevibed

# Enable auto-download
export DAEMON_ALLOW_DOWNLOAD_BINARIES=true
```

---

## Secure key backup

### Backup Commands

```bash
# Backup genesis
cp /var/lib/wevibed/config/genesis.json /backup/genesis-$(date +%Y%m%d).json

# Backup validator state
tar -czf /backup/priv_validator_state-$(date +%Y%m%d).tar.gz \
  -C /var/lib/wevibed/data .
```

### Automated Backup Script

```bash
#!/bin/bash
# /usr/local/bin/backup-wevibed.sh

DATE=$(date +%Y%m%d-%H%M)
BACKUP_DIR=/backup/wevibed
KEEP_DAYS=7

mkdir -p $BACKUP_DIR

# Stop node
sudo systemctl stop wevibed

# Backup data
tar -czf $BACKUP_DIR/wevibed-data-$DATE.tar.gz /var/lib/wevibed/data

# Backup genesis
cp /var/lib/wevibed/config/genesis.json $BACKUP_DIR/genesis-$DATE.json

# Start node
sudo systemctl start wevibed

# Cleanup old backups
find $BACKUP_DIR -name "*.tar.gz" -mtime +$KEEP_DAYS -delete
find $BACKUP_DIR -name "genesis-*.json" -mtime +$KEEP_DAYS -delete
```

### Recovery Procedure

```bash
# 1. Stop node
sudo systemctl stop wevibed

# 2. Restore data
tar -xzf /backup/wevibed-data-YYYYMMDD.tar.gz -C /var/lib/wevibed/data

# 3. Verify
wevibed genesis validate-genesis /var/lib/wevibed/config/genesis.json

# 4. Start node
sudo systemctl start wevibed
```

---

## Governance participation

### tx gov submit-proposal

Submit a governance proposal.

```bash
wevibed tx gov submit-proposal [proposal_type] [proposal_json_or_file] [flags]
```

### tx gov vote

Vote on a proposal.

```bash
wevibed tx gov vote [proposal_id] [option] [flags]
```

**Options:** `yes`, `no`, `abstain`, `no_with_veto`

### tx gov deposit

Deposit to a proposal.

```bash
wevibed tx gov deposit [proposal_id] [amount] [flags]
```

### query gov proposals

List proposals.

```bash
wevibed query gov proposals [flags]
```

### query gov proposal

Get proposal details.

```bash
wevibed query gov proposal [proposal_id] [flags]
```

### query gov deposits

Get proposal deposits.

```bash
wevibed query gov deposits [proposal_id] [flags]
```

### query gov votes

Get proposal votes.

```bash
wevibed query gov votes [proposal_id] [flags]
```

---
