# Domain 5: Context & Reliability
## Weight: 15% — Foundational (Failures Here Cascade Everywhere)

---

## What This Domain Tests

Though the smallest domain by weight, **failures in context and reliability cascade into every other domain**.
This domain tests your ability to build agents that don't lose work, handle errors gracefully,
track information provenance, and optimize costs.

---

## 5.1 State Management in Long-Running Agents

### The Core Rule

**Never use Claude's context window as your database.**

The context window is volatile:
- It disappears when the session ends
- It can be lost if the process crashes
- It cannot be resumed by another process

### External State Design

```python
import redis
import json
import uuid
from datetime import datetime
from enum import Enum

class TaskStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    AWAITING_HUMAN = "awaiting_human"

class AgentStateManager:
    """
    External state management for long-running agents.
    Uses Redis as the state store — any persistent storage works.
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.KEY_PREFIX = "agent:session:"

    def create_session(self, user_task: str, user_id: str) -> str:
        """Start a new agent session with a unique, resumable ID."""
        session_id = str(uuid.uuid4())

        state = {
            "session_id": session_id,
            "user_id": user_id,
            "original_task": user_task,
            "status": TaskStatus.PENDING.value,
            "subtasks": [],
            "completed_subtask_ids": [],
            "failed_subtask_ids": [],
            "results": {},
            "created_at": datetime.utcnow().isoformat(),
            "updated_at": datetime.utcnow().isoformat(),
            "checkpoint": None
        }

        self.redis.set(
            f"{self.KEY_PREFIX}{session_id}",
            json.dumps(state),
            ex=86400  # Expire after 24 hours
        )
        return session_id

    def register_subtasks(self, session_id: str, subtasks: list):
        """Register all subtasks before starting execution."""
        state = self._load(session_id)
        state["subtasks"] = subtasks
        state["status"] = TaskStatus.IN_PROGRESS.value
        self._save(session_id, state)

    def checkpoint(self, session_id: str, subtask_id: str, result: dict):
        """
        Save progress after each subtask completes.
        This is the key to resumability — call after EVERY subtask.
        """
        state = self._load(session_id)
        state["completed_subtask_ids"].append(subtask_id)
        state["results"][subtask_id] = result
        state["checkpoint"] = {
            "last_completed_subtask": subtask_id,
            "timestamp": datetime.utcnow().isoformat()
        }
        state["updated_at"] = datetime.utcnow().isoformat()
        self._save(session_id, state)

    def resume(self, session_id: str) -> dict:
        """
        Resume a session after failure or restart.
        Returns only the REMAINING (uncompleted) subtasks.
        """
        state = self._load(session_id)

        completed_ids = set(state["completed_subtask_ids"])
        remaining_subtasks = [
            st for st in state["subtasks"]
            if st["id"] not in completed_ids
        ]

        return {
            "session_id": session_id,
            "status": state["status"],
            "remaining_subtasks": remaining_subtasks,
            "completed_results": state["results"],
            "last_checkpoint": state["checkpoint"]
        }

    def mark_complete(self, session_id: str, final_result: str):
        state = self._load(session_id)
        state["status"] = TaskStatus.COMPLETED.value
        state["final_result"] = final_result
        self._save(session_id, state)

    def _load(self, session_id: str) -> dict:
        raw = self.redis.get(f"{self.KEY_PREFIX}{session_id}")
        if not raw:
            raise ValueError(f"Session not found: {session_id}")
        return json.loads(raw)

    def _save(self, session_id: str, state: dict):
        state["updated_at"] = datetime.utcnow().isoformat()
        self.redis.set(
            f"{self.KEY_PREFIX}{session_id}",
            json.dumps(state),
            ex=86400
        )
```

### Idempotent Tool Calls

```python
# Idempotent = safe to call multiple times without changing the result
# Critical for retry logic — you must be able to retry safely

# NOT idempotent — calling twice sends two emails
def send_email_bad(user_id: str, message: str):
    email_service.send(user_id, message)  # Sends every time called

# IDEMPOTENT — calling twice only sends one email
def send_email_idempotent(user_id: str, message: str, idempotency_key: str):
    # Check if already sent with this key
    if redis.get(f"sent:{idempotency_key}"):
        return {"status": "already_sent", "skipped": True}

    email_service.send(user_id, message)
    redis.set(f"sent:{idempotency_key}", "1", ex=3600)
    return {"status": "sent"}

# When building tools: prefer idempotent designs
# Database upsert (INSERT OR REPLACE) is idempotent
# Append-only operations are NOT idempotent
```

---

## 5.2 Structured Error Propagation

### The Error Taxonomy

Every error in your system must be classified. The classification determines the action.

```
ERROR CLASSIFICATION
│
├── TRANSIENT (retry eligible)
│   ├── Timeout → exponential backoff, then retry
│   ├── Rate limit (429) → wait, then retry
│   └── Connection error → retry immediately, then backoff
│
├── TERMINAL (do NOT retry)
│   ├── Permission denied (403) → escalate to human
│   ├── Not found (404) → handle as valid empty state
│   ├── Validation error → fix input, do not retry same input
│   └── Quota exceeded → escalate, do not retry
│
└── VALID EMPTY RESULT (not an error at all)
    ├── "No results found" → return empty list, handle in app logic
    ├── "No records match" → valid response
    └── "Queue is empty" → valid state
```

