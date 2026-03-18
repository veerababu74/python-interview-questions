# 🧠 RAG (Retrieval-Augmented Generation) — Interview Questions & Answers

> Comprehensive interview preparation covering architecture, chunking, embeddings, hybrid search, re-ranking, context optimization, hallucination reduction, and evaluation metrics.

---

## Q1 — RAG Architecture & Pipeline Design 🟢 Easy

### 🔹 Conceptual Question

**Q: Explain the end-to-end architecture of a RAG system. Why is RAG preferred over fine-tuning for knowledge-intensive tasks?**

### 🔹 Answer

RAG combines a **retrieval** component with a **generative** LLM. At query time, relevant documents are fetched from an external knowledge base and injected into the LLM's prompt as context, enabling factual, up-to-date responses without retraining.

**Why RAG over fine-tuning:**

| Dimension | RAG | Fine-Tuning |
|-----------|-----|-------------|
| Data freshness | Real-time updates by updating the index | Requires retraining |
| Cost | Low — no GPU training | High — needs compute for training |
| Hallucination control | Grounded in retrieved docs | Model may still hallucinate |
| Transparency | Can cite sources | Black-box answers |
| Domain adaptation | Swap the knowledge base | Requires domain-specific data + training |

**Trade-offs:** RAG adds retrieval latency and depends on retrieval quality. If retrieval fails, the LLM generates without grounding. Fine-tuning bakes knowledge into weights, which can be faster at inference but stale.

### 🔹 Example

An enterprise legal assistant uses RAG to answer questions about company policies. The knowledge base (vector store) is updated nightly with new policy documents. Instead of fine-tuning GPT-4 on policies (expensive, stale quickly), the system retrieves the top-5 relevant policy chunks and passes them as context to the LLM.

### 🔹 Visual Explanation (Text-Based)

```
User Query
    │
    ▼
┌─────────────┐     ┌──────────────────┐
│  Embedding   │────▶│  Vector Database  │
│   Model      │     │  (FAISS/Pinecone) │
└─────────────┘     └──────┬───────────┘
                           │ Top-K docs
                           ▼
                    ┌──────────────┐
                    │ Prompt Builder│
                    │ (Query+Context)│
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │     LLM      │
                    │ (GPT-4/Claude)│
                    └──────┬───────┘
                           │
                           ▼
                      Response + Citations
```

---

### 🔹 Scenario-Based Question

**Q: Your RAG system is returning outdated information even though you've updated the knowledge base. Users report seeing answers referencing old product pricing. How do you debug and fix this?**

### 🔹 Scenario Answer

**Step-by-step debugging:**

1. **Check ingestion pipeline:** Verify the updated documents were actually re-embedded and upserted into the vector DB. Common issue: the ingestion job failed silently.
2. **Check chunk overlap/duplication:** Old chunks may still exist alongside new ones. If the old chunk has a higher similarity score, it gets retrieved.
3. **Verify embedding timestamps:** Add metadata (e.g., `last_updated`) to each chunk. Filter retrieval to prioritize recent documents.
4. **Inspect retrieval results:** Log the retrieved chunks before they reach the LLM. If old chunks appear, the issue is in retrieval, not generation.
5. **Cache invalidation:** If you're caching embeddings or responses, stale cache entries may serve old answers.

**Fix:**
- Implement **versioned ingestion**: delete old chunks for updated documents before inserting new ones.
- Add **metadata filtering** (`WHERE updated_at > X`) in vector DB queries.
- Add **TTL (time-to-live)** on cached responses.

### 🔹 Scenario Example

A SaaS company updates pricing quarterly. After a price change, the ingestion pipeline re-embedded the new pricing page but didn't delete the old chunks. The vector DB now has two versions of the pricing page. The old chunk (embedded with a previous model version) has a slightly different vector, and depending on the query, sometimes the old chunk ranks higher. The fix: use document-level deduplication keyed on `doc_id` — upsert replaces old chunks with the same key.

### 🔹 Visual Explanation (Scenario)

