# EXTRACTION-FLOW.md — Ground-Truth Reference for the WeVibe Memory-Extraction Pipeline

> **Status:** LOCAL orchestration doc (uncommitted, alongside AGENTS.md / SESSIONCONTINUANCE.md). NOT inside any git repo.
> **Authored:** 06-07-26 (Jul 6 2026) by the extraction-investigation orchestrator.
> **Updated 2026-07-25:** the zero-progress extraction gate is now BUILT + LIVE (PART B step 4 — was "gate ABSENT";
> canonical mcp `c5304d9` + bench clone `1a04bae`); memory-side keyword-labeling gap added as §(4a); invariant
> citation corrected to `WP-DESIGN-SPEC.md:918`.
> **Scope:** the complete memory-EXTRACTION path — what the contributor SEES + how the backend works — plus the definitive resolution of the `classified:[]` / `org=-` question (see §CRUX).
> **All `file:line` anchors below were opened on-disk and verified live** (the extraction files are UNCOMMITTED and drifting; MCP `src` ≈ `dist` semantically, `dist` line numbers differ — `src` cited unless noted). Spot-checked by the orchestrator (R-29): MCP `extraction.ts:919-943`, `288-337`, `1049-1058`; `http-server.ts:437-472`; dashboard `route.ts:201-233`; `DECISIONS.md:590-613`.

## LEGEND — every claim is labelled
- **[CURRENT-TRUTH]** = verified against the running/on-disk code today.
- **[DESIGN-INTENT]** = from canon (DECISIONS.md / CANONICALUX.md); may or may not be built as written.

---

## PART A — THE UX FLOW (what the contributor SEES) [CURRENT-TRUTH]

Paths under `wevibe-server/wevibe-dashboard/`.

1. **Sessions list.** Contributor opens `app/(dashboard)/sessions/page.tsx`, picks a locally-captured coding session (transcript loaded via `loadSessionDetail`). An **"Extract" button** renders at `sessions/page.tsx:718-724` (label `queueCtaLabel`, `:126`).
2. **Click → client queue.** `requestActiveSessionExtraction` (`sessions/page.tsx:455-473`) → `enqueueActiveSession` (`:409-422`) → `enqueueExtraction({ sessionId, pubkeyHex, transcript, title, directory, model })` (`:414`) in the client singleton `lib/extraction-queue.ts:313`.
3. **POST to dashboard route.** `runJob` (`extraction-queue.ts:138`) POSTs `/api/extract` with body `{ transcript, title, directory, model, session_id }` — **no org field** (`:140-152`).
4. **Progress toast.** A persistent toast shows chunk progress (`updatePersistentToast`, `extraction-queue.ts:109-136`); the sessions page shows "Extracting…" (`sessions/page.tsx:652,701`).
5. **Poll loop.** On `202 {job_id}` the client polls `GET /api/extract/status?job_id=…` every 4 s (`extraction-queue.ts:193-279`; `POLL_INTERVAL_MS=4000` `:51`, 20-min deadline `:52`).
6. **Draft saved.** On `status:'done'`, `saveDraft(pubkeyHex, …)` writes the draft to **localStorage** key `wevibe.drafts.v1.<pubkeyHex>` (`extraction-queue.ts:266-273` → `lib/draft-store.ts:131-143`). Toast flips to "Extraction complete — review your memories".
7. **Review page.** Contributor opens **`/sessions/extracted`** (`app/(dashboard)/sessions/extracted/page.tsx`), which loads drafts (`:48`) and renders one `<MemoryReview>` per draft (`:134-142`).
8. **Review + Submit.** In `components/sessions/memory-review.tsx` the contributor sees, per memory: a "Memory" badge (`:471-475`), a **near-dup amber badge** when present ("Possible duplicate · of an injected/extracted memory · NN%", `:476-482`), the `implement` text (`:516-518`), Context (`:520-523`), Don't (`:526-530`), and **stack pills** (`:532-543`). They select memories, optionally pick an org per memory, and click **Submit** (`:551-564` → `submitSelected` `:171-329`).

**KEY UX FINDING [CURRENT-TRUTH]:** the contributor-facing UI renders **NO keyword pills**. `memory.keywords` is only passed to `buildSubmitMemoryPayload` at submit (`memory-review.tsx:260`); it is never displayed. Keyword pills (classified/suggestions, 3-color provenance) exist only on the **moderation/leader side** — `components/moderation/leader-pipeline-panel.tsx:1351-1359` and `components/moderation/moderator-review-panel.tsx:396-400` — a different data shape. The near-dup flag IS shown to the contributor; keywords are not.

---

## PART B — HOW THE BACKEND WORKS (end-to-end) [CURRENT-TRUTH]

