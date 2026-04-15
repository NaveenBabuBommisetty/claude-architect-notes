# Domain 1: Agentic Architecture & Orchestration
## Weight: 27% — Most Important Domain

---

## What This Domain Tests

Your ability to **design, build, and debug multi-agent systems** that work reliably in production.
Think of this as the "system design" interview for Claude-based applications.

---

## 1.1 The Agentic Loop — The Core Mental Model

### What Is an Agentic Loop?

An **agentic loop** is the cycle Claude runs through when completing a task autonomously:

```
┌─────────────────────────────────────────────────────┐
│                   AGENTIC LOOP                      │
│                                                     │
│  User Task → Claude Reasons → Tool Call?            │
│                    ↓              ↓                 │
│               Respond         Execute Tool          │
│               Directly        ↓                     │
│                           Tool Result               │
│                               ↓                     │
│                        Continue Reasoning           │
│                               ↓                     │
│                    Task Complete / Escalate          │
└─────────────────────────────────────────────────────┘
```

### The 4 Stop Reasons — MEMORIZE THESE

When Claude stops generating, there are exactly 4 reasons. The exam tests all of them.

| `stop_reason` | What It Means | What You Should Do |
|---------------|---------------|-------------------|
| `end_turn` | Claude finished naturally | Return response to user |
| `tool_use` | Claude wants to call a tool | Execute the tool, return result |
| `max_tokens` | Hit token limit mid-response | Increase limit or handle truncation |
| `stop_sequence` | Hit a custom stop sequence | Handle based on your stop sequence logic |

### Code Example — Handling the Agentic Loop

```python
import anthropic
import json

client = anthropic.Anthropic()

def run_agentic_loop(user_task: str, tools: list, max_iterations: int = 10):
    """
    A production-safe agentic loop with iteration guard.
    """
    messages = [{"role": "user", "content": user_task}]
    iteration = 0

    while iteration < max_iterations:
        iteration += 1

        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        # Check stop reason — this is what the exam focuses on
        stop_reason = response.stop_reason

        if stop_reason == "end_turn":
            # Claude is done — extract final text response
            final_text = next(
                block.text for block in response.content
                if block.type == "text"
            )
            return {"status": "complete", "result": final_text}

        elif stop_reason == "tool_use":
            # Claude wants a tool — find it and call it
            tool_use_block = next(
                block for block in response.content
                if block.type == "tool_use"
            )

            tool_name = tool_use_block.name
            tool_input = tool_use_block.input
            tool_use_id = tool_use_block.id

            # Execute the tool
            tool_result = execute_tool(tool_name, tool_input)

            # Append Claude's response and tool result to message history
            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": tool_use_id,
                    "content": json.dumps(tool_result)
                }]
            })
            # Loop continues — Claude will reason about the tool result

        elif stop_reason == "max_tokens":
            # Handle truncation — production critical
            return {"status": "error", "reason": "max_tokens_exceeded"}

        else:
            # stop_sequence or unexpected
            return {"status": "stopped", "reason": stop_reason}

    # Safety exit — prevents infinite loops in production
    return {"status": "error", "reason": "max_iterations_exceeded"}


def execute_tool(name: str, input_data: dict) -> dict:
    """Dispatch tool calls to actual implementations."""
    # Your tool registry here
    tool_registry = {
        "search_web": search_web,
        "read_file": read_file,
    }
    if name in tool_registry:
        return tool_registry[name](**input_data)
    return {"error": f"Unknown tool: {name}"}
```

### Key Concept — Loop Termination Guard

**Always implement a max_iterations guard.** An infinite agentic loop is a production failure.
The exam will present scenarios where infinite loops occur — you must identify the missing guard.

---

## 1.2 Hub-and-Spoke Orchestration

### The Pattern

This is the **dominant architecture pattern** for complex multi-agent systems. The exam heavily tests this.

