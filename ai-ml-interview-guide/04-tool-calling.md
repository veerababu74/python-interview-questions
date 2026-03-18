# 🔧 Tool Calling / Function Calling — Interview Questions & Answers

> Comprehensive interview preparation covering API orchestration, tool selection strategies, failure handling, and parallel tool execution.

---

## Q1 — Function Calling Fundamentals 🟢 Easy

### 🔹 Conceptual Question

**Q: What is function calling (tool calling) in LLMs, and how does it work under the hood? How does the LLM "know" which function to call and what arguments to pass?**

### 🔹 Answer

**Function calling** allows LLMs to generate structured outputs that map to predefined API calls, rather than generating free-form text.

**How it works:**
1. You define available tools with their schemas (name, description, parameters, types).
2. The user sends a query.
3. The LLM analyzes the query and decides whether to call a tool.
4. If yes, the LLM outputs a structured JSON with the function name and arguments.
5. Your application executes the function and returns the result.
6. The LLM incorporates the result into its final response.

**Key insight:** The LLM doesn't actually execute functions — it generates a JSON object that describes the function call. Your application code handles execution.

**How does the LLM "know"?**
- The tool schemas are injected into the system prompt (or via special tokens in fine-tuned models).
- The LLM is trained (or fine-tuned) on examples of tool use.
- The descriptions act as "documentation" the LLM reads to understand each tool.

**Trade-offs:**
- Function calling is more reliable than asking the LLM to output code (structured vs. free-form)
- Tool descriptions consume context window tokens
- Not all LLMs support native function calling (GPT-4, Claude 3, Gemini do; many open-source models need fine-tuning)

### 🔹 Example

User: "What's the weather in Tokyo?"

LLM with function calling:
```json
{
  "function": "get_weather",
  "arguments": {"city": "Tokyo", "units": "celsius"}
}
```

Application executes `get_weather("Tokyo", "celsius")` → `{"temp": 22, "condition": "cloudy"}`

LLM receives result and generates: "The current weather in Tokyo is 22°C and cloudy."

### 🔹 Visual Explanation (Text-Based)

```
Function Calling Flow:

  User: "Weather in Tokyo?"
       │
       ▼
  ┌─────────┐     Tool Schemas:
  │   LLM   │◄──  [get_weather(city, units),
  │         │      send_email(to, subject, body),
  └────┬────┘      search_flights(from, to, date)]
       │
       ▼
  Decision: Call get_weather
  Output: {"function": "get_weather", "args": {"city": "Tokyo"}}
       │
       ▼
  ┌───────────────┐
  │  Application  │──▶ API call ──▶ Weather API
  │  (executes)   │◄── {"temp": 22, "condition": "cloudy"}
  └───────┬───────┘
          │
          ▼
  ┌─────────┐
  │   LLM   │──▶ "It's 22°C and cloudy in Tokyo."
  └─────────┘
```

---

### 🔹 Scenario-Based Question

**Q: Your LLM application has 50 tool definitions, but you notice the LLM's tool selection accuracy drops significantly compared to when you had 10 tools. Response quality also degrades. What's happening and how do you fix it?**

### 🔹 Scenario Answer

1. **Root cause:** Too many tool definitions overwhelm the LLM's context window and decision-making ability. 50 tool schemas consume ~5,000-10,000 tokens of context, leaving less room for the actual conversation.

2. **Fixes:**

   **Dynamic tool loading:**
   - Don't send all 50 tools every time. Use a lightweight classifier to predict which 5-10 tools are relevant for the current query.
   - Example: A billing query only needs `get_invoice`, `process_refund`, `check_balance` — not the 47 other tools.

   **Tool categorization:**
   - Group tools into categories. First, the LLM selects a category (e.g., "billing"), then sees only tools in that category.

   **Tool description compression:**
   - Shorten descriptions to essential info. Remove verbose examples from schemas.
   - Use concise parameter names and descriptions.

   **Hierarchical tool calling:**
   - Create "meta-tools" that group related functions: `billing_operations(action, params)` instead of 10 separate billing tools.

   **Fine-tuning:**
   - If using an open-source model, fine-tune on your specific tool set for better accuracy.

