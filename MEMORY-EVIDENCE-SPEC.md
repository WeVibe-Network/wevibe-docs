# Memory Evidence & Benchmark Independence

**Canonical specification of the memory-evidence model.**

Status: 2026-08-21. This is the durable, authoritative statement of what counts as evidence in the WeVibe memory system, how evidence is bounded, how the evidence machinery degrades outside its home environment, and what guards keep the system independent of any evaluation harness. It was established by reading running code and measuring real output, not from the docs alone.

Relation to sibling documents:
- `RECALL-PIVOT-SPEC.md` — the event/boundary spec: what goes on-chain and how standing is computed at the edge. Orthogonal to this spec, which is about evidence and independence, not chain events.
- `MEMORY-LIFECYCLE.md` — the status/traps companion: the running-system state, traps, and what is built versus not. It covers parts of §2/§3/§8 in operational form; this spec is the canonical statement and does not repeat that prose.

Marking convention (unmissable):
- `STATUS: BUILT` — current behavior, verified against running code.
- `STATUS: DESIGN INTENT — NOT BUILT` — specified but not wired / not yet influencing behavior.

A reader must never mistake design intent for implementation.

## §1 Independence boundary

The memory system is a standalone product. It must not depend on — or be tuned to — the output, wording, or structure of any evaluation harness. The memory system and the evaluation harness are separate products with separate lifecycles: one measures, the other is measured.

The ban has two prongs:

1. **No harness vocabulary in the product's prompts.** Vocabulary that the evaluation harness itself smuggles into the product's prompts is forbidden. Prompts sent to the extraction model must contain no evaluation-harness vocabulary and no task-domain vocabulary drawn from whatever task the harness happens to use.
2. **No reading of harness-originated output.** The product must not detect, parse, or pattern-match anything that originates from a harness — not by verbatim string, not by approximate phrasing, not by a flag the harness sets on its own output.

Out of scope — the org's own vocabulary is legitimate substrate:

- The ORG'S OWN deliberately-configured vocabulary — `trajectory.txt` / `keywords.txt`, seeded by a human at join time — is the memory system's legitimate substrate and subject matter. It is NOT a harness coupling, and it is explicitly outside the ban's scope. The ban targets vocabulary the harness itself injects and the product reading harness output; it does not target the memory system's own human-seeded domain vocabulary.

Why this matters — the failure mode to name:

- **Coupling by shared phrasing.** Two independently written components can agree on a phrase, work correctly, and give every appearance of a general mechanism — while in fact being a private agreement that breaks silently the moment either side is reworded. An evaluation harness that shapes the system it measures invalidates the measurement: the system's results then describe the pairing (system + harness), not the memory system.

## §2 Two kinds of failure, and only one is evidence

Two classes of failure exist and must never be conflated:

- **own-failure** — the agent's own actions failing: its tools erroring, its commands exiting non-zero, its own tests failing. Produced by the agent's own actions inside the session.
- **external judgment** — any outside party (an evaluation harness, a reviewer, a user) asserting that the work is wrong.

Only own-failure constitutes evidence that something was hard to learn.

External judgment is a report of a symptom, never a record of work. It says the outcome was wrong; it carries no information about what was attempted. Attributing a memory to external judgment is circular whenever that judge is also the scorer — the verdict becomes evidence about the very work the verdict grades.

The positive rule, on its own reasoning (not because any harness does anything):

- **Inbound messages are not failure evidence.** A message asserting that the work is wrong is a symptom report, not a record of work. The agent acts on the report, and that action — its own commands, its own tests, its own tool errors — produces genuine evidence through the agent's own tooling. Nothing is lost by ignoring the report itself.

This rule stands on its own. It is not, and must not be, justified by reference to any evaluation harness — a rule justified that way is itself a coupling.

(Reference, not duplication: `MEMORY-LIFECYCLE.md` §1 "The four signal kinds, and why they are not interchangeable" describes the operational form — `test_failure`, `tool_error`, `command_failure` are citable; `user_feedback` is withheld as the grader speaking.)

## §3 Struggle — what it means and what it is for

Purpose: struggle is a **lift predictor**.

- Relevance answers "is this memory about my problem."
- Struggle answers "is this worth one of a small number of injection slots."

A lesson the agent derived easily, it will derive easily again — recalling it produces no benefit. Struggle is the signal that says a memory earned its slot.

STATUS: DESIGN INTENT — NOT BUILT.

