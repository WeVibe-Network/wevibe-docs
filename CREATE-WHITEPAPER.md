# CREATE-WHITEPAPER.md — Whitepaper Authoring Spec & Thinking-Line

> **Purpose.** This is the durable rationale behind the WeVibe whitepaper's framing and the
> canonical product/economic/social model. It captures the **what AND the why** for every decision
> solidified in the design session of 2026-06-02, so that any future update to `WHITEPAPER.md` (or to
> the model itself) starts from the complete reasoning — not just the conclusions. When something new
> is brought to us, consult this first: it tells you *why* each choice was made and which alternatives
> were already considered and rejected, so we don't relitigate settled ground or accidentally break a
> load-bearing intent.
>
> **Authority:** the binding decisions live in `DECISIONS.md` (§19: D-SG-1/2/3, D-ECON-CANON,
> D-ATTEST-ROADMAP; plus D-S32-TOKENOMICS-LOCKED, GAP-RARITY-1). The layer map lives in `TOPOLOGY.md`.
> This file is the *narrative rationale* that ties them together. If this file and DECISIONS.md ever
> disagree, DECISIONS.md wins and this file must be corrected.

---

## 0. How to use this spec when writing/updating the whitepaper

- Frame everything **as it ships in alpha** (near-term, present-tense voice), NOT as distant someday.
  WeVibe is **pre-alpha, stealth, zero DAU** — but the badges + social rep + economy are being built
  now and the whitepaper describes the alpha product, with the post-mainnet items clearly marked roadmap.
- Be **honest about build status**: the economic model is **DECIDED**, but the demand-leg (VIBE
  subscriptions → treasury), the protocol burn, and reward settlement are **not yet built** (CO-047
  `org_credits` is only a skeleton). Describe the alpha product confidently; mark unbuilt mechanisms as
  near-term, never as already-live. Do not overclaim.
- Lead with the **social-first** wedge; let memory be the bonus. See §2.

---

## 1. The core repositioning (what changed and why)

**WHAT:** The whitepaper used to be "Plugin-Gated Shared Memory for AI Coding Agents" — memory as the
headline, orgs framed as Bittensor-style economic subnets, with enterprise features (SLA-backed
retrieval, dedicated infra). We are reframing to a **social-first tool for individual vibe coders and
small crews**, where **memories are a bonus**, not the headline.

**WHY:**
- The **go-to-market wedge is social reputation, not memory infrastructure.** The GTM is: free sign-ups
  for vibe coders who already exist today, an easy plugin + "go" mechanics, and a daily **dopamine hook
  — the coder watches their reputation build every day** from their own coding exhaust. Memory recall is
  the *bonus* that makes their agent better; the *hook* is the social/reputation layer.
- The old enterprise/org-scale framing (orgs as Bittensor subnets, SLA-backed enterprise features)
  **diverged from what the product actually is** (MASTER.md frames it for individual devs + small orgs)
  and from the wedge. Audience is **individual vibe coders + small crews, NOT multi-billion-dollar
  coding organizations.**
- **Orgs = domain-expert-run memory collections you join.** A leader is a domain expert who oversees
  memory commitment/quality; orgs span the full range (inexperienced → experienced). The org is the
  collaboration + economic *container*, not the protagonist. The protagonist is the vibe coder.
- **Plaintext keywords are an accepted non-issue.** The audience is vibe coders, not enterprises
  protecting secrets from validators. Do not alarm about keyword exposure or "blind-token" weakness;
  treat plaintext keywords as the discovery feature they are. Do not over-engineer keyword privacy.

---

## 2. The flywheel (the "why" the whole economy exists)

```
free sign-up + easy plugin
      → vibe coder contributes memories from daily coding
      → reputation + badges build publicly (DOPAMINE)
      → more users
      → more leaders buy/run orgs (to monetize a valuable memory base)
      → memory quality ↑
      → more users  (repeat)
```

- The **VIBE token bootstraps the supply side**: contributors are paid by the network for contributing,
  even before any org has paying customers. This is the cold-start labor subsidy — it is *why* we don't
  need "contractor" contributors (see §5).
- The **demand side closes the loop**: users pay orgs in VIBE for memory access; that revenue funds
  leaders, who pay moderators; leaders/validators stake and secure the chain. Value is meant to be
  **retained inside a closed economy**, everything denominated in VIBE.

---

## 3. Closed-economy model (the loop, and why it balances)