### Structured Error Response Format

```python
from dataclasses import dataclass, asdict
from typing import Optional, Any
from enum import Enum

class ErrorCode(Enum):
    # Transient
    TIMEOUT = "TIMEOUT"
    RATE_LIMITED = "RATE_LIMITED"
    CONNECTION_ERROR = "CONNECTION_ERROR"

    # Terminal
    PERMISSION_DENIED = "PERMISSION_DENIED"
    INVALID_INPUT = "INVALID_INPUT"
    QUOTA_EXCEEDED = "QUOTA_EXCEEDED"

    # Not errors
    NOT_FOUND = "NOT_FOUND"  # Handle as valid empty result


@dataclass
class ToolError:
    error_code: str           # Machine-readable (for routing logic)
    message: str              # Human-readable (for logging/display)
    retry_eligible: bool      # True = transient, retry; False = terminal
    details: Optional[dict] = None


def handle_database_error(exception: Exception) -> ToolError:
    """
    Convert raw exceptions to structured ToolError responses.
    Never let raw exceptions propagate to Claude.
    """
    exc_type = type(exception).__name__
    exc_msg = str(exception)

    if "timeout" in exc_msg.lower():
        return ToolError(
            error_code=ErrorCode.TIMEOUT.value,
            message=f"Database query timed out: {exc_msg}",
            retry_eligible=True,
            details={"exception_type": exc_type}
        )

    elif "permission" in exc_msg.lower() or "403" in exc_msg:
        return ToolError(
            error_code=ErrorCode.PERMISSION_DENIED.value,
            message=f"Access denied: {exc_msg}",
            retry_eligible=False,   # Do NOT retry permission errors
            details={"exception_type": exc_type}
        )

    elif "not found" in exc_msg.lower() or "404" in exc_msg:
        # NOT_FOUND is a valid empty result — not a true error
        return ToolError(
            error_code=ErrorCode.NOT_FOUND.value,
            message="Record not found",
            retry_eligible=False,
            details={"note": "This is a valid empty result, not an error"}
        )

    else:
        return ToolError(
            error_code="UNKNOWN_ERROR",
            message=f"Unexpected error: {exc_msg}",
            retry_eligible=False,  # Unknown errors: escalate
        )
```

### Error Decision Matrix — Memorize for Exam

| Error Type | Retry? | Action |
|------------|--------|--------|
| Timeout | Yes | Exponential backoff (1s, 2s, 4s) then retry |
| Rate limit (429) | Yes | Wait for retry-after header, then retry |
| Permission denied (403) | No | Escalate to human immediately |
| Not found (404) | No | Return empty result — handle in app logic |
| Validation error | No | Fix the input, log, return error to caller |
| Quota exceeded | No | Escalate to human — requires admin action |

---

## 5.3 Provenance Tracking

### What Is Provenance and Why Does It Matter?

In multi-agent systems, information passes through multiple agents, tools, and sources.
**Provenance tracking** records WHICH agent, WHICH tool, and WHICH source produced each piece of data.

Without provenance, you cannot:
- Verify a claim's source
- Debug why an agent made a wrong decision
- Audit actions for compliance
- Re-run a single step in a pipeline

### Provenance Data Model

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
import hashlib

@dataclass
class ProvenanceTag:
    """
    Attached to every piece of information in a multi-agent pipeline.
    """
    source_agent_id: str      # Which agent produced this
    tool_call_id: str         # Which specific tool call produced this
    tool_name: str            # Which tool was called
    timestamp: str            # When it was produced
    input_hash: str           # Hash of the input that produced this output
    parent_provenance_id: Optional[str] = None  # For chained derivations
    confidence: Optional[float] = None


@dataclass
class ProvenanceItem:
    """A data item with its provenance attached."""
    content: Any
    provenance: ProvenanceTag

    @property
    def id(self) -> str:
        return hashlib.md5(
            f"{self.provenance.source_agent_id}:{self.provenance.tool_call_id}".encode()
        ).hexdigest()


class ProvenanceTracker:
    """Tracks data provenance through a multi-agent pipeline."""

    def __init__(self):
        self.audit_log = []

    def tag(
        self,
        content: Any,
        agent_id: str,
        tool_call_id: str,
        tool_name: str,
        input_data: dict,
        confidence: float = None
    ) -> ProvenanceItem:
        """
        Wrap any data item with its provenance metadata.
        Call this EVERY time a tool produces a result.
        """
        prov = ProvenanceTag(
            source_agent_id=agent_id,
            tool_call_id=tool_call_id,
            tool_name=tool_name,
            timestamp=datetime.utcnow().isoformat(),
            input_hash=hashlib.md5(
                json.dumps(input_data, sort_keys=True).encode()
            ).hexdigest(),
            confidence=confidence
        )
        item = ProvenanceItem(content=content, provenance=prov)
        self.audit_log.append(asdict(item))
        return item

    def build_citation(self, item: ProvenanceItem) -> str:
        """
        Generate a human-readable citation for a provenance item.
        Include in final outputs so humans can verify claims.
        """
        p = item.provenance
        return (
            f"[Source: Agent {p.source_agent_id} | "
            f"Tool: {p.tool_name} | "
            f"Tool Call ID: {p.tool_call_id} | "
            f"Time: {p.timestamp}"
            + (f" | Confidence: {p.confidence:.0%}" if p.confidence else "")
            + "]"
        )

    def get_audit_trail(self) -> list:
        """Return the full audit trail for compliance or debugging."""
        return self.audit_log


