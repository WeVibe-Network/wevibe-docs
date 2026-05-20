# DEPLOYMENTCHECKLIST.md

# WeVibe Network — Deployment Checklist

Three phases. Each phase builds on the prior. Nothing in a later phase ships until everything in the current phase is checked off.

---

## Phase 1: Closed Alpha

**Who:** Walter + 5-10 invited developers
**Duration:** 2-4 weeks
**Goal:** Validate the full pipeline with real humans doing real coding work

### Infrastructure

- [ ] **VPS provisioned** — 1 machine, 4-8 vCPU, 16-32GB RAM, 200GB NVMe SSD (Hetzner CPX41 or equivalent, ~€30/month)
- [ ] **Domain configured** — `alpha.echo.network` A record pointing to VPS IP
- [ ] **TLS via Caddy** — Caddy reverse proxy in front of hub (`:4440`) + dashboard (`:3000`) + social-graph (`:4470`). Auto-TLS via Let's Encrypt. Verify `wevibe-server/echo-infra/Caddyfile` covers all public endpoints.
- [ ] **Firewall rules** — only expose: 443 (HTTPS), 26656 (CometBFT P2P), 26657 (CometBFT RPC). All Docker service ports (4440, 4450, 4470, 3000, 5432, 6333) internal only.
- [ ] **Docker + Docker Compose installed** on VPS
- [ ] **All 8 Docker services running** — postgres, qdrant, wevibed, hub, dashboard, umbral-sidecar, wevibe-mcp, social-graph
- [ ] **Ollama decision** — either: (a) each tester runs Ollama locally for extraction (250MB model download), or (b) run Ollama on VPS without GPU (slower but removes tester friction). Document choice.
- [ ] **Uptime monitor** — free tier (UptimeRobot or Healthchecks.io) pinging `https://alpha.echo.network/health` and alerting Walter

### Chain Genesis

- [ ] **chain_id** — `echo-alpha-1`
- [ ] **Block time** — 6 seconds (CometBFT default, instant finality)
- [ ] **Chain epoch duration** — 12 hours (testable decay/payout cycles within a day)
- [ ] **min_gas_price** — 0.01 uecho
- [ ] **Daily emission** — set a number (e.g., 1000 ECHO/day). Test tokens, no real value.
- [ ] **validator_share** — 10% (Walter is sole validator, 90% to org contributor pools)
- [ ] **Walter's validator stake** — genesis allocation (e.g., 100,000 ECHO)
- [ ] **Tester faucet allocation** — e.g., 1,000 ECHO per tester for gas fees. Manual transfer or simple faucet script.
- [ ] **Genesis file generated** — `wevibed init` + genesis params configured + validator created
- [ ] **Validator key backed up** — separate from VPS (offline copy of validator key + node key)

### Security

- [ ] **No default passwords** — fresh generation at deploy time for: PostgreSQL password, Qdrant API key, hub delegate key, validator key
- [ ] **Session token** — wevibe-mcp container writes token internally; dogfood test reads via `docker exec`. Verify this works on VPS.
- [ ] **Hub auth** — all endpoints require WeVibe-Signed header (delegated key auth)
- [ ] **wevibe-mcp auth** — all endpoints require Bearer token (D-12.5a)
- [ ] **Qdrant** — API key auth enabled, not exposed to public network
- [ ] **PostgreSQL** — not exposed to public network, password auth required
- [ ] **Secret management** — all credentials stored as env vars in a `.env` file on VPS (not committed to git). `.env` file permissions 0600.
- [ ] **PRE keys** — epoch keypair generated at org creation, Shamir 2-of-3 shares distributed (Walter holds 2 shares for alpha — acceptable for single-operator phase)

### Backup

- [ ] **Daily PostgreSQL backup** — cron job running `pg_dump` to `/backups/pg/`, retained 7 days
- [ ] **Daily Qdrant snapshot** — Qdrant snapshot API call via cron, retained 7 days
- [ ] **Chain state** — `wevibed export` weekly (chain state can be rebuilt from genesis, but export is faster)
- [ ] **Backup verification** — test restore from backup at least once before inviting testers

### Tester Onboarding

- [ ] **Quickstart doc written** — step-by-step: install Keplr → get test ECHO → join org → install OpenCode + plugin → configure wevibe-mcp → extract first memory
- [ ] **Wallet setup guide** — how to add Echo alpha chain to Keplr/Leap (custom chain config with RPC endpoint)
- [ ] **Plugin installation guide** — how to install wevibe-guard plugin in OpenCode, configure `~/.echo/plugin-config.json`
- [ ] **wevibe-mcp connection config** — how to point wevibe-mcp at `alpha.echo.network` instead of localhost
- [ ] **Org creation** — Walter creates the initial org(s) before testers arrive. Testers join via `/discover` + join request.
- [ ] **Communication channel** — private Discord server or Slack workspace. Channels: #general, #bugs, #feedback
- [ ] **Known limitations doc** — what works, what doesn't, what's expected to break