**THE CANONICAL LOOP:**
```
emission → contributors (contribution-only) + validators/stakers   [mint → sell pressure]
        → users buy VIBE
        → users pay orgs in VIBE for recall access  (hub-accounted, leader-set model & price)
        → SMALL PROTOCOL BURN (the sink) + remainder → org treasury
        → leader withdraws revenue → pays moderators
        → leaders/validators/stakers stake & secure the network
        → repeat
```

**WHY it balances:**
- **new issuance / "the bleed":** the emission — contributor pool + validator pool. This is
  where VIBE enters circulation and where sell pressure originates.
- **offset the issuance:** (1) **org-creation burn**, (2) the **small protocol burn on subscription revenue**. 
  burns are what keep a closed economy from only emitting + inflating supply -> bleeding to zero.
- **Demand recirculation:** users buying VIBE to pay orgs creates the *demand* that absorbs contributor
  sell pressure and funds leaders. A token with only a faucet and no sink/demand dies — the burns +
  subscription demand are the deliberate counterweight.

**IMPORTANT — testnet vs mainnet:** the **faucet service is testnet-only gas scaffolding** (standard
for any Cosmos testnet). It is **NOT part of mainnet economics** and must never be reasoned about as
such. On mainnet, gas is paid in real VIBE.

---

## 4. The five layers (what each is, and why it's drawn this way)

| Layer | What | Why this boundary |
|---|---|---|
| **1. Chain** | Source of truth: provenance, author attribution, serve/denial **aggregate** counters (contributor + org), approved-memory counts, membership/roles, per-memory rarity tier, economic state. | Immutability + provenance must be tamper-proof. Economics reads **only** contribution counts + the network threshold — never serve counts (see §6). |
| **2. RPC** | The read contract: exposes raw chain counts/inputs to any client. | A clean public contract lets *anyone* build on the data; it's the seam that makes the social graph forkable. |
| **3. Social graph** | **Open-source, forkable, self-hostable** display client over RPC. Renders public profiles: serve counts, reputation, badges. | It's display, not truth — so it can be permissionless and forkable. This forces badge criteria to be canonical (see §7) so a tier means the same thing on every fork. |
| **4. Economy** | VIBE flows (see §3). | Kept minimal on-chain (treasury + burn primitives); the *model* is pushed to the leader/hub (see §8). |
| **5. Future attestation** | Pluggable verification of session claims, post-mainnet. | Load-bearing vision; infra not ready. Kept as a clearly-marked roadmap so it isn't lost (see §9). |

---

### 4.1 Canonical alpha software architecture (hub + MCP, NOT wevibe-client) — decided 2026-06-02

**WHAT:** The alpha system is three software pieces: **wevibe-chain** (source of truth),
**wevibe-hub** (`wevibe-server`: coordination, accounting, AND retrieval — the Qdrant vector recall
runs *here*), and the **MCP server + plugin** (local guard/sanitization, the human-gate approval UX,
and context injection near the coder).

**WHY / correction:** An earlier v2.0 design described a `wevibe-client` that performed *local* vector
retrieval and "eliminated the hub." That client **was never built — there is no `wevibe-client` repo**
(the repos are chain, server, mcp, sdk, guard, umbral, protocol, social-graph, sim, meta), and the
live, gated retrieval path runs **in the hub** (`wevibe-server/wevibe-hub/internal/retrieval` — the
exact path the empirical matrix exercises). So: the canonical alpha architecture is **hub + MCP**, the
hub is **live** (not eliminated), and the `wevibe-client` local-retrieval narrative is **dropped
entirely** (not even kept as roadmap — Walter, 2026-06-02). The whitepaper's §2 ("software pieces"),
§4 ("Local Client Architecture"), and §8 ("Storage … Local Only") must be rewritten to hub+MCP
retrieval. Related economic stale-aside in §3.9 ("orgs pay for contributors … never hold tokens")
must be corrected to the closed-loop economy (members pay in VIBE for access; contributors paid by the
network; see §3, §8 of this spec).

**Grounded recall data flow (code-verified 2026-06-02):** (1) the MCP/plugin computes the QUERY
embedding LOCALLY via Ollama (nomic-embed); (2) it POSTs the query VECTOR to the hub
(`/v1/orgs/{org}/query`); (3) the hub's **Qdrant** does standard (non-encrypted) vector search and
returns memory **ids + metadata + matched keywords** (NOT plaintext, NOT ciphertext, NOT raw vectors);
(4) the MCP fetches each memory's **ciphertext** from the hub and **decrypts it LOCALLY** via the Umbral
re-encryption sidecar; (5) the MCP runs **wevibe-guard** sanitization, the human-gate approval, and
context injection — all local.

