# 🤖 Agentic AI Systems — Interview Questions & Answers

> Comprehensive interview preparation covering AI agents vs pipelines, planning vs reactive agents, ReAct, multi-agent collaboration, task decomposition, and reflection loops.

---

## Q1 — AI Agents vs Traditional Pipelines 🟢 Easy

### 🔹 Conceptual Question

**Q: What distinguishes an AI agent from a traditional LLM pipeline? When should you use an agent architecture versus a deterministic pipeline?**

### 🔹 Answer

| Dimension | Traditional Pipeline | AI Agent |
|-----------|---------------------|----------|
| Control flow | Fixed, deterministic DAG | Dynamic, LLM-decided |
| Decision making | Pre-coded branching | LLM reasons at runtime |
| Tool use | Hardcoded sequence | Agent selects tools dynamically |
| Error handling | Predefined fallbacks | Self-correcting loops |
| Iteration | Single pass | Multi-step reasoning loops |

**When to use agents:**
- Tasks with unpredictable inputs requiring flexible reasoning
- Multi-step tasks where the next action depends on previous results
- Problems requiring tool selection from a large toolkit
- Exploratory tasks (research, debugging, data analysis)

**When to use pipelines:**
- Well-defined, repeatable workflows (e.g., summarize → translate → format)
- Latency-sensitive applications (agents are slower due to multiple LLM calls)
- High-reliability requirements (agents can go off-track)
- Cost-sensitive applications (agents use more tokens)

**Trade-off:** Agents are powerful but unpredictable. Pipelines are reliable but rigid. In production, most teams use **hybrid approaches** — agentic loops within a structured pipeline framework.

### 🔹 Example

A customer support system: The **pipeline** approach routes tickets by category (billing → billing template, technical → tech FAQ search). The **agent** approach reads the ticket, decides whether to search the knowledge base, check the user's account, or escalate to a human — adapting to each unique query. The hybrid approach uses a pipeline for common queries (80%) and an agent for complex/novel queries (20%).

### 🔹 Visual Explanation (Text-Based)

```
Traditional Pipeline:
  Input ──▶ Step1 ──▶ Step2 ──▶ Step3 ──▶ Output
  (Fixed order, no branching decisions by LLM)

AI Agent:
  Input ──▶ LLM Thinks ──▶ Choose Action ──▶ Execute ──▶ Observe
                ▲                                          │
                └──────────── Loop until done ◄────────────┘
```

---

### 🔹 Scenario-Based Question

**Q: Your team built an agentic system for automated code review. It works great in demos but in production, it sometimes enters infinite loops — repeatedly calling the same tool or oscillating between two actions. How do you diagnose and prevent this?**

### 🔹 Scenario Answer

1. **Diagnosis:**
   - Log every agent step: action chosen, tool called, observation received, reasoning trace.
   - Identify the loop pattern: Is the agent getting the same observation and making the same decision? Or is the observation changing but the agent doesn't recognize it's making progress?

2. **Root causes:**
   - **Ambiguous tool output:** The tool returns a result the agent doesn't know how to interpret, so it retries.
   - **Missing stop condition:** The agent doesn't have a clear "done" signal.
   - **Context overflow:** After many steps, early context is lost, and the agent forgets it already tried an action.

3. **Prevention strategies:**
   - **Max iteration limit:** Hard cap at N steps (e.g., 10). Return best-effort result or escalate.
   - **Action deduplication:** Track action history. If the same (action, params) is repeated 2+ times, force a different action or terminate.
   - **Monotonic progress check:** After each step, verify the agent is making progress toward the goal. If not, intervene.
   - **Structured output:** Force the agent to output a structured decision with an explicit "DONE" or "CONTINUE" field.
   - **Summarize history:** Periodically compress action history into a summary to prevent context overflow.

### 🔹 Scenario Example

The code review agent loop: `read_file("auth.py")` → `analyze_function("login")` → `read_file("auth.py")` → `analyze_function("login")` → ... The observation from `analyze_function` returns "no issues found" but the agent's prompt says "review all functions." It doesn't realize it already reviewed `login` and starts over. Fix: Maintain a "reviewed functions" list in the agent state. After reviewing `login`, add it to the list and exclude it from future reviews.

### 🔹 Visual Explanation (Scenario)

```
BEFORE (Infinite Loop):
  Think ──▶ read_file ──▶ analyze ──▶ Think ──▶ read_file ──▶ ...
                                       │ (forgets it already read!)
                                       └─── Lost context

AFTER (With Safeguards):
  Think ──▶ read_file ──▶ analyze ──▶ Update State ──▶ Think
    │                                  {"reviewed": ["login"]}    │
    │                                                             │
    └── Max 10 steps ── Action dedup check ── Progress check ─────┘
                              │
                        If stuck → TERMINATE with partial result
```

