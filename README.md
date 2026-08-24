<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:02100a,100:2fe07a&height=160&section=header&text=wevibe-docs&fontColor=54f59a&fontSize=42&fontAlignY=40&desc=Architecture%20security%20and%20runbooks&descAlignY=64&descSize=16" alt="wevibe-docs" width="100%" />

![Markdown](https://img.shields.io/badge/Markdown-000000?style=flat-square&logo=markdown&logoColor=white)
[![status-alpha](https://img.shields.io/badge/status-alpha-ffc266?style=flat-square)](https://github.com/WeVibe-Network)
[![license-Apache--2.0](https://img.shields.io/badge/license-Apache--2.0-82aaff?style=flat-square)](LICENSE)
[![docs-wevibe-docs](https://img.shields.io/badge/docs-wevibe--docs-54f59a?style=flat-square)](https://github.com/WeVibe-Network/wevibe-docs)
[![%40WeVibe__Network](https://img.shields.io/badge/%40WeVibe__Network-0a0a0a?style=flat-square&logo=x&logoColor=white)](https://x.com/WeVibe_Network)

</div>

---

Canonical architecture and documentation for the [WeVibe Network](https://github.com/WeVibe-Network). WeVibe is a memory layer for AI coding agents: attributed, encrypted memories delivered across trust boundaries, decrypted and sanitized locally so every recalled item passes human review before reaching an agent's context.
Code lives in the other WeVibe Network repositories; this repository contains documentation only. The project is in active alpha development — interfaces and operational procedures can change.

This repository is the single place pinning the three things that must not drift apart:

- **System intent** — [`WHITEPAPER.md`](WHITEPAPER.md) is the architecture contract: the normative
  specification the network is built and audited against.
- **System reality** — [`TOPOLOGY.md`](TOPOLOGY.md), generated 2026-08-18 and superseding the 2026-06-14
  edition, re-derives the cross-module topology from audited current source rather than carrying prior
  documentation forward; [`CHAIN-SPEC.md`](CHAIN-SPEC.md) is the chain's as-built surface and integration spec.
- **The live pivot** — [`RECALL-PIVOT-SPEC.md`](RECALL-PIVOT-SPEC.md) is the LIVE recall/standing
  specification: the chain stores content-free events, and standing is computed at the edge against
  `edge-policy-v1`, anchored on-chain at height 45.

## Documentation map

Every file listed below exists in this repository and is grouped by role.

**Specification**

- [`WHITEPAPER.md`](WHITEPAPER.md) — normative architecture contract; [`WeVibe-Whitepaper.pdf`](WeVibe-Whitepaper.pdf) is its typeset rendering.
- [`WP-DESIGN-SPEC.md`](WP-DESIGN-SPEC.md) — whitepaper typesetting source: supplies the printed wording and layout for the PDF. The normative role belongs to `WHITEPAPER.md`.
- [`TOPOLOGY.md`](TOPOLOGY.md) — cross-module topology, generated 2026-08-18 from current source.
- [`CHAIN-SPEC.md`](CHAIN-SPEC.md) — chain surface and integration specification, as built.
- [`MEMORY-EVIDENCE-SPEC.md`](MEMORY-EVIDENCE-SPEC.md) — memory evidence and benchmark independence.
- [`HUB-REBUILD.md`](HUB-REBUILD.md) — hub retrieval-plane rebuild specification.

**Recall, matching and extraction**

- [`RECALL-PIVOT-SPEC.md`](RECALL-PIVOT-SPEC.md) — the LIVE recall/standing pivot spec (`edge-policy-v1` anchored at height 45).
- [`RECALL-PIVOT.md`](RECALL-PIVOT.md) — canonized 2026-07-28 pivot decisions (pivot LIVE as of 2026-07-30).
- [`RECALL-SYSTEM.md`](RECALL-SYSTEM.md) — project-wide recall reference.
- [`MATCHING_ENGINE.md`](MATCHING_ENGINE.md) — matching-engine architecture reference, framed
  pre-pivot. Current retrieval truth lives in `RECALL-PIVOT-SPEC.md`, not here.
- [`EXTRACTION-FLOW.md`](EXTRACTION-FLOW.md) — ground-truth reference for the memory-extraction pipeline.
- [`DECISION-NOTE-anticipated-need-delete.md`](DECISION-NOTE-anticipated-need-delete.md) — ratified decision note removing `anticipated_need` from the retrieval representation.

**Security and keys**

- [`SECURITY.md`](SECURITY.md) — security policy.
- [`SECURITY-MODEL.md`](SECURITY-MODEL.md) — security model.
- [`KEY_MANAGEMENT.md`](KEY_MANAGEMENT.md) — key management.

**Running WeVibe**

- [`QUICKSTART.md`](QUICKSTART.md) — quick start.
- [`LEADER_QUICKSTART.md`](LEADER_QUICKSTART.md) — org-leader quick start.
- [`VALIDATOR_QUICKSTART.md`](VALIDATOR_QUICKSTART.md) — validator setup runbook.
- [`SELF-HOSTING.md`](SELF-HOSTING.md) — self-hosting guide.
- [`MCP-TOOLS.md`](MCP-TOOLS.md) — MCP tool reference.

**Operational runbooks**

- [`RECOVERY_RUNBOOK.md`](RECOVERY_RUNBOOK.md) — recovery runbook.
- [`RUNBOOK-PRE-RECOVERY.md`](RUNBOOK-PRE-RECOVERY.md) — pre-recovery runbook.
- [`RUNBOOK-EPOCH-SK-COMPROMISE.md`](RUNBOOK-EPOCH-SK-COMPROMISE.md) — epoch SK compromise response.

**Project**

- [`ROADMAP.md`](ROADMAP.md) — roadmap.
- [`CHANGELOG.md`](CHANGELOG.md) — changelog.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — contributing guide.

Internal design and strategy documents (`MASTER.md`, `DECISIONS.md`) are kept local and are
deliberately not published in this repository.

## License

Apache-2.0. See [LICENSE](./LICENSE).

## Links

- Organization: https://github.com/WeVibe-Network
- X / Twitter: https://x.com/WeVibe_Network