**PRIVACY POSTURE (the honest, accurate claim):** the hub stores **embedding vectors + keyword
metadata** (Qdrant) and **ciphertext** (PostgreSQL/chain) and performs the vector search — so vectors
and keyword metadata DO live in the hub (NOT "local only"). But the hub **NEVER holds decrypted
plaintext**; decryption is client-side (Umbral). The correct public claim is therefore *"the hub never
sees your decrypted memory content"* — NOT *"nothing leaves your machine."* Encrypted vector search was
evaluated and **rejected as infeasible** (`EVAL-ENCRYPTED-VECTOR-SEARCH.md`, DOC-ONLY); Qdrant holds
plaintext float32 vectors. The whitepaper's §3.5/§3.6/§3.8/§8.3 "Local Only" claims must be corrected to
this posture, and §4 "Local Client Architecture" rewritten as: local MCP/plugin (query embedding,
decryption, guard, human-gate, injection) + hub (Qdrant vector search, ciphertext storage/serving).

---

## 5. Membership & contribution (what, and the alternative we rejected)

**WHAT (decided):** Contributors are **paid by the network per approved memory** (contribution-only),
gated by a **network-set** threshold. Whether a contributor must *also* pay for recall access is a
**leader-configured** policy (see §8). One person can be both contributor and consumer.

**WHY, and the rejected alternative:**
- We considered **"non-member contractor contributors"** (people who work for an org as contributors
  without being members). **Rejected** because: (a) it breaks the membership-gated architecture (big
  rebuild), and (b) it **breaks the GTM dopamine hook** — the viral loop is *the individual vibe coder
  watching their own rep climb from their own daily coding*, which is a member, not a gig contractor.
- The **cold-start objection** ("an org has no memories, why would anyone subscribe?") is already solved
  by the **network contribution subsidy**: early members contribute for VIBE + rep *before* any
  subscriber exists. The mint is the bootstrap labor budget, so contractors aren't needed.

---

## 6. Serve/retrieval attribution = SOCIAL, not economic (the key decoupling)

**WHAT:** When a memory is served, it increments **aggregate** served-counters for **both** the
contributing author **and** the org (not per-memory "cards"). These counts power profiles + badges and
are **public**. Source of truth = chain; the social graph displays.

**WHY this is decoupled by *purpose*, not deleted:**
- The **old system tied rewards to retrievals/serves** baked into the economics. That is **gameable**
  (you can manufacture retrievals to farm rewards), so it was **stripped from the ECONOMICS.**
- But the **attribution itself is KEPT** — as a **social/status/reputation signal**, which *cannot be
  gamed for profit* because no VIBE is attached. This is the crucial distinction: we removed the
  *economic coupling*, not the *signal*. **Do not strip the serve-attribution code.**
- Therefore: economics reads contribution counts only; the social graph reads serve counts. Two
  different consumers of two different signals.

---

## 7. Badges (what, scoping, and why each constraint)

**WHAT:** Multiple badge **families** — (a) **serve-milestone** (your memories served N times),
(b) **rarity-tier** (per-memory keyword supply/demand, computed once at commit, frozen **on-chain** —
GAP-RARITY-1), (c) **contribution-volume**. Scoped **per-org** with a profile breakdown.
**Status-only, no reward.**

**WHY each choice:**
- **No cross-org leaderboard** → avoids toxic competition; per-org breakdown keeps it celebratory, not
  zero-sum.
- **Canonical-spec-in-display for serve/contribution criteria** (rarity is on-chain) → because the
  social graph is forkable, "Legendary" must mean the same thing on every host. The chain exposes raw
  counts; a **canonical spec** defines the thresholds the reference client applies. Putting *all* badge
  state on-chain was rejected (too rigid — every criteria tweak would need a chain upgrade); fully
  per-fork was rejected (fragments the gamified identity).
- **No reward attached** → keeps economics contribution-only and anti-game; badges are pure
  status/dopamine.
- **Aggregate counters, not per-memory collectible cards** → Walter's call; simpler, and the profile is
  the aggregated view.

---