```
Ingestion Pipeline (BEFORE fix):
  New Doc ──▶ Embed ──▶ INSERT into VectorDB
  (Old chunks remain — DUPLICATES!)

Ingestion Pipeline (AFTER fix):
  New Doc ──▶ DELETE WHERE doc_id = X
          ──▶ Embed ──▶ UPSERT into VectorDB
          ──▶ Invalidate response cache for doc_id = X
```

---

## Q2 — Chunking Strategies 🟡 Medium

### 🔹 Conceptual Question

**Q: Compare fixed-size chunking, semantic chunking, and recursive chunking. When would you choose each, and what are the failure modes?**

### 🔹 Answer

| Strategy | How It Works | Best For | Failure Mode |
|----------|-------------|----------|-------------|
| **Fixed-size** | Split by character/token count (e.g., 512 tokens) with overlap | Simple docs, uniform structure | Splits mid-sentence, breaks context |
| **Semantic** | Split at natural boundaries (paragraphs, topic shifts) using embeddings | Long-form content, articles | Uneven chunk sizes, some too large for context window |
| **Recursive** | Hierarchically split: first by `\n\n`, then `\n`, then sentence, then word | Code, mixed-format docs | Complex implementation, can still produce odd splits |

**Key considerations:**
- **Chunk size vs. retrieval precision:** Smaller chunks = more precise retrieval but lose context. Larger chunks = more context but dilute relevance signal.
- **Overlap:** 10-20% overlap between chunks prevents context loss at boundaries.
- **Metadata enrichment:** Attach section headers, page numbers, and parent document info to each chunk for better retrieval and re-ranking.

**Trade-off:** Semantic chunking produces the most meaningful units but is slower (requires embedding computation at ingestion) and harder to implement. Fixed-size is fast and simple but brittle.

### 🔹 Example

A healthcare company processes medical research papers. Fixed-size chunking at 500 tokens sometimes splits a drug interaction table across two chunks, making both useless. They switch to semantic chunking: using sentence-transformers to detect topic boundaries. Tables and lists are kept as single chunks. This improves retrieval accuracy by 23% on their evaluation set.

### 🔹 Visual Explanation (Text-Based)

```
Original Document:
┌──────────────────────────────────────────────┐
│ Section 1: Introduction (300 tokens)          │
│ Section 2: Methods (800 tokens)               │
│ Section 3: Results Table (200 tokens)         │
│ Section 4: Discussion (600 tokens)            │
└──────────────────────────────────────────────┘

Fixed-Size (512 tokens):
  [Chunk1: Intro + half of Methods]
  [Chunk2: rest of Methods + Results]  ← Table split!
  [Chunk3: Discussion]

Semantic Chunking:
  [Chunk1: Introduction]
  [Chunk2: Methods]
  [Chunk3: Results Table]  ← Preserved!
  [Chunk4: Discussion]

Recursive Chunking:
  Split by \n\n → [Section1] [Section2] [Section3] [Section4]
  If Section2 > max_size → split by \n → sub-chunks
```

---

### 🔹 Scenario-Based Question

**Q: You've deployed a RAG system for a code documentation assistant. Users complain that answers about function signatures are incomplete — the system returns partial function definitions. How do you diagnose and fix this?**

### 🔹 Scenario Answer

1. **Root cause:** Fixed-size chunking splits code blocks mid-function. A function spanning 80 lines gets split into two chunks, and only one is retrieved.
2. **Diagnosis:** Log retrieved chunks for failing queries. Confirm chunks contain partial code blocks.
3. **Fix — Use code-aware chunking:**
   - Parse the AST (Abstract Syntax Tree) to identify function/class boundaries.
   - Each function becomes one chunk (with its docstring).
   - For very long functions, split at logical sub-blocks (e.g., by method within a class).
4. **Add parent-child chunking:** Store both the full file and individual function chunks. When a function chunk is retrieved, also include the file-level summary for context.
5. **Overlap with header context:** Prepend each chunk with the file path, class name, and import statements.

### 🔹 Scenario Example

```python
# BEFORE: Fixed chunking splits this function
# Chunk 1: lines 1-50 (function header + first half)
# Chunk 2: lines 51-100 (second half — no function name!)

# AFTER: AST-aware chunking
# Chunk: entire function as one unit
# Metadata: {"file": "auth.py", "class": "AuthService", "function": "validate_token"}
```