# Usage in a multi-agent pipeline
tracker = ProvenanceTracker()

def research_agent(query: str, agent_id: str) -> ProvenanceItem:
    # Agent calls a search tool
    search_result = search_tool(query)
    tool_call_id = f"tc_{uuid.uuid4().hex[:8]}"

    # Tag the result with provenance
    return tracker.tag(
        content=search_result,
        agent_id=agent_id,
        tool_call_id=tool_call_id,
        tool_name="search_tool",
        input_data={"query": query},
        confidence=0.92
    )

# Final output includes provenance
def synthesize_with_citations(items: list[ProvenanceItem]) -> str:
    output = "## Analysis\n\n"
    for item in items:
        output += f"- {item.content}\n"
        output += f"  {tracker.build_citation(item)}\n\n"
    return output
```

---

## 5.4 Prompt Caching for Cost Optimization

### What Is Prompt Caching?

Prompt caching stores repeated content (system prompts, large documents, few-shot examples)
so that subsequent API calls reuse the cached version instead of re-processing it.

**Result: significant reduction in input token costs** for applications with repeated large contexts.

### When to Use Caching

| Content Type | Cache? | Why |
|-------------|--------|-----|
| System prompts (long) | Yes | Identical across every request |
| Large reference documents | Yes | Same document read many times |
| Few-shot example sets | Yes | Same examples every request |
| User-specific input | No | Different every request |
| Conversation history | Depends | Only if history is stable |

### Cache Control Implementation

```python
import anthropic

client = anthropic.Anthropic()

# Large system prompt that is the same for every request
LARGE_SYSTEM_PROMPT = """
You are an expert customer support analyst for Acme Corp...
[Imagine 5,000 tokens of product documentation, policies,
 troubleshooting guides, and few-shot examples here]
...
Always respond in JSON format using the schema provided.
"""

def analyze_ticket_with_caching(ticket: str) -> dict:
    """
    Use cache_control to mark the large system prompt for caching.
    First call: processes normally (cache miss)
    Subsequent calls: uses cached version (cache hit → cheaper)
    """
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=512,
        system=[
            {
                "type": "text",
                "text": LARGE_SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"}  # ← Mark for caching
            }
        ],
        messages=[{"role": "user", "content": ticket}]
    )
    return json.loads(response.content[0].text)


# Document analysis with caching
def analyze_document_batch(document: str, queries: list[str]) -> list[dict]:
    """
    Cache a large document and run multiple queries against it.
    The document is processed once, cached, then reused for each query.
    """
    results = []

    for query in queries:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=1024,
            messages=[
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "text",
                            "text": f"Document to analyze:\n\n{document}",
                            "cache_control": {"type": "ephemeral"}  # ← Cache the document
                        },
                        {
                            "type": "text",
                            "text": f"Question: {query}"
                        }
                    ]
                }
            ]
        )
        results.append({"query": query, "answer": response.content[0].text})

    return results
```

### Caching Rules for Exam

| Rule | Detail |
|------|--------|
| Use `cache_control: {type: "ephemeral"}` | Mark content for caching |
| Cache large, stable content | System prompts, documents, few-shot sets |
| Don't cache user input | Changes every request |
| First call is a cache miss | Normal cost |
| Subsequent calls are cache hits | Cheaper input tokens |
| Best for high-volume apps | More requests = more savings |

---

## Domain 5 — Concept Check Questions

1. Why should you never store agent state in Claude's context window?
2. What is an idempotent tool call? Give an example of a non-idempotent vs. idempotent design.
3. Name 3 transient errors. Name 3 terminal errors.
4. "No results found" — is this an error? Should the agent retry?
5. What is provenance tracking? Why does a multi-agent system need it?
6. What does `cache_control: {type: "ephemeral"}` do?
7. What types of content should you cache? What should you NOT cache?
8. What are the 3 required fields in a structured error response?

---

## Domain 5 Summary

| Rule | Detail |
|------|--------|
| External state only | Redis/DB for session state, not context window |
| Checkpoint after every subtask | Enables resume after failure |
| Idempotent tools where possible | Safe to retry without side effects |
| Transient → retry | Timeout, rate limit: exponential backoff |
| Terminal → escalate | Permission denied, quota exceeded |
| Empty ≠ error | "Not found" is a valid result |
| Provenance on every item | Source agent + tool call ID + timestamp |
| Cache stable content | System prompts, docs, few-shot examples |
| Don't cache user input | It changes every request |
| 3 required error fields | error_code + message + retry_eligible |