---

## Q2 — ReAct Framework 🟡 Medium

### 🔹 Conceptual Question

**Q: Explain the ReAct (Reasoning + Acting) framework. How does it differ from Chain-of-Thought (CoT) prompting, and why is it effective for tool-using agents?**

### 🔹 Answer

**ReAct** interleaves reasoning traces (thinking) with actions (tool calls) in a loop:
1. **Thought:** The agent reasons about what to do next
2. **Action:** The agent calls a tool or takes an action
3. **Observation:** The agent receives the tool's output
4. Repeat until the task is complete

**ReAct vs Chain-of-Thought:**

| Aspect | CoT | ReAct |
|--------|-----|-------|
| Reasoning | Internal only (think step-by-step) | Interleaved with actions |
| External info | None (uses parametric knowledge) | Retrieves from tools/APIs |
| Grounding | No (can hallucinate facts) | Yes (grounded in observations) |
| Adaptability | One-shot reasoning | Iterative, adaptive |

**Why ReAct is effective:**
- **Grounded reasoning:** Each thought is informed by real observations, not just parametric memory.
- **Traceable:** You can inspect the thought-action-observation trace for debugging.
- **Error recovery:** If an action fails, the agent can reason about why and try differently.

**Limitations:**
- More token usage (thoughts + actions + observations consume context)
- Latency increases with each loop iteration
- Quality depends heavily on the LLM's reasoning ability

### 🔹 Example

Query: "What's the current stock price of Apple and how does it compare to its 52-week average?"

ReAct trace:
- **Thought:** I need to look up Apple's current stock price. Let me use the stock API.
- **Action:** `get_stock_price("AAPL")`
- **Observation:** Current price: $178.50
- **Thought:** Now I need the 52-week average. Let me query historical data.
- **Action:** `get_52week_stats("AAPL")`
- **Observation:** 52-week high: $199.62, low: $143.90, average: $171.76
- **Thought:** Apple is trading at $178.50, which is 3.9% above its 52-week average of $171.76.
- **Action:** Return final answer

### 🔹 Visual Explanation (Text-Based)

```
ReAct Loop:

┌─────────────────────────────────────────────┐
│                                             │
│   Thought ──▶ Action ──▶ Observation        │
│      │                        │             │
│      └────────────────────────┘             │
│           (repeat until done)               │
│                                             │
│   Thought: "I need the stock price"         │
│   Action:  get_stock_price("AAPL")          │
│   Obs:     $178.50                          │
│   Thought: "Now I need 52-week avg"         │
│   Action:  get_52week_stats("AAPL")         │
│   Obs:     avg: $171.76                     │
│   Thought: "3.9% above average"             │
│   Action:  FINISH(answer)                   │
│                                             │
└─────────────────────────────────────────────┘
```

---

### 🔹 Scenario-Based Question

**Q: You're building a ReAct agent for a data analysis task. The agent has access to a SQL tool, a Python executor, and a charting tool. In testing, the agent frequently generates incorrect SQL queries, gets error messages, but fails to correct itself — it keeps generating variations of the same wrong query. How do you fix this?**

### 🔹 Scenario Answer

1. **Root cause analysis:** The agent isn't learning from error observations. It sees "SQL syntax error near..." but lacks the SQL knowledge to fix it.

2. **Solutions:**
   - **Error-specific prompting:** Include examples of common SQL errors and their fixes in the system prompt. Show a "bad query → error → corrected query" example.
   - **Schema awareness:** Inject the database schema (table names, column names, types) into the context. Most SQL errors come from wrong table/column names.
   - **Query validation step:** Before executing SQL, have the agent validate the query against the schema (a lightweight check).
   - **Fallback escalation:** After 3 failed SQL attempts, switch strategy — ask the agent to describe what it wants in natural language, then use a text-to-SQL model to generate the query.
   - **Error categorization:** Classify errors (syntax, permission, timeout, no results) and provide specific recovery instructions for each category.

3. **Testing:** Create a benchmark of 50 data analysis questions with known SQL solutions. Measure: (a) success rate, (b) average attempts before success, (c) error recovery rate.

### 🔹 Scenario Example

```
Attempt 1:
  Thought: "Get monthly revenue from the sales table"
  Action: SQL("SELECT month, SUM(revenue) FROM sales GROUP BY month")
  Obs: ERROR - column "revenue" not found

Attempt 2 (without fix):
  Thought: "Let me try again"
  Action: SQL("SELECT month, SUM(total_revenue) FROM sales GROUP BY month")
  Obs: ERROR - column "total_revenue" not found

After fix (with schema injection):
  Context: Schema: sales(id, date, amount, product_id, customer_id)
  Thought: "The revenue column is called 'amount' based on the schema"
  Action: SQL("SELECT DATE_TRUNC('month', date), SUM(amount) FROM sales GROUP BY 1")
  Obs: SUCCESS - [{"month": "2024-01", "sum": 150000}, ...]
```