### 🔹 Scenario Example

```
BEFORE: 50 tools in every request
  Context usage: 8,000 tokens for tools + 4,000 for conversation = 12,000 total
  Tool selection accuracy: 62%

AFTER: Dynamic loading (max 8 tools per request)
  Context usage: 1,500 tokens for tools + 4,000 for conversation = 5,500 total
  Tool selection accuracy: 91%

Dynamic Loading Logic:
  Query: "Refund my last order" ──▶ Classifier: "billing" category
  Loaded tools: [get_order, process_refund, check_balance, get_invoice, update_payment]
  (5 tools instead of 50)
```

### 🔹 Visual Explanation (Scenario)

```
Dynamic Tool Loading Architecture:

  User Query ──▶ Tool Relevance Classifier
                       │
                  ┌────┴────┐
                  ▼         ▼
            Category    Top-K Tools
            "billing"   by relevance
                  │
                  ▼
            Load 5-8 tools
            from category
                  │
                  ▼
            ┌──────────┐
            │   LLM    │ (sees only relevant tools)
            │ + 5 tools│
            └────┬─────┘
                 │
                 ▼
            Tool call + response
```

---

## Q2 — API Orchestration 🟡 Medium

### 🔹 Conceptual Question

**Q: How do you orchestrate multiple API calls in a tool-calling agent when the calls have dependencies? What patterns exist for managing complex API workflows?**

### 🔹 Answer

**Orchestration patterns:**

1. **Sequential chaining:** Output of API A becomes input to API B.
   - Example: `get_user(id)` → `get_orders(user_id)` → `get_shipping_status(order_id)`
   - ✅ Simple, predictable
   - ❌ Slow (no parallelism)

2. **Parallel fan-out:** Independent API calls run simultaneously.
   - Example: `get_weather(city)` || `get_flights(city)` || `get_hotels(city)`
   - ✅ Fast
   - ❌ Error handling is complex (what if one fails?)

3. **DAG execution:** Mix of sequential and parallel based on dependencies.
   - Model API calls as a directed acyclic graph
   - Topological sort determines execution order
   - Independent nodes run in parallel

4. **Iterative/conditional:** Agent decides next API call based on previous results.
   - Example: Search → if no results, broaden search → if still no results, try alternative API
   - ✅ Adaptive
   - ❌ Unpredictable execution time

**Implementation considerations:**
- **Timeout management:** Set per-call and total-workflow timeouts
- **Retry logic:** Exponential backoff with jitter for transient failures
- **Circuit breaker:** After N failures, stop calling a failing API
- **Result caching:** Cache API results to avoid redundant calls

### 🔹 Example

Travel booking orchestration:
```
Step 1 (parallel): get_flights(NYC, LON, Dec15) || get_exchange_rate(USD, GBP)
Step 2 (depends on Step 1): get_hotels(LON, flight_arrival_time)
Step 3 (depends on Step 1+2): calculate_total(flight_price, hotel_price, exchange_rate)
Step 4 (depends on Step 3): book_itinerary(flight_id, hotel_id, total)
```

### 🔹 Visual Explanation (Text-Based)

```
API Orchestration DAG:

  ┌──────────────┐     ┌──────────────────┐
  │ get_flights  │     │ get_exchange_rate │
  │   (NYC→LON)  │     │   (USD→GBP)      │
  └──────┬───────┘     └────────┬─────────┘
         │                      │
         │ (parallel)           │
         ▼                      │
  ┌──────────────┐              │
  │ get_hotels   │              │
  │ (LON, arrival)│             │
  └──────┬───────┘              │
         │                      │
         └──────────┬───────────┘
                    ▼
         ┌──────────────────┐
         │ calculate_total  │
         └────────┬─────────┘
                  ▼
         ┌──────────────────┐
         │ book_itinerary   │
         └──────────────────┘
```

---

### 🔹 Scenario-Based Question

