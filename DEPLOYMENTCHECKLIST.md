# Deployment Checklist — WeVibe Network

Last reviewed: Sprint 32 (2026-06)

## Alpha Dogfood (Single VPS, Leader + Moderator Cohort)

### Infrastructure

- [ ] **VPS provisioned** — 1 machine, 4-8 vCPU, 16-32GB RAM, 200GB NVMe SSD (Hetzner CPX41 or equivalent, ~€30/month)
- [ ] **Domain configured** — `alpha.wevibe.network` A record pointing to VPS IP
- [ ] **TLS via Caddy** — Caddy reverse proxy in front of hub (`:4440`) + dashboard (`:3000`) + social-graph (`:4470`). Auto-TLS via Let's Encrypt. Verify `wevibe-server/wevibe-infra/Caddyfile` covers all public endpoints.
- [ ] **Firewall rules** — only expose: 443 (HTTPS), 26656 (CometBFT P2P), 26657 (CometBFT RPC). All Docker service ports (4440, 4450, 4470, 3000, 5432, 6333) internal only.
- [ ] **Docker + Docker Compose installed** on VPS
- [ ] **All 9 Docker services running** — postgres, qdrant, wevibe-chain, wevibe-umbral, wevibe-social-graph, wevibe-faucet, wevibe-hub, wevibe-mcp, wevibe-dashboard
- [ ] **Ollama decision** — either: (a) each tester runs Ollama locally for extraction (250MB model download), or (b) run Ollama on VPS without GPU (slower but removes tester friction). Document choice.
- [ ] **Uptime monitor** — free tier (UptimeRobot or Healthchecks.io) pinging `https://alpha.wevibe.network/health` and alerting Walter

### Chain Genesis

- [ ] **chain_id** — `wevibe-alpha-1`
- [ ] **Block time** — 6 seconds (CometBFT default, instant finality)
- [ ] **Chain epoch duration** — 12 hours (testable decay/payout cycles within a day)
- [ ] **min_gas_price** — 0.01 uvibe
- [ ] **Daily emission** — set a number (e.g., 1000 VIBE/day). Test tokens, no real value.
- [ ] **validator_share** — 10% (Walter is sole validator, 90% to org contributor pools)
- [ ] **Walter's validator stake** — genesis allocation (e.g., 100,000 VIBE)
- [ ] **Tester faucet allocation** — e.g., 1,000 VIBE per tester for gas fees. Manual transfer or simple faucet script.
- [ ] **Genesis file generated** — `wevibed init` + genesis params configured + validator created
- [ ] **Validator key backed up** — separate from VPS (offline copy of validator key + node key)

### Tester Onboarding

- [ ] **Quickstart doc written** — step-by-step: install Keplr → get test VIBE → join org → install OpenCode + plugin → configure wevibe-mcp → extract first memory
- [ ] **Wallet setup guide** — how to add WeVibe alpha chain to Keplr/Leap (custom chain config with RPC endpoint)
- [ ] **Plugin installation guide** — how to install wevibe-guard plugin in OpenCode, configure `~/.wevibe/plugin-config.json`
- [ ] **wevibe-mcp connection config** — how to point wevibe-mcp at `alpha.wevibe.network` instead of localhost
- [ ] **Org creation** — Walter creates the initial org(s) before testers arrive. Testers join via `/discover` + join request.
- [ ] **Communication channel** — private Discord server or Slack workspace. Channels: #general, #bugs, #feedback
- [ ] **Known limitations doc** — what works, what doesn't, what's expected to break
- [ ] **Tester tracking** — spreadsheet or Notion with onboarding status, invite sent, credits allocated, feedback recorded

### Data Hygiene

- [ ] **Moderation queue clean** — no pending submissions lingering from internal test runs
- [ ] **Org credits loaded** — each org has initial credit balance for serve queries
- [ ] **Keyword taxonomy** — initial vocabulary seeded (if applicable) or leaders briefed on manual seeding

### Incident Response