### 🔹 Visual Explanation (Scenario)

```
Improved ReAct with Error Recovery:

  Thought ──▶ Generate SQL ──▶ Validate vs Schema
                                    │
                              ┌─────┴─────┐
                              ▼           ▼
                           Valid       Invalid
                              │           │
                              ▼           ▼
                         Execute      Fix with schema
                              │        hints, retry
                         ┌────┴────┐
                         ▼         ▼
                      Success    Error
                         │         │
                         ▼         ▼
                      Return    Classify error
                      result    │
                                ├─ Syntax → auto-fix
                                ├─ No results → broaden query
                                └─ 3 failures → fallback to NL2SQL model
```

---

## Q3 — Multi-Agent Collaboration 🔴 Hard

### 🔹 Conceptual Question

**Q: How do you design a multi-agent system where multiple specialized LLM agents collaborate on a complex task? What are the key architectural patterns and their trade-offs?**

### 🔹 Answer

**Multi-agent collaboration patterns:**

1. **Hierarchical (Manager-Worker):** A manager agent decomposes the task and delegates to specialized worker agents.
   - ✅ Clear control flow, easy to debug
   - ❌ Manager is a bottleneck, single point of failure

2. **Peer-to-Peer (Debate/Discussion):** Agents discuss and critique each other's outputs.
   - ✅ Better for tasks requiring diverse perspectives (e.g., code review)
   - ❌ High token cost, can get stuck in disagreements

3. **Pipeline (Sequential):** Each agent handles one phase and passes output to the next.
   - ✅ Simple, predictable, easy to monitor
   - ❌ No backtracking, errors propagate

4. **Blackboard (Shared Memory):** Agents read from and write to a shared state space.
   - ✅ Flexible, agents can work in parallel
   - ❌ Complex concurrency management, harder to debug

**Key design decisions:**
- **Communication protocol:** Structured messages (JSON) vs. natural language
- **Conflict resolution:** Voting, authority hierarchy, or consensus
- **State management:** Shared vs. local state
- **Termination:** How do agents know when to stop?

### 🔹 Example

An automated research assistant with three agents:
- **Planner Agent:** Breaks down "Research the impact of AI on healthcare" into subtasks
- **Researcher Agent:** Searches papers, extracts findings for each subtask
- **Writer Agent:** Synthesizes findings into a coherent report
- **Reviewer Agent:** Reviews the report, sends feedback to the Writer

This uses a hybrid hierarchical-pipeline pattern: Planner manages the workflow, Researcher and Writer execute sequentially, and Reviewer provides feedback loops.

### 🔹 Visual Explanation (Text-Based)

```
Pattern 1: Hierarchical                Pattern 2: Peer-to-Peer
┌──────────┐                          ┌────────┐   ┌────────┐
│  Manager │                          │ Agent1 │◄─▶│ Agent2 │
│  Agent   │                          └───┬────┘   └────┬───┘
└──┬───┬───┘                              │             │
   │   │                                  └──────┬──────┘
   ▼   ▼                                        ▼
┌─────┐ ┌─────┐                          ┌────────┐
│ W1  │ │ W2  │                          │ Agent3 │
└─────┘ └─────┘                          └────────┘

Pattern 3: Pipeline                    Pattern 4: Blackboard
A1 ──▶ A2 ──▶ A3 ──▶ Output           ┌──────────────┐
                                       │  Shared State │
                                       └──┬──┬──┬─────┘
                                          │  │  │
                                         A1 A2 A3
```

---

### 🔹 Scenario-Based Question

**Q: You're designing a multi-agent code generation system. One agent writes code, another writes tests, and a third reviews the code. In production, you notice the agents sometimes produce inconsistent results — the test agent writes tests for a different interface than the code agent implemented. How do you ensure consistency?**

### 🔹 Scenario Answer

1. **Root cause:** Each agent works with an incomplete view of what the others produced. The test agent may start before the code agent finishes, or it may not receive the final interface definition.

2. **Solution — Shared Contract Pattern:**
   - Before coding begins, an **Architect agent** produces an interface specification (function signatures, types, expected behavior).
   - Both the Code agent and Test agent receive this shared contract as input.
   - The Review agent checks both code and tests against the contract.

3. **Implementation:**
   - **Step 1:** Architect generates `interface_spec.json` with function signatures, input/output types, and edge cases.
   - **Step 2:** Code agent implements against the spec. Test agent writes tests against the spec. These can run in parallel.
   - **Step 3:** Reviewer checks: Does code implement the spec? Do tests cover the spec? Are code and tests compatible?
   - **Step 4:** If review fails, send specific feedback back to the failing agent with the contract and the mismatch.