### Transport & services
- Dashboard route `app/api/extract/route.ts` → MCP `http://127.0.0.1:4450` (`lib/config.ts:15,60-65`), target `POST /v1/extract` (`route.ts:161`).
- Dashboard route `app/api/extract/status/route.ts` → MCP `GET /v1/extract/status/<jobId>` (`:104`).
- MCP `wevibe-mcp/src/http-server.ts` registers `POST /v1/extract`→`handleExtract` (`:1068-1071`) and `GET /v1/extract/status/:id`→`handleExtractStatus` (`:1063-1066`).
- Hub Go `wevibe-server/wevibe-hub/` serves org vocab, submit, verify, and the commit-stage vocab join.
- Umbral sidecar `:4451` performs encrypt-at-verify (leader side).

### The MCP async job (transport-only, per report `06-07-26-0230`)
`handleExtract` (`http-server.ts` ~410-500): parse body → `jobId=randomUUID(); createJob(jobId, trace)` (`:476-477`) → return **202 `{job_id,status:'accepted'}`** (`:499`) → detached worker `void (async()=>{ …extractMemories… completeJob/failJob })()` (`:481-496`). Job registry = `src/extract-jobs.ts` (in-memory Map + disk `~/.wevibe/jobs/<id>.json`, R-32 resumable). Progress hook `onProgress(done,total)` → `recordChunkProgress` (`extract-jobs.ts:154-177`).

### `extractMemories` (the core, `src/extraction.ts`)
1. **Org context resolve** (`:919-943`) — **gated on `options.orgContext`** (see §CRUX): `getOrgInfo` (`:923`), `getOrgKeywords` (`:926`, FATAL on failure `:927-930`), `getOrgKeywordCandidates`→emerging terms (`:933`, non-fatal). When `orgContext` undefined → all stay empty (`orgVocabulary=[]`).
2. **Context window resolve** (`:948-959`) — remote min-across-providers window (`getModelMinContextWindow`), 75% input budget (`budgetChars`). Fail-closed (report `06-07-26-0051`).
3. **Substrate build** **[DESIGN-INTENT; D-SESSION-SUBSTRATE (`DECISIONS.md:892`)]** — extraction substrate = the full local session event stream captured by the plugin (user messages incl. repair/feedback, assistant text, tool calls + tool outputs including test/build/error output, and file-edit events), built by ONE shared substrate builder that is byte-identical between the dashboard Extract path and benchmark harnesses; benchmark-private oracle/gate internals are permanently excluded.
4. **Episode segmentation pre-pass** **[DESIGN-INTENT; D-FAILURE-EPISODES (`DECISIONS.md:905`)]** — deterministic code-only pre-pass (no LLM) runs before `planChunkSlices` (`:700-720`): failing signal → following edits → signal disappearance (resolved) or persistence (unresolved). Resolved episodes yield symptom→diff→validation evidence spans; unresolved episodes yield dnd-only negative candidates ("tried X, symptom Y, unresolved") **— but only within a session that resolved ≥1 problem (see the progress invariant below)**. E2 evidence-bounding gates stay unchanged (this widens what extraction sees, never what it may claim); attribution uses the full attempt diff and discloses coincidental flips.

**EXTRACTION PROGRESS INVARIANT [DESIGN-INTENT; D-BENCH-CUMULATIVE-LOOP-2026-07-23 §2, Walter-ratified 2026-07-23 — normative text at `wevibe-docs/WP-DESIGN-SPEC.md:918`; the decision bodies are NOT in `DECISIONS.md` (citation drift noted 25-07-26-0330)].** A session resolving ZERO objective problems/errors MUST produce ZERO memories — extracting a standalone memory (positive OR negative) from a wholly-unresolved session is a HARD INTEGRITY FAILURE (the benchmark halts; extraction must be fixed). A session resolving ≥1 problem enters extraction even if not fully green; net-new committed corpus delta may legitimately be zero when all candidates duplicate injected/existing knowledge. **[CURRENT-TRUTH 2026-07-25] the gate is now BUILT and LIVE:** shipped in canonical `wevibe-mcp c5304d9` (`src/http-server.ts` fail-closed + gate; `tests/extraction-zero-progress-gate.test.ts`) and the bench clone `1a04bae` (live on :4550/:4451) — in `handleExtract`, after episodes are computed+logged and after `createJob`, before the free-model preflight/`runExtractionJob`: if `resolved === 0` → log `op=extract phase=zero_progress`, `completeJob(jobId, {memories: [], meta: {emptyReason: 'zero_progress'}})`, emit the terminal `extraction.integrity` record (`outcome:'completed'`, resolved 0, emitted 0, `empty_reason:'zero_progress'`, `invariant_violation:false` — byte-compatible with the coordinator contract), respond with the unchanged `202 {job_id, status:'accepted'}` shape; NO LLM call, no preflight. **Fail-closed:** segmentation (`segmentFailureEpisodes`+render) is wrapped — on throw: ERROR op-log, integrity `outcome:'failed'` + `emptyReason:'segmentation_error'`, `500 extraction_segmentation_failed`, no job, no LLM. Live-proven $0 at :4451 (the same resolved=0/unresolved=1 session shape that emitted 11 confabulated candidates on 25-07 03:56 now emits 0). The external benchmark-coordinator abort (missing/uncorrelatable record, or `invariant_violation:true`) remains as the BACKSTOP. (Report `25-07-26-0330-inproduct-zero-progress-gate.md`.)
5. **Chunk plan** — `planChunkSlices` (`:700-720`, overlap-staggered), invoked `:1113`.
6. **Concurrent fan-out** — `runBounded` (`:724-751`), invoked `:1126`, `concurrency = isLocal?1:4` (`REMOTE_CHUNK_CONCURRENCY`, R-33 honored). Each chunk → `llm.chat(...)` (`:1129`).
7. **LLM candidate schema** — `MEMORY_EXTRACTION_SCHEMA` (`:660-694`): model returns a **FLAT `keywords:[{keyword,weight}]`** array (prompt `prompts/extraction.md:37` "Do NOT split into classified/suggestions").
8. **Keyword routing** — `routeKeywordCandidates(flatCandidates, orgVocabulary)` (`:288-337`), invoked `:1181-1184`, builds the `{classified,suggestions}` split (see §CRUX).
9. **Exact-hash dedup** — `dedupeOverlapCandidatesByExtractionHash` (`:504-524`), invoked `:1162`; hash = `computeExtractionHash` sha256 over `{implement,context,dnd,stack}` (`:165-182`).
10. **Semantic near-dup annotate** — `annotateNearDuplicates` (`:775-901`), threshold `NEAR_DUP_COSINE_THRESHOLD=0.93` (`:70`), invoked `:1189-1192`. **Flags, never drops** — attaches `near_dup` per memory.
11. **Complete** — `completeJob` (`extract-jobs.ts:179-202`) stores `result`.