- [ ] **Pager on-call** — Slack/Discord notifications configured for downtime alerts
- [ ] **Fallback plan** — ability to pause submissions if something goes wrong (disable wevibe-mcp write path via feature flag or env var)
- [ ] **Support workflow** — testers report bugs via dedicated channel, scribe captures reproduction steps + logs

## Testnet Launch (Public, Multitenant)

### Infrastructure

- [ ] **VPS 1 — Validator** — dedicated machine, 4 vCPU, 8GB RAM, 500GB SSD. Runs only `wevibed`. Hardened: no SSH password auth, fail2ban, unattended-upgrades.
- [ ] **VPS 2 — Services** — 8 vCPU, 32GB RAM, 500GB SSD. Runs: hub, dashboard, social-graph, wevibe-mcp, postgres, qdrant, umbral-sidecar.
- [ ] **VPS 3 — Second validator (optional)** — community-run or WeVibe-run. 4 vCPU, 8GB RAM, 500GB SSD. Provides basic fault tolerance (chain continues if one validator is down).
- [ ] **Domain** — `testnet.wevibe.network` for services, `rpc.testnet.wevibe.network` for chain RPC
- [ ] **TLS** — Caddy on services VPS, separate TLS for validator RPC if publicly exposed
- [ ] **CDN for dashboard** — consider static export + CDN (Cloudflare, Vercel) instead of running Next.js server. Reduces VPS load.
- [ ] **Monitoring stack** — Prometheus + Grafana (or equivalent). Dashboards for: block height, TX throughput, hub request rate, Qdrant query latency, PostgreSQL connection pool, memory commit rate, emission payout totals. Alert on: service down, disk >80%, memory >90%, chain halted, hub error rate spike.

### Chain

- [ ] **chain_id** — `wevibe-testnet-1`
- [ ] **Genesis ceremony** — documented procedure for generating genesis with multiple validators
- [ ] **Validator set** — minimum 2, target 4+ for meaningful decentralization. Each validator operator has their own machine.
- [ ] **Emission schedule finalized** — daily emission amount, halving/decay schedule, total supply trajectory
- [ ] **Bootstrap credit program parameters** — how many credits, duration, who qualifies (per DECISIONS.md D-8 area)
- [ ] **Chain epoch duration** — evaluate whether 12h is right for testnet or adjust
- [ ] **Governance module** — `x/gov` proposals enabled for parameter changes (emission rate, required_approvals defaults, etc.)
- [ ] **Chain explorer** — Ping.pub (open source Cosmos explorer) or custom. Deployed at `explorer.testnet.wevibe.network`. Shows: blocks, transactions, validators, account balances, memory commitments.

### Security — Hardened

- [ ] **ARCH-G9 implemented** — identity is passkey-first and the PRE key is client-generated (not wallet-derived), per DECISIONS.md `D-IDENTITY-PROGRESSIVE-CUSTODY` (amends D-1.4). This is a security requirement before public exposure.
- [ ] **Rate limiting hardened** — per-wallet, per-org, per-endpoint limits on hub. DDoS protection via Caddy rate limiting or Cloudflare proxy.
- [ ] **Sybil mitigation** — stake requirement for org creation (minimum VIBE stake to create an org). Minimum member count (e.g., 3) before org qualifies for emissions.
- [ ] **Spam mitigation** — gas costs on chain submission are the primary defense. Hub-side rate limiting on submit endpoint as secondary.
- [ ] **Report abuse mitigation** — D-7.6 (sword cuts both ways) is already designed. Verify it works: dismissed report count increments, malicious reporter can be removed.
- [ ] **Embedding inversion** — Gaussian noise is now DISABLED by default (σ=0, D-9.5 — it cost ~20pp recall and had no rationale). Active mitigations: API key auth + Qdrant internal-only. Document limitations publicly.

### Token Economics