4. **Additional safeguards:**
   - Run the tests against the code as a hard validation step (not just LLM review).
   - Use type checking (e.g., mypy) to catch interface mismatches automatically.

### 🔹 Scenario Example

```
Without Shared Contract:
  Code Agent: def calculate_tax(income: float) -> float
  Test Agent: def test_tax(): assert calculate_tax(50000, "US") == 12500
  ❌ Test fails — different function signature!

With Shared Contract:
  Contract: calculate_tax(income: float, country: str = "US") -> float
  Code Agent: implements calculate_tax with both params
  Test Agent: tests with both params
  ✅ Compatible!
```

### 🔹 Visual Explanation (Scenario)

```
Shared Contract Multi-Agent Pattern:

            ┌─────────────┐
            │  Architect   │
            │    Agent     │
            └──────┬──────┘
                   │
                   ▼
          ┌─────────────────┐
          │ Interface Spec  │ (shared contract)
          │ {functions,     │
          │  types, edge    │
          │  cases}         │
          └───┬────────┬───┘
              │        │
              ▼        ▼
        ┌──────┐  ┌──────┐
        │ Code │  │ Test │  (parallel)
        │ Agent│  │ Agent│
        └──┬───┘  └──┬───┘
           │         │
           ▼         ▼
        ┌───────────────┐
        │ Review Agent  │──▶ Check vs contract
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │ Run Tests     │──▶ Hard validation
        └───────────────┘
```

---

## Q4 — Task Decomposition 🟡 Medium

### 🔹 Conceptual Question

**Q: How does an AI agent decompose complex tasks into subtasks? What are the strategies for task decomposition, and how do you handle dependencies between subtasks?**

### 🔹 Answer

**Task decomposition strategies:**

1. **LLM-based decomposition:** Prompt the LLM to break down the task into steps. Most flexible but can produce poor decompositions for unfamiliar tasks.
   - Prompt: "Break down this task into sequential subtasks: [task]"

2. **Predefined templates:** For known task categories, use hardcoded decomposition templates. Reliable but not generalizable.
   - Example: "Write a report" → [research, outline, draft, review, finalize]

3. **Recursive decomposition:** Decompose into subtasks, then decompose each subtask further until reaching atomic actions. Good for deeply complex tasks.

4. **Plan-and-Solve:** Generate a high-level plan first, then solve each step. Variant of CoT with explicit planning.

**Handling dependencies:**
- **Sequential dependencies:** Subtask B needs the output of subtask A → execute in order.
- **Parallel opportunities:** Subtasks A and B are independent → execute simultaneously to save time.
- **Conditional dependencies:** Execute subtask C only if subtask A's result meets criteria X.
- **DAG representation:** Model subtasks as a directed acyclic graph. Topological sort gives execution order.

**Trade-offs:**
- Over-decomposition creates unnecessary overhead and context usage
- Under-decomposition leads to subtasks that are too complex for single LLM calls
- Dynamic re-planning is essential — initial decomposition may be wrong

### 🔹 Example

Task: "Analyze our competitor's pricing strategy and recommend adjustments."

Decomposition:
1. Identify top 5 competitors (independent research tasks → parallel)
2. For each competitor, gather pricing data (5 parallel tasks)
3. Analyze pricing patterns across competitors (depends on step 2)
4. Compare with our current pricing (depends on step 3 + internal data)
5. Generate recommendations (depends on step 4)

### 🔹 Visual Explanation (Text-Based)

```
Task Decomposition DAG:

"Analyze competitor pricing & recommend"
                │
        ┌───────┴───────┐
        ▼               ▼
  [Identify          [Get our
   competitors]       pricing data]
        │                  │
   ┌────┼────┐             │
   ▼    ▼    ▼             │
 [Get  [Get  [Get          │
  C1]   C2]   C3]          │
  prices prices prices     │
   │     │     │           │
   └─────┼─────┘           │
         ▼                 │
   [Analyze patterns]◄─────┘
         │
         ▼
   [Generate recommendations]

Parallel: C1, C2, C3 pricing can run simultaneously
Sequential: Analysis depends on all pricing data
```

---

### 🔹 Scenario-Based Question

**Q: Your agentic system decomposes a user request into 15 subtasks, but execution takes 3 minutes due to sequential processing. The user expects results in under 30 seconds. How do you optimize the task execution?**

### 🔹 Scenario Answer

1. **Analyze the dependency graph:** Map which subtasks depend on each other. Often, 15 sequential tasks can be restructured into 4-5 parallel stages.

