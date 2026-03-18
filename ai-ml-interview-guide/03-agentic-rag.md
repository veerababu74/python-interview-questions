# 🔁 Agentic RAG — Interview Questions & Answers

> Comprehensive interview preparation covering RAG vs Agentic RAG, multi-hop reasoning, dynamic retrieval planning, and tool-augmented retrieval.

---

## Q1 — RAG vs Agentic RAG 🟢 Easy

### 🔹 Conceptual Question

**Q: What is the difference between standard RAG and Agentic RAG? When would you choose Agentic RAG over a standard RAG pipeline?**

### 🔹 Answer

| Dimension | Standard RAG | Agentic RAG |
|-----------|-------------|-------------|
| Retrieval | Single-pass: retrieve once, generate once | Multi-pass: iterative retrieval based on reasoning |
| Decision making | Fixed pipeline (retrieve → generate) | Agent decides when, what, and how to retrieve |
| Query handling | Single query to vector DB | Reformulates queries, uses multiple sources |
| Complex questions | Struggles with multi-hop reasoning | Handles "gather info from A, then use it to find B" |
| Tool use | Vector DB only | Vector DB + APIs + web search + SQL + more |
| Adaptability | Static flow | Dynamic — adapts retrieval strategy per query |

**When to choose Agentic RAG:**
- Questions require information from multiple sources or documents
- The user's question is ambiguous and needs clarification or reformulation
- The answer requires reasoning over retrieved facts (not just extraction)
- You need to combine structured data (SQL) with unstructured data (documents)
- Complex multi-step queries: "Compare policy A from 2022 with the current version and summarize changes"

**Trade-offs:** Agentic RAG is more powerful but slower (multiple retrieval rounds), more expensive (more LLM calls), and harder to debug (non-deterministic retrieval paths).

### 🔹 Example

Query: "How did our customer churn rate change after we launched the loyalty program in Q3?"

**Standard RAG:** Searches vector DB for "customer churn rate loyalty program Q3" → might retrieve the loyalty program launch announcement but miss the churn metrics document.

**Agentic RAG:**
1. Agent identifies two sub-questions: (a) "What was the churn rate before Q3?" (b) "What was the churn rate after the loyalty program launch?"
2. Retrieves churn metrics for Q2 and Q4 separately
3. Also queries the structured analytics database for exact churn numbers
4. Synthesizes: "Churn decreased from 5.2% (Q2) to 3.8% (Q4), a 27% improvement after the loyalty program launch."

### 🔹 Visual Explanation (Text-Based)

```
Standard RAG:
  Query ──▶ Embed ──▶ Vector Search ──▶ Top-K Docs ──▶ LLM ──▶ Answer
  (single pass, one retrieval)

Agentic RAG:
  Query ──▶ Agent (LLM)
              │
              ├──▶ "I need churn data" ──▶ Vector Search ──▶ Docs
              │
              ├──▶ "I need exact numbers" ──▶ SQL Query ──▶ Data
              │
              ├──▶ "I need program details" ──▶ API Call ──▶ Info
              │
              └──▶ Synthesize all sources ──▶ Final Answer
              (multiple passes, adaptive retrieval)
```

---

### 🔹 Scenario-Based Question

**Q: Your standard RAG system handles 90% of queries well, but the remaining 10% are complex multi-part questions that require information from multiple documents. Your manager asks you to upgrade to Agentic RAG for ALL queries. Why might this be a bad idea, and what would you propose instead?**

### 🔹 Scenario Answer

**Why upgrading everything to Agentic RAG is problematic:**
1. **Cost:** Agentic RAG uses 3-5x more LLM tokens per query (multiple reasoning + retrieval steps). For 90% of queries that standard RAG handles fine, this is wasted cost.
2. **Latency:** Each agent reasoning loop adds 1-3 seconds. Simple queries that could be answered in 1s now take 5-10s.
3. **Reliability:** More LLM calls = more chances for errors, hallucinations, or infinite loops.
4. **Complexity:** Harder to debug, monitor, and maintain.