### 🔹 Visual Explanation (Scenario)

```
Code-Aware Chunking Pipeline:
  Source File ──▶ AST Parser ──▶ Extract Functions/Classes
       │                              │
       ▼                              ▼
  File-level summary chunk     Function-level chunks
  (imports, class hierarchy)   (complete function + docstring)
       │                              │
       └──────────┬───────────────────┘
                  ▼
           Vector Database
           (with metadata: file, class, function, line_range)
```

---

## Q3 — Hybrid Search (BM25 + Vector) 🟡 Medium

### 🔹 Conceptual Question

**Q: Why would you use hybrid search (BM25 + vector similarity) instead of pure vector search in a RAG system? How do you combine the scores?**

### 🔹 Answer

**Why hybrid search:**
- **Vector search** excels at semantic similarity (finding paraphrases, conceptual matches) but struggles with exact keyword matching (product IDs, error codes, acronyms).
- **BM25 (lexical search)** excels at exact term matching but misses semantic relationships ("automobile" vs "car").
- **Hybrid** combines both, covering each other's weaknesses.

**Score combination methods:**

1. **Reciprocal Rank Fusion (RRF):** `score = Σ 1/(k + rank_i)` where `k` is a constant (typically 60). Simple, no score normalization needed.
2. **Weighted linear combination:** `final = α * vector_score + (1-α) * bm25_score`. Requires score normalization (min-max or z-score). `α` is tuned on evaluation data.
3. **Re-ranking:** Retrieve top-N from each method, merge, then re-rank with a cross-encoder.

**Trade-offs:**
- RRF is simple and robust but ignores score magnitude.
- Linear combination requires careful tuning of `α` and score normalization.
- Re-ranking is most accurate but adds latency (cross-encoder inference).

### 🔹 Example

An e-commerce product search system. Query: "Nike Air Max 270 size 10". Pure vector search returns semantically similar shoes (Adidas running shoes) but misses the exact product. Pure BM25 finds "Nike Air Max 270" but ranks poorly for "comfortable running shoe" queries. Hybrid search with RRF handles both cases, improving nDCG@10 by 15%.

### 🔹 Visual Explanation (Text-Based)

```
User Query: "error code NX-4012 authentication failure"
         │
    ┌────┴─────┐
    ▼          ▼
┌────────┐ ┌─────────┐
│  BM25  │ │ Vector  │
│ Search │ │ Search  │
└───┬────┘ └────┬────┘
    │            │
    │ Exact:     │ Semantic:
    │ "NX-4012"  │ "auth failure"
    │ ranked #1  │ context matches
    │            │
    └─────┬──────┘
          ▼
   ┌──────────────┐
   │ RRF / Fusion │
   │ Score Merge  │
   └──────┬───────┘
          ▼
    Top-K Results
```

---

### 🔹 Scenario-Based Question

**Q: Your hybrid search RAG system works well for English queries, but after expanding to support Japanese and Korean, retrieval quality drops significantly. What's happening, and how do you fix it?**

### 🔹 Scenario Answer

1. **Diagnosis:** BM25 relies on tokenization. Standard whitespace tokenization fails for CJK languages (Japanese, Korean, Chinese) because words aren't space-separated.
2. **Root cause breakdown:**
   - BM25 component: poor tokenization → bad term matching
   - Vector component: if the embedding model isn't multilingual, semantic matching degrades
3. **Fixes:**
   - **BM25:** Use language-specific tokenizers (MeCab for Japanese, Komoran for Korean). Elasticsearch has built-in CJK analyzers.
   - **Embeddings:** Switch to a multilingual embedding model (e.g., `multilingual-e5-large`, `cohere-multilingual-v3`).
   - **Fusion weight adjustment:** The optimal `α` between BM25 and vector may differ per language. Tune separately.
   - **Query language detection:** Route queries to language-specific BM25 indexes.

### 🔹 Scenario Example