2. **Optimization strategies:**
   - **Parallelize independent subtasks:** If tasks 2, 3, 4 are independent, run them simultaneously.
   - **Merge small subtasks:** Combine simple subtasks into a single LLM call (e.g., "Extract name, email, and phone" instead of three separate calls).
   - **Cache common subtasks:** If "get company info" is a common subtask, cache results.
   - **Streaming partial results:** Return results as subtasks complete rather than waiting for all.
   - **Reduce decomposition granularity:** Maybe 15 subtasks is over-decomposed. 5-7 larger subtasks might suffice.
   - **Use lighter models for simple subtasks:** Route simple extraction tasks to GPT-3.5 (faster, cheaper) and complex reasoning to GPT-4.

3. **Measure and iterate:** Profile each subtask's latency. Focus optimization on the critical path (longest sequential chain).

### 🔹 Scenario Example

```
BEFORE (15 sequential tasks, 3 min total):
  T1(12s) → T2(8s) → T3(15s) → ... → T15(10s) = 180s

AFTER (Parallelized + Merged):
  Stage 1: T1 (12s)
  Stage 2: [T2, T3, T4] parallel (15s max)
  Stage 3: [T5+T6 merged, T7+T8 merged] parallel (18s max)
  Stage 4: T9 (10s)
  Total: 12 + 15 + 18 + 10 = 55s

Further optimized (lighter models for simple tasks):
  Stage 1: T1 with GPT-3.5 (4s)
  Stage 2: [T2, T3, T4] parallel with GPT-3.5 (6s max)
  Stage 3: [T5+T6, T7+T8] parallel with GPT-4 (18s max)
  Stage 4: T9 with GPT-4 (10s)
  Total: 4 + 6 + 18 + 10 = 38s ✓ Under target?

  Need more? → Cache T1 result, pre-compute common subtasks.
```

### 🔹 Visual Explanation (Scenario)

```
Critical Path Optimization:

BEFORE:
T1──T2──T3──T4──T5──T6──T7──T8──T9──T10──T11──T12──T13──T14──T15
└────────────────── 180 seconds ──────────────────────────────────┘

AFTER:
Stage 1: T1 ─────────────────────────────────── (12s)
Stage 2: T2 ─┐
         T3 ─┼── parallel ──────────────────── (15s)
         T4 ─┘
Stage 3: T5+T6 ─┐
         T7+T8 ─┼── parallel, merged ───────── (18s)
Stage 4: T9 ─────────────────────────────────── (10s)
└────────────── 55 seconds ────────────────────┘
```

---

## Q5 — Reflection & Self-Improvement 🔴 Hard

### 🔹 Conceptual Question

**Q: What are reflection and self-improvement loops in AI agents? How do you implement them, and what are the risks?**

### 🔹 Answer

**Reflection** is when an agent evaluates its own output and reasoning, then iterates to improve. It's a meta-cognitive capability that mimics how humans review and revise their work.

**Implementation patterns:**

1. **Self-critique:** After generating output, the agent is prompted to critique its own work: "Review your answer. What's wrong? How can you improve it?"
2. **Reflexion framework:** Agent acts, evaluates the outcome, generates a textual "reflection" stored in memory, and uses past reflections to improve future attempts.
3. **Verifier-guided improvement:** A separate verifier model scores the output. If the score is below a threshold, the agent revises.
4. **Test-driven iteration:** For code generation, run tests after each attempt. Use test failures as feedback for revision.

**Risks:**
- **Overthinking:** Too many reflection loops waste tokens and time without improving quality. Diminishing returns after 2-3 iterations.
- **Regression:** The agent "fixes" something that was correct, making it worse. Always compare revision quality to original.
- **Hallucinated self-critique:** The agent generates false critiques, "fixing" non-existent problems.
- **Cost explosion:** Each reflection loop multiplies token usage.

**Best practices:**
- Cap reflection iterations (2-3 max)
- Use objective metrics (test results, score thresholds) rather than purely subjective self-critique
- Track quality per iteration — stop if quality plateaus or degrades

### 🔹 Example

A code generation agent writes a sorting function. Self-improvement loop:
1. **Generate:** Writes bubble sort
2. **Reflect:** "Bubble sort is O(n²). The task asks for an efficient sort. I should use a better algorithm."
3. **Revise:** Rewrites using merge sort
4. **Verify:** Runs unit tests — all pass. Time complexity meets requirements.
5. **Stop:** Quality threshold met.

### 🔹 Visual Explanation (Text-Based)