**Proposed hybrid architecture:**
1. **Query classifier:** Use a lightweight model to classify incoming queries as "simple" or "complex."
2. **Route simple queries → Standard RAG** (fast, cheap, reliable)
3. **Route complex queries → Agentic RAG** (powerful, multi-hop)
4. **Fallback:** If standard RAG's confidence score is low, escalate to Agentic RAG.

### 🔹 Scenario Example

```
Query Classification Examples:
  "What is our refund policy?" → Simple → Standard RAG (200ms, $0.002)
  "Compare Q1 and Q2 revenue by region" → Complex → Agentic RAG (5s, $0.02)

Cost Impact:
  All Agentic RAG: 10K queries/day × $0.02 = $200/day
  Hybrid: 9K × $0.002 + 1K × $0.02 = $18 + $20 = $38/day (81% savings)
```

### 🔹 Visual Explanation (Scenario)

```
Hybrid RAG Architecture:

  User Query ──▶ Query Classifier (lightweight LLM or rule-based)
                      │
               ┌──────┴──────┐
               ▼              ▼
          Simple (90%)    Complex (10%)
               │              │
               ▼              ▼
         Standard RAG    Agentic RAG
         (single pass)   (multi-step)
               │              │
               ▼              ▼
          ┌────────────────────┐
          │ Confidence Check   │
          │ If low → escalate  │
          │ to Agentic RAG     │
          └────────┬───────────┘
                   ▼
              Final Answer
```

---

## Q2 — Multi-Hop Reasoning 🟡 Medium

### 🔹 Conceptual Question

**Q: What is multi-hop reasoning in the context of Agentic RAG? How does an agent handle questions that require chaining information across multiple documents?**

### 🔹 Answer

**Multi-hop reasoning** requires the agent to:
1. Retrieve information from document A
2. Use that information to formulate a new query
3. Retrieve from document B based on the result from A
4. Combine findings to produce the final answer

**Example:** "Who is the CEO of the company that acquired Fitbit?"
- Hop 1: "Which company acquired Fitbit?" → Google
- Hop 2: "Who is the CEO of Google?" → Sundar Pichai

**Implementation approaches:**

1. **Iterative retrieval:** Agent retrieves, reasons, generates a follow-up query, retrieves again. Simple but sequential.
2. **Query decomposition:** Break the question into sub-questions upfront, retrieve for each, then synthesize. Can parallelize independent sub-questions.
3. **Chain-of-retrieval:** Each retrieval step's output becomes input for the next retrieval. The agent maintains a "reasoning chain."

**Challenges:**
- **Error propagation:** If hop 1 retrieves wrong info, all subsequent hops are wrong.
- **Latency:** Each hop adds retrieval + LLM latency (typically 1-3s per hop).
- **Context management:** The agent must carry forward relevant info from previous hops without exceeding context limits.

### 🔹 Example

Legal due diligence query: "What are the environmental compliance violations of subsidiaries of companies listed in Portfolio X?"

Multi-hop chain:
1. Retrieve Portfolio X → list of companies [CompA, CompB, CompC]
2. For each company, retrieve subsidiaries → [SubA1, SubA2, SubB1, ...]
3. For each subsidiary, search for environmental violations → results
4. Synthesize into a compliance report

This is a 3-hop query with fan-out at each hop — Agentic RAG handles it naturally.

### 🔹 Visual Explanation (Text-Based)

```
Multi-Hop Reasoning:

"Who is the CEO of the company that acquired Fitbit?"

  Hop 1: Query="Who acquired Fitbit?"
         ──▶ Retrieve ──▶ "Google acquired Fitbit in 2021"
                                │
                                ▼
  Hop 2: Query="Who is the CEO of Google?"  (derived from Hop 1)
         ──▶ Retrieve ──▶ "Sundar Pichai is CEO of Google"
                                │
                                ▼
  Answer: "Sundar Pichai is the CEO of Google, the company that acquired Fitbit."
```

---

### 🔹 Scenario-Based Question

**Q: Your Agentic RAG system performs multi-hop reasoning for research queries. You notice that for 3+ hop queries, accuracy drops from 85% (1-hop) to 45% (3-hop). How do you improve multi-hop accuracy?**

### 🔹 Scenario Answer