## 8. Leader as economic operator (what, and the alternatives rejected)

**WHAT:** **Leaders earn NO emissions.** Leader revenue comes solely from the **org demand-leg**:
members pay the org in VIBE for recall access, settling to the org treasury; the leader withdraws
revenue (`MsgWithdrawTreasury`) and pays moderators at discretion. The **access/payment model is
leader-configured** (pricing market-driven, cadence and contributor-access policy set by the leader)
via the **hub accounting layer** (CO-047 `org_credits`). The protocol's only rule on this leg is a
**small burn on subscription revenue at settlement**, remainder to the leader.

**WHY:**
- Pushing the model to the **leader + hub** keeps the **chain minimal** and makes orgs **sovereign
  little businesses** that compete on price vs memory quality — a free market, not a protocol-dictated
  tariff.
- **Leaders excluded from emissions** so the mint stays clean (contributors + validators only) and
  leader reward is tied to **real demand**, not issuance.
- The **small burn** is the sink that closes the loop and offsets contributor sell pressure.

**Alternatives considered and rejected:**
- **Leader emissions / "org tax" (cut of contributor mint):** taxes the dopamine loop and modifies
  locked tokenomics (D-S32-TOKENOMICS-LOCKED) — rejected.
- **Per-serve royalty / usage metering:** CO-047 deliberately *removed* per-query metering (privacy +
  simplicity) and serve-based economics are gameable — rejected.
- **Stake-to-create + emissions yield (Bittensor-style):** bigger build, deferred (could revisit
  long-term as an additional sink, but not the alpha mechanism).
- **Protocol-enforced moderator revenue split:** rigid; moderator pay is leader-discretionary instead.

---

## 9. Future pluggable attestation (roadmap — do not drop)

**WHAT:** Post-mainnet, evolve the optional §3.10 Session Attestation + §3.11 Two-Layer Difficulty
Scoring into a **pluggable attestation framework**: separate components plug into the chain to validate
— cryptographically OR via API — session claims like *"user X using LLM model Y took N turns to solve
problem Z."* How it enhances the economic/social layers is **undetermined (TBD).**

**WHY:** This is a **major, load-bearing future vision** and was at risk of being lost from the docs.
Infra is not there yet, so it is explicitly **post-mainnet roadmap** — carried, not claimed as built.

---

## 10. Code remnants to be aware of (leftover from old systems/pivots)

These exist in the code today, contradict the canonical model, and are **documentation flags / future
cleanup targets** (NOT changed by the whitepaper pass; verify before acting). They explain why earlier
reads of the code were misleading:
1. **Contributor payout sourced from org treasury** (`x/emissions/keeper/epoch_hooks.go` ~L147,
   `DebitTreasury`) — canonical source is the **network** contributor-emission pool.
2. **Operator/leader emission machinery** (`OperatorShare`, `DistributeOperatorRewards`, `opreward/`,
   `ComputeWorkScore`) — dead; contradicts "leaders earn no emissions"; remove in cleanup.
3. **Retrieval term in `WorkScore`** (`x/emissions/types/keys.go` ~L48, `retrievalScore`) — reclassify
   as social, not economic. **The serve-attribution code itself is KEPT, not stripped** (see §6).

Also factual staleness in the current whitepaper to fix: **L55 "wevibe-hub eliminated"** is wrong (the
hub is live — it's the component under the empirical gate); **Phase II "Sprint 23-25"** is stale vs the
current sprint.

---

## 11. Decision ledger (where each "what" is locked)

| Topic | Binding source |
|---|---|
| Serve attribution = social, kept, chain-truth | DECISIONS D-SG-1 |
| Social graph = open-source forkable display over RPC | DECISIONS D-SG-2 |
| Badges (families, per-org, canonical-spec-in-display, status-only) | DECISIONS D-SG-3, GAP-RARITY-1 |
| Closed economy (contribution-only, leaders-no-emissions, leader-as-operator, burn sink, demand-leg) | DECISIONS D-ECON-CANON |
| Tokenomics constants (1B, pools, 32yr, caps) | DECISIONS D-S32-TOKENOMICS-LOCKED |
| Future pluggable attestation | DECISIONS D-ATTEST-ROADMAP |
| 5-layer architecture map | TOPOLOGY.md "Canonical 5-Layer Architecture" |

---

*End of CREATE-WHITEPAPER.md — the thinking-line for the whitepaper and the canonical model.*