Japanese query: "認証エラー NX-4012" (Authentication error NX-4012). Without MeCab, BM25 treats the entire Japanese string as one token, matching nothing. With MeCab: ["認証", "エラー", "NX-4012"] → BM25 now matches the error code. The multilingual embedding model captures the semantic meaning of "authentication error" across languages.

### 🔹 Visual Explanation (Scenario)

```
Multi-Language Hybrid Search:

Query ──▶ Language Detector ──▶ Route to:
                                    │
                    ┌───────────────┼────────────────┐
                    ▼               ▼                ▼
              EN Pipeline     JA Pipeline      KO Pipeline
              │                │                │
              ├─ BM25(EN)      ├─ BM25(MeCab)   ├─ BM25(Komoran)
              ├─ Vector(EN)    ├─ Vector(multi)  ├─ Vector(multi)
              └─ RRF(α=0.6)   └─ RRF(α=0.4)    └─ RRF(α=0.5)
                    │               │                │
                    └───────────────┼────────────────┘
                                    ▼
                              Merged Results
```

---

## Q4 — Re-Ranking 🟡 Medium

### 🔹 Conceptual Question

**Q: What is re-ranking in RAG, and why is a two-stage retrieval approach (retrieve then re-rank) more effective than single-stage retrieval?**

### 🔹 Answer

**Re-ranking** is a second-stage scoring pass where a more powerful model (typically a cross-encoder) re-scores the candidate documents retrieved in the first stage.

**Why two stages:**
- **Stage 1 (Retrieval):** Fast but approximate. Bi-encoders encode query and document independently — they can't model fine-grained query-document interactions. Searches millions of documents in milliseconds.
- **Stage 2 (Re-ranking):** Slow but precise. Cross-encoders process (query, document) pairs jointly, capturing token-level interactions. Only runs on top-50 to top-100 candidates.

**Key insight:** You can't run a cross-encoder over millions of documents (O(N) full inference). The two-stage approach gives you cross-encoder quality at bi-encoder speed.

**Popular re-rankers:** Cohere Rerank, `cross-encoder/ms-marco-MiniLM-L-6-v2`, Jina Reranker, ColBERT.

**Trade-offs:**
- Re-ranking adds 50-200ms latency per query
- Cross-encoders are more accurate but don't scale to large candidate sets
- ColBERT offers a middle ground (late interaction) but requires more storage

### 🔹 Example

A customer support RAG system retrieves top-50 documents using FAISS (fast vector search). Without re-ranking, the most relevant article ranks #7. After applying Cohere Rerank, it moves to #1. This improves answer accuracy from 72% to 89% on their test set, at a cost of ~120ms additional latency.

### 🔹 Visual Explanation (Text-Based)

```
             Stage 1: Retrieval (Fast)           Stage 2: Re-Ranking (Precise)
             ─────────────────────────           ──────────────────────────────
Query ──▶ Bi-Encoder ──▶ ANN Search ──▶ Top-50 ──▶ Cross-Encoder ──▶ Top-5
          (encode once)   (milliseconds)   docs     (score each pair)   docs
                                                    (+100-200ms)

Bi-Encoder:                    Cross-Encoder:
  Q ──▶ [vec_q]                  (Q, D) ──▶ [joint encoding] ──▶ score
  D ──▶ [vec_d]                  Sees token-level interactions
  score = cosine(vec_q, vec_d)   More accurate, but O(N) per query
```

---

### 🔹 Scenario-Based Question

**Q: After adding a cross-encoder re-ranker to your RAG pipeline, P95 latency increased from 800ms to 2.5s. Your SLA requires < 1.5s. How do you optimize the re-ranking stage without significantly sacrificing quality?**

### 🔹 Scenario Answer

1. **Reduce candidate set:** Drop from top-50 to top-20 re-ranking candidates. Measure quality impact — often top-20 captures 95% of relevant documents.
2. **Use a smaller re-ranker:** Switch from a large cross-encoder to a distilled model (e.g., MiniLM-L-6 instead of L-12). ~2x speedup with ~3% quality drop.
3. **Batch and parallelize:** Process re-ranking in batches on GPU. If using CPU, switch to ONNX Runtime for 3-5x speedup.
4. **Async re-ranking with streaming:** Start streaming the LLM response using top-1 from retrieval while re-ranking runs in parallel. Update if re-ranking changes the top result.
5. **Cache frequent re-ranking results:** For common queries, cache the re-ranked order.
6. **ColBERT as alternative:** Use late-interaction models that pre-compute document token embeddings, making re-ranking much faster.

