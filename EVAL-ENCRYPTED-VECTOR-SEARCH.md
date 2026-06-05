> **HISTORICAL DECISION RECORD — CO-267 Task E.** AlloyDB Encrypted Vector Search was evaluated and REJECTED (Go SDK gap, AGPL licensing, unproven perf, no epoch-key integration). The current path is Qdrant + the WeVibe epoch-key hierarchy. Retained for historical reference; not a live spec.

# Evaluation: Encrypted Vector Search (CO-267 Task E)

## Summary

AlloyDB Encrypted Vector Search is a potential option for secure vector storage. Task E required a doc-only evaluation of Alloy’s suitability for WeVibe Network’s encrypted retrieval path.

## Findings

### 1) Language support

- AlloyDB client libraries exist for Rust, Java, Kotlin, Python. There is no Go SDK.
- wevibe-hub is Go-centric (Go API server). Introducing a Rust sidecar purely for Alloy integration defeats the purpose of simplifying infra.

**Conclusion:** Lack of Go SDK is a blocker.

### 2) Licensing

- AlloyDB Encrypted Vector Search ships under AGPL-3.0 unless a commercial license is purchased.
- WeVibe Network is Apache-2.0. Integrating AGPL code into the hub (or any process linked at runtime) forces the entire codebase under AGPL.
- SaaS Shield (their managed service) avoids AGPL, but puts critical retrieval components on a vendor-managed path.

**Conclusion:** Licensing incompatible without commercial agreement.

### 3) Performance claims

- Marketing claims: `<5%` recall loss vs. plaintext, `<2x` latency vs. plaintext (vector search).
- Independent verification requires a WeVibe-specific benchmark. No published data for WeVibe workloads.
- Rust benchmarks indicate low microsecond-range vector encrypt/roundtrip costs in standalone mode and higher but still sub-millisecond TSP-backed operations in provided examples.

**Gaps / risks**
- Vendor benchmark numbers are not end-to-end WeVibe query path measurements.
- Qdrant query latency impact depends on extra pre-query encryption, network/TSP hops, and metadata decrypt path.
- No explicit vendor claim that complete retrieval pipeline overhead is always `<2x`.

**Recommendation**
- Treat `<2x` as unproven until measured in WeVibe-like deployment conditions.

### 4) Qdrant compatibility

- Alloy encrypts vectors before storage. Qdrant expect plain float arrays.
- No documented turnkey integration path. Would require custom bridging layers.

**Conclusion:** Additional engineering required; not drop-in.

### 5) Key management

- Alloy supports SaaS Shield (Tenant Security Proxy) and standalone secret management, including key rotation support.

**Gaps / risks**
- No documented native integration with WeVibe epoch-key semantics.
- Would require a mapping/bridge layer between epoch key derivation/rotation policy and Alloy secret paths/tenant metadata.

**Alternatives**
- Use existing WeVibe epoch key hierarchy with local vector storage (current approach).
- If Alloy is ever adopted, the TSP model (SaaS Shield) is more compatible with existing zero-trust assumptions than client libraries (due to licensing + SDK gap).

## Recommendation

Decision: **DOC-ONLY** (no integration stub added).

Rationale:
- Not clearly feasible under current constraints due to Go SDK gap + licensing gate + unproven `<5%` recall and `<2x` latency under WeVibe-specific conditions + missing epoch-key integration design.

## Source Links

- AlloyDB Encrypted Vector Search announcement — https://cloud.google.com/blog/products/databases/introducing-alloydb-encrypted-vector-search
- SaaS Shield reference — https://cloud.google.com/alloydb/docs/vector-search/tenant-security-proxy
- Licensing FAQ — https://cloud.google.com/alloydb/docs/vector-search/license
