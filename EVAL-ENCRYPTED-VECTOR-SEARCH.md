# CO-267 Task E: Encrypted Vector Search Feasibility (IronCore Alloy Focus)

Date: 2026-05-19

## Executive Conclusion

Using IronCore Alloy for encrypted vector search is technically possible today for a Rust-based path and has a documented Qdrant example, but it is **not clearly feasible for this codebase as a production default** yet because:

1. There is no Alloy Go SDK (Rust/Java/Kotlin/Python only), while `wevibe-hub` is Go-centric.
2. Alloy is AGPL-3.0-only by default (commercial license required for most closed/proprietary deployment patterns).
3. Recall impact is workload-dependent and can exceed the `<5%` target on published examples.
4. The key-management model is SaaS Shield/standalone secrets, not directly integrated with Echo epoch-key lifecycle.

Recommendation: **doc-only for CO-267 Task E** (no integration stub), and defer implementation until licensing and key-management alignment are resolved.

## Requirement-by-Requirement Assessment

### 1) Property-preserving encrypted ANN

**What Alloy provides**
- Cloaked AI explicitly describes property-preserving vector encryption that preserves distance relationships for nearest-neighbor search.
- Algorithm basis is documented (distance-comparison-preserving symmetric encryption family) and exposed through Alloy vector APIs.

**Gaps / risks**
- This is not semantic security equivalent to standard encryption; it intentionally preserves geometric structure.
- Security/accuracy trade-off must be tuned by approximation factor and embedding distribution.

**Alternatives**
- Microsoft SEAL (CKKS/BFV/BGV): supports homomorphic arithmetic but not a drop-in ANN index flow; high complexity and overhead.
- TFHE-rs: strong encrypted computation model, but practical ANN retrieval stacks are not turnkey and overhead is typically much higher.
- Noise-based/perturbation and LSH-style methods: pragmatic for approximate privacy, but security guarantees are weaker and highly model-dependent.

**Recommendation**
- Alloy is currently the most practical option among reviewed candidates for "encrypted embeddings + ANN-like retrieval" in standard vector DB pipelines.

### 2) Recall degradation target (<5%)

**What Alloy provides**
- IronCore publishes benchmark-style relevance impacts (precision/recall/NDCG/MAP) and shows some cases near the target.

**Gaps / risks**
- Published examples include recall loss above 5% (for example 6%, 8%, 13% depending on model/dataset).
- Therefore `<5%` cannot be treated as guaranteed; it is model/data/parameter dependent.

**Alternatives**
- SEAL/TFHE paths can in theory preserve correctness differently, but end-to-end ANN practical throughput/latency usually degrades far more.
- Noise-only approaches may hit recall targets with careful tuning, but often weaken privacy guarantees.

**Recommendation**
- Require workload-specific evaluation before any production commitment; do not assume `<5%` globally from vendor docs.

### 3) Latency overhead target (<2x)

**What Alloy provides**
- Rust benchmarks indicate low microsecond-range vector encrypt/roundtrip costs in standalone mode and higher but still sub-millisecond TSP-backed operations in provided examples.

**Gaps / risks**
- Vendor benchmark numbers are not end-to-end Echo query path measurements.
- Qdrant query latency impact depends on extra pre-query encryption, network/TSP hops, and metadata decrypt path.
- No explicit vendor claim that complete retrieval pipeline overhead is always `<2x`.

**Alternatives**
- SEAL/TFHE approaches are usually worse for latency at ANN serving scale.
- Approximate privacy transforms can be faster but generally provide weaker protections.

**Recommendation**
- Treat `<2x` as unproven until measured in Echo-like deployment conditions.

### 4) Qdrant compatibility

**What Alloy provides**
- IronCore docs include Qdrant under supported integration examples.
- The `ironcore-alloy` repository includes a Rust Qdrant integration example demonstrating encrypted vector insert/search and encrypted payload handling.