- [ ] **Economics paper published** — total supply (cap or inflationary), emission schedule, validator share, contributor share, treasury share, tier structure, qualification gates. Publicly available at `wevibe.network/economics` or in the docs.
- [ ] **Faucet service** — automated faucet (web page or Discord bot) that dispenses test VIBE to new wallets. Rate limited (1 request per wallet per 24h).
- [ ] **Bootstrap credits** — year-one genesis-funded service credits operational. New orgs receive X credits without purchasing.
- [ ] **Storage-market slot auction live** — org slot auction configured with caps 32 / 320 / 3200 (DECISIONS.md `D-ECON-STORAGE-MARKET`).
- [ ] **Org purchase split enforced** — org slot purchase routes 50% to burn and 50% to the org account (DECISIONS.md `D-ECON-STORAGE-MARKET`).
- [ ] **Per-memory storage deposit active** — each committed memory requires a VIBE storage deposit (DECISIONS.md `D-ECON-STORAGE-MARKET`).
- [ ] **Org-account feegrant configured** — org account feegrant covers serve and deny flows (DECISIONS.md `D-ECON-STORAGE-MARKET`).

### Data Migration

- [ ] **Zero stale data** — wipe any leftover testnet data before opening to public testers
- [ ] **Migration scripts** — if schema changes are required, have clear wipe + migration plan (R-NO-DB-HACKS)

### Documentation

- [ ] **Documentation site** — `docs.wevibe.network`. Covers: architecture overview, quickstart, contributor guide, moderator guide, leader guide, API reference, self-hosting guide, security model.
- [ ] **API reference** — OpenAPI spec for hub endpoints, published and versioned
- [ ] **Self-hosting guide** — how to run your own hub (D-12.1 — hub is open-source). Prerequisites, Docker setup, chain connection, Qdrant setup.
- [ ] **Validator guide** — how to run a validator node. Hardware requirements, genesis procedure, key management, monitoring.

### Legal / Policy (Testnet)

- [ ] **Terms of Service** — covers: test token disclaimer (no monetary value), data handling, acceptable use, liability limitations
- [ ] **Privacy Policy** — covers: what data is stored (encrypted memories, wallet addresses, reputation data, email for notifications), how it's processed, GDPR compliance if EU-accessible, data deletion rights
- [ ] **Open source licenses verified** — Apache-2.0 for public repo (WeVibe), GPL boundary correctly isolated for umbral-sidecar (D-2.2), all dependencies license-compatible

### Operations

- [ ] **CI/CD pipeline** — automated build + test + deploy. GitHub Actions or equivalent. Push to main → build Docker images → deploy to testnet (with manual approval gate).
- [ ] **Zero-downtime deploy strategy** — rolling update for hub (at minimum, drain connections before restart). Chain validators handle this natively (miss a few blocks during restart, catch up).
- [ ] **Incident response runbook** — who gets alerted, escalation path, communication template for public status page
- [ ] **Status page** — `status.wevibe.network` showing service health. Can be as simple as a static page updated by monitoring.
- [ ] **Backup strategy upgraded** — PostgreSQL streaming replication or WAL archiving. Qdrant snapshots to off-site storage. Chain state backed up daily.

### Testnet Exit Criteria

- [ ] **Moderation SLA** — <24h average approval time
- [ ] **Serve latency** — <500ms P95 for wevibe-mcp HTTP recall path under expected load
- [ ] **Chain stability** — 7 days without halting
- [ ] **No critical security findings outstanding** — internal or external audit
- [ ] **Validator community ready** — documentation + onboarding for external operators

## Mainnet Launch (Production, Tokenized)

### Infrastructure

- [ ] **Multi-region deployment** — at least two regions for redundancy (e.g., us-west + eu-central)
- [ ] **Load balancer** — global load balancer in front of hub + dashboard, with per-region health checks
- [ ] **Hardened validator set** — multiple independent operators, key ceremony complete
- [ ] **Secrets management** — HashiCorp Vault / AWS KMS / GCP KMS for hub secrets
- [ ] **Observability stack** — Prometheus, Grafana, Loki, Alertmanager, on-call rotation
- [ ] **Disaster recovery** — tested restores from backups (Postgres, Qdrant, chain snapshots)
- [ ] **Penetration test** — external audit of hub + dashboard + wevibe-mcp HTTP surface