### 🔹 Scenario Example

Optimization results:
- Top-50 → Top-20 candidates: latency 2.5s → 1.6s, quality -2%
- ONNX Runtime quantized model: latency 1.6s → 1.1s, quality -1%
- Combined: latency 1.1s (under SLA), total quality impact: -3%

### 🔹 Visual Explanation (Scenario)

```
BEFORE (P95 = 2.5s):
  Retrieve(50) ──▶ CrossEncoder(50 pairs) ──▶ LLM ──▶ Response
    200ms              1800ms                 500ms

AFTER (P95 = 1.1s):
  Retrieve(20) ──▶ ONNX-CrossEncoder(20 pairs) ──▶ LLM ──▶ Response
    200ms              400ms                        500ms

ALTERNATIVE (Streaming):
  Retrieve(20) ──▶ Start LLM with Top-1 ──▶ Stream partial response
       │                                         │
       └──▶ Re-rank in parallel ──▶ If Top-1 changed → regenerate
```

---

## Q5 — Context Window Optimization 🔴 Hard

### 🔹 Conceptual Question

**Q: How do you optimize context window usage in a RAG system when you have more relevant documents than can fit in the LLM's context window? What are the key strategies and their trade-offs?**

### 🔹 Answer

**The core tension:** More context = better grounding, but LLMs have finite context windows and exhibit "lost in the middle" effects (information in the middle of context is less attended to).

**Strategies:**

1. **Document compression/summarization:** Use a smaller LLM to summarize each retrieved chunk before passing to the main LLM. Reduces tokens but may lose critical details.

2. **Relevant passage extraction:** Instead of full chunks, extract only the most relevant sentences using extractive methods (sentence-level re-ranking). Preserves exact wording.

3. **Map-Reduce over chunks:** Process each chunk independently ("map"), then synthesize answers ("reduce"). Handles unlimited chunks but increases latency and cost.

4. **Hierarchical retrieval:** Retrieve at multiple granularities — document summary first, then drill into specific sections. Efficient context usage.

5. **Context ordering:** Place the most relevant documents at the beginning and end of the context (primacy and recency bias). Less relevant ones in the middle.

6. **Iterative retrieval:** Start with top-3 chunks. If the LLM indicates insufficient context, retrieve more. Adaptive context sizing.

**Trade-offs:**
- Compression saves tokens but risks losing nuance
- Map-Reduce handles scale but 3-5x cost increase
- Hierarchical retrieval is elegant but complex to implement

### 🔹 Example

A financial analyst assistant needs to answer questions across 50-page quarterly reports. The context window holds ~15 pages. Solution: Hierarchical retrieval — first retrieve the relevant section (e.g., "Revenue Breakdown"), then extract the specific paragraphs about the queried metric. This fits in context while maintaining precision.

### 🔹 Visual Explanation (Text-Based)

```
Strategy: Hierarchical Retrieval + Compression

50-page document
    │
    ▼
┌──────────────────┐
│ Document-level    │
│ summary index     │──▶ Match query to section: "Revenue Q3"
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Section-level     │
│ chunk index       │──▶ Retrieve 5 relevant paragraphs
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ LLM Compressor   │──▶ Compress 5 paragraphs → key facts (30% of tokens)
└──────────────────┘
    │
    ▼
┌──────────────────┐
│ Main LLM         │──▶ Answer with compressed, relevant context
└──────────────────┘
```

---

### 🔹 Scenario-Based Question

**Q: Your RAG system for a medical knowledge base retrieves 10 relevant chunks per query, but you notice the LLM ignores important information in chunks 4-7 (the "lost in the middle" problem). How do you address this while maintaining answer quality?**

### 🔹 Scenario Answer

1. **Confirm the problem:** Run an evaluation where the ground-truth answer is only in chunk positions 4-7. Measure accuracy by position — expect a U-shaped curve (high at positions 1-2 and 9-10, low in the middle).