**Gaps / risks**
- Integration is application-side encryption; Qdrant itself is not doing homomorphic search over ciphertext semantics.
- No first-party Go Alloy SDK, so direct integration in Go services is a blocker without Rust sidecar/FFI design.

**Alternatives**
- Build a Rust sidecar for encryption/query-vector generation and keep Go service as orchestrator.
- Use non-Alloy obfuscation layer (weaker guarantees) directly in Go.

**Recommendation**
- Qdrant compatibility is **yes** at the workflow level, but language and licensing constraints block immediate adoption in this repo.

### 5) Key management integration with epoch keys

**What Alloy provides**
- Alloy supports SaaS Shield (Tenant Security Proxy) and standalone secret management, including key rotation support.

**Gaps / risks**
- No documented native integration with Echo epoch-key semantics.
- Would require a mapping/bridge layer between epoch key derivation/rotation policy and Alloy secret paths/tenant metadata.

**Alternatives**
- Continue current epoch-key handling and evaluate whether Alloy standalone secrets can be deterministically derived from epoch material.
- Use an internal KMS abstraction and wrap Alloy usage behind a dedicated key-provider adapter.

**Recommendation**
- Do not proceed without explicit key-lifecycle design proving epoch rotation, revocation, and audit requirements are preserved.

### 6) Licensing compatibility

**What Alloy provides**
- Public SDK is AGPL-3.0-only; vendor also offers commercial licenses.

**Gaps / risks**
- AGPL is generally incompatible with many proprietary/internal service distribution models unless full obligations are accepted.
- This is a hard gate for adoption unless legal approves AGPL usage or a commercial license is procured.

**Alternatives**
- Microsoft SEAL is MIT licensed (more permissive), but not operationally equivalent for drop-in encrypted ANN.
- TFHE-rs uses BSD-3-Clause-Clear with additional patent/commercial terms from vendor FAQ; also requires legal review.

**Recommendation**
- Resolve legal path first (commercial Alloy license or approved alternative) before any code integration.

## Feasibility Decision for CO-267 Task E

Decision: **DOC-ONLY** (no integration stub added).

Rationale:
- Not clearly feasible under current constraints due to Go SDK gap + licensing gate + unproven `<5%` recall and `<2x` latency under Echo-specific conditions + missing epoch-key integration design.

## Source Links

IronCore / Alloy / Cloaked AI
- https://github.com/IronCoreLabs/ironcore-alloy
- https://raw.githubusercontent.com/IronCoreLabs/ironcore-alloy/main/README.md
- https://ironcorelabs.com/docs/ironcore-alloy/
- https://ironcorelabs.com/docs/ironcore-alloy/rust/
- https://crates.io/crates/ironcore-alloy
- https://docs.rs/ironcore-alloy/latest/ironcore_alloy/
- https://ironcorelabs.com/docs/cloaked-ai/how-it-works
- https://ironcorelabs.com/docs/cloaked-ai/search-accuracy/
- https://ironcorelabs.com/docs/cloaked-ai/integrations/
- https://raw.githubusercontent.com/IronCoreLabs/ironcore-alloy/main/example-integrations/qdrant/README.md
- https://raw.githubusercontent.com/IronCoreLabs/ironcore-alloy/main/example-integrations/qdrant/src/main.rs
- https://raw.githubusercontent.com/IronCoreLabs/ironcore-alloy/main/Cargo.toml
- https://raw.githubusercontent.com/IronCoreLabs/ironcore-alloy/main/benches/README.md

Qdrant / language package checks
- https://qdrant.tech/documentation/security/
- https://pkg.go.dev/search?q=ironcore+alloy
- https://proxy.golang.org/ironcorelabs.com/alloy/@v/list

Alternatives
- https://github.com/microsoft/SEAL
- https://github.com/zama-ai/tfhe-rs
- https://en.wikipedia.org/wiki/Locality-sensitive_hashing
