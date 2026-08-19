# Failure Analysis

## Case 1: Flat RAG fails on multi-hop evidence linking
Question: `G5000-05` (Aeris–Ericsson transaction) and `G5000-06` (ServiceNow + NVIDIA + Accenture timeline).

Root cause:
- The relevant facts are spread across multiple articles and dates.
- Flat RAG ranks chunks by semantic similarity, but each article carries only a fragment of the situation.
- Vector retrieval has no explicit notion of the event chain connecting seller, acquirer, and resulting footprint.

Result:
- The model sometimes returns an incomplete answer: it names one company but omits the scale or the series of business transfers.

Mitigation:
- Graph traversal with seed entities and BFS preserves the chain `Ericsson -> IoT business -> Aeris -> 100M devices`.
- Temporal filtering ensures the modern evidence is prioritized without losing earlier event state.

## Case 2: GraphRAG fails from false merge / noisy edges
A failure mode appears when entity resolution is too aggressive. If `Amazon` and `Amazon AI` or `Microsoft` and `Microsoft Copilot` are collapsed together without lexical guard, the graph will produce a broad but wrong relationship neighborhood. The model may then answer a question about a product partnership using company-level facts that are not actually connected.

Root cause:
- vector similarity is high on token overlap, but the semantic roles differ.
- The system treats product entities and company entities as the same canonical node.

Mitigation:
- Use a lexical guard that strips suffixes and compares normalized names before merging.
- Reject suspicious merges and add an audit table documenting the reason (`MERGE_MANUAL`, `MERGE_VECTOR`, `REJECT_GUARD`).

## Case 3: Super-node explosion
If a high-degree node like Microsoft or Google is included in a query without a cap, the graph expands too widely and produces massive context windows.

Root cause:
- The graph contains dense clusters of relationships for large technology vendors.
- Unbounded BFS can exceed the prompt token budget.

Mitigation:
- Cap each node at a maximum of 50 newest edges and enforce a global edge cap of 250.
- Keep the query to the relevant subgraph and trim historical noise while preserving temporal relevance.

## Overall lesson
The core defect is not merely retrieval but reasoning under incomplete provenance. Both systems fail when context is missing, overly noisy, or semantically merged incorrectly. The fix is to protect provenance, constrain growth, and maintain a clear entity identity policy.