```
Reflection Loop:

  Generate Answer
       │
       ▼
  Self-Critique ──▶ "Is this good enough?"
       │
  ┌────┴────┐
  ▼         ▼
 Yes       No
  │         │
  ▼         ▼
 Return   Revise ──▶ Generate Improved Answer
 Answer              │
                     ▼
               Self-Critique (iteration 2)
                     │
                ┌────┴────┐
                ▼         ▼
               Yes       No (max iterations?)
                │         │
                ▼         ▼
              Return    Return best attempt
```

---

### 🔹 Scenario-Based Question

**Q: You've implemented a reflection loop in your writing agent. Users report that the agent's final output is often worse than its first draft — it "over-edits" and loses the original meaning. How do you fix this?**

### 🔹 Scenario Answer

1. **Diagnosis:** The reflection prompt is too aggressive — it always finds something to "improve," even when the original is good. The agent makes unnecessary changes.

2. **Fixes:**
   - **Quality scoring:** Assign numerical scores to each iteration. Only accept the revision if `score(revision) > score(original) + threshold`. This prevents marginal "improvements" that actually degrade quality.
   - **Targeted reflection:** Instead of "what's wrong?", ask specific questions: "Are there factual errors? Is the tone appropriate? Is anything missing?" This prevents vague over-editing.
   - **Diff-based review:** Show the agent the diff between original and revision. Ask "Is this change an improvement?" for each change. Revert changes that aren't clear improvements.
   - **Preserve original:** Always keep the first draft. Use an evaluation model (or human) to choose between original and revised versions.
   - **Early stopping:** If the reflection says "no significant issues found," stop immediately.

3. **Evaluation:** Create a benchmark of writing tasks. Compare: (a) first draft quality, (b) quality after 1 reflection, (c) quality after 2 reflections. Find the optimal number of iterations.

### 🔹 Scenario Example

```
First draft (quality: 8.5/10):
  "Our new product reduces data processing time by 60%,
   enabling real-time analytics for enterprise customers."

After reflection 1 (quality: 9.0/10):
  "Our new product reduces data processing time by 60%,
   enabling real-time analytics for enterprise customers,
   with support for petabyte-scale datasets."  ← Good addition

After reflection 2 (quality: 7.5/10):  ← WORSE!
  "Our innovative, cutting-edge product dramatically slashes
   data processing time by an impressive 60%, revolutionizing
   the way enterprise customers experience real-time analytics."
   ← Over-edited: fluffy adjectives, lost clarity

Fix: Accept revision 1, reject revision 2 (score dropped).
```

### 🔹 Visual Explanation (Scenario)

```
Quality-Gated Reflection:

  Draft_v1 ──▶ Score: 8.5
       │
       ▼
  Reflect & Revise
       │
       ▼
  Draft_v2 ──▶ Score: 9.0 ──▶ 9.0 > 8.5 + 0.3? YES ──▶ Accept v2
       │
       ▼
  Reflect & Revise
       │
       ▼
  Draft_v3 ──▶ Score: 7.5 ──▶ 7.5 > 9.0 + 0.3? NO ──▶ Reject v3
                                                         Return v2 ✅
```

---

## Q6 — Planning Agents 🟡 Medium

### 🔹 Conceptual Question

**Q: Compare planning-based agents (like plan-and-execute) with reactive agents (like ReAct). When is each approach appropriate?**

### 🔹 Answer

| Aspect | Planning Agent | Reactive Agent |
|--------|---------------|----------------|
| Approach | Plan all steps upfront, then execute | Decide next action based on current observation |
| Best for | Well-structured, multi-step tasks | Dynamic, exploratory tasks |
| Adaptability | Needs re-planning if environment changes | Naturally adaptive |
| Efficiency | Can parallelize planned steps | Sequential by nature |
| Failure handling | Can anticipate and plan around failures | Handles failures as they occur |
| Traceability | Full plan visible upfront | Emergent behavior, harder to predict |

**Planning agent variants:**
- **Plan-and-Execute:** Generate full plan → execute each step → re-plan if needed
- **Tree of Thought (ToT):** Explore multiple reasoning paths in a tree structure
- **LATS (Language Agent Tree Search):** Combine Monte Carlo Tree Search with LLM reasoning

**When to use planning:**
- Tasks with clear sub-goals (e.g., "book a trip" = flights + hotel + car)
- When parallelization is important
- When you need to estimate completion time/cost upfront

**When to use reactive:**
- Tasks with high uncertainty (debugging, research)
- When the environment changes rapidly
- When the full task scope isn't known upfront

### 🔹 Example

**Planning agent for travel booking:** Plans all steps upfront (search flights, compare prices, book cheapest, book hotel near airport, book car rental). Can execute flight and hotel searches in parallel.

**Reactive agent for debugging:** "My app is crashing." Agent reads error logs → forms hypothesis → checks code → tests fix → observes result → adjusts approach. Can't plan upfront because each step depends on what it discovers.

### 🔹 Visual Explanation (Text-Based)