**Q: Your tool-calling agent orchestrates 5 API calls for a customer request. API #3 (payment processing) intermittently fails with a 503 error. The agent retries indefinitely, causing the user to wait 30+ seconds. How do you make the orchestration resilient?**

### 🔹 Scenario Answer

1. **Immediate fixes:**
   - **Retry limit:** Max 3 retries with exponential backoff (1s, 2s, 4s)
   - **Per-call timeout:** 5 seconds per API call
   - **Total workflow timeout:** 15 seconds max. Return partial results if exceeded.

2. **Circuit breaker pattern:**
   - Track failure rate for each API over a sliding window
   - If payment API fails > 50% in last 5 minutes, "open" the circuit — skip the call entirely
   - Return a graceful degradation: "Payment processing is temporarily unavailable. Your order has been saved, and we'll process payment when the service recovers."

3. **Fallback strategies:**
   - Primary: Stripe API → Fallback: PayPal API
   - If both fail: Queue the payment for async processing, confirm the order tentatively.

4. **User experience:**
   - Stream partial results: "I've found your flights and hotels. Processing payment..." (don't make the user wait in silence)
   - Provide estimated wait time and timeout behavior

### 🔹 Scenario Example

```
Resilient API Orchestration:

  API #1: get_user() ──▶ ✅ (200ms)
  API #2: get_order() ──▶ ✅ (350ms)
  API #3: process_payment()
          Attempt 1: 503 (1s timeout)
          Wait 1s...
          Attempt 2: 503 (1s timeout)
          Wait 2s...
          Attempt 3: ✅ (800ms)
  API #4: send_confirmation() ──▶ ✅ (200ms)
  
  Total: ~6s (acceptable)

  If all 3 retries fail:
  → Circuit breaker opens
  → Fallback: queue_payment_async()
  → Return: "Order confirmed! Payment will be processed shortly."
```

### 🔹 Visual Explanation (Scenario)

```
Resilient Orchestration Pattern:

  API Call ──▶ Execute
                │
           ┌────┴────┐
           ▼         ▼
        Success    Failure
           │         │
           ▼         ▼
        Return    Retry? (count < max?)
        result       │
                ┌────┴────┐
                ▼         ▼
              Yes        No
                │         │
                ▼         ▼
           Backoff    Circuit Open?
           & Retry       │
                    ┌────┴────┐
                    ▼         ▼
                  Yes        No
                    │         │
                    ▼         ▼
              Skip call   Fallback API
              (degraded)     │
                    │    ┌───┴───┐
                    │    ▼       ▼
                    │  Success  Fail
                    │    │       │
                    │    ▼       ▼
                    │  Return  Queue
                    └────┤    async
                         ▼
                   Return partial
                   result to user
```

---

## Q3 — Parallel Tool Execution 🟡 Medium

### 🔹 Conceptual Question

**Q: How do modern LLMs support parallel tool calling, and what are the challenges of executing multiple tool calls simultaneously?**

### 🔹 Answer

**Parallel tool calling** allows the LLM to request multiple tool executions in a single turn, rather than one at a time.

**How it works (OpenAI example):**
- The LLM returns an array of tool calls instead of a single one
- The application executes all calls in parallel
- All results are returned to the LLM in a single message
- The LLM synthesizes the results

**Benefits:**
- Reduces round trips between LLM and application (latency savings)
- Takes advantage of independent data fetches
- More efficient token usage (one reasoning step instead of multiple)

**Challenges:**

1. **Dependency detection:** The LLM must correctly identify which calls are independent. If it parallelizes dependent calls, results will be wrong.
2. **Error handling:** If 2 of 3 parallel calls succeed and 1 fails, how do you handle it? Options: return partial results, retry failed call, or fail the whole batch.
3. **Rate limiting:** Parallel calls may hit API rate limits simultaneously.
4. **Result aggregation:** The LLM must correctly map results back to the right tool call (especially when results are similar in structure).
5. **Cost amplification:** Parallel calls mean more API costs simultaneously.

**Best practices:**
- Validate that parallel calls are truly independent before executing
- Implement per-tool rate limiting
- Return results with clear labels mapping to the original call
- Set reasonable parallelism limits (e.g., max 5 concurrent calls)

### 🔹 Example

User: "Compare the weather in Tokyo, London, and New York."

Sequential (3 round trips):
- Call 1: get_weather("Tokyo") → wait → return → LLM → Call 2: get_weather("London") → ...
- Total: ~9 seconds (3 × 3s)

Parallel (1 round trip):
- LLM returns: [get_weather("Tokyo"), get_weather("London"), get_weather("New York")]
- Execute all 3 simultaneously
- Return all results to LLM
- Total: ~4 seconds (3s parallel + 1s LLM)

### 🔹 Visual Explanation (Text-Based)

```
Sequential Tool Calling:
  LLM → Tool1 → LLM → Tool2 → LLM → Tool3 → LLM → Answer
  |--3s--|--1s--|--3s--|--1s--|--3s--|--1s--|  = 12s total

Parallel Tool Calling:
  LLM → [Tool1, Tool2, Tool3] → LLM → Answer
  |--1s--|------3s------|--1s--|  = 5s total
         (all 3 execute simultaneously)
```

---

### 🔹 Scenario-Based Question

**Q: Your LLM agent uses parallel tool calling to fetch data from 4 microservices. Under load, one microservice becomes slow (P99 latency jumps from 200ms to 5s). Since you're calling all 4 in parallel, the entire request is bottlenecked by the slowest service. How do you handle this?**

### 🔹 Scenario Answer

1. **Individual timeouts:** Set a 2-second timeout per service call. If the slow service doesn't respond in 2s, return a timeout error for that specific call.

2. **Partial result handling:** Configure the LLM to handle partial data gracefully:
   - System prompt: "If some data sources are unavailable, provide the best answer with available data and note what's missing."
   - Example: "Here's the analysis based on 3 of 4 data sources. Customer satisfaction data is temporarily unavailable."

3. **Deadline propagation:** Set an overall deadline (e.g., 3 seconds for all parallel calls). As the deadline approaches, cancel remaining slow calls.

4. **Priority-based execution:** Not all data is equally important. Categorize:
   - **Critical:** Must have for a response (e.g., user account data)
   - **Nice-to-have:** Enhances response but not required (e.g., recommendations)
   - If critical services respond but nice-to-have times out → proceed without it.

5. **Async backfill:** For the slow service, return a partial response immediately and update asynchronously when the slow data arrives (works for streaming responses or UI updates).

### 🔹 Scenario Example

```
Parallel call with timeout handling:

  ┌──── Service A: 150ms ✅ ─────────────────────────┐
  ├──── Service B: 180ms ✅ ─────────────────────────┤
  ├──── Service C: 5000ms ⏰ timeout at 2000ms ─────┤
  └──── Service D: 220ms ✅ ─────────────────────────┘
                                                      │
  Total wait: 2000ms (capped by timeout)              │
                                                      ▼
  LLM receives: {A: data, B: data, C: "timeout", D: data}
  Response: "Based on available data (A, B, D)... Note: Service C data 
             is temporarily unavailable."
```

### 🔹 Visual Explanation (Scenario)

```
Timeout-Aware Parallel Execution:

  Time: 0ms                    2000ms (deadline)      5000ms
        │                         │                      │
  Svc A ████ 150ms ✅              │                      │
  Svc B █████ 180ms ✅             │                      │
  Svc C ██████████████████████████ ⏰ cancelled           │████ (would finish here)
  Svc D ██████ 220ms ✅            │                      │
        │                         │                      │
        └─── Return partial ──────┘                      
             results at 2s
```

---

## Q4 — Tool Selection Strategies 🔴 Hard

### 🔹 Conceptual Question

**Q: What strategies can you use to improve an LLM's tool selection accuracy in a system with many available tools? Compare prompt-based, embedding-based, and fine-tuning approaches.**

### 🔹 Answer

| Strategy | How It Works | Accuracy | Latency | Effort |
|----------|-------------|----------|---------|--------|
| **Prompt engineering** | Detailed tool descriptions + few-shot examples in the prompt | Medium-High | Low (no extra step) | Low |
| **Embedding-based routing** | Embed user query, match to tool description embeddings, send top-K tools | High | +50-100ms | Medium |
| **Classifier-based** | Train a classifier to predict the correct tool(s) from the query | High | +20-50ms | High |
| **Fine-tuning** | Fine-tune the LLM on (query, correct_tool) examples | Highest | None (built in) | Highest |
| **Hierarchical selection** | Category first, then specific tool | High | +1 LLM call | Medium |

**Prompt engineering best practices:**
- Use contrastive descriptions: "Use X for A, NOT for B"
- Include 3-5 few-shot examples of correct tool selection
- Group related tools logically

**Embedding-based routing:**
- Pre-compute embeddings for each tool description
- At query time, embed the query and find top-5 most similar tools
- Send only those 5 tools to the LLM
- Fast, scalable, works well for 100+ tools

**Fine-tuning approach:**
- Collect (query, selected_tool, arguments) training data from production logs
- Fine-tune the model on tool selection task
- Highest accuracy but requires ongoing data collection and retraining

**Hybrid recommendation:** Start with prompt engineering. If accuracy < 85%, add embedding-based routing. If still insufficient, fine-tune.

### 🔹 Example

System with 100 tools, embedding-based routing:
```
Query: "Send an email to John about the Q3 report"
Query embedding → cosine similarity with 100 tool embeddings
Top 5 matches: [send_email (0.92), create_draft (0.85), get_contacts (0.78), 
                schedule_meeting (0.65), search_docs (0.61)]
Send these 5 to LLM → LLM correctly selects send_email
```

### 🔹 Visual Explanation (Text-Based)

```
Embedding-Based Tool Selection:

  100 Tool Descriptions ──▶ Pre-compute embeddings ──▶ Tool Embedding Index
                                                            │
  User Query ──▶ Embed query ──▶ Cosine similarity ─────────┘
                                        │
                                        ▼
                                  Top-5 tools by similarity
                                        │
                                        ▼
                                  ┌─────────┐
                                  │   LLM   │──▶ Select tool + generate args
                                  │ (5 tools)│
                                  └─────────┘
```

---

### 🔹 Scenario-Based Question

**Q: You're building a customer service agent with 30 tools. During A/B testing, you find that the agent selects the correct tool 85% of the time, but when it selects the wrong tool, the resulting customer experience is terrible (wrong actions taken on their account). How do you get to 99% tool selection accuracy?**

### 🔹 Scenario Answer

1. **Analyze the 15% failure cases:**
   - Are there specific tool pairs that get confused? (e.g., `cancel_order` vs `return_order`)
   - Are failures concentrated on certain query types?
   - Is the LLM selecting a wrong but semantically similar tool?

2. **Layered approach to reach 99%:**

   **Layer 1 — Better descriptions (85% → 90%):**
   - Make confusable tools' descriptions explicitly contrastive
   - Add "when NOT to use this tool" to each description

   **Layer 2 — Confirmation for destructive actions (90% → 95%):**
   - Classify tools by risk: `cancel_order` = high risk, `get_order_status` = low risk
   - For high-risk tools, require the agent to confirm with the user before executing
   - "I'm going to cancel order #12345. Is that correct?"

   **Layer 3 — Validation rules (95% → 98%):**
   - Add business logic validation before execution
   - Example: Can't call `process_refund` if order status isn't "delivered"
   - Catch logically impossible tool calls before they execute

   **Layer 4 — Monitoring & feedback (98% → 99%):**
   - Log every tool selection with outcome
   - Weekly review of misselections, update descriptions
   - A/B test description changes

3. **Fallback for the remaining 1%:**
   - If tool execution produces an unexpected error, don't surface it to the user
   - Graceful degradation: "I wasn't able to complete that action. Let me connect you with a human agent."

### 🔹 Scenario Example

```
Confusion pair: cancel_order vs return_order

BEFORE:
  cancel_order: "Cancel a customer's order"
  return_order: "Process a return for a customer"
  Query: "I want to send back my order" → Agent calls cancel_order ❌

AFTER (contrastive descriptions):
  cancel_order: "Cancel an order BEFORE it ships. Use when the order 
                 hasn't been delivered yet. NOT for returns."
  return_order: "Process a return AFTER delivery. Use when customer 
                 received the item and wants to send it back."
  Query: "I want to send back my order" → Agent calls return_order ✅

Additional safeguard:
  Validation rule: cancel_order requires order.status == "processing"
  If order.status == "delivered" → block cancel, suggest return_order
```

### 🔹 Visual Explanation (Scenario)

```
Multi-Layer Tool Selection Safety:

  User Query ──▶ LLM Tool Selection
                       │
                       ▼
               ┌──────────────┐
               │ Layer 1:     │
               │ Improved     │──▶ Correct tool? (90% yes)
               │ Descriptions │
               └──────┬───────┘
                      │
                      ▼
               ┌──────────────┐
               │ Layer 2:     │
               │ Risk-based   │──▶ High risk? → Confirm with user
               │ Confirmation │
               └──────┬───────┘
                      │
                      ▼
               ┌──────────────┐
               │ Layer 3:     │
               │ Business     │──▶ Valid action? (status checks)
               │ Validation   │
               └──────┬───────┘
                      │
                      ▼
               ┌──────────────┐
               │ Layer 4:     │
               │ Execute +    │──▶ Monitor outcome, log for analysis
               │ Monitor      │
               └──────────────┘
```

---

## Q5 — Failure Handling in Tool Calling 🔴 Hard

### 🔹 Conceptual Question

**Q: What are the common failure modes in LLM tool calling systems, and how do you build a robust error-handling strategy?**

### 🔹 Answer

**Common failure modes:**

1. **Wrong tool selected:** LLM picks `delete_user` instead of `deactivate_user`
2. **Invalid arguments:** LLM passes wrong types or missing required params
3. **Tool execution failure:** API returns 500, timeout, or rate limit error
4. **Hallucinated tools:** LLM calls a function that doesn't exist
5. **Infinite tool loops:** LLM keeps calling the same tool without progress
6. **Partial failures:** In parallel execution, some tools succeed and some fail
7. **Stale results:** Cached tool results are outdated

**Robust error-handling strategy:**

| Failure | Prevention | Recovery |
|---------|-----------|----------|
| Wrong tool | Better descriptions, validation layer | Undo action, retry with correct tool |
| Invalid args | Schema validation before execution | Return clear error to LLM, let it retry |
| Execution failure | Timeouts, circuit breaker | Retry with backoff, fallback tool |
| Hallucinated tool | Strict allowlist validation | Return "tool not found" to LLM |
| Infinite loop | Max iteration limit, action dedup | Terminate, return best-effort answer |
| Partial failure | Independent error handling per call | Proceed with partial results |
| Stale results | TTL on cache, invalidation triggers | Re-fetch if staleness detected |

**Implementation pattern:**
```
try:
    validate_tool_name(tool_call)      # Is this a real tool?
    validate_arguments(tool_call)       # Are args valid?
    result = execute_with_timeout(tool_call, timeout=5s)
    validate_result(result)             # Is result reasonable?
    return result
except ToolNotFoundError:
    return "Tool does not exist. Available tools: [...]"
except ValidationError as e:
    return f"Invalid arguments: {e}. Expected schema: {schema}"
except TimeoutError:
    return "Tool call timed out. Try a simpler query or different tool."
except APIError:
    if retries < max_retries:
        retry with backoff
    else:
        return fallback_result or "Service temporarily unavailable"
```

### 🔹 Example

A financial agent calls `transfer_funds(from="checking", to="savings", amount="one thousand")`. The validation layer catches that `amount` should be a number, not a string. It returns to the LLM: "Invalid argument: amount must be a number. You provided 'one thousand'." The LLM retries with `amount=1000`.

### 🔹 Visual Explanation (Text-Based)

```
Robust Tool Execution Pipeline:

  LLM Output: {tool: "transfer_funds", args: {...}}
       │
       ▼
  ┌──────────────┐
  │ Tool Exists? │──▶ No ──▶ Return "unknown tool" to LLM
  └──────┬───────┘
         │ Yes
         ▼
  ┌──────────────┐
  │ Args Valid?  │──▶ No ──▶ Return schema + error to LLM
  └──────┬───────┘
         │ Yes
         ▼
  ┌──────────────┐
  │ Execute      │──▶ Timeout ──▶ Retry (max 3) ──▶ Fallback
  │ (with timeout)│
  └──────┬───────┘──▶ API Error ──▶ Retry ──▶ Circuit breaker
         │ Success
         ▼
  ┌──────────────┐
  │ Result Valid?│──▶ No ──▶ Return "unexpected result" to LLM
  └──────┬───────┘
         │ Yes
         ▼
  Return result to LLM
```

---

### 🔹 Scenario-Based Question

**Q: Your production agent processes 10,000 tool calls per day. You discover that 3% of tool calls fail silently — the LLM generates a plausible-looking response even when the tool call returns an error. Users don't realize they're getting wrong information. How do you detect and prevent silent failures?**

### 🔹 Scenario Answer

1. **Detection:**
   - **Structured result validation:** Every tool must return results in a standard format: `{status: "success|error", data: {...}, error_message: "..."}`
   - **Result-response correlation:** Compare tool results with the LLM's final response. If the tool returned an error but the response doesn't mention it, flag as a silent failure.
   - **Anomaly detection:** Track tool success rates. If a tool's success rate drops, alert immediately.

2. **Prevention:**
   - **Force error acknowledgment:** In the system prompt: "If ANY tool call fails or returns an error, you MUST inform the user. Never fabricate data to replace a failed tool call."
   - **Error injection in prompt:** Include the error status prominently in the tool result: "⚠️ ERROR: This tool call failed. Do not use fabricated data."
   - **Post-generation verification:** After the LLM generates a response, check: did any tool calls fail? Does the response acknowledge the failure? If not, regenerate with an explicit error reminder.

3. **Monitoring dashboard:**
   - Track: tool call success rate, silent failure rate, user-reported issues
   - Alert on: success rate drop, silent failure rate > 1%, anomalous patterns

### 🔹 Scenario Example

```
Silent Failure Example:
  Tool call: get_account_balance("user_123")
  Tool result: {status: "error", error: "database timeout"}
  LLM response: "Your account balance is $5,432.10" ← FABRICATED! 💀

After Fix:
  Tool result: "⚠️ TOOL ERROR: get_account_balance failed (database timeout). 
                DO NOT fabricate a balance. Inform the user."
  LLM response: "I'm sorry, I'm unable to retrieve your account balance 
                  right now due to a temporary system issue. Please try 
                  again in a few minutes or contact support."
```

### 🔹 Visual Explanation (Scenario)

```
Silent Failure Detection Pipeline:

  Tool Call ──▶ Execute ──▶ Result
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                 Success             Error
                    │                   │
                    ▼                   ▼
              Return data        Return error with ⚠️ prefix
                    │                   │
                    └─────────┬─────────┘
                              ▼
                        LLM Response
                              │
                              ▼
                    ┌──────────────────┐
                    │ Post-Generation  │
                    │ Validator        │
                    ├──────────────────┤
                    │ Any tool errors? │
                    │ Response acks    │
                    │ the error?       │
                    └────────┬─────────┘
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
                  Yes               No (silent failure!)
                    │                 │
                    ▼                 ▼
                 Return          Regenerate with
                 response        explicit error context
```

---

*Follow-up Questions to Expect:*
- "How do you handle tool calling with streaming responses?" → Buffer tool calls until complete, execute tools, then stream the final response. Or use a two-phase approach: Phase 1 (non-streaming) for tool calls, Phase 2 (streaming) for the response.
- "What's the token overhead of function calling?" → Each tool definition uses ~100-300 tokens. 10 tools = ~1,000-3,000 tokens. Use dynamic tool loading to minimize overhead.
- "How do you test tool calling agents?" → Mock tools with predefined responses, test edge cases (empty results, errors, timeouts), use trajectory evaluation (was the right sequence of tools called?).