2. **Solutions by effectiveness:**
   - **Reorder chunks:** Place chunks in relevance-descending order alternating between beginning and end: [#1, #3, #5, #7, #9, #10, #8, #6, #4, #2]. The most important chunks are at the edges.
   - **Reduce to top-5:** Fewer chunks = less "middle" to lose. Use aggressive re-ranking to ensure top-5 are truly the best.
   - **Chunk fusion:** Merge highly relevant chunks into a single, coherent passage using a summarization step.
   - **Use a model with better long-context handling:** GPT-4-Turbo and Claude 3 have improved middle-attention. Alternatively, use models trained with position-interpolated attention.
   - **Explicit instruction:** Add to the prompt: "Important: carefully consider ALL provided context sections, especially sections 4-7, before answering."

3. **Measure impact:** Re-run the position-based evaluation after each fix.

### 🔹 Scenario Example

Before fix: accuracy when answer is in position 1 = 94%, position 5 = 61%, position 10 = 88%.
After interleaved reordering: position 1 = 93%, position 5 = 82%, position 10 = 87%.
After reducing to top-5 with re-ranking: all positions > 88%.

### 🔹 Visual Explanation (Scenario)

```
"Lost in the Middle" Attention Pattern:

Position:   1    2    3    4    5    6    7    8    9   10
Attention: ███  ██   █    ▪    ▪    ▪    ▪    █   ██  ███

Fix — Interleaved Ordering:
  Original rank: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  Reordered:     [1, 3, 5, 7, 9, 10, 8, 6, 4, 2]
                  ▲ Most relevant at edges ▲

Position:   1    2    3    4    5    6    7    8    9   10
Content:   R1   R3   R5   R7   R9  R10   R8   R6   R4   R2
Attention: ███  ██   █    ▪    ▪    ▪    ▪    █   ██  ███
           ▲High attention on R1,R3        High attention on R4,R2▲
```

---

## Q6 — Hallucination Reduction 🔴 Hard

### 🔹 Conceptual Question

**Q: What techniques can you use to reduce hallucinations in a RAG system? How do you measure whether your RAG system is hallucinating?**

### 🔹 Answer

**Hallucination types in RAG:**
1. **Intrinsic:** LLM contradicts the retrieved context
2. **Extrinsic:** LLM adds information not present in retrieved context
3. **Retrieval failure:** No relevant docs retrieved, LLM answers from parametric memory

**Reduction techniques:**

1. **Faithful prompting:** Instruct the LLM: "Answer ONLY based on the provided context. If the context doesn't contain the answer, say 'I don't know.'"
2. **Citation enforcement:** Require the LLM to cite specific chunks: "For each claim, reference [Source X]." Verify citations programmatically.
3. **Self-consistency checking:** Generate multiple answers (temperature > 0), check agreement. Disagreement signals potential hallucination.
4. **Retrieval quality gates:** If the top retrieval score is below a threshold, refuse to answer rather than hallucinate.
5. **Chain-of-verification:** LLM generates an answer, then generates verification questions, retrieves evidence for those questions, and revises the answer.
6. **Constrained decoding:** Limit output tokens to vocabulary present in retrieved context (aggressive but effective for extractive tasks).

**Measurement metrics:**
- **Faithfulness (RAGAS):** Does the answer align with retrieved context?
- **Answer relevance:** Does the answer address the query?
- **Context relevance:** Are the retrieved docs relevant?
- **Groundedness (LLM-as-judge):** Use a second LLM to verify each claim against context.

### 🔹 Example

A pharmaceutical company's drug information assistant uses RAG. A user asks about drug interactions. The system retrieves the correct drug monograph but the LLM adds a dosage recommendation not present in the context (extrinsic hallucination). Fix: Implement citation enforcement + retrieval score gating (refuse to answer if cosine similarity < 0.75).

### 🔹 Visual Explanation (Text-Based)

```
Hallucination Detection Pipeline:

User Query ──▶ Retrieve Docs ──▶ Generate Answer
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ Claim Extractor  │
                              │ (split into      │
                              │  atomic claims)  │
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │ For each claim:  │
                              │ Is it supported  │
                              │ by context?      │
                              │ (NLI model or    │
                              │  LLM-as-judge)   │
                              └────────┬────────┘
                                       │
                        ┌──────────────┼──────────────┐
                        ▼              ▼              ▼
                   Supported     Not Supported    Not Verifiable
                   (keep)        (flag/remove)    (flag/warn)
```

---

### 🔹 Scenario-Based Question

**Q: Your RAG-based customer support bot is hallucinating product features that don't exist — telling customers about features your product doesn't have, leading to customer complaints. You need to fix this within a week. What's your action plan?**

### 🔹 Scenario Answer

**Immediate (Day 1-2):**
1. Add a retrieval confidence threshold. If no chunk scores above 0.7 cosine similarity, return "I'll connect you with a human agent" instead of generating.
2. Update the system prompt: "ONLY describe features explicitly mentioned in the provided context. NEVER speculate about features."

**Short-term (Day 3-5):**
3. Implement a post-generation verification step: extract claims from the response, check each claim against a structured product feature database (not just vector search — use exact matching against a feature list).
4. Add a blocklist of commonly hallucinated features (from customer complaints).
5. Log all responses with retrieved context for offline analysis.

**Medium-term (Day 5-7):**
6. Build an evaluation dataset from customer complaints. Run the updated system against it.
7. Implement human-in-the-loop for low-confidence responses.
8. Set up automated hallucination monitoring (daily RAGAS faithfulness score on sampled responses).

### 🔹 Scenario Example

Hallucination: "Our Pro plan includes unlimited API calls." (Reality: Pro plan has a 10K/month limit.)
Root cause: The retrieved chunk mentions "generous API limits" and the LLM interpolates "generous" as "unlimited."
Fix: The verification step checks "unlimited API calls" against the feature database, finds no match, and either removes the claim or replaces it with the actual limit.

### 🔹 Visual Explanation (Scenario)

```
Anti-Hallucination Pipeline:

Response ──▶ Claim Extractor ──▶ ["Pro plan has unlimited API calls",
                                  "Supports 5 languages", ...]
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ Feature Database │
                              │ (structured,     │
                              │  ground truth)   │
                              └────────┬────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
              ✅ Verified         ❌ Contradicted     ⚠️ Not Found
              "5 languages"      "unlimited API"      "24/7 support"
                    │                  │                  │
                    ▼                  ▼                  ▼
                Keep             Replace with         Flag for
                                 correct info         human review
```

---

## Q7 — RAG Evaluation Metrics 🟡 Medium

### 🔹 Conceptual Question

**Q: How do you evaluate a RAG system end-to-end? What metrics would you track, and how do you distinguish between retrieval failures and generation failures?**

### 🔹 Answer

**RAG evaluation must assess three components independently:**

| Component | Metrics | What It Measures |
|-----------|---------|-----------------|
| **Retrieval** | Hit Rate, MRR, nDCG@K, Recall@K | Does the retriever find relevant documents? |
| **Generation** | Faithfulness, Answer Relevance, Completeness | Does the LLM use context correctly? |
| **End-to-End** | Correctness, User Satisfaction, Task Completion | Does the system solve the user's problem? |

**Key framework — RAGAS metrics:**
1. **Faithfulness:** Fraction of claims in the answer supported by context (detects hallucination)
2. **Answer Relevance:** How well does the answer address the question
3. **Context Precision:** Are the retrieved docs actually relevant to the query
4. **Context Recall:** Does the retrieved context contain the information needed to answer

**Distinguishing failures:**
- **Retrieval failure + generation looks fine:** The LLM generates a plausible-sounding but wrong answer from parametric memory. Context precision is low.
- **Retrieval success + generation failure:** Correct docs retrieved but the LLM misinterprets or ignores them. Faithfulness is low.
- **Both fail:** Bad query understanding. Check query reformulation.

**Trade-off:** LLM-as-judge evaluation is scalable but can be unreliable. Human evaluation is gold standard but expensive. Use LLM-as-judge for continuous monitoring and human evaluation for periodic benchmarks.

### 🔹 Example

An internal knowledge base RAG system shows 85% end-to-end accuracy. Debugging: Retrieval recall@5 is 92% (good), but faithfulness is only 78%. The LLM is retrieving the right docs but adding information from its training data. Fix: Improve the system prompt for faithfulness. Post-fix: faithfulness jumps to 91%, end-to-end accuracy reaches 93%.

### 🔹 Visual Explanation (Text-Based)

```
RAG Evaluation Framework:

                    ┌─────────────────────┐
                    │    Ground Truth      │
                    │  (Q, A, Source Doc)  │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  Retrieval   │    │  Generation  │    │  End-to-End  │
  │  Evaluation  │    │  Evaluation  │    │  Evaluation  │
  ├──────────────┤    ├──────────────┤    ├──────────────┤
  │ Hit Rate     │    │ Faithfulness │    │ Correctness  │
  │ MRR          │    │ Relevance    │    │ Latency      │
  │ nDCG@K       │    │ Completeness │    │ User Rating  │
  │ Recall@K     │    │ Halluc. Rate │    │ Task Compl.  │
  └──────────────┘    └──────────────┘    └──────────────┘

  Failure Diagnosis:
  Low Retrieval + High Generation = Retrieval Problem
  High Retrieval + Low Generation = Prompt/Model Problem
  Low Both = Query Understanding Problem
```

---

### 🔹 Scenario-Based Question

**Q: You're tasked with setting up a continuous evaluation pipeline for a production RAG system serving 10K queries/day. You can't manually evaluate every response. Design an automated evaluation system.**

### 🔹 Scenario Answer

**Architecture:**

1. **Sampling:** Randomly sample 2-5% of daily queries (200-500 queries/day) for evaluation.
2. **Stratified sampling:** Ensure samples cover different query types, topics, and confidence levels.
3. **Automated evaluation pipeline:**
   - Run RAGAS metrics (faithfulness, relevance, context precision) using an LLM judge (GPT-4) on sampled responses.
   - Store scores in a time-series database.
   - Set up alerts when faithfulness drops below 80% or relevance drops below 75%.
4. **Weekly human evaluation:** Have domain experts evaluate 50-100 randomly sampled responses. Compare LLM-judge scores with human scores to calibrate.
5. **Regression testing:** Maintain a "golden set" of 200 curated (query, expected_answer) pairs. Run daily on the RAG system. Alert on accuracy drops.
6. **User feedback loop:** Track thumbs up/down, escalation to human agents, and repeat queries (signal of unsatisfactory answers).

**Cost considerations:** LLM-as-judge for 500 queries/day ≈ $5-10/day with GPT-4. Human evaluation at $50-100/week.

### 🔹 Scenario Example

Dashboard alerts: Faithfulness score dropped from 88% to 71% over 3 days. Investigation reveals a new batch of documents was ingested with formatting issues (HTML tags left in text), confusing the LLM. Fix: Add a text cleaning step in the ingestion pipeline. Faithfulness recovers to 87% the next day.

### 🔹 Visual Explanation (Scenario)

```
Continuous RAG Evaluation Pipeline:

Production Traffic (10K/day)
         │
         ├──▶ 100% ──▶ Serve Response ──▶ User
         │
         └──▶ 3% sample ──▶ Evaluation Pipeline
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
              LLM-as-Judge   Golden Set Test  User Feedback
              (RAGAS daily)  (200 Q&A daily)  (thumbs up/down)
                    │              │              │
                    └──────────────┼──────────────┘
                                   ▼
                          ┌──────────────┐
                          │  Monitoring  │
                          │  Dashboard   │
                          │  + Alerts    │
                          └──────────────┘
                                   │
                          Weekly Human Review
                          (calibrate LLM judge)
```

---

*Follow-up Questions to Expect:*
- "How do you handle evaluation when you don't have ground-truth answers?" → Use LLM-as-judge with reference-free metrics or human evaluation.
- "What's the cost of running RAGAS at scale?" → ~$0.01-0.02 per evaluation with GPT-4, so 500/day ≈ $5-10/day.
- "How do you evaluate multi-turn RAG conversations?" → Track conversation-level metrics: task completion rate, turns to resolution, context carryover accuracy.