```
Planning Agent:
  Task ──▶ Generate Plan ──▶ [Step1, Step2, Step3, Step4]
                                 │      │      │      │
                                 ▼      ▼      ▼      ▼
                              Execute Execute Execute Execute
                                 │      │      │      │
                                 ▼      ▼      ▼      ▼
                              Result Result Result Result
                                 │
                                 ▼
                            Synthesize Final Answer

Reactive Agent:
  Task ──▶ Observe ──▶ Think ──▶ Act ──▶ Observe ──▶ Think ──▶ Act
           (no upfront plan — each step is decided dynamically)
```

---

### 🔹 Scenario-Based Question

**Q: Your planning agent generates a 10-step plan for a data analysis task, but at step 4, the data format is different than expected. The remaining steps are now invalid. How do you handle mid-execution plan failures?**

### 🔹 Scenario Answer

1. **Detection:** Step 4 returns an unexpected result. The agent needs a mechanism to detect plan invalidation.

2. **Strategies:**
   - **Full re-planning:** Discard remaining steps. Feed the current state (steps 1-4 results) back to the planner. Generate a new plan for the remaining work. Safe but expensive.
   - **Partial re-planning:** Only re-plan the affected steps. If step 4 was "parse CSV" and the file is actually JSON, replace step 4 with "parse JSON" and adjust downstream steps that depend on the CSV structure.
   - **Adaptive execution:** Add pre-conditions to each step. Before executing step 5, check if its pre-conditions are met. If not, trigger targeted re-planning for that step.
   - **Plan checkpointing:** Save state after each step. On failure, roll back to the last successful checkpoint and re-plan from there.

3. **Best practice:** Use **partial re-planning with pre-conditions**. It minimizes wasted work while being adaptive enough to handle surprises.

### 🔹 Scenario Example

```
Original Plan:
  1. Load CSV file ✅
  2. Clean data ✅
  3. Compute statistics ✅
  4. Generate pivot table ❌ (error: column "revenue" not found)
  5. Create visualization (depends on step 4)
  6. Write summary (depends on steps 4-5)

Re-planning from step 4:
  Current state: Data loaded, cleaned, stats computed
  New info: Available columns are ["sales_amount", "region", "date"]
  
  4a. Map columns: sales_amount → revenue equivalent
  4b. Generate pivot table with sales_amount
  5. Create visualization (updated column names)
  6. Write summary
```

### 🔹 Visual Explanation (Scenario)

```
Plan Execution with Re-Planning:

  Plan: [S1] ──▶ [S2] ──▶ [S3] ──▶ [S4] ──▶ [S5] ──▶ [S6]
         ✅       ✅       ✅       ❌
                                     │
                                     ▼
                              Detect Failure
                                     │
                                     ▼
                              Checkpoint: {S1-S3 results}
                                     │
                                     ▼
                              Re-Plan from S4
                                     │
                                     ▼
                              [S4a] ──▶ [S4b] ──▶ [S5'] ──▶ [S6']
                               ✅        ✅        ✅        ✅
```

---

## Q7 — Agent Safety & Controllability 🔴 Hard

### 🔹 Conceptual Question

**Q: How do you ensure an AI agent operating autonomously doesn't take harmful or unintended actions? What guardrails and safety mechanisms should be in place for production agents?**

### 🔹 Answer

**Layers of agent safety:**

1. **Action space restriction:** Limit the set of tools/actions the agent can take. Don't give a customer support agent access to the billing system's delete function.

2. **Permission levels:** Categorize actions by risk:
   - 🟢 **Low risk (auto-approve):** Read data, search, compute
   - 🟡 **Medium risk (log & monitor):** Send emails, update records
   - 🔴 **High risk (require approval):** Delete data, make payments, modify configs

3. **Input/output validation:** Validate tool inputs before execution. Validate outputs before returning to users.

4. **Budget constraints:** Limit max tokens, max API calls, max cost per agent run. Kill the agent if it exceeds budget.

5. **Sandboxing:** Execute agent actions in a sandboxed environment. Code execution in containers. Database operations in transactions (rollback on failure).

6. **Human-in-the-loop:** For high-stakes decisions, require human approval before proceeding.

7. **Audit logging:** Log every action, decision, and reasoning trace. Essential for post-incident analysis.

8. **Timeout and kill switches:** Set maximum execution time. Provide manual kill switch for operators.

**Trade-off:** More guardrails = safer but slower and less autonomous. The art is calibrating safety to the risk level of the use case.

### 🔹 Example

A financial trading agent has three safety layers:
1. **Action restriction:** Can place trades but can't withdraw funds
2. **Budget limit:** Max $10K per trade, $50K daily limit
3. **Human approval:** Any trade > $5K requires human confirmation via Slack notification
4. **Audit log:** Every trade decision logged with reasoning trace