### Operations

- [ ] **Deployment script** — `deploy.sh` that does: git pull, docker compose down, docker compose up --build -d, verify health
- [ ] **Rollback plan** — if a deploy breaks: `docker compose down`, restore from backup, redeploy previous commit
- [ ] **Upgrade communication** — message testers in Discord before any downtime ("going down 5 min for update")
- [ ] **Log access** — `docker logs wevibe-hub`, `docker logs wevibe-mcp`, etc. No structured logging needed at alpha.

### Alpha Exit Criteria

Before moving to public testnet, alpha must demonstrate:

- [ ] At least 3 testers have completed the full loop: extract → submit → moderate → commit → recall
- [ ] At least 50 memories committed to chain
- [ ] At least 1 report filed and resolved
- [ ] Keyword weight decay observable (at least one memory with decayed weights)
- [ ] Emission payouts distributed to at least 2 contributors
- [ ] No critical bugs open in MASTER.md gap log
- [ ] Tester feedback collected and triaged into gap log

---

## Phase 2: Public Testnet

**Who:** Anyone. Public signup.
**Duration:** 2-6 months
**Goal:** Stress test at scale, validate economics, find adversarial bugs, build community

### Infrastructure

- [ ] **VPS 1 — Validator** — dedicated machine, 4 vCPU, 8GB RAM, 500GB SSD. Runs only `wevibed`. Hardened: no SSH password auth, fail2ban, unattended-upgrades.
- [ ] **VPS 2 — Services** — 8 vCPU, 32GB RAM, 500GB SSD. Runs: hub, dashboard, social-graph, wevibe-mcp, postgres, qdrant, umbral-sidecar.
- [ ] **VPS 3 — Second validator (optional)** — community-run or Echo-run. 4 vCPU, 8GB RAM, 500GB SSD. Provides basic fault tolerance (chain continues if one validator is down).
- [ ] **Domain** — `testnet.echo.network` for services, `rpc.testnet.echo.network` for chain RPC
- [ ] **TLS** — Caddy on services VPS, separate TLS for validator RPC if publicly exposed
- [ ] **CDN for dashboard** — consider static export + CDN (Cloudflare, Vercel) instead of running Next.js server. Reduces VPS load.
- [ ] **Monitoring stack** — Prometheus + Grafana (or equivalent). Dashboards for: block height, TX throughput, hub request rate, Qdrant query latency, PostgreSQL connection pool, memory commit rate, emission payout totals. Alert on: service down, disk >80%, memory >90%, chain halted, hub error rate spike.

### Chain

- [ ] **chain_id** — `echo-testnet-1`
- [ ] **Genesis ceremony** — documented procedure for generating genesis with multiple validators
- [ ] **Validator set** — minimum 2, target 4+ for meaningful decentralization. Each validator operator has their own machine.
- [ ] **Emission schedule finalized** — daily emission amount, halving/decay schedule, total supply trajectory
- [ ] **validator_share / contributor_share / treasury_share** — ratios locked and documented
- [ ] **Bootstrap credit program parameters** — how many credits, duration, who qualifies (per DECISIONS.md D-8 area)
- [ ] **Chain epoch duration** — evaluate whether 12h is right for testnet or adjust
- [ ] **Governance module** — `x/gov` proposals enabled for parameter changes (emission rate, required_approvals defaults, etc.)
- [ ] **Chain explorer** — Ping.pub (open source Cosmos explorer) or custom. Deployed at `explorer.testnet.echo.network`. Shows: blocks, transactions, validators, account balances, memory commitments.

### Security — Hardened

- [ ] **ARCH-G9 implemented** — BIP-32 key hierarchy separation (PRE identity derived from wallet, not shared). This is a security requirement before public exposure.
- [ ] **Rate limiting hardened** — per-wallet, per-org, per-endpoint limits on hub. DDoS protection via Caddy rate limiting or Cloudflare proxy.
- [ ] **Sybil mitigation** — stake requirement for org creation (minimum ECHO stake to create an org). Minimum member count (e.g., 3) before org qualifies for emissions.
- [ ] **Spam mitigation** — gas costs on chain submission are the primary defense. Hub-side rate limiting on submit endpoint as secondary.
- [ ] **Report abuse mitigation** — D-7.6 (sword cuts both ways) is already designed. Verify it works: dismissed report count increments, malicious reporter can be removed.
- [ ] **Embedding inversion** — Phase 1 mitigations confirmed working (Gaussian noise, API key auth, Qdrant internal-only). Document limitations publicly.
- [ ] **Security review** — minimum: 2-3 experienced developers attempt to break the system over 1-2 weeks. Document findings. Address critical findings before testnet launch.
- [ ] **Vulnerability disclosure policy** — published page explaining how to report security issues (email, PGP key, response timeline, bounty if applicable)
- [ ] **Key rotation tested** — at least one epoch rotation performed and verified during alpha. Recovery procedure (Shamir 2-of-3 reconstruction) tested at least once.