### Chain

- [ ] **chain_id** — `wevibe-1` (mainnet)
- [ ] **Genesis ceremony** — formal, documented, multi-party. Genesis file signed by founding validators. Published and verifiable.
- [ ] **Token distribution event** — how VIBE tokens reach the public. Decision: ICO, airdrop, liquidity mining, validator staking, or combination. Legal review required.
- [ ] **Emission schedule final** — immutable (or governance-changeable with high threshold). Published in economics paper.
- [ ] **Governance live** — `x/gov` with reasonable thresholds. Parameter changes require supermajority. Emergency pause capability for critical bugs.
- [ ] **IBC enabled** (optional) — Cosmos IBC for cross-chain token transfers. Enables VIBE to be traded on DEXes.
- [ ] **Chain explorer** — production explorer at `explorer.wevibe.network`. Full-featured: TX search, account history, validator dashboard, governance proposals.

### Security — Production Grade

- [ ] **Smart contract audits** (if any on-chain Wasm contracts) — completed by external firm
- [ ] **wevibe-hub audit** — independent security review
- [ ] **wevibe-mcp audit** — independent security review
- [ ] **wevibe-guard audit** — independent security review
- [ ] **SOC2 / ISO27001 roadmap** — if targeting enterprise customers
- [ ] **Bug bounty program** — defined scope, payouts, responsible disclosure policy
- [ ] **Incident response drills** — at least one tabletop exercise covering chain halt, key compromise, data leak

### Token Economics — Live

- [ ] **Token listed** — on at least one exchange or DEX for price discovery
- [ ] **Staking operational** — validators and delegators staking VIBE, earning rewards
- [ ] **Treasury funded** — org treasuries hold real VIBE, contributor payouts have real value
- [ ] **Bootstrap credits program** — operational with clear end date (year one only, per design)
- [ ] **Economic monitoring** — dashboards tracking: total staked, emission rate, contributor payouts, org treasury balances, token velocity

### Legal / Compliance

- [ ] **Terms of Service** — covers real token value, liability, dispute resolution, jurisdiction
- [ ] **Privacy Policy** — GDPR + CCPA compliant. Data processing agreements where required.
- [ ] **Token legal opinion** — legal review confirming VIBE token classification in relevant jurisdictions (utility token, security, commodity)
- [ ] **Corporate structure** — entity that operates the network, holds treasury, employs team. Jurisdiction selected.
- [ ] **Insurance** — consider: cyber insurance, directors & officers insurance, errors & omissions

### Documentation — Complete

- [ ] **Full documentation site** — comprehensive, versioned, searchable
- [ ] **Developer SDK docs** — for third parties building on WeVibe (if applicable)
- [ ] **Whitepaper final version** — updated from current WHITEPAPER.md to reflect production architecture
- [ ] **Validator operator manual** — hardware requirements, setup, monitoring, upgrade procedures, key management
- [ ] **Troubleshooting guide** — common issues and resolutions for all participant roles
- [ ] **Migration guide** — procedures for orgs upgrading from testnet to mainnet

### Operations — Production Ready

- [ ] **24/7 on-call rotation** — staffed, documented runbooks for common incidents
- [ ] **Change management** — deployment approvals, rollback plan, postmortem process
- [ ] **Capacity planning** — forecasting for hub (requests/sec), wevibe-chain (TPS), Qdrant (vector ops)
- [ ] **Customer support** — ticketing system, SLAs by customer tier
- [ ] **Status page** — live at `status.wevibe.network`, auto-updated via monitoring

### Post-Launch Checklist

- [ ] **Validator onboarding program** — documentation + staking incentives
- [ ] **Org onboarding flow** — self-serve or curated
- [ ] **Partner integrations** — e.g., IDE plugins, LLM vendors
- [ ] **Observability dashboards shared** — with community (read-only)
- [ ] **Retro** — schedule post-launch retrospective within first month