1. **Error analysis:** Categorize failures:
   - **Retrieval failure at hop N:** Correct reasoning but wrong documents retrieved
   - **Reasoning failure:** Wrong sub-question generated from previous hop's results
   - **Information loss:** Key info from early hops lost by hop 3 (context window)

2. **Fixes by failure type:**

   **Retrieval failures:**
   - Improve embedding quality for intermediate queries (which are often reformulated and less natural)
   - Add query expansion: generate multiple phrasings of each sub-question
   - Use hybrid search for intermediate hops (exact matches matter for entity names)

   **Reasoning failures:**
   - Add verification steps: after each hop, verify the intermediate result before proceeding
   - Use chain-of-thought prompting explicitly at each hop
   - Include examples of multi-hop reasoning in the system prompt

   **Information loss:**
   - Maintain a "scratchpad" — a structured summary of key facts from each hop
   - Don't carry raw retrieved text; extract and compress key facts
   - Use the scratchpad as context for each subsequent hop

3. **Architectural improvement:** Implement **beam search over retrieval paths** — maintain top-3 reasoning chains in parallel, evaluate which chain is most consistent at the end.

### 🔹 Scenario Example

```
BEFORE (45% accuracy on 3-hop):
  Hop 1: "Which company acquired Fitbit?" → "Google" ✅
  Hop 2: "Who leads Google's hardware division?" → "Rick Osterloh" ✅
  Hop 3: "What's Rick Osterloh's background?" → Retrieves wrong "Rick Osterloh"
         from a different context ❌

AFTER (with scratchpad + verification):
  Hop 1: "Which company acquired Fitbit?" → "Google" ✅
    Scratchpad: {company: "Google/Alphabet", event: "Fitbit acquisition 2021"}
  Hop 2: "Who leads Google's hardware division?" → "Rick Osterloh" ✅
    Verify: Rick Osterloh + Google hardware → confirmed ✅
    Scratchpad: {company: "Google", person: "Rick Osterloh", role: "SVP Devices & Services"}
  Hop 3: "What's Rick Osterloh's background at Google?" → Correct context ✅
    (Query includes "Google" from scratchpad, improving retrieval precision)
```

### 🔹 Visual Explanation (Scenario)

```
Multi-Hop with Scratchpad:

  Query ──▶ Decompose into sub-questions
               │
               ▼
  ┌─────────────────────────────┐
  │         Scratchpad          │ (persistent across hops)
  │  {facts: [], entities: []}  │
  └─────────────┬───────────────┘
                │
     Hop 1 ──▶ Retrieve ──▶ Extract facts ──▶ Update scratchpad
                │                                    │
                ▼                                    ▼
     Verify intermediate result ←──── scratchpad context
                │
     Hop 2 ──▶ Retrieve (query enriched with scratchpad entities)
                │                                    │
                ▼                                    ▼
     Verify ──▶ Update scratchpad                    │
                │                                    │
     Hop 3 ──▶ Retrieve (full scratchpad context)    │
                │                                    │
                ▼                                    ▼
     Synthesize final answer from scratchpad
```

---

## Q3 — Dynamic Retrieval Planning 🔴 Hard

### 🔹 Conceptual Question

**Q: How does an Agentic RAG system dynamically decide its retrieval strategy at runtime? What factors influence whether the agent retrieves from a vector DB, queries a SQL database, calls an API, or uses a web search?**

### 🔹 Answer

**Dynamic retrieval planning** means the agent selects the optimal retrieval source and strategy based on:

1. **Query type analysis:**
   - Factual lookup → Vector DB or knowledge graph
   - Numerical/analytical → SQL database
   - Current events → Web search
   - Structured data → API call
   - Comparison → Multiple sources

2. **Source metadata awareness:** The agent knows what's in each source:
   - "Product docs are in the vector DB"
   - "Sales data is in PostgreSQL"
   - "Real-time pricing is via the pricing API"

3. **Query complexity assessment:**
   - Simple fact → single retrieval
   - Multi-part → decompose and route each part to the best source
   - Ambiguous → retrieve from multiple sources and cross-reference

4. **Confidence-based iteration:**
   - If first retrieval yields low-confidence results, try alternative sources
   - If results conflict across sources, retrieve more evidence to resolve

