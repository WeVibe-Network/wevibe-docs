# Changelog

All notable documentation changes to this repository are tracked in this file by sprint.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
with sprint-based milestones (no software version releases).

## [Unreleased]

### Sprint-33
- Recall-trigger supersession canon landed (2026-08-08): recall no longer fires per user prompt — it fires on ONE gated condition (second failure under the same stable failureKey while still red, D-RECALL-TRIGGER-REPEAT) and on nothing else; the four-button gate remains as specified but appears on the repeat-failure trigger and blocks (D-RECALL-GATE-BLOCKS). Supersession notes added across RECALL-PIVOT-SPEC, RECALL-SYSTEM, CANONICALUX, EXTRACTION-FLOW, MATCHING_ENGINE, MCP-TOOLS, RUNBOOK, DECISIONS, WHITEPAPER, WP-DESIGN-SPEC.

### Sprint-32
- Identity/onboarding architecture locked: passkey-first, wallet-optional.
- Storage-market economy folded into DECISIONS: slot auction, 50/50 burn/account split, per-memory deposit, and org-account feegrant.
- Hub remains records-only; dashboard path uses CosmJS direct signing.
- Whitepaper repositioned to social-first framing.
- TOPOLOGY/MASTER synchronized to the current architecture.
- Attestation remains disabled-but-wired.

### Sprint <=31
- Prior sprint history tracked in PDP (now archived) and DECISIONS.md.
