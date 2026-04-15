# The 7 Critical Anti-Patterns
## Exam Trap Reference Guide

---

## Why Anti-Patterns Matter on the Exam

The exam guide explicitly states:
> "Many wrong exam answers are not random — they are architectural anti-patterns that
>  seem reasonable before you understand production implications."

The correct answer to many exam questions is: **identify which anti-pattern the wrong options represent.**
Study all 7 until you can immediately recognize them.

---

## Anti-Pattern 1: Passing Full Hub Context to Subagents

### What It Is
Giving subagents the entire coordinator (hub) conversation history instead of only
the context they need for their specific subtask.

### Why It Happens
It seems "helpful" to give subagents more context. Developers assume more context = better results.

### Why It's Wrong
- **Context bloat**: Subagents waste tokens processing irrelevant information
- **Reasoning degradation**: Irrelevant context confuses subagent reasoning
- **Cost increase**: Every extra token costs money in both input processing and output quality
- **Context leakage**: Sensitive information from other subtasks flows to unrelated subagents

### The Fix

```python
# WRONG
def invoke_subagent(hub_messages, subtask):
    messages = hub_messages + [{"role": "user", "content": subtask}]
    # ← Subagent gets 10,000 tokens of irrelevant hub history

# CORRECT
def invoke_subagent(subtask, required_context):
    system = f"You are a specialist. Context: {required_context}"
    messages = [{"role": "user", "content": subtask}]
    # ← Subagent gets only what it needs
```

### Exam Recognition
If an answer says "pass the full conversation history to the subagent" → it's **wrong**.

---

## Anti-Pattern 2: Treating Empty Results as Errors

### What It Is
Triggering retry logic or error handling when a query returns zero results.

### Why It Happens
Developers think "no results" means something went wrong.

### Why It's Wrong
- `"No customers found"` is a **valid application state**, not a failure
- Retrying empty results wastes tokens and hits rate limits
- The application should handle empty results explicitly in its own logic
- Retry logic should be reserved for actual errors (timeouts, permission failures)

### The Fix

```python
# WRONG — treats empty as error
def search_customers(query):
    results = db.search(query)
    if not results:
        raise ValueError("No results found")  # ← Wrong! This is valid!
    return results

# CORRECT — handles empty as valid state
def search_customers(query):
    results = db.search(query)
    return {
        "results": results,
        "count": len(results),
        "message": "No customers found matching your query" if not results else None
        # ← No error raised — caller handles the empty case
    }
```

### Exam Recognition
If an answer says "retry when no results are returned" → it's **wrong**.

---

## Anti-Pattern 3: Vague Calibration Instructions

### What It Is
Using instructions like "be conservative", "only flag high-confidence issues",
or "use your judgment" instead of specific, measurable criteria.

### Why It Happens
Developers write these because they sound like good guidelines. In human communication, they work.
With Claude, they don't — Claude cannot calibrate vague directives.

### Why It's Wrong
- Claude has no way to determine what "conservative" means in your specific context
- Results become unpredictable and non-reproducible
- The same input produces different outputs across runs
- "High-confidence" means different things in different contexts

### The Fix

```python
# WRONG
system = """
Review code for security issues.
Be conservative. Only flag high-confidence vulnerabilities.
"""

# CORRECT — specific, measurable, reproducible
system = """
Review code for security vulnerabilities.

Flag as CRITICAL if:
- User input is concatenated directly into a SQL query (SQL injection)
- User input is rendered as HTML without sanitization (XSS)
- API keys or passwords are hardcoded as string literals

Flag as WARNING if:
- A function has no input validation for external data
- Error messages expose internal file paths or stack traces

Do NOT flag:
- Code style or best-practice issues
- Performance concerns
- Missing documentation

Return findings as JSON: [{"severity": "CRITICAL|WARNING", "line": N, "issue": "..."}]
"""
```

### Exam Recognition
Answer choices with "be conservative", "use good judgment", or "only high-confidence findings"
are **always wrong** on this exam.

---

## Anti-Pattern 4: Multi-Purpose Tools

### What It Is
Creating tools that perform multiple unrelated functions (e.g., `search_and_update_customer`).

### Why It Happens
Seems efficient to combine related operations into one tool.

### Why It's Wrong
- Ambiguous descriptions: Claude doesn't know when it should search vs. update
- Risk of unintended side effects: Claude might trigger the update when it only meant to search
- Violates single responsibility — harder to test, debug, and audit
- Tool reasoning becomes unreliable with ambiguous tools

### The Fix

```python
# WRONG — multi-purpose
multi_tool = {
    "name": "customer_operations",
    "description": "Search, update, or delete a customer based on the operation parameter",
    # Claude won't know when to search vs. update vs. delete
}

# CORRECT — single purpose tools
tools = [
    {
        "name": "search_customer",
        "description": "Search for a customer by email or ID. READ-ONLY. No side effects.",
    },
    {
        "name": "update_customer_plan",
        "description": "Update a customer's subscription plan. WRITE operation.",
    },
    {
        "name": "deactivate_customer",
        "description": "Deactivate a customer account. IRREVERSIBLE. Requires confirmation.",
    }
]
```

