# Echo Matching Engine — Architecture Decision Record

**ADR-025: Contested Memory Disambiguation + Moderation Similarity Detection**
**Date:** 2026-04-09
**Status:** Accepted
**Sprint:** 13 — Matching Engine Refinement

---

## The Problem

A scoring engine ranks memories. It does not understand them. When two memories score similarly but recommend different approaches, the scoring engine picks one arbitrarily based on decimal-place differences in keyword overlap and vector similarity. The user gets a confident-sounding answer that might be the wrong answer for their situation.

This is the silent failure mode. The user never knows a better memory existed. The agent never questions its choice. Nobody complains because nobody knows what they missed.

At scale (2,000+ memories), this becomes the dominant failure. More memories means more near-misses, more competing approaches to the same problem, more opportunities for the wrong memory to surface.

---

## The Consumer Side

### When results are clear

The developer types a prompt. The agent calls `echo_recall`. One memory scores well above the rest. The MCP client returns it. The agent weaves it into the response. The developer never sees Echo. This is the common case and it stays exactly as it is today.

### When results are contested

The developer types a prompt. The agent calls `echo_recall`. Two or three memories score within striking distance of each other. They address the same problem with different approaches.

The MCP client detects this. It calls the local LLM to read the competing memories and produce three things for each: a one-line summary, a "best when" statement, and the key tradeoff. This gets returned to the agent as structured guidance.

The agent reads the guidance. It asks the developer 2-3 clarifying questions — the kind it would ask anyway when a problem has multiple valid solutions. "Are you optimizing for security isolation or latency?" "Is this for a VPS or a cloud deployment?" Based on the answers, the agent picks the right memory and proceeds.

The developer experiences this as a normal conversation with a thoughtful agent. They don't see scoring breakdowns. They don't see "ECHO WARNING." They see an agent that asks good questions before giving advice. The 5-10 seconds spent answering questions save 30 minutes of following the wrong approach.

### What the developer never does

- Search for memories manually
- Tag or categorize memories
- Choose between memories in a UI
- Configure scoring weights
- Interact with Echo directly in any way

The agent is the interface. Echo is invisible infrastructure.

---

## The Moderation Side

### When a new memory comes in

The moderator sees the pending submission in the queue: the decrypted plaintext, the contributor, the stack hint. This is unchanged.

What's new: before the moderator approves, the system runs the new memory's keywords and embedding against the existing index. If similar memories already exist in the org (score above threshold), they're surfaced alongside the new submission.

The moderator sees:

- The new memory
- 1-3 existing memories that cover similar ground
- For each existing memory: its content, when it was approved, how often it's been retrieved

The moderator makes one of three decisions:

1. **Approve as distinct.** The new memory covers different ground despite keyword overlap. Both belong in the system.
2. **Supersede.** The new memory is a better version of an existing one. Approve the new memory, deprecate the old one.
3. **Deny as duplicate.** The existing memory already covers this. The new submission adds nothing.

There is no AI making this decision. The moderator has domain knowledge the system doesn't. The system's job is to surface the comparison — the human's job is to judge it.

### Optional: LLM-assisted comparison

The moderator can ask the local LLM to compare the new memory against the similar existing ones. The LLM produces a structured diff: what's the same, what's different, which is more specific, which is more general. This is a convenience for moderators reviewing unfamiliar domains. It's not required. A moderator who knows the subject can skip it and judge directly.

### What the moderator never does

- Assign keywords manually (the LLM does this at approval)
- Set keyword weights (the LLM assigns percentage weights)
- Tune scoring parameters (that's the system's job)
- Decide memory ranking (that happens at query time, not approval time)

The moderator is a quality gate for content. The system handles indexing and retrieval.

---

## How It Works

### Recall flow (updated)

```
Agent calls echo_recall(query)
        │
        ▼
MCP extracts keywords + embedding from query
(LLM-based extraction, not deterministic splitting)
        │
        ▼
Hub queries Qdrant: vector similarity + keyword scoring
Hub computes combined score per memory
Hub detects contested: score gap between #1 and #2 < threshold
        │
        ▼
Hub returns: ranked results + contested flag
        │
        ▼
MCP decrypts all returned memories
        │
        ├── NOT contested ──▶ Return top memory to agent. Done.
        │
        └── CONTESTED ──▶ Call local LLM with all decrypted memories
                          LLM produces: summary, best-when, tradeoffs
                          MCP returns structured disambiguation to agent
                                  │
                                  ▼
                          Agent asks user 2-3 clarifying questions
                          Agent picks the right memory
                          Agent proceeds with correct context
```

### Approval flow (updated)

```
Moderator decrypts pending submission
        │
        ▼
System runs similarity query against existing index
using new memory's keywords + embedding
        │
        ├── No similar memories ──▶ Normal approval flow
        │
        └── Similar memories found ──▶ Surface them to moderator
                                       Moderator reviews side-by-side
                                       Decides: distinct / supersede / deny
                                              │
                                              ▼
                                       On approve: LLM extracts keywords,
                                       embedding computed, indexed as today
```

### Where each component has authority