```
                    ┌──────────────┐
                    │   HUB AGENT  │
                    │(Coordinator) │
                    │  - Receives  │
                    │    task      │
                    │  - Decomposes│
                    │  - Synthesizes│
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │  SUBAGENT A │  │  SUBAGENT B │  │  SUBAGENT C │
   │  (Research) │  │  (Analysis) │  │  (Writing)  │
   │             │  │             │  │             │
   │ Scoped tools│  │ Scoped tools│  │ Scoped tools│
   │ Scoped ctx  │  │ Scoped ctx  │  │ Scoped ctx  │
   └─────────────┘  └─────────────┘  └─────────────┘
```

### Rules of Hub-and-Spoke

1. **Hub decomposes tasks** — takes complex user requests, breaks into subtasks
2. **Hub delegates to subagents** — each subagent handles ONE specific subtask
3. **Hub synthesizes results** — collects subagent outputs, produces final result
4. **Context isolation is mandatory** — subagents get ONLY what they need (NOT the full hub context)

### Why Context Isolation Matters (Exam Critical)

```python
# WRONG — Anti-Pattern 1: Passing full hub context to subagent
def invoke_subagent_wrong(hub_messages: list, subtask: str):
    subagent_messages = hub_messages  # ← NEVER do this
    subagent_messages.append({"role": "user", "content": subtask})
    # Problem: subagent gets irrelevant context → reasoning degrades
    # Problem: tokens wasted on irrelevant history → higher cost


# CORRECT — Context isolation: subagent gets only what it needs
def invoke_subagent_correct(subtask: str, required_context: dict):
    # Build a fresh, minimal context for this specific subtask
    system_prompt = f"""You are a specialist agent for: {subtask}
    Relevant context only: {required_context}
    Do not ask for more information. Complete only your assigned subtask."""

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        system=system_prompt,
        messages=[{"role": "user", "content": subtask}]
    )
    return response
```

### Full Hub-and-Spoke Example

```python
import anthropic
import json
from concurrent.futures import ThreadPoolExecutor

client = anthropic.Anthropic()

def hub_agent(user_request: str) -> str:
    """
    Coordinator agent — decomposes task, delegates to subagents,
    synthesizes final response.
    """

    # Step 1: Hub decomposes the task
    decomposition_prompt = f"""
    Analyze this task and decompose it into independent subtasks.
    Return JSON with format:
    {{"subtasks": [{{"id": "1", "name": "...", "description": "...", "required_context": {{}}}}]}}

    Task: {user_request}
    """

    decomp_response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": decomposition_prompt}]
    )
    subtasks = json.loads(decomp_response.content[0].text)["subtasks"]

    # Step 2: Identify independent vs. dependent subtasks
    # Independent subtasks → run in parallel (ThreadPoolExecutor)
    # Dependent subtasks → run sequentially

    # Step 3: Invoke subagents (parallel for independent tasks)
    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = {
            executor.submit(subagent, st["description"], st["required_context"]): st
            for st in subtasks
        }
        results = {}
        for future, subtask in futures.items():
            results[subtask["id"]] = future.result()

    # Step 4: Hub synthesizes results
    synthesis_prompt = f"""
    Synthesize these subagent results into a final response for the user.

    Original request: {user_request}
    Subagent results: {json.dumps(results, indent=2)}

    Produce a coherent, unified response.
    """

    final_response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": synthesis_prompt}]
    )
    return final_response.content[0].text


def subagent(subtask: str, context: dict) -> str:
    """
    Specialist agent — receives ONLY its specific subtask and minimal context.
    No access to hub conversation history.
    """
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        system=f"You are a specialist agent. Context: {json.dumps(context)}",
        messages=[{"role": "user", "content": subtask}]
    )
    return response.content[0].text
```

---

## 1.3 Session State & Task Decomposition

### Why External State Management Matters

Claude's context window is **not a database**. If a long-running agent crashes or times out, you lose all progress. Production agents must store state externally.

