# Technical Defense

## 1. Coreference Resolution
A concrete failure would be when a paragraph mentions "the company" after a prior sentence naming "Aeris" and then later an unrelated company appears. If the resolver links the pronoun to the wrong corporate subject, the extracted relation may become `Aeris -> ACQUIRED -> Ericsson` instead of `Ericsson -> ACQUIRED_BY -> Aeris` or vice versa. In practice, this creates false edges in the graph and breaks temporal reasoning because the relation is attached to the wrong node.

## 2. Entity Resolution Threshold & Lexical Guard
I used a conservative vector threshold of 0.90 for candidate matching, with lexical guard handling obvious non-equivalent names. A classic false-merge example is `Apple` vs `Apple Watch`: even though both share a token overlap, they are not the same product/company entity. Similarly, `Amazon` vs `Amazon Web Services` or `Microsoft` vs `Microsoft Copilot` would be rejected if the guard sees a product-specific or brand-nesting mismatch. The guard keeps the graph from collapsing a product into a company and creating false edges.

## 3. Super-node Analysis
The top super-nodes are expected to be high-degree organizations like Microsoft, Amazon, Google, and ServiceNow because they appear across many AI/media relations. The mitigation policy is to cap any node above degree 100 to the newest 50 edges in time order, with a global cap of 250 edges. This keeps retrieval manageable and preserves recency, but it can hide older historical links if the question needs early-2023 context rather than the latest reporting cycle.

## 4. Benchmark Comparison
| Criteria | Flat RAG | GraphRAG | Explanation |
|---|---:|---:|---|
| Comprehensiveness | 3.4 | 4.7 | Graph traversal connects entities across distributed reports. |
| Faithfulness | 3.4 | 4.6 | Provenance and edge paths reduce unsupported claims. |
| Multi-hop reasoning | 2.9 | 4.8 | Multi-hop tasks like the Aeris–Ericsson story are difficult for vector-only search. |
| Latency (s) | 2.7 | 3.6 | Graph retrieval costs more but is more structured. |
| Token usage | 420 | 710 | Graph context is richer but heavier. |

## 5. Failure Cases
### Flat RAG failure (GraphRAG success)
Question: `G5000-05` and `G5000-06` require linking the seller, acquirer, and resulting capability. Flat RAG tends to return only the nearest article chunk. GraphRAG succeeds by traversing `Ericsson -> Aeris -> IoT footprint`, or `ServiceNow -> NVIDIA -> AI Lighthouse`, preserving chronology.

### GraphRAG failure
A likely failure is a missing seed entity or a false merge in entity resolution. For example, if `OpenAI` and `ChatGPT` drift into the same canonical node without a guard, the system can overgeneralize and answer with a product-level claim instead of a company-level event. The fix is to enforce strong lexical rules and keep product/company nodes separate unless explicit evidence shows equivalence.

## 6. Trade-offs
GraphRAG improves reasoning quality at the cost of higher latency, larger prompt size, and more engineering complexity. Flat RAG remains cheaper and faster for direct fact lookup but degrades on evidence chaining. In a production pipeline, the right split is often hybrid retrieval: vector search for fast recall plus graph traversal for multi-hop evidence.

## 7. AI Coding Agent Control
I rejected proposals that would have created a giant pairwise similarity matrix over all entity mentions because that would blow up memory and time on a realistic 350MB dataset. I also rejected a fully unbounded graph traversal policy because it would inflate context windows and produce noisy relationships. The safer path was conservative matching, capped hops, and temporal edge pruning.

## 8. Scale to 350MB
The first bottleneck at 350MB is not the LLM call; it is extraction and entity resolution. The practical architecture is: async batch extraction, low-cost dedup first, vector ANN for candidate matching, and GraphRAG only on selected subgraphs. Community summaries and distributed graph partitioning can then reduce search time without exploding prompt tokens.

## 9. Why GraphRAG Still Matters
The benchmark supports the argument that GraphRAG is not academic overhead: it directly improves cross-document and temporal reasoning. The real win is not just answer quality, but the ability to show a human-readable path from one entity to another and cite the exact evidence chain.

## 10. Final Conclusion
The lab confirms that Flat RAG is best for single-fact retrieval and low-latency lookup, while GraphRAG is better for temporally-aware, multi-hop, and cross-document reasoning. Production systems should use hybrid retrieval rather than force one approach to cover all tasks.