**Implementation:** Give the agent a "tool description" for each retrieval source. The tool description explains what data the source contains and when to use it. The agent's reasoning selects the appropriate tool.

### 🔹 Example

Query: "What's the revenue impact of our new pricing tier launched last month?"

Agent planning:
1. "Last month's pricing tier" → API call to product service (get tier details)
2. "Revenue impact" → SQL query to analytics DB (`SELECT revenue WHERE tier = 'new' AND date > launch_date`)
3. "Context about the launch" → Vector DB search in internal communications
4. Synthesize: combine numerical data with qualitative context

### 🔹 Visual Explanation (Text-Based)

```
Dynamic Retrieval Planning:

  User Query ──▶ Agent (LLM with tool descriptions)
                      │
                      ▼
               Query Analysis:
               "What type of info do I need?"
                      │
        ┌─────────────┼─────────────┬──────────────┐
        ▼             ▼             ▼              ▼
   Factual?      Numerical?    Real-time?     Structured?
        │             │             │              │
        ▼             ▼             ▼              ▼
   Vector DB      SQL DB       Web Search      API Call
        │             │             │              │
        └─────────────┼─────────────┴──────────────┘
                      ▼
              Merge & Synthesize
                      │
                      ▼
              Confidence Check
                      │
               ┌──────┴──────┐
               ▼              ▼
           High conf.    Low conf.
               │              │
               ▼              ▼
           Return        Try alternative
           answer        source & retry
```

---

### 🔹 Scenario-Based Question

**Q: You're building an Agentic RAG system for a financial services company. The system needs to access 5 data sources: (1) regulatory documents (vector DB), (2) client portfolios (SQL), (3) market data (REST API), (4) internal memos (vector DB), and (5) news (web search). How do you design the retrieval routing so the agent efficiently selects the right source?**

### 🔹 Scenario Answer

1. **Tool description engineering:** Write precise descriptions for each source:
   ```
   Tool: regulatory_search - Search SEC filings, compliance rules, regulations
   Tool: portfolio_query - Query client portfolio data (holdings, returns, risk)
   Tool: market_data - Get real-time stock prices, indices, forex rates
   Tool: memo_search - Search internal research memos and analyst notes
   Tool: news_search - Search recent financial news and press releases
   ```

2. **Query routing heuristics (as fallback):**
   - Keywords like "regulation", "compliance", "SEC" → regulatory_search
   - Client names, account numbers → portfolio_query
   - Stock tickers, "current price" → market_data
   - "Our analysis", "team's view" → memo_search
   - "Latest news", "today" → news_search

3. **Multi-source orchestration:**
   - For complex queries, the agent retrieves from multiple sources in parallel
   - Set a max of 3 source queries per turn to control cost/latency
   - Implement source-specific result formatters (SQL results as tables, docs as text)

4. **Evaluation:** Track per-source hit rates. If the agent consistently ignores a source or misroutes queries, refine the tool descriptions.

### 🔹 Scenario Example

Query: "Is our client John Smith's portfolio compliant with the new SEC diversification rules?"

Agent routing:
1. `portfolio_query("John Smith holdings")` → Returns portfolio composition
2. `regulatory_search("SEC diversification requirements 2024")` → Returns the rule
3. Agent compares portfolio against rule → "Portfolio has 45% in one sector. SEC rule requires max 25%. Non-compliant."

### 🔹 Visual Explanation (Scenario)

```
Financial Agentic RAG:

  "Is John Smith's portfolio compliant with new SEC rules?"
       │
       ▼
  Agent Plans: Need (1) portfolio data + (2) SEC rules
       │
       ├──▶ portfolio_query("John Smith") ──▶ {holdings: [...]}
       │                                         │
       ├──▶ regulatory_search("SEC diversification") ──▶ {rules: [...]}
       │                                                    │
       └──▶ Synthesize:                                     │
            Compare holdings vs rules ◄─────────────────────┘
                 │
                 ▼
            "Non-compliant: 45% concentration vs 25% max"
```

---

## Q4 — Tool-Augmented Retrieval 🟡 Medium