- The signal is produced (derived, never asked of the model), carried, and displayed. It does not influence retrieval today.
- Produced: `wevibe-mcp/src/extraction.ts:909` (`struggled: outcomes.includes('unresolved') || attemptCount > 1`).
- Carried: stamped into the extraction envelope (`extraction.ts:1602`) → dashboard (`session-types.ts:62,80` → `extract-shared.ts:202`).
- Displayed: `wevibe-server/wevibe-dashboard/components/sessions/memory-review.tsx:631`.
- Not consumed by retrieval: hub ranking `RankCandidate` (`wevibe-server/wevibe-hub/internal/retrieval/ranking_core.go:9–17`) has no struggle/episode field; the score formula (`ranking_core.go:182–191`) is vector + keyword-boost + standing only.

Finding that nothing consumes the stored field is expected while the ranking work is outstanding. It must be reported as an unbuilt gap — never treated as dead plumbing to remove. (See `MEMORY-LIFECYCLE.md` trap 1.)

## §4 Evidence-window semantics

A failure's attempt window is bounded in time:

- A window **opens at the failure** and **closes when the failure is observed to be gone.**
- A failure that is never observed to clear has no natural close. Left unbounded, its window runs to the end of the session and absorbs unrelated work. The resulting count then measures *when the failure happened*, not how hard it was fought — and multiple such failures each absorb the same tail.
- The window must therefore be **bounded at both ends in all cases.**

Direction to err: **undercounting is correct.** The count feeds a worth-remembering decision; an inflated count promotes a memory that did not earn its slot, which is worse than omitting one that did.

Where windows can overlap, effort is counted as **distinct work items, not as a sum across failures** — otherwise shared work is counted more than once.

STATUS: DESIGN INTENT — NOT BUILT.

- Committed code still absorbs the tail: `wevibe-mcp/src/failure-episodes.ts` bounds the window as `episode.validationIndex ?? ordered.length` — an unresolved episode with no validation falls through to the end of the transcript.
- A bounding fix is WRITTEN but UNCOMMITTED (working tree, `failure-episodes.ts:363`): `endExclusive = episode.validationIndex ?? nextSignalIndex ?? ordered.length`, which closes an unresolved episode's window at the next failure signal. Even this fix leaves a residual gap: the FINAL unresolved episode has no subsequent signal and still falls back to `ordered.length`, so "bounded at both ends in all cases" is not yet fully realized even in the working tree. This section remains DESIGN INTENT until that residual is closed and the fix is committed.

## §5 Durability tiers — what will and will not survive contact with other users

The failure-detection machinery is classified by how it degrades outside the environment it was written in, so future maintainers know which parts are safe to build on. Two named tiers:

- **Mechanism (durable).** Timeline arithmetic, window bounding, effort counting, process exit status. Independent of language, ecosystem, tooling, and human language. Fixes here are permanent and benefit everyone.
- **Content matching (fragile).** Recognising failure by the *text* a tool printed. Inherently specific to ecosystem, tool version, and human language. Every addition here is a patch that the next ecosystem will need again.

Measured degradation behavior:

- Text recognition currently identifies the **majority of common test runners precisely**; the remainder fall back to a generic failure classification via process exit status. **Detection is preserved; only label precision is lost.**
- **That fallback is load-bearing and single-threaded.** Where exit status is not propagated by the surrounding tooling, the ecosystems that rely on the fallback become undetectable — not vaguer, **absent**. This dependency must be stated.
- Coverage was **measured, not assumed**, and the measurement is **reproducible without running an evaluation**.

## §6 Stored error text must be selected, not truncated

The retained excerpt of a failure is chosen by **locating the part of the output that indicates the failure** — not by taking the beginning of the output.

- Many toolchains emit banners, version headers, or runtime warnings before any real output. Taking the head of the output stores that preamble instead of the error.
- This is **not specific to any one runtime.** Fixing a named runtime's preamble is precisely the fragile-tier patch (see §5) that the next ecosystem must redo. The rule is therefore stated at the **strategy level**, so it holds for toolchains not yet seen.

STATUS: DESIGN INTENT — NOT BUILT.

- Committed code head-truncates: `wevibe-mcp/src/failure-episodes.ts:73` — `toExcerpt` slices the first `MAX_EXCERPT_CHARS` of the normalized output. It is applied to `command_failure` (`failure-episodes.ts:162`) and `tool_error` (`:173`).
- A partial fix is WRITTEN but UNCOMMITTED: `test_failure` routes through `excerptFromFailure` (`failure-episodes.ts:89–95`, applied at `:150`), which starts the slice at the line where the failure was detected. But `command_failure` and `tool_error` still head-truncate, and even the `test_failure` slice head-cuts if more than ~300 chars remain after the failure line. The strategy-level rule is therefore not yet built.