### Exam Recognition
If a tool description says "...and also...", "...or...", "...depending on..." → anti-pattern.

---

## Anti-Pattern 5: No CLAUDE.md Project Context

### What It Is
Running Claude Code without a CLAUDE.md file in the project.

### Why It Happens
Developers don't know about CLAUDE.md, or think Claude will figure it out from the code.

### Why It's Wrong
- Every session starts without architectural context
- Claude generates code inconsistent with your project's conventions
- Naming violations accumulate over time
- Every session requires re-explaining the same rules

### The Fix

```markdown
# CLAUDE.md — Always include at minimum:
## Tech Stack
[Your languages, frameworks, major libraries]

## Architectural Rules
[Critical constraints, patterns, forbidden approaches]

## File Naming Conventions
[Exactly how files should be named]

## Test Standards
[How tests should be written, where they go, what to mock]

## Forbidden Patterns
[What NOT to do — with examples]
```

### Exam Recognition
If a scenario describes a project without CLAUDE.md and asks what's wrong → **missing CLAUDE.md**.
If an answer choice suggests "describe your project at the start of each session" → wrong approach.

---

## Anti-Pattern 6: Synchronous API for Batch Jobs

### What It Is
Using the real-time synchronous API to process large volumes of non-urgent requests
(e.g., nightly analysis of 10,000 documents).

### Why It Happens
The synchronous API is the default. Developers use it for everything without considering cost.

### Why It's Wrong
- Synchronous API costs significantly more than Batch API for high-volume processing
- Batch API is designed exactly for this use case — async, polled, cheaper
- Blocking on synchronous calls for batch work wastes compute and slows pipelines

### The Fix

```python
# WRONG — synchronous for batch (expensive, wrong tool)
def process_10000_documents(documents):
    results = []
    for doc in documents:
        # This calls the synchronous API 10,000 times
        result = client.messages.create(...)
        results.append(result)
    return results


# CORRECT — batch API for non-urgent bulk processing
def process_10000_documents(documents):
    # Submit all 10,000 in a single batch request
    batch = client.beta.messages.batches.create(
        requests=[
            {"custom_id": f"doc_{i}", "params": {...}}
            for i, doc in enumerate(documents)
        ]
    )
    batch_id = batch.id

    # Come back later to collect results (async)
    # In production: schedule a follow-up job to poll
    return batch_id
```

### Decision Rule

```
Is the user waiting for a response RIGHT NOW?
  YES → Synchronous API

Is this background processing, a batch job, or not time-sensitive?
  YES → Batch API
```

### Exam Recognition
Scenarios with nightly jobs, bulk processing, dataset generation → **Batch API**.
Answer choices with synchronous API for these → **wrong**.

---

## Anti-Pattern 7: Reviewing Code in the Same Session That Wrote It

### What It Is
Using the same Claude Code session to both write and review code.

### Why It Happens
Seems efficient — Claude already has context from writing the code.

### Why It's Wrong
- **Context bias**: The session "knows" WHY it made each decision
- This bias makes Claude less likely to question those decisions during review
- It will overlook issues that a fresh perspective would catch
- Code review effectiveness degrades significantly with context bias

### The Fix

```bash
# WRONG
claude> "Write the authentication module"
# ... Claude writes auth.py ...
claude> "Now review auth.py for security issues"
# ← Same session — Claude reviews its own reasoning, not the code

# CORRECT — use separate sessions
# Session 1: Write
claude> "Write the authentication module"
# End session

# Session 2: Review (fresh context)
claude> "Review api/auth/auth.py for security vulnerabilities and edge cases"
# ← No memory of writing it — evaluates purely on code quality
```

### Exam Recognition
If a question asks the best way to review code → **fresh session**.
If an answer says "review in the same session for continuity" → **wrong**.

---

## Quick Reference — All 7 Anti-Patterns

| # | Anti-Pattern | Symptom | Fix |
|---|-------------|---------|-----|
| 1 | Full hub context to subagents | Context bloat, high cost, degraded reasoning | Pass only task-relevant context |
| 2 | Empty results as errors | Unnecessary retries, rate limiting | Handle empty as valid state |
| 3 | Vague calibration | Unpredictable, non-reproducible output | Use specific, measurable criteria |
| 4 | Multi-purpose tools | Wrong tool selected, unintended side effects | One tool = one responsibility |
| 5 | No CLAUDE.md | Inconsistent code, re-explained conventions | Always create CLAUDE.md |
| 6 | Sync API for batch | High cost, inefficient pipeline | Use Batch API |
| 7 | Review in same session | Context bias, missed issues | Always use a fresh session for review |

---

## Exam Strategy for Anti-Pattern Questions

When you see a question about "what is wrong with this design" or "which is the best approach":

1. **Check if the scenario describes one of the 7 anti-patterns** — they appear frequently
2. **Eliminate answers that contain vague language** ("be conservative", "use judgment")
3. **Eliminate answers that pass full context to subagents**
4. **Eliminate answers that treat empty results as errors**
5. **Eliminate multi-purpose tool designs**
6. **Pick the answer that uses Batch API for non-real-time scenarios**
7. **Pick the answer that uses a fresh session for code review**