### Token Economics

- [ ] **Economics paper published** — total supply (cap or inflationary), emission schedule, validator share, contributor share, treasury share, tier structure, qualification gates. Publicly available at `echo.network/economics` or in the docs.
- [ ] **Faucet service** — automated faucet (web page or Discord bot) that dispenses test ECHO to new wallets. Rate limited (1 request per wallet per 24h).
- [ ] **Bootstrap credits** — year-one genesis-funded service credits operational. New orgs receive X credits without purchasing.

### Data Migration

- [ ] **Alpha → testnet decision made** — either: (a) testnet starts fresh (clean genesis, alpha data discarded), or (b) alpha data migrated. Recommendation: start fresh, grandfather alpha participants with a reputation bonus (e.g., "Alpha Tester" badge, bonus starting reputation).
- [ ] **Migration path documented** — if carrying data forward: chain state export/import, PostgreSQL migration, Qdrant re-index.

### Documentation

- [ ] **Documentation site** — `docs.echo.network`. Covers: architecture overview, quickstart, contributor guide, moderator guide, leader guide, API reference, self-hosting guide, security model.
- [ ] **API reference** — OpenAPI spec for hub endpoints, published and versioned
- [ ] **Self-hosting guide** — how to run your own hub (D-12.1 — hub is open-source). Prerequisites, Docker setup, chain connection, Qdrant setup.
- [ ] **Validator guide** — how to run a validator node. Hardware requirements, genesis procedure, key management, monitoring.

### Legal

- [ ] **Terms of Service** — covers: test token disclaimer (no monetary value), data handling, acceptable use, liability limitations
- [ ] **Privacy Policy** — covers: what data is stored (encrypted memories, wallet addresses, reputation data, email for notifications), how it's processed, GDPR compliance if EU-accessible, data deletion rights
- [ ] **Open source licenses verified** — Apache-2.0 for public repo (Echo), GPL boundary correctly isolated for umbral-sidecar (D-2.2), all dependencies license-compatible

### Operations

- [ ] **CI/CD pipeline** — automated build + test + deploy. GitHub Actions or equivalent. Push to main → build Docker images → deploy to testnet (with manual approval gate).
- [ ] **Zero-downtime deploy strategy** — rolling update for hub (at minimum, drain connections before restart). Chain validators handle this natively (miss a few blocks during restart, catch up).
- [ ] **Incident response runbook** — who gets alerted, escalation path, communication template for public status page
- [ ] **Status page** — `status.echo.network` showing service health. Can be as simple as a static page updated by monitoring.
- [ ] **Backup strategy upgraded** — PostgreSQL streaming replication or WAL archiving. Qdrant snapshots to off-site storage. Chain state backed up daily.

### Testnet Exit Criteria

Before moving to mainnet:

- [ ] Testnet has run for at least 2 months with no chain halt
- [ ] At least 5 orgs with active contributors
- [ ] At least 500 memories committed
- [ ] At least 3 independent validators running
- [ ] Emission payouts distributed correctly for at least 30 chain epochs
- [ ] No critical security findings from review remain unaddressed
- [ ] Token economics validated: tier payouts, qualification gates, decay model all producing intended behavior
- [ ] Self-hosted hub successfully deployed by at least 1 external operator
- [ ] Formal security audit scheduled or completed

---

## Phase 3: Public Mainnet

**Who:** Everyone. Real tokens, real value.
**Duration:** Forever
**Goal:** Production system with real economic incentives

### Infrastructure

- [ ] **Validator set** — 10-20+ independent operators, geographically distributed. Each on their own infrastructure (not all on the same cloud provider).
- [ ] **Hub high availability** — 2+ hub instances behind load balancer. Shared PostgreSQL (primary + read replica). Health-check-based failover.
- [ ] **Qdrant cluster** — 3-node minimum for durability. Per-org collections replicated across nodes.
- [ ] **PostgreSQL** — primary + streaming replica + WAL archiving to off-site storage
- [ ] **Dashboard** — static export on CDN (Cloudflare Pages, Vercel, Netlify). No server-side rendering in production.
- [ ] **Social Graph Service** — 2+ instances behind load balancer. SQLite replaced with PostgreSQL for durability.
- [ ] **Monitoring** — full observability: Prometheus + Grafana + Loki (logs) + alerting to PagerDuty/Opsgenie. SLA: 99.9% uptime for hub, 99.99% for chain.
- [ ] **DDoS protection** — Cloudflare or equivalent in front of all public endpoints
- [ ] **Geographic redundancy** — services in at least 2 regions (active-passive or active-active)