```python
import uuid
import json
import redis  # or any key-value store

redis_client = redis.Redis()

def start_agent_session(user_task: str) -> str:
    """Create a resumable agent session."""
    session_id = str(uuid.uuid4())

    session_state = {
        "session_id": session_id,
        "original_task": user_task,
        "status": "in_progress",
        "subtasks": [],
        "completed_subtasks": [],
        "checkpoint": None
    }

    redis_client.set(f"session:{session_id}", json.dumps(session_state))
    return session_id


def checkpoint_session(session_id: str, completed_subtask_id: str, result: dict):
    """Save progress so the agent can resume after failure."""
    raw = redis_client.get(f"session:{session_id}")
    state = json.loads(raw)

    state["completed_subtasks"].append(completed_subtask_id)
    state["checkpoint"] = {
        "last_completed": completed_subtask_id,
        "result": result
    }

    redis_client.set(f"session:{session_id}", json.dumps(state))


def resume_agent_session(session_id: str) -> dict:
    """Resume an interrupted session from last checkpoint."""
    raw = redis_client.get(f"session:{session_id}")
    if not raw:
        raise ValueError(f"Session {session_id} not found")

    state = json.loads(raw)
    completed = set(state["completed_subtasks"])

    # Filter to only uncompleted subtasks
    remaining = [st for st in state["subtasks"] if st["id"] not in completed]

    return {
        "session_id": session_id,
        "remaining_subtasks": remaining,
        "checkpoint": state["checkpoint"]
    }
```

### Parallel vs. Sequential Subtasks

```
Task: "Research competitors AND write a report based on the research"

     ┌─────────────────┐
     │  Research Task  │ ← Must complete FIRST (dependency)
     └────────┬────────┘
              │ result feeds into
     ┌────────▼────────┐
     │  Writing Task   │ ← Sequential (depends on research)
     └─────────────────┘

Task: "Research competitors AND analyze our financials"

     ┌─────────────────┐    ┌─────────────────┐
     │  Research Task  │    │ Financial Task  │ ← Both independent
     └────────┬────────┘    └────────┬────────┘   Run in PARALLEL
              └───────────┬──────────┘
                   ┌──────▼──────┐
                   │    Hub      │
                   │ Synthesizes │
                   └─────────────┘
```

---

## 1.4 Lifecycle Hooks

### Three Hook Types

Lifecycle hooks run at specific points in the agent execution lifecycle. Design them to be **side-effect free and fast**.

```python
from typing import Callable, Any
from datetime import datetime
import logging

class AgentLifecycleHooks:
    """
    Lifecycle hooks for production agent execution.
    Pre → Execute → Post (or Error hook on failure)
    """

    def __init__(self, agent_id: str):
        self.agent_id = agent_id
        self.logger = logging.getLogger(f"agent.{agent_id}")

    # ── PRE-EXECUTION HOOKS ──────────────────────────────────────────

    def pre_execution(self, task: dict, user_id: str) -> None:
        """
        Runs BEFORE agent starts.
        Use for: input validation, permission checks, task logging.
        """
        # 1. Validate inputs
        self._validate_task_inputs(task)

        # 2. Check permissions
        self._check_user_permissions(user_id, task["required_permissions"])

        # 3. Log task initiation (for audit trail)
        self.logger.info({
            "event": "task_started",
            "agent_id": self.agent_id,
            "task_id": task["id"],
            "user_id": user_id,
            "timestamp": datetime.utcnow().isoformat()
        })

    # ── POST-EXECUTION HOOKS ─────────────────────────────────────────

    def post_execution(self, task: dict, result: dict) -> None:
        """
        Runs AFTER agent completes successfully.
        Use for: output validation, downstream triggers, result storage.
        """
        # 1. Validate output
        self._validate_output(result)

        # 2. Store result
        self._store_result(task["id"], result)

        # 3. Trigger downstream agents if needed
        if result.get("requires_followup"):
            self._trigger_downstream_agent(task, result)

        # 4. Log completion
        self.logger.info({
            "event": "task_completed",
            "task_id": task["id"],
            "result_status": result["status"]
        })

    # ── ERROR HOOKS ──────────────────────────────────────────────────

    def on_error(self, task: dict, error: Exception, retry_count: int) -> str:
        """
        Runs when the agent encounters an error.
        Returns: 'retry', 'escalate', or 'abort'
        """
        self.logger.error({
            "event": "task_error",
            "task_id": task["id"],
            "error": str(error),
            "retry_count": retry_count
        })

        # Classify error type
        if self._is_transient_error(error) and retry_count < 3:
            return "retry"  # Retry with exponential backoff
        elif self._requires_human_decision(error):
            self._escalate_to_human(task, error)
            return "escalate"
        else:
            return "abort"

    def _is_transient_error(self, error: Exception) -> bool:
        """Timeouts, rate limits = transient (retry eligible)."""
        transient_types = ["TimeoutError", "RateLimitError", "ConnectionError"]
        return type(error).__name__ in transient_types

    def _requires_human_decision(self, error: Exception) -> bool:
        """Permission errors, scope violations = escalate."""
        return "PermissionError" in type(error).__name__
```