### 🔹 Conceptual Question

**Q: What is tool-augmented retrieval in Agentic RAG? How does it extend beyond traditional retrieval methods, and what challenges does it introduce?**

### 🔹 Answer

**Tool-augmented retrieval** extends Agentic RAG by giving the agent access to computational tools during the retrieval process — not just search, but also data transformation, calculation, and API interactions.

**Extensions beyond traditional retrieval:**

| Capability | Traditional Retrieval | Tool-Augmented Retrieval |
|------------|----------------------|--------------------------|
| Search | Vector similarity search | + web search, SQL, graph queries |
| Computation | None | Calculator, code execution, aggregation |
| Transformation | None | Data parsing, format conversion, summarization |
| Verification | None | Fact-checking against authoritative APIs |
| Action | Read-only | Can write data, trigger workflows |

**Key tools for Agentic RAG:**
1. **Search tools:** Vector DB, BM25, web search, knowledge graph queries
2. **Data tools:** SQL queries, spreadsheet analysis, data extraction
3. **Compute tools:** Calculator, Python code execution, statistical analysis
4. **API tools:** CRM lookup, inventory check, pricing API
5. **Verification tools:** Fact-checking, citation verification

**Challenges:**
- **Tool selection errors:** Agent picks the wrong tool for a subtask
- **Tool composition:** Chaining multiple tools correctly is error-prone
- **Security:** Giving an agent SQL or code execution access requires sandboxing
- **Error handling:** Tools can fail, return empty results, or time out
- **Cost:** Each tool call may have API costs, latency overhead

### 🔹 Example

Query to a healthcare RAG system: "What's the recommended dosage of metformin for a 75kg diabetic patient with mild renal impairment?"

Tool-augmented retrieval:
1. `drug_db_search("metformin dosage guidelines")` → retrieval
2. `calculate_gfr(age=65, weight=75, creatinine=1.5)` → computation
3. `dosage_adjustment_table(drug="metformin", gfr=52)` → structured lookup
4. Synthesize: "For GFR 45-59 (mild-moderate impairment), max metformin dose is 1000mg/day"

### 🔹 Visual Explanation (Text-Based)

```
Tool-Augmented Retrieval Pipeline:

  Query ──▶ Agent Reasoning
              │
              ├──▶ 🔍 Search Tool ──▶ Retrieve drug guidelines
              │
              ├──▶ 🧮 Compute Tool ──▶ Calculate GFR from patient data
              │
              ├──▶ 📊 Lookup Tool ──▶ Cross-reference dosage table
              │
              ├──▶ ✅ Verify Tool ──▶ Check against contraindications
              │
              └──▶ 📝 Synthesize ──▶ Final recommendation with sources
```

---

### 🔹 Scenario-Based Question

**Q: Your Agentic RAG system has 12 tools available, but the agent frequently selects the wrong tool or calls tools unnecessarily. This causes slow responses and incorrect answers. How do you improve tool selection accuracy?**

### 🔹 Scenario Answer

1. **Diagnose tool selection errors:**
   - Log every tool selection with the agent's reasoning trace
   - Categorize errors: wrong tool, unnecessary tool call, missing tool call, wrong parameters
   - Identify patterns: specific query types that confuse the agent

2. **Improvement strategies:**

   **Tool description optimization:**
   - Make descriptions more specific and contrastive ("Use this for X, NOT for Y")
   - Include example queries for each tool
   - Add negative examples: "Do NOT use this tool for real-time data"

   **Tool consolidation:**
   - 12 tools may be too many. Merge similar tools (e.g., combine 3 different search tools into one with a `source` parameter)
   - Group tools into categories, use a two-step selection: first pick category, then pick tool

   **Few-shot tool selection:**
   - Include 3-5 examples of correct tool selection in the system prompt
   - Show the reasoning: "Query is about pricing → this requires the pricing API → use `get_pricing()`, NOT `search_docs()`"

   **Validation layer:**
   - Before executing a tool call, validate: is this tool appropriate for this query type?
   - Use a lightweight classifier to verify tool selection before execution

   **Feedback loop:**
   - When tool calls fail or return irrelevant results, feed this back to the agent: "The previous tool call returned no useful results. Consider using a different tool."

### 🔹 Scenario Example

```
BEFORE (poor tool descriptions):
  Tool: "search" - "Search for information"
  Tool: "query_db" - "Query the database"
  Agent: Query="latest stock price AAPL" → Calls "search" (vector DB)
         → Returns old article about Apple stock ❌

AFTER (precise tool descriptions):
  Tool: "doc_search" - "Search internal documents, policies, guides.
        Use for: questions about company procedures, historical docs.
        DO NOT use for: real-time data, prices, current status."
  Tool: "market_data_api" - "Get real-time stock prices, market indices.
        Use for: current prices, today's data, market status.
        Input: ticker symbol (e.g., 'AAPL')"
  Agent: Query="latest stock price AAPL" → Calls "market_data_api('AAPL')"
         → Returns $178.50 ✅
```

### 🔹 Visual Explanation (Scenario)

```
Improved Tool Selection Architecture:

  User Query ──▶ Agent Reasoning
                      │
                      ▼
               Tool Selection
                      │
                      ▼
              ┌───────────────┐
              │  Validation   │
              │  Layer        │──▶ "Is search_docs correct
              │               │     for a real-time price query?"
              └───────┬───────┘     NO → suggest market_data_api
                      │
                      ▼
              Execute Correct Tool
                      │
                      ▼
              ┌───────────────┐
              │ Result Check  │──▶ Empty/irrelevant? → Retry different tool
              └───────┬───────┘
                      │
                      ▼
               Use result in answer
```

---

## Q5 — Agentic RAG for Complex Workflows 🔴 Hard

### 🔹 Conceptual Question

**Q: How do you design an Agentic RAG system that handles both conversational context (multi-turn) and complex analytical queries? What architectural components are needed?**

### 🔹 Answer

**Architectural components:**

1. **Conversation memory:** Maintain chat history and extracted entities across turns. Store in short-term memory (context window) and long-term memory (vector DB or key-value store).

2. **Query understanding module:**
   - Coreference resolution: "What about its competitors?" → resolve "its" to the previously discussed company
   - Query reformulation: Combine current question with conversation context into a standalone query for retrieval
   - Intent classification: Is this a follow-up, a new topic, or a clarification?

3. **Adaptive retrieval planner:**
   - Decides retrieval strategy based on query complexity
   - For simple follow-ups: use cached context from previous retrievals
   - For new topics: perform fresh retrieval
   - For analytical questions: route to SQL + computation tools

4. **State management:**
   - Track the "research state" — what has been retrieved, what's been answered, what's still unknown
   - Enable the agent to say "Based on what I've already found..." without re-retrieving

5. **Response generation with context fusion:**
   - Merge conversation history, new retrievals, and computed results
   - Prioritize recent context but include relevant historical context

**Trade-offs:**
- Carrying too much conversation context dilutes retrieval relevance
- Too little context causes the agent to lose track of the conversation
- Balancing act between context window usage and retrieval freshness

### 🔹 Example

Multi-turn analytical conversation:
- Turn 1: "What was our Q3 revenue?" → SQL query → "$12.5M"
- Turn 2: "How does that compare to Q2?" → Agent remembers Q3=$12.5M, queries Q2 → "$11.2M, 11.6% growth"
- Turn 3: "What drove the growth?" → Agent retrieves product launch docs from Q3, sales reports → "New enterprise tier launch drove 60% of incremental revenue"
- Turn 4: "Any risks to Q4?" → Agent searches risk assessments, market reports → synthesizes risks

### 🔹 Visual Explanation (Text-Based)

```
Multi-Turn Agentic RAG Architecture:

  User Message ──▶ Query Understanding
                        │
                   ┌────┴────┐
                   ▼         ▼
             Follow-up?   New topic?
                   │         │
                   ▼         ▼
            Resolve refs  Fresh retrieval
            from history  planning
                   │         │
                   └────┬────┘
                        ▼
                  Adaptive Retriever
                  (Vector/SQL/API/Cache)
                        │
                        ▼
              ┌──────────────────┐
              │ Context Fusion   │
              │ • Chat history   │
              │ • New retrievals │
              │ • Computed data  │
              │ • State/memory   │
              └────────┬─────────┘
                       ▼
                 Generate Response
                       │
                       ▼
                 Update State & Memory
```