### 🔹 Visual Explanation (Text-Based)

```
Agent Safety Architecture:

  Agent Decision
       │
       ▼
  ┌──────────────┐
  │ Action Space │──▶ Is this action allowed? NO ──▶ Block
  │ Validator    │
  └──────┬───────┘
         │ YES
         ▼
  ┌──────────────┐
  │ Risk Level   │──▶ 🔴 High Risk ──▶ Human Approval ──▶ Approve/Deny
  │ Classifier   │──▶ 🟡 Medium   ──▶ Log & Proceed
  └──────┬───────┘──▶ 🟢 Low      ──▶ Auto-proceed
         │
         ▼
  ┌──────────────┐
  │ Budget Check │──▶ Over budget? ──▶ Kill agent
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Sandbox      │──▶ Execute in isolated environment
  │ Execution    │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Output       │──▶ Validate & sanitize before returning
  │ Validator    │
  └──────────────┘
```

---

### 🔹 Scenario-Based Question

**Q: An AI agent in your system was given access to a code repository and a deployment pipeline. During a routine task, it accidentally deployed untested code to production, causing a 2-hour outage. How would you redesign the system to prevent this?**

### 🔹 Scenario Answer

1. **Post-mortem analysis:**
   - The agent had `deploy` as an available tool without sufficient guardrails.
   - It generated code, skipped testing, and deployed directly.
   - No human approval was required for deployment.

2. **Redesign:**
   - **Remove direct deploy access:** The agent can create PRs and trigger CI, but cannot deploy to production.
   - **Mandatory pipeline gates:** Code must pass tests, linting, and security scans before any deployment. The agent cannot bypass these gates.
   - **Environment restrictions:** Agent can only deploy to `dev` and `staging` environments. Production deployment requires human approval via a separate workflow.
   - **Deployment approval workflow:** Agent creates a deployment request. A human reviews and approves. Deployment is executed by the CI/CD system, not the agent.
   - **Rollback capability:** Automated rollback if health checks fail post-deployment.
   - **Rate limiting:** Max 1 deployment per hour to staging. No more than 3 PRs per day.

3. **Monitoring:**
   - Alert when the agent attempts actions outside its permitted scope.
   - Dashboard showing all agent-initiated deployments and their outcomes.

### 🔹 Scenario Example

```
BEFORE (Unsafe):
  Agent ──▶ write_code() ──▶ deploy_to_production()
                              └──▶ 💥 Outage!

AFTER (Safe):
  Agent ──▶ write_code() ──▶ create_pr() ──▶ CI Pipeline
                                               │
                                          ┌────┴────┐
                                          ▼         ▼
                                       Tests     Security
                                       Pass?      Scan OK?
                                          │         │
                                          └────┬────┘
                                               ▼
                                         Deploy to Staging
                                               │
                                          Health Check OK?
                                               │
                                               ▼
                                     Human Approval Required
                                               │
                                               ▼
                                      Deploy to Production
                                               │
                                          Auto-rollback
                                          if errors > 1%
```

### 🔹 Visual Explanation (Scenario)

```
Safe Agent Deployment Architecture:

┌─────────────┐     ┌──────────────┐     ┌───────────┐
│   Agent      │────▶│  Code Repo   │────▶│    CI/CD  │
│ (write only) │     │  (PR only)   │     │  Pipeline │
└─────────────┘     └──────────────┘     └─────┬─────┘
                                               │
                                    ┌──────────┼──────────┐
                                    ▼          ▼          ▼
                                  Tests     Linting    Security
                                    │          │          │
                                    └──────────┼──────────┘
                                               │ All pass?
                                               ▼
                                         ┌──────────┐
                                         │ Staging  │
                                         └────┬─────┘
                                              │ Health OK?
                                              ▼
                                    ┌──────────────────┐
                                    │ Human Approval   │
                                    │ (Slack/PagerDuty)│
                                    └────────┬─────────┘
                                             │
                                             ▼
                                       ┌──────────┐
                                       │Production│
                                       └──────────┘
```

---

*Follow-up Questions to Expect:*
- "How do you handle agent state persistence across sessions?" → Use a state store (Redis, database) keyed by session ID. Serialize agent memory, action history, and current plan.
- "What's the difference between AutoGPT and BabyAGI architectures?" → AutoGPT uses a single agent with a fixed loop (think-act-observe). BabyAGI uses a task queue pattern — a planner creates tasks, an executor runs them, a prioritizer reorders remaining tasks.
- "How do you test agentic systems?" → Simulation environments with known answers, trajectory evaluation (was the sequence of actions reasonable?), and outcome-based evaluation (did it achieve the goal?).