### Status poll + normalize
`app/api/extract/status/route.ts` (`GET :41`) proxies MCP status; on `done`, loops `result.memories` and calls `normalizeMemoryCandidate` (`lib/extract-shared.ts:144-196`) per candidate (`:290-314`), returning normalized memories (`:342`). Client `saveDraft`s them to localStorage.

---

## STAGE-BY-STAGE TABLE

| # | Stage | Trigger | Service / fn (file:line) | Inputs | Outputs | Canon | What-to-log-to-measure |
|---|---|---|---|---|---|---|---|
| 1 | Session capture (local) | coding session | plugin → `~/.wevibe/served-memories.json` + transcripts | session I/O | local transcript | D-5.7 | plugin served_store writes |
| 2 | Enqueue (client) | click "Extract" | `extraction-queue.ts:313` `enqueueExtraction` | sessionId, transcript, model | queued job | D-5.7 | (browser, no op-log) |
| 3 | Dashboard extract route | POST `/api/extract` | `app/api/extract/route.ts:205-253` | transcript, model, **`org_id` from `settings.org_id` only if non-empty (:227-229)** | MCP body → `/v1/extract` | D-5.7 | `logOp('dashboard.extract')` entry `:126`, outcome `:371` |
| 4 | MCP enqueue | `POST /v1/extract` | `http-server.ts:437-499` `handleExtract` | body (`org_id?` read `:437-439`) | 202 `{job_id}`; detached worker | — | `logOp('extract.job',phase:'accepted')` `:498` |
| 5 | Org context resolve | worker start | `extraction.ts:919-943` (gated `if(options.orgContext)`) | orgContext (from `org_id`) | orgVocabulary[], emergingTerms[], orgInfo | **D-KEYWORD-AT-EXTRACTION cl.1** | `extract` `phase:'entry'` **`org=`** `:1049-1058` |
| 6 | Context window + budget | after org resolve | `extraction.ts:948-959` | model slug, transcript len | char budget | — | `extract phase:'context_resolve'` `:960` |
| 7 | Substrate build (shared) **[DESIGN-INTENT]** | budget resolved | planned shared substrate builder (dashboard Extract + benchmark harnesses) | full local session event stream (messages/assistant/tools/tool outputs/file edits) | canonical substrate bytes/events (benchmark-private oracle/gate internals excluded) | **D-SESSION-SUBSTRATE (`DECISIONS.md:892`)** | substrate event-counts + builder fingerprint parity (Extract vs benchmark) |
| 8 | Episode segmentation pre-pass **[DESIGN-INTENT]** | substrate ready | deterministic code-only pre-pass before `planChunkSlices` `:700-720` | substrate events + failing signals | resolved episodes (symptom→diff→validation spans) + unresolved dnd-only negative candidates **(zero-resolution session ⇒ zero memories — D-BENCH-CUMULATIVE-LOOP §2; gate LIVE 25-07: mcp `c5304d9` + clone `1a04bae`, fail-closed, no LLM on resolved=0)** | **D-FAILURE-EPISODES (`DECISIONS.md:905`)** | episode counts (resolved/unresolved), evidence-span bounds, coincidental-flip disclosures |
| 9 | Chunk plan | segmentation done | `planChunkSlices` `:700-720` (call `:1113`) | transcript/substrate, budget, overlap | chunk slices | — | `extract phase:'chunk_fanout'` |
| 10 | Concurrent LLM fan-out | slices ready | `runBounded` `:724-751` (call `:1126`); `llm.chat` `:1129` | slices, model, VOCABULARY prompt | flat candidates/chunk | D-5.6 | per-chunk log |
| 11 | Keyword routing | per candidate | `routeKeywordCandidates` `:288-337` (call `:1181`) | flat `{keyword,weight}`, `orgVocabulary` | `{classified,suggestions}` | **D-KEYWORD-AT-EXTRACTION cl.1** | (no dedicated log — GAP, see §MEASURE) |
| 12 | Exact-hash dedup | chunks merged | `dedupeOverlapCandidatesByExtractionHash` `:504-524` (call `:1162`) | all candidates | deduped set | — | `extract phase:'dedup'` `:1164` |
| 13 | Semantic near-dup | after dedup | `annotateNearDuplicates` `:775-901` (call `:1189`) | candidates + injected mem texts | `near_dup` flags (≥0.93) | D-RECALL-ALIGNMENT | `phase:'near_dup'` `:878` + `near_dup_summary` `:890` |
| 14 | Complete job | worker done | `completeJob` `extract-jobs.ts:179-202` | result | job status `done` | — | `phase:'done'{kept}` |
| 15 | Status poll + normalize | client 4s poll | `status/route.ts:290-314` → `normalizeMemoryCandidate` `extract-shared.ts:144` | MCP result | normalized memories | — | `logOp('dashboard.extract_status')` done `:324` |
| 16 | Draft → localStorage | `status:'done'` | `extraction-queue.ts:266-273` → `draft-store.ts:131` | normalized memories | draft in `wevibe.drafts.v1.<pk>` | D-5.7 | (browser) |
| 17 | Render review (pills=none) | open `/sessions/extracted` | `memory-review.tsx:471-543` | draft | cards + near-dup badge | D-5.7 | (client) |
| 18 | Submit | click Submit | `submitSelected` `memory-review.tsx:171-329` → `POST /v1/orgs/{org}/submit` | encrypted memory + keywords | pending row | D-5.7 | hub `http.request` |
| 19 | Land pending_keyword + candidates | submit received | hub `moderation.go:~110-157` | submission | `pending_keyword`; `keyword_candidates` rows | D-MODERATION-ADVISORY | hub log |
| 20 | Leader curate | leader review | `SubmitKeywordResults` `keyword_extraction.go:108`; `UpdateKeywords` `:447` | curated `extraction_result` | persisted on pending row | D-KEYWORD-AT-EXTRACTION cl.4 | hub log |
| 21 | Encrypt-at-verify | leader Verify | `VerifyKeywords` `keyword_extraction.go:209` (umbral `:4451`) | capsule+ciphertext, embedding | `pending_keyword→pending_chain` | D-MODERATION-ADVISORY | hub log |
| 22 | Chain commit | leader broadcast | `MsgApproveSubmission` (`verify/canonical.go:174`) | signed approve | on-chain tx | D-6.1 | chain |
| 23 | Vocab join | watcher observes tx | `processApproveMemoryBookkeeping` `watcher_memory.go:148-181` | on-chain keywords | upsert `org_keywords` + insert `memory_keywords`; `→committed` | **D-KEYWORD-AT-EXTRACTION cl.5** | hub watcher log |

---

## SUB-SYSTEMS

### (1) Async job model [CURRENT-TRUTH]
Sync `/v1/extract` was replaced by enqueue+poll (report `06-07-26-0230`) because the blocking call exceeded undici's 300 s `headersTimeout` → 502. `POST /v1/extract` returns 202 immediately; worker runs detached; `GET /v1/extract/status/:id` returns `{status, chunks_done, chunks_total, result?, error?}`. Registry `src/extract-jobs.ts` (Map + disk `~/.wevibe/jobs/<id>.json`). Client polls 4 s, 20-min deadline (`extraction-queue.ts:51-52`).

### (2) Context-window resolution + chunk planning + concurrent fan-out [CURRENT-TRUTH]
(reports `06-07-26-0051`, `04-07-26-1237`.) Budget = 75% of the extracting model's **min-across-providers** window (`getModelMinContextWindow`; kimi-k2.6 min≈96000 across 20 providers, not top 262144). No `max_tokens` cap. `planChunkSlices` (`:700`, overlap 8000) → `runBounded` (`:724`, concurrency 4 remote / 1 local, R-33) → merge → exact-hash dedup. Each chunk's LLM sees only its slice + Block A (already-known served memories) + VOCABULARY.

### (3) LLM extraction call + candidate schema [CURRENT-TRUTH]
Prompt scaffold `extraction.ts:1029-1044`: `Project / Stack / Directory / {orgContextBlock} VOCABULARY:\n{vocabularyBlock} / {emergingTermsBlock} / KEYWORD OUTPUT CONTRACT / transcript`. `vocabularyBlock = orgVocabulary.join('\n')` or literal `(none)` (`:973-975`). Structured output `MEMORY_EXTRACTION_SCHEMA` (`:660-694`): each memory requires `implement, context, dnd, stack, memory_type, preference_confidence, keywords`, where `keywords` is a **flat** `[{keyword,weight}]` (model told NOT to split). Candidate shape `MemoryCandidate` (`:87-100`); `ClassifiedKeyword` / `SuggestedKeyword` (`:26-37`) — suggestions carry `rationale`.

### (4) The KEYWORD sub-pipeline [CURRENT-TRUTH + DESIGN-INTENT] — THE CRUX
See §CRUX below for the full verdict. Mechanics:
- **[DESIGN-INTENT, DECISIONS.md:594]** Extraction fetches `GET /v1/orgs/{org}/keywords` and classifies each keyword: in-vocab → `classified`, not-yet-in-vocab → `suggested-new`.
- **[CURRENT-TRUTH]** The MCP implements exactly this: `getOrgKeywords` (`extraction.ts:926` → `org-client.ts:1209-1223`, drops `deprecated`) supplies `orgVocabulary`; `routeKeywordCandidates` (`:288-337`) splits: `vocabularySet.has(keyword)` → `classified` (`:297-304`), else pattern/underscore filter → `suggestions` (`:317-322`). Vocab is injected into the prompt (`:1033-1034`).
- **[CURRENT-TRUTH]** BUT the whole org block is gated `if (options.orgContext)` (`:922`), which requires `org_id` in the `/v1/extract` body. `classified` is initialized `[]` (`:381-384`) and stays `[]` whenever `orgVocabulary` is empty.

### (4a) Memory-side keyword-LABELING gap [GAP — 2026-07-24, root cause of the bench vector-only serve]

Separate from the org-resolution crux above (which is about WHERE classification runs): even when extraction runs with a resolved org, the **keywords the LLM assigns to memories are labeled by build-tool, not by domain**. Empirical (gather-verified 2026-07-24 on the 17-memory bench corpus): extraction tagged memories `type_stripping,frontend`-style (the tool used) instead of `backgammon,doubling,cube`-style (the problem domain) — **8/17 corpus memories mislabeled**. Consequence: a consumer's domain-rich need-card (the query side is HEALTHY — 4–13 rich domain terms; digest = intent+task only by design; keywords boost-never-gate) finds NO keyword overlap with the on-target memory, so the decisive hit arrives **vector-only** (the intended hybrid tail), its serve receipt then 400s on empty `matched_keywords` (D-4.2 frozen), and the plugin's fire-and-forget drops it silently (bench defect D-C — OPEN Walter canon question: serve-count-only receipt = chain change, vs drop-with-eyes-open; stamping the memory's own keywords as matched = RULED OUT). **Fix direction = extraction-side keyword quality (prompt/labeling guidance toward domain terms), NOT need-card redesign.** Standing metric: keyword-match-rate reported on every bench run (derivable from logs, ~free).

### (5) Semantic near-dup detect/flag/display (0.93 cosine) [CURRENT-TRUTH]
(report `06-07-26-0344`.) Concurrent chunks under-dedup vs the old single pass, so `annotateNearDuplicates` (`:775-901`) embeds candidate texts + injected already-known memory texts, computes best cosine, and attaches `near_dup {source:'injected_memory'|'intra_session', matched, score}` when `score ≥ 0.93` (`:70`). **DETECT+FLAG+DISPLAY, never auto-drop.** Dashboard `normalizeNearDup` (`extract-shared.ts:119-142`) whitelist-copies it; `memory-review.tsx:476-482` renders the amber badge. No DB change (candidates never persisted server-side).

### (6) Exact-hash dedup [CURRENT-TRUTH]
`dedupeOverlapCandidatesByExtractionHash` (`:504-524`, call `:1162`) collapses identical candidates emitted by overlapping chunks; hash = sha256 over canonical-sorted `{implement,context,dnd,stack}` (`computeExtractionHash` `:165-182`). Logs `phase:'dedup'` with collapsed count.

### (6a) Corpus-aware dedup canon note [DESIGN-INTENT]
The extraction-context dedup path above is structurally limited to what extraction can see in-session (chunk outputs plus the ALREADY-KNOWN injected-memory pool), so it is blind to committed org-corpus memories that were never injected for that run (for example OFF/capability-filtered memories). Canon now pins corpus-aware behavior in **D-BENCH-CORPUS-PRESERVATION (2026-07-23)**: exact reruns must yield corpus delta zero, genuinely new knowledge must be preserved, no single cosine threshold is sufficient (nomic-768 duplicate band ~0.82-0.89 overlaps a distinct band ≤~0.8047), ciphertext-family hashes cannot do content dedup, and coverage must include memories outside the producing session's injected set. Mechanism is a downstream backend-first implementation program; baseline evidence is `22-07-26-1636-corpus-rerun-dedup-baseline.md`.

### (6b) Extraction-session is a provenance LEG [DESIGN-INTENT + CURRENT-TRUTH]
Extraction is **post-hoc, user-triggered, and is itself an LLM session** (the fan-out above): the USER supplies the extraction model/provider/key (remote OpenRouter default or local), ONE call only if the transcript fits, else overlapping chunk fan-out (8000-char overlap, remote concurrency 4 / local 1). **NEVER describe this as one call, and NEVER as a WeVibe-controlled small extractor** — both claims are false. Remote extraction sends transcript slices **unredacted** to the user-chosen provider; the hub never sees plaintext. **[CURRENT-TRUTH]** production submits `attestation:null` with no session binding (no producer/extraction evidence flows today).

Canon (**D-PROVENANCE-ADMISSIBILITY-2026-07-23**, DECISIONS §22) requires a servable memory to carry admissible provenance for BOTH legs, joined by welds: (a) **producer-evidence → deterministic substrate** (the substrate corresponds to the committed/witnessed producer trajectory, not doctored local events); (b) **substrate/chunk-plan → all extraction requests** (bind exact slices, scaffold/config/model, the COMPLETE overlapping chunk set — the multi-call fan-out is what makes weld (b) non-trivial); (c) **extraction-responses → submitted memory** (bind structured outputs, re-run deterministic post-processing, commit final plaintext hash + keyword/semantic-shadow inputs + extraction-output hash + source trajectory root). The producer trajectory root is an ordered event/turn commitment (versioned hash chain OR Merkle accumulator; primitive not locked) replacing the inadequate flat session hash. Local extraction requires P1-compatible verifiable execution (CommitLLM-class gap) before its memories are admissible. Provenance proves exact inputs/responses/identities + deterministic derivation — **it cannot prove the synthesized memory is semantically faithful or correct.** All UNBUILT; canon contract only.

### (7) Return → status-poll → normalize → localStorage → render [CURRENT-TRUTH]
Status route normalizes (`normalizeMemoryCandidate` `extract-shared.ts:144`; keyword shape validated by `normalizeKeywords` `:92-117`). Client `saveDraft` → localStorage `wevibe.drafts.v1.<pubkeyHex>` (`draft-store.ts:131`). `/sessions/extracted` renders `<MemoryReview>` — near-dup badge shown, **no keyword pills** (keywords only ride the submit payload).

### (8) Downstream after Submit [CURRENT-TRUTH, shallower] — D-5.7 manual submit
Lifecycle `pending_keyword → pending_chain → committed` (schema `db/schema.sql:133-136`; no `pending` gate — D-MODERATION-ADVISORY). Submit → client encrypt (DEK sealed to mod pubkey) + sign → `POST /v1/orgs/{org}/submit` (`SubmitMemory`) → `moderation.go:~126` inserts `pending_keyword` + `moderation.go:132-157` captures `suggestions` into `keyword_candidates`. Leader curates (deselect) via `SubmitKeywordResults` (`keyword_extraction.go:108`) / `UpdateKeywords` (`:447`) — these persist `extraction_result` and **do NOT read `org_keywords`** (no vocab check — D-KEYWORD-AT-EXTRACTION cl.6). `VerifyKeywords` (`:209`) validates format+weight-sum(1.0)+count, stores umbral capsule/ciphertext + embedding, transitions `pending_keyword→pending_chain`. Leader signs `MsgApproveSubmission`; `ChainWatcher.processApproveMemoryBookkeeping` (`watcher_memory.go:148-181`) upserts `org_keywords ON CONFLICT DO UPDATE SET deprecated=false` (`:150-154`) + inserts `memory_keywords` (`:159-163`) — the **vocab JOIN** — then `→committed`. This commit-stage upsert is exactly how `wevibe-org-0` accumulated its 128 active keywords + 185 `memory_keywords` rows.

---

## CANON MAPPING [DESIGN-INTENT — verbatim, verified DECISIONS.md]
- **D-KEYWORD-AT-EXTRACTION** (`DECISIONS.md:590-613`). Cl.1 (`:594`): *"Extraction fetches the org's current vocabulary (`GET /v1/orgs/{org}/keywords`, readable by any active member) and classifies against it… Terms already in vocabulary are in-vocab; terms not yet in vocabulary are emitted as suggested-new."* Cl.5 (`:598`): vocab-join is **at commit** (`processApproveMemoryBookkeeping`). Cl.6 (`:599`): leader `verify-keywords`/`update-keywords` do NOT check vocab. → **Classification is a design-intended EXTRACTION-time step; the vocab JOIN is a separate commit-time step; the leader neither classifies nor joins.**
- **D-MODERATION-ADVISORY** (`DECISIONS.md:1022-1045`): moderation is always-on advisory, never a gate; `pending` dropped; `pending_keyword` is the submit-landing state.
- **D-5.7** (`DECISIONS.md:642-651`): contribution is manual/dashboard-driven; nothing leaves the machine without explicit Extract then Submit.
- **D-ATOMIC-CONFORMANCE** (`DECISIONS.md:920`): one insight = one memory object end-to-end (including benchmark harnesses); collapsing atomic memories into a blob is a canon violation (reaffirms §5.1 atomic format + D-MEMORY-OBJECT-IMPLEMENT-DND).
- **D-RECALL-ALIGNMENT** (`DECISIONS.md:670-701`): query keywords share the controlled vocabulary; keywords boost-not-gate; embedder pinned `nomic-embed-text:v1.5` both sides.
- **D-MISSION-INVARIANT** (`DECISIONS.md:3160-3176`): no single party can READ plaintext / WITHHOLD / REWRITE / KILL. Extraction runs on plaintext the contributor already holds, locally; the hub never sees plaintext (keywords + clean embeddings are a disclosed semantic shadow, per D-EMBEDDING-HONEST-CLAIM — the hub is NOT zero-knowledge about content).

---

## CRUX — WHY `classified:[]` + `org=-` FOR A RUN WITH 128 ORG KEYWORDS

**Live fact:** trace `e248225f`, job `28080a1d`, model `moonshotai/kimi-k2.6`, 9 memories kept, every candidate `{classified:[], suggestions:[…]}`, op-log `org=-`; hub `org_keywords` has 128 active for `wevibe-org-0`, `memory_keywords` 185 rows.

### VERDICT: **(C) Partial wiring.** The vocab-classify machinery is fully built and canon-intended to run AT EXTRACTION, but the ORG WAS NEVER RESOLVED for this invocation, so the machinery never fired.

**Evidence chain (each hop verified):**
1. **Request field is `org_id`, not `org`.** MCP reads `org_id` from the body (`http-server.ts:437-439`); absent/empty → `orgId = undefined`. There is NO default/fallback org.
2. **`org=-` is a rendering of `undefined`.** Entry log emits `org: options.orgContext?.orgId` (`extraction.ts:1049-1058`); when `orgContext` is undefined the logger renders `undefined` as `-` (`logger.ts:114-117`). So `org=-` literally means "no `org_id` was sent."
3. **The org block is gated on `orgContext`.** `if (options.orgContext)` (`extraction.ts:922`) — and `orgContext` is set only when `org_id` present (`http-server.ts:468-470`). No `org_id` → block skipped → `orgVocabulary = []`, prompt VOCABULARY renders `(none)` (`:973-975`).
4. **Empty vocab ⇒ everything is a suggestion.** `routeKeywordCandidates` (`:288-337`): with an empty `vocabularySet`, `vocabularySet.has(...)` is never true, so no candidate reaches `classified` (`:297`); all pass to `suggestions` (`:317`). `classified` stays its initialized `[]` (`:381-384`). **This is exactly the observed shape.**
5. **The missing hop is the DASHBOARD.** `app/api/extract/route.ts:201` sources org ONLY from `settings.org_id` (the server-side on-disk dashboard settings file, default `''`), and sends `org_id` only if non-empty (`:227-229`). The client POST body carries no org (`extraction-queue.ts:145-151`). The contributor's active org context (`useOrgContext`/`activeOrg`, used AT SUBMIT in `memory-review.tsx:123,176-198`) is **never wired into extraction.** So unless the user manually set `org_id` in Profile settings, extraction runs org-less — which is what happened.

**Answers to the four sub-questions:**
1. **Does `/api/extract` + `:4450 /v1/extract` carry an org?** The MCP body has `org_id` (`http-server.ts:135,437-439`); the dashboard sends it only from `settings.org_id` when non-empty (`route.ts:227-229`). It is NEVER sourced from the client request, session, cookie, header, or the contributor's active org. The live run logged `org=-` because `settings.org_id` was empty → `org_id` omitted → MCP `orgContext` undefined.
2. **Is org vocab fetched + passed to the LLM?** YES, and fully — `getOrgKeywords` (`extraction.ts:926` → `org-client.ts:1209`, `GET /v1/orgs/{org}/keywords`) → injected as the prompt `VOCABULARY:` block (`:1033-1034`) and used by `routeKeywordCandidates`. **But only inside the `if(options.orgContext)` gate (`:922`)** — so it did NOT run this time.
3. **What populates `suggestions` vs `classified`?** `routeKeywordCandidates` (`:288-337`) builds both: in-vocab → `classified`, else → `suggestions`. The LLM returns only a flat `[{keyword,weight}]` list (schema `:660-694`; prompt `extraction.md:37`); the split is a pure code post-process. Per canon (D-KEYWORD-AT-EXTRACTION cl.1), `classified` is MEANT to be filled at EXTRACTION (in-vocab terms), NOT later — the leader curates but does not classify (cl.6); the commit-stage watcher joins new terms into `org_keywords` (cl.5) but that is a JOIN, not classification.
4. **A / B / C verdict:** **(C)**, with certainty. NOT (A): canon puts classification at extraction, not at a later stage, so `classified:[]` when the org holds 128 vocab terms is not the designed steady state. NOT (B): the MCP path is present and correct end-to-end (fetch + prompt + split + fail-closed). It is (C): the vocab is *meant* to be sent, the machinery exists, but the org was not resolved for this invocation — the **exact missing hop is `app/api/extract/route.ts:201-229`**, which should resolve the contributor's active org (or the request should carry it) but instead reads only an often-empty `settings.org_id`. Fix surface is the CALLER, not the MCP. Whether extraction should auto-resolve the contributor's current org when `org_id` is omitted is a UX decision (R-CONFIG-SoT-IS-UX / escalate to Walter).

### On the continuance's "keyword-pipeline observability" claim
The claim ("EXISTS+wired: vocab in hub, sent+fail-closed, LLM sees it") is **TRUE about the MCP hop and INCOMPLETE about the caller.**
- TRUE: vocab lives in the hub (`org_keywords` + `ListKeywords`); the MCP fetches it and **fail-closes** (aborts extraction if the vocab fetch throws, `extraction.ts:927-930`); the LLM sees it via the `VOCABULARY:` block.
- MISSING: the "sent" only happens when `org_id` reaches the MCP, and the dashboard resolves `org_id` solely from `settings.org_id` (empty by default). So for a normal contributor the fully-wired path **never fires** — the org is dropped one hop upstream of everything the claim describes. The claim is not wrong about the MCP-side wiring; it just doesn't cover the dashboard org-resolution gap that produced `org=-`.

### RESOLVED-BY-DECISION (2026-07-22)
**[DESIGN-INTENT]** Walter resolved the open UX question via **D-EXTRACT-ORG-AUTORESOLVE** (`DECISIONS.md:930`): when `/api/extract` receives no explicit org, extraction auto-resolves the contributor's ACTIVE org (the same `useOrgContext` / `activeOrg` submit context in `memory-review.tsx:123,176-198`); explicit `settings.org_id` still wins when set. As documented above, the fix surface remains the dashboard caller `app/api/extract/route.ts:201-229` until that caller change ships.

---

## HOW TO MEASURE AGAINST THIS

| Stage | Signal to check | Where |
|---|---|---|
| Dashboard extract | did it send org? | `wevibe-meta/.logs/ops/dashboard.extract-<YYYYMMDD>.log` — entry line; and inspect the outbound `org_id` (present only if `settings.org_id` set) |
| MCP org resolve | **`org=<slug>` vs `org=-`** | `wevibe-meta/.logs/ops/extract-<YYYYMMDD>.log`, `phase=entry`. `org=-` ⇒ no org_id sent ⇒ classification will be empty |
| Vocab fetch | did the hub serve vocab? | hub `http.request` log for `GET /v1/orgs/{org}/keywords`; MCP throws (fail-closed) if it fails |
| Context/budget | window + budget chosen | `extract` `phase=context_resolve` |
| Chunk fan-out | chunk count + concurrency | `extract` `phase=chunk_fanout` + per-chunk lines; job `chunks_done/chunks_total` |
| Keyword routing | classified vs suggestions counts | **GAP — no dedicated log today.** Proxy: inspect the returned/normalized candidates (draft or status response). Recommend adding a `phase=keyword_route {classified_n,suggestions_n,vocab_n}` log at `extraction.ts:1181` |
| Exact-hash dedup | collapsed count | `extract` `phase=dedup` |
| Near-dup | flagged count + scores | `extract` `phase=near_dup` + `near_dup_summary` (≥0.93) |
| Job done | kept count | `extract.job`/job `phase=done {kept}`; status route `dashboard.extract_status` done line |
| Draft | draft present | localStorage `wevibe.drafts.v1.<pubkeyHex>` |
| Submit landing | pending row + candidates | hub DB `pending_submissions.status='pending_keyword'`; `keyword_candidates` rows for the org |
| Verify | pending_chain + umbral | hub DB row transition + `umbral_capsule/ciphertext` populated |
| Commit / vocab join | new `org_keywords` + `memory_keywords` | hub watcher log; DB row counts for the org (this is what grew wevibe-org-0 to 128/185) |

**Fastest single check for the crux symptom:** grep today's `extract-<YYYYMMDD>.log` for `phase=entry` — if `org=-`, the org was never resolved and `classified` will necessarily be empty regardless of how much vocab the hub holds.