### Chain

- [ ] **chain_id** — `echo-1` (mainnet)
- [ ] **Genesis ceremony** — formal, documented, multi-party. Genesis file signed by founding validators. Published and verifiable.
- [ ] **Token distribution event** — how ECHO tokens reach the public. Decision: ICO, airdrop, liquidity mining, validator staking, or combination. Legal review required.
- [ ] **Emission schedule final** — immutable (or governance-changeable with high threshold). Published in economics paper.
- [ ] **Governance live** — `x/gov` with reasonable thresholds. Parameter changes require supermajority. Emergency pause capability for critical bugs.
- [ ] **IBC enabled** (optional) — Cosmos IBC for cross-chain token transfers. Enables ECHO to be traded on DEXes.
- [ ] **Chain explorer** — production explorer at `explorer.echo.network`. Full-featured: TX search, account history, validator dashboard, governance proposals.

### Security — Production Grade

- [ ] **Formal security audit completed** — by a reputable firm (Trail of Bits, Zellic, Halborn, Oak Security). Covers: chain modules, PRE implementation, hub auth, wevibe-guard, key management, token economics. Budget: $50-200K.
- [ ] **Audit findings addressed** — all critical and high findings fixed. Medium findings documented with mitigation timeline.
- [ ] **Bug bounty program** — published scope, payout tiers, submission process. Platforms: Immunefi, HackerOne, or self-hosted.
- [ ] **Key management ceremony** — documented procedure for genesis key generation, Shamir share distribution, validator key rotation. Multiple key holders, no single point of failure.
- [ ] **Incident response team** — named individuals, on-call rotation, communication protocols, post-incident review process
- [ ] **Penetration test** — external pentest of all public-facing surfaces (hub API, dashboard, chain RPC, social graph)

### Token Economics — Live

- [ ] **Token listed** — on at least one exchange or DEX for price discovery
- [ ] **Staking operational** — validators and delegators staking ECHO, earning rewards
- [ ] **Treasury funded** — org treasuries hold real ECHO, contributor payouts have real value
- [ ] **Bootstrap credits program** — operational with clear end date (year one only, per design)
- [ ] **Economic monitoring** — dashboards tracking: total staked, emission rate, contributor payouts, org treasury balances, token velocity

### Operations — Production

- [ ] **SLA published** — uptime commitment, response time for incidents, planned maintenance windows
- [ ] **On-call rotation** — at least 2 people, 24/7 coverage for critical alerts
- [ ] **Runbooks operational** — PRE recovery (ARCH-G8), epoch SK compromise (OQ-6), chain halt, hub failover, database recovery, validator key rotation
- [ ] **Disaster recovery tested** — full DR drill: simulate VPS loss, restore from backup, verify chain and hub resume correctly
- [ ] **Change management** — all production changes go through PR review, staging environment, canary deploy before full rollout
- [ ] **Audit logging** — all administrative actions logged immutably. Who changed what config, when, from where.

### Legal — Full

- [ ] **Terms of Service** — covers real token value, liability, dispute resolution, jurisdiction
- [ ] **Privacy Policy** — GDPR + CCPA compliant. Data processing agreements where required.
- [ ] **Token legal opinion** — legal review confirming ECHO token classification in relevant jurisdictions (utility token, security, commodity)
- [ ] **Corporate structure** — entity that operates the network, holds treasury, employs team. Jurisdiction selected.
- [ ] **Insurance** — consider: cyber insurance, directors & officers insurance, errors & omissions

### Documentation — Complete

- [ ] **Full documentation site** — comprehensive, versioned, searchable
- [ ] **Developer SDK docs** — for third parties building on Echo (if applicable)
- [ ] **Whitepaper final version** — updated from current WHITEPAPER.md to reflect production architecture
- [ ] **Validator operator manual** — hardware requirements, setup, monitoring, upgrade procedures, key management
- [ ] **Troubleshooting guide** — common issues and resolutions for all participant roles

---

## Quick Reference: What Blocks What

| Blocker | Blocks |
|---------|--------|
| VPS provisioned + Docker running | Everything |
| Genesis params decided | Chain start |
| TLS + domain configured | Tester access |
| Quickstart doc written | Tester onboarding |
| Faucet (manual or automated) | Testers submitting TXs |
| ARCH-G9 (BIP-32 keys) | Public testnet security |
| Security review | Public testnet launch |
| Economics paper | Public testnet community trust |
| Formal audit | Mainnet launch |
| Token legal opinion | Mainnet token distribution |
| Multi-validator genesis | Testnet/mainnet decentralization |

---

*This document is the deployment truth. Update it as items are checked off. Items without a check are not done.*