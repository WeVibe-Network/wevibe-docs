# Decision Note — DELETE `anticipated_need` from the retrieval representation

**Status:** RATIFIED (Walter, 2026-07-03). Code landed 2026-07-03; canon reconciled 2026-07-04.
**Scope:** amends `D-RECALL-ALIGNMENT`; closes `CANONICALUX` open lock #2.

## Decision

DELETE `anticipated_need` from the retrieval representation. The retrieval card is now a **pure
deterministic template on BOTH sides** — the doc side (leader/Verify, `search_document:` prefix) and the
query side (consumer/recall, `search_query:` prefix). No per-memory LLM sentence is generated or stored.

Live card = `Applies when / Stack / Implement / Avoid` (the `Avoid` line omitted when `dnd` is null).

## Evidence (measured)

Manager-run ablation on the recall sim — cell **C3 (with-need)** vs **C3_BASE (without-need)**; held
corpus / fixed seed / pinned `nomic-embed-text:v1.5` (768-d); ranker–query parity `cos = 1.00000`.

| cell | recall@1 | recall@5 |
|---|---:|---:|
| C3 (with anticipated_need) | 0.967 | 1.000 |
| C3_BASE (no anticipated_need) | 0.950 | 1.000 |

- Δrecall@1 = **+1.7pp on exactly ONE scenario**; Δrecall@5 = **0.000** (noise-floor).
- Results: `wevibe-sim/recall-sim/results/2026-07-03T15-30-24-085Z/`.

The only measured "benefit" of `anticipated_need` is a single-scenario recall@1 blip with no recall@5
movement — not enough to justify a non-deterministic, LLM-dependent doc-side field.

## Rationale

Deleting `anticipated_need` makes the card **deterministic on both sides**, therefore
**reproducible-on-rebuild**. This satisfies **D-HUB-REBUILDABLE where rebuildability = TOP-K PARITY, not
bit-identity**: a rebuilt hub, re-deriving cards from on-chain ciphertext + leader keys + the anchored model
id, produces byte-identical card text and thus retrieval-equivalent vectors — no statistical drift from a
regenerated LLM sentence.

Query↔doc symmetry is preserved: the query side (`buildNeedCard`) was already deterministic / LLM-free;
`anticipated_need` was a **doc-side-only** field, so removing it does not break the symmetric-embedding
invariant (D-RECALL-ALIGNMENT point 2).

## Lock #2 resolution

This CLOSES **CANONICALUX open lock #2** ("anticipated_need persistence") as **DELETE-AND-CLOSE**. The lock
asked whether to persist the generated sentence for deterministic rebuild vs accept statistical parity;
deleting the field dissolves the question — a rebuilt hub need not reproduce any `anticipated_need` sentence,
and conforming top-k is enough.

## Option space (A–G collapsed to X vs Y)

The prior option space collapsed to two live choices:

- **X — generate at EXTRACTION** on the contributor's own model and **freeze it as a stored field**
  (deterministic-on-rebuild because persisted).
- **Y — DELETE** (chosen).

Status-quo (generate-at-verify, unpersisted) was **OFF the table** — it is exactly the non-reproducible
doc-side LLM dependency the rebuildable contract forbids. **Y chosen** on the measured evidence (no material
recall benefit) + determinism.

## Re-add shape (IF telemetry ever demands)

If future telemetry justifies re-adding it, the ONLY admissible shape is **option X**: contributor-generated
at EXTRACTION, frozen as a stored field — nothing else (never generate-at-verify, never unpersisted).

**Revisit triggers:**
- a Phase-D BM25 / lexical layer lands (doc2query expansion may earn its keep against a sparse index);
- the extraction-model-provenance model shifts (e.g. a change to who/what generates extraction fields);
- a failed disposability top-k drill traces to card representation.

## Code already landed

wevibe-mcp commit **`d683baa`** removed `buildAnticipatedNeed` and `ANTICIPATED_NEED_SYSTEM_PROMPT`
(`retrieval-card.ts`), and rewired `embed-card.ts` `embedRetrievalCard` to a deterministic
`(structured) -> {vector, cardText, embeddingModelId}` path. `tsc` clean; vitest green.

## Canon reconciled (2026-07-04)

- `D-RECALL-ALIGNMENT` (amendment header + point 1 + Q-B schema + supersedes/refines line).
- `WHITEPAPER.md` §3 (retrieval-card description — `anticipated_need` line removed).
- `CANONICALUX.md` §1 (C1 model role retired), step 12 + rebuild note (deterministic card), open lock #2
  (marked CLOSED / DELETE-AND-CLOSE).