---

### 🔹 Scenario-Based Question

**Q: Your Agentic RAG system for a consulting firm needs to generate a comprehensive competitive analysis report. The report requires data from 20+ sources, involves financial calculations, and must cross-reference information. A single agent run times out after 5 minutes. How do you architect this for reliability and speed?**

### 🔹 Scenario Answer

1. **Break into orchestrated sub-agents:**
   - **Planner agent:** Decomposes "competitive analysis" into sections (market overview, competitor profiles, financial comparison, SWOT, recommendations)
   - **Research agents (parallel):** One per competitor, each gathering data independently
   - **Analysis agent:** Processes gathered data, runs financial calculations
   - **Writer agent:** Composes the final report from analyzed data

2. **Reliability improvements:**
   - **Checkpointing:** Save progress after each section. If it fails at section 4, restart from section 4, not from scratch.
   - **Timeouts per sub-task:** Each sub-agent gets a 60-second timeout. If it fails, retry once, then skip with a "[data unavailable]" placeholder.
   - **Partial results:** Return completed sections immediately, mark incomplete ones as "in progress."

3. **Speed improvements:**
   - **Parallel research:** All competitor research agents run simultaneously
   - **Cached data:** Cache competitor profiles with a 24-hour TTL. Only refresh financial data.
   - **Pre-computed templates:** Use report templates so the writer agent fills in sections rather than generating from scratch.

4. **Architecture:**
   - Total time: Planner (10s) + Parallel research (60s max) + Analysis (30s) + Writing (30s) = ~2.5 min (under timeout)

### 🔹 Scenario Example

```
Report Generation Pipeline:

  Planner: "Analyze competitors: [CompA, CompB, CompC, CompD, CompE]"
  
  Parallel Research (60s timeout each):
    Agent-A: CompA profile, financials, products ──▶ ✅ (45s)
    Agent-B: CompB profile, financials, products ──▶ ✅ (38s)
    Agent-C: CompC profile, financials, products ──▶ ✅ (52s)
    Agent-D: CompD profile, financials, products ──▶ ❌ (timeout → retry → ✅ 55s)
    Agent-E: CompE profile, financials, products ──▶ ✅ (41s)
  
  Analysis Agent: Cross-reference, compute market shares, growth rates (30s)
  Writer Agent: Compile into report template (30s)
  
  Total: ~2.5 minutes ✅
```

### 🔹 Visual Explanation (Scenario)

```
Orchestrated Agentic RAG for Report Generation:

  ┌──────────┐
  │ Planner  │──▶ Decompose into 5 competitor profiles + analysis + report
  └────┬─────┘
       │
       ▼
  ┌─────────────────────────────────────────────┐
  │           Parallel Research Phase            │
  │                                              │
  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │
  │  │Comp A│ │Comp B│ │Comp C│ │Comp D│ │Comp E│ │
  │  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ │
  │     │        │        │        │        │     │
  └─────┼────────┼────────┼────────┼────────┼─────┘
        └────────┼────────┼────────┼────────┘
                 ▼
          ┌──────────┐
          │ Analysis │──▶ Checkpoint: save intermediate results
          │  Agent   │
          └────┬─────┘
               ▼
          ┌──────────┐
          │  Writer  │──▶ Final Report (from template)
          │  Agent   │
          └──────────┘
```

---

*Follow-up Questions to Expect:*
- "How do you handle contradictory information from multiple retrieval sources?" → Implement conflict resolution: prioritize by source authority (official docs > memos), recency, and consistency with other sources. Flag contradictions for human review.
- "What's the latency overhead of Agentic RAG vs standard RAG?" → Typically 3-10x (2-3 LLM reasoning calls + multiple retrievals). Mitigate with parallel retrieval, caching, and lightweight planning models.
- "How do you evaluate Agentic RAG vs standard RAG?" → Use multi-hop QA benchmarks (HotpotQA, MuSiQue). Measure both accuracy AND efficiency (tokens used, retrieval calls, latency).