---

## 1.5 Human-in-the-Loop Escalation

### The 3 Valid Escalation Triggers — MEMORIZE FOR EXAM

The exam specifically tests these three. Other answers will look plausible — they are traps.

| # | Trigger | Example |
|---|---------|---------|
| 1 | Tool call fails with non-retriable error after retry exhaustion | API returns 403 after 3 retries |
| 2 | Confidence score falls below threshold on a sensitive action | Score: 0.45 for a financial transaction |
| 3 | Task requires decision outside agent's authorized scope | Agent asked to delete production data |

```python
class HumanEscalationManager:
    """
    Determines when to escalate to a human operator.
    Production systems must define clear escalation triggers.
    """

    CONFIDENCE_THRESHOLD = 0.80  # Below this → escalate on sensitive actions
    MAX_RETRIES = 3

    def should_escalate(
        self,
        action_type: str,
        confidence: float,
        error_history: list,
        action_scope: str,
        authorized_scopes: list
    ) -> tuple[bool, str]:
        """
        Returns (should_escalate: bool, reason: str)
        """

        # Trigger 1: Non-retriable error after retry exhaustion
        non_retriable_errors = [e for e in error_history if not e["retriable"]]
        if len(error_history) >= self.MAX_RETRIES or non_retriable_errors:
            return True, "retry_exhaustion"

        # Trigger 2: Low confidence on sensitive action
        sensitive_actions = ["financial_transaction", "data_deletion", "email_send"]
        if action_type in sensitive_actions and confidence < self.CONFIDENCE_THRESHOLD:
            return True, f"low_confidence_{confidence:.2f}"

        # Trigger 3: Outside authorized scope
        if action_scope not in authorized_scopes:
            return True, f"unauthorized_scope_{action_scope}"

        return False, "none"


    def escalate(self, task: dict, reason: str, context: dict):
        """
        Pause the agent and notify a human operator.
        The agent MUST NOT proceed until a human responds.
        """
        escalation_event = {
            "task_id": task["id"],
            "reason": reason,
            "context": context,
            "status": "awaiting_human_decision",
            "created_at": datetime.utcnow().isoformat()
        }

        # In production: push to Slack/PagerDuty/email
        # Store escalation state so agent can resume after human responds
        self._notify_human_operator(escalation_event)
        self._pause_agent(task["id"])
```

### What Does NOT Trigger Escalation (Exam Trap)

- Agent successfully completing a task with moderate confidence ← NOT escalation
- Empty result returned from a tool ← NOT an error, NOT escalation
- A long-running task taking more time than expected ← NOT escalation
- A tool returning a structured error that is retry-eligible ← Retry first, escalate only after exhaustion

---

## Domain 1 — Concept Check Questions

Test yourself before moving to the next domain.

1. What are the 4 values of `stop_reason`? What should your code do for each?
2. In hub-and-spoke, what context should a subagent receive?
3. Why is the max_iterations guard critical in an agentic loop?
4. Name the 3 valid human escalation triggers.
5. What is the difference between parallel and sequential subtask execution? When do you use each?
6. Where should long-running agent session state be stored? Why not in Claude's context window?
7. What hook runs before execution? What does it do?

---

## Domain 1 Summary — Key Rules

| Rule | Detail |
|------|--------|
| Always guard loops | `max_iterations` prevents infinite loops |
| Isolate subagent context | Never pass full hub context to subagents |
| Store state externally | Redis/DB, not context window |
| Parallel for independent | ThreadPoolExecutor for independent subtasks |
| Sequential for dependent | Task B depends on Task A's output |
| 3 escalation triggers only | Error exhaustion, low confidence, scope violation |
| Pre/Post/Error hooks | Validate, log, trigger, retry, escalate |
