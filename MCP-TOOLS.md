# MCP Tool Reference

WeVibe Network exposes five tools via the Model Context Protocol.

## `wevibe_context`

Register the current task and surface relevant memories.

**Parameters:**
| Name | Type | Required | Description |
|---|---|---|---|
| `task` | string | yes | Current task description |
| `stack` | string[] | no | Technology stack hints (e.g. `["go", "postgres"]`) |

**Returns:** Relevant memories (up to 3) and detected stack.

**When to call:** At the start of a coding session or when switching tasks.

---

## `wevibe_recall`

Explicit memory search.

**Parameters:**
| Name | Type | Required | Description |
|---|---|---|---|
| `query` | string | yes | Search query |
| `limit` | number | no | Max results (default: 5) |
| `org_id` | string | no | Specific org (defaults to first membership) |

**Returns:** Matching memories with epoch, freshness score, and CID.

---

## `wevibe_contribute`

Submit encrypted session notes for moderation.

**Parameters:**
| Name | Type | Required | Description |
|---|---|---|---|
| `raw_notes` | string | yes | Session notes (min 50 chars) |

**Returns:** Submission hash and pending status, or rejection reason if scanner blocked it.

**Security:** Notes are scanned by wevibe-guard before encryption. Credentials, API keys,
and prompt injection patterns cause immediate rejection — nothing is sent to wevibe-chain or any optional analytics surface.

---

## `wevibe_reject`

Flag a returned memory as not useful.

**Parameters:**
| Name | Type | Required | Description |
|---|---|---|---|
| `pack_id` | string | yes | Memory CID to reject |
| `reason` | string | no | Rejection reason |

**Effect:** Memory is added to your local blacklist. If you mirror events into an analytics surface it may increment rejection metrics, but wevibe-chain state only changes when you submit a moderation transaction. Memories with high rejection counts can be quarantined by org leaders.

---

## `wevibe_orgs`

List current org memberships.

**Parameters:** None.

**Returns:** Org IDs, roles, epochs, and egress modes.