## §7 Guards that must exist, and what they protect

An invariant that is only written down is not enforced. Where a violation would be silent, a guard is required — and the guard's existence belongs in this documentation, next to the rule. Three guards, each with the failure it prevents and its honest enforcement status:

1. **No evaluation-harness or task-domain vocabulary in any prompt sent to the extraction model.** Prevents **recoupling**. The violation is invisible on inspection and produces no error, so it must be enforced automatically.
   - STATUS: DESIGN INTENT — NOT BUILT. It exists as a test — `wevibe-mcp/tests/prompt-independence.test.ts` (untracked) asserts no prompt file or preset contains evaluator vocabulary or task-domain vocabulary — but it is not wired into any enforcement gate (`wevibe-meta/scripts/verify-clean.sh` does not reference it).

2. **Cross-ecosystem detection coverage pinned by an executable check.** Prevents the coverage claim in documentation from silently rotting; changes to detection are measured rather than assumed.
   - STATUS: DESIGN INTENT — NOT BUILT. It exists as a test — `wevibe-mcp/tests/ecosystem-coverage.test.ts` (untracked) asserts precise runners are labelled `test_failure` and that exit-status-only runners degrade to `NONE` when exit status is absent (pinning the §5 fallback) — but it is not wired into any enforcement gate.

3. **Citations validated against real records.** A citation naming something that does not exist is discarded rather than becoming evidence.
   - STATUS: BUILT. `wevibe-mcp/src/extraction.ts` `deriveEpisodeEvidence` (`:864–875`) matches cited refs against the real episode list and drops any that name no real episode; wired into extraction at `extraction.ts:1595`.

Wiring the first two guards into the enforcement gate is the outstanding gap that a commit will close. Honest nuance: both tests run under `npm test` (vitest `include: ['tests/**/*.test.ts']`), so they are exercised by the repo's test command — but "runs in a test suite" is not "enforced at the seam."

## §8 Verification discipline

These four practices are working rules, each established by a real error that cost time:

1. **Verify against the artifact, not the summary.** Read the stored record, the database, or the running process. A rendered label can disagree with the data behind it.
2. **A test that passes both before and after a change proves nothing.** Any test written to pin a fix must be shown to fail against the unfixed code.
3. **Distinguish what reaches the model from what does not.** Explanatory comments in source code do not affect behavior; strings assembled into prompts do. A survey that does not separate these will badly misstate the size of a problem.
4. **Measure blast radius before concluding severity** — in either direction. Alarm and reassurance are both worthless without a count.

(Reference, not duplication: `MEMORY-LIFECYCLE.md` operational note "Verify against the artifact, not the summary.")

## §9 Terminology

Pinned names for concepts that have been conflated. Use these consistently; do not invent alternatives.

| Term | Meaning | Must never be confused with |
| --- | --- | --- |
| **own-failure** | The agent's OWN actions failing: tools erroring, commands exiting non-zero, its own tests failing. The ONLY class that constitutes evidence. Test sub-case: **own test failure** (code signal kind `test_failure`). | — (this is the evidence class) |
| **external judgment** | Any OUTSIDE party (evaluation harness, reviewer, user) asserting the work is wrong. A symptom report, never a record of work. Harness sub-case: **harness verdict**. | own-failure |
| **selection threshold** | A quality bar applied when SELECTING memories. | harness check |
| **harness check** | A check a harness applies to a FINISHED ARTIFACT. | selection threshold |

Prohibited forms: never use bare **"test failure" / "tests failing"** for the harness sense (that sense is "harness verdict"); never use bare **"quality check"** ambiguously (say "selection threshold" or "harness check").

## §10 Corrections record

Wherever existing documentation or source commentary explains a memory-system rule by reference to an evaluation harness, that explanation is being (or must be) rewritten to stand on its own reasoning. Rationale expressed in harness terms is a **latent coupling**: it teaches the next maintainer that the harness is a legitimate consideration inside the memory system, and the next fix made under that assumption becomes a real dependency rather than a comment.

Any statement that a rule exists "because the harness does X" must be restated as the general property that makes the rule correct for every user.

Target locations (record kept complete):
- `DECISIONS.md`
- `WHITEPAPER.md`
- `WP-DESIGN-SPEC.md`
- `MATCHING_ENGINE.md`