| Component | Decides | Does not decide |
|-----------|---------|-----------------|
| Hub | Scoring, ranking, contested detection | Memory content, which memory is "better" |
| MCP client | Keyword extraction, embedding, disambiguation formatting, encryption/decryption | Scoring weights, what to surface |
| Local LLM | Keyword generation, memory summaries, comparison analysis | Approval/denial, ranking |
| Moderator | Content quality, duplicate resolution, supersession | Keyword weights, scoring parameters |
| Agent | Which memory to apply based on user context, what to ask the user | What memories exist, how they scored |
| User | Clarifying questions that determine which approach fits their situation | Everything else |

---

## What This Does

1. **Prevents silent wrong-memory delivery.** When the system isn't confident, it says so — through the agent, not through warnings.

2. **Prevents contradictory memories from accumulating.** Moderators see similar existing memories before approving new ones. Duplicates and near-duplicates get caught at the gate.

3. **Uses the user's context to break ties the scoring engine can't.** "Are you on a VPS or cloud?" is worth more than 0.03 points of cosine similarity. The user already has this context. Asking costs 10 seconds. Not asking costs 30 minutes of wrong advice.

4. **Works across every MCP client identically.** The disambiguation is text returned by the tool. Claude Code, OpenCode, Codex, Cursor — they all read text, they all know how to ask clarifying questions. No client-specific integration needed.

5. **Keeps Echo invisible in the common case.** Clear winner → silent injection. Only contested results trigger the disambiguation path. Most recalls won't be contested.

## What This Does Not Do

1. **Does not replace the scoring engine.** Keyword extraction quality, α/β weights, and embedding quality still matter. They determine what's contested vs. clear. Better scoring means fewer contested results, which means fewer user interruptions.

2. **Does not automate moderation decisions.** The LLM can compare memories. It cannot decide which one the org should keep. That's a domain judgment that requires human authority.

3. **Does not guarantee the right memory wins.** If the user answers clarifying questions wrong, or if the agent misinterprets the disambiguation guidance, the wrong memory can still be applied. This reduces the failure rate — it doesn't eliminate it.

4. **Does not handle memory staleness.** A memory that was correct 6 months ago might be wrong today (API changed, library deprecated, better approach discovered). Contested detection catches memories that disagree at the same point in time. It doesn't catch memories that were once correct and are now outdated. That's a separate problem.

5. **Does not work without a capable local LLM.** The disambiguation path requires qwen3.5-128k (or equivalent) running on Ollama. If Ollama is down, the system falls back to today's behavior: return top result, no disambiguation. This is acceptable degradation — the system is no worse than it is right now.

---

## What To Watch For

### Short term (Sprint 13-14)

- **Contested threshold tuning.** Starting at 0.15 score gap. Too tight = everything is contested, user gets asked questions on every recall (annoying). Too loose = real conflicts slip through silently. The adversarial benchmark measures this directly — run it at different thresholds, find where Gold@1 peaks.

- **Disambiguation LLM latency.** The local LLM call adds 3-5 seconds to contested recalls. Monitor what percentage of recalls trigger it. If >20% of recalls are contested, the scoring engine needs work — that many ties means the keywords aren't discriminating enough.

- **Moderation UX for similarity.** The similarity results need to be useful, not noisy. If every new memory surfaces 5 "similar" results that are actually unrelated, moderators will ignore the feature. Start with a high similarity threshold (0.6+) and lower it based on moderator feedback.

### Medium term (post-launch)

- **Memory supersession chains.** When memory B supersedes memory A, and later memory C supersedes memory B, the system needs to handle this cleanly. Deprecated memories should not appear in recall results or similarity comparisons. Track supersession as a first-class relationship, not just a deprecation flag.

- **Contested patterns as signal.** If the same query triggers contested results repeatedly across different users, that's a signal that the org needs to consolidate those memories. Surface this to the org leader: "These 3 memories keep competing. Consider merging them or adding differentiating keywords."

- **Agent capability variance.** The disambiguation guidance is text that a capable agent interprets. GPT-4.1 and Claude will ask good clarifying questions. Qwen3 Coder 30B might not. Consider a "disambiguation quality" metric in the benchmark: given contested results and guidance, does the agent ask the right questions?

### Long term (scale)

- **Contested rate vs. memory count.** As the org grows from 100 to 10,000 memories, the contested rate will rise because more memories means more near-misses. If contested rate grows linearly with memory count, the disambiguation path becomes the primary experience rather than the exception. At that point, the system needs better pre-filtering (keyword gates before vector search) or hierarchical memory organization (topics/categories) to reduce the candidate set before scoring.

- **Cross-org memory sharing.** If orgs ever share memories (federation), the contested detection becomes critical — memories from different teams with different conventions will frequently conflict. The disambiguation path is the interface between "our way" and "their way."

---

## Implementation Scope

This ADR covers the architectural decisions. Implementation is split across COs:

- **CO-078:** LLM-based query keyword extraction (replaces deterministic splitter in echo_recall/echo_context)
- **CO-079:** Extraction prompt tuning (mandatory keyword categories + few-shot examples)
- **CO-080:** Hub contested detection (score gap threshold, contested flag in response)
- **CO-081:** MCP disambiguation flow (local LLM comparison when contested, structured response format)
- **CO-082:** Moderation similarity detection (pre-approval similarity query, side-by-side display)
- **CO-083:** Adversarial benchmark re-run (full measurement after all changes)

Standing rules R-01 through R-14 apply to all COs. R-14 (benchmark before and after) applies to any CO that changes scoring behavior.