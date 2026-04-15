# Domain 4: Prompt Engineering & Structured Output
## Weight: 20%

---

## What This Domain Tests

This domain is described in the exam guide as "tricky — the wrong answers look like good engineering."
It tests **precise knowledge** of how to get reliable, structured, production-quality outputs from Claude.

---

## 4.1 Few-Shot Prompting

### What Is Few-Shot Prompting?

Providing **2–3 example input/output pairs** to show Claude exactly what behavior you expect.
It is more reliable than instruction-only prompts for complex or ambiguous tasks.

### When to Use Few-Shot

- Complex output formats (structured JSON, specific markdown patterns)
- Classification tasks (show an example of each category)
- Tasks where "good" is ambiguous (show what "good" looks like)
- Edge cases are important (show how to handle them)

### Anatomy of a Good Few-Shot Prompt

```python
import anthropic

client = anthropic.Anthropic()

FEW_SHOT_SYSTEM = """You classify customer support tickets into categories.

Categories:
- BILLING: Payment, invoice, refund, pricing questions
- TECHNICAL: Bugs, errors, crashes, feature not working
- ACCOUNT: Login, password, profile, account settings
- FEATURE_REQUEST: Requests for new functionality
- OTHER: Does not fit above categories

Examples:

Input: "My payment was charged twice this month"
Output: {"category": "BILLING", "confidence": 0.97, "reasoning": "Duplicate charge is a billing issue"}

Input: "The app crashes when I try to upload a file larger than 10MB"
Output: {"category": "TECHNICAL", "confidence": 0.95, "reasoning": "App crash is a technical bug"}

Input: "Can you add dark mode to the mobile app?"
Output: {"category": "FEATURE_REQUEST", "confidence": 0.99, "reasoning": "Dark mode is a new feature request"}

Input: "I can't log in, it says my password is wrong but I just reset it"
Output: {"category": "ACCOUNT", "confidence": 0.92, "reasoning": "Login issue is an account problem"}

Input: "When is your office open?"
Output: {"category": "OTHER", "confidence": 0.88, "reasoning": "Business hours question doesn't fit defined categories"}

Return ONLY valid JSON matching the output format shown above. No additional text."""


def classify_ticket(ticket_text: str) -> dict:
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=256,
        system=FEW_SHOT_SYSTEM,
        messages=[{"role": "user", "content": ticket_text}]
    )
    import json
    return json.loads(response.content[0].text)
```

### Few-Shot Rules for Exam

| Rule | Why It Matters |
|------|---------------|
| 2–3 examples minimum | Too few = ambiguous behavior |
| Cover ALL categories | Missing category → poor classification |
| Cover edge cases | Edge cases in examples = edge cases handled |
| Bad examples are worse than none | Wrong examples teach wrong patterns |
| Examples must match your exact output format | Claude mirrors the format of your examples |

```python
# WRONG — only one example, no edge cases, vague format
bad_few_shot = """
Example:
Input: "My bill is wrong"
Output: billing issue
"""

# CORRECT — 2+ examples, all categories, exact output format
good_few_shot = """
Example 1:
Input: "My bill is wrong"
Output: {"category": "BILLING", "confidence": 0.95}

Example 2:
Input: "The app won't load"
Output: {"category": "TECHNICAL", "confidence": 0.93}

Example 3 (edge case — ambiguous):
Input: "I was charged but my account is still locked"
Output: {"category": "BILLING", "confidence": 0.72, "note": "billing issue takes precedence"}
"""
```

---

## 4.2 JSON Schema for Structured Output

### Why Use JSON Schema?

When Claude must return structured JSON, a schema:
1. Enforces required fields
2. Specifies field types
3. Guides Claude's reasoning (via field descriptions)
4. Makes validation automatic

### Complete JSON Schema Example

```python
import anthropic
import json
from jsonschema import validate, ValidationError

client = anthropic.Anthropic()

# Define the expected output schema
ANALYSIS_SCHEMA = {
    "type": "object",
    "properties": {
        "summary": {
            "type": "string",
            "description": "One-sentence summary of the customer issue (max 100 chars)"
        },
        "severity": {
            "type": "string",
            "enum": ["low", "medium", "high", "critical"],
            "description": "Severity based on business impact: critical=data loss or outage"
        },
        "category": {
            "type": "string",
            "enum": ["billing", "technical", "account", "feature_request", "other"],
            "description": "The most specific applicable category"
        },
        "requires_escalation": {
            "type": "boolean",
            "description": "True if this ticket needs immediate human review"
        },
        "suggested_response": {
            "type": ["string", "null"],  # ← nullable optional field
            "description": "Draft response to send to customer, or null if escalation required"
        },
        "tags": {
            "type": "array",
            "items": {"type": "string"},
            "description": "Relevant tags for this ticket (e.g., ['refund', 'mobile-app'])"
        },
        "confidence": {
            "type": "number",
            "minimum": 0.0,
            "maximum": 1.0,
            "description": "Confidence in this classification (0.0 = uncertain, 1.0 = certain)"
        }
    },
    "required": [
        "summary",
        "severity",
        "category",
        "requires_escalation",
        "tags",
        "confidence"
    ],
    "additionalProperties": False  # ← Reject unexpected fields
}


SYSTEM_PROMPT = f"""Analyze customer support tickets and return structured JSON analysis.

Required output format (JSON Schema):
{json.dumps(ANALYSIS_SCHEMA, indent=2)}

Rules:
- severity=critical: data loss, system outage, security breach
- severity=high: feature completely broken, blocking user workflow
- severity=medium: degraded functionality, workaround exists
- severity=low: cosmetic issues, minor inconvenience
- requires_escalation=true when: severity=critical, legal/compliance mentioned, or customer explicitly angry
- Return ONLY valid JSON. No markdown, no explanation text."""


def analyze_ticket(ticket: str) -> dict:
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=512,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": ticket}]
    )
    result = json.loads(response.content[0].text)

    # Always validate against schema
    validate(instance=result, schema=ANALYSIS_SCHEMA)
    return result
```

### Nullable Optional Fields

```python
# Pattern for optional fields in JSON schema
{
    "suggested_response": {
        "type": ["string", "null"],   # ← Two types: string OR null
        "description": "Response draft, or null if human review needed"
    }
}

# In your prompt, be explicit:
# "Set suggested_response to null when requires_escalation is true"
```

---

## 4.3 Validation & Retry Loops

### Why Validation Is Non-Negotiable

Claude may occasionally:
- Return JSON with a missing required field
- Use a value outside an allowed enum
- Return text outside the JSON structure

**Production systems must validate and retry.**

### The Production Validation-Retry Pattern

```python
import json
import time
from jsonschema import validate, ValidationError

MAX_RETRIES = 3

def analyze_with_retry(ticket: str) -> dict:
    """
    Production-safe Claude invocation with validation and retry.
    """
    last_output = None
    last_error = None

    for attempt in range(1, MAX_RETRIES + 1):
        try:
            response = client.messages.create(
                model="claude-opus-4-6",
                max_tokens=512,
                system=build_system_prompt(last_error, last_output),  # ← Feed error back
                messages=[{"role": "user", "content": ticket}]
            )

            raw_text = response.content[0].text

            # Step 1: Parse JSON
            try:
                result = json.loads(raw_text)
            except json.JSONDecodeError as e:
                last_error = f"Invalid JSON: {str(e)}. Raw output was: {raw_text[:200]}"
                last_output = raw_text
                continue  # Retry

            # Step 2: Validate against schema
            try:
                validate(instance=result, schema=ANALYSIS_SCHEMA)
                # Success — log and return
                if attempt > 1:
                    log_validation_failure(ticket, attempt, last_error)
                return result

            except ValidationError as e:
                last_error = f"Schema validation failed: {e.message}"
                last_output = json.dumps(result)
                continue  # Retry

        except Exception as e:
            last_error = f"API error: {str(e)}"
            time.sleep(2 ** attempt)  # Exponential backoff
            continue

    # Exhausted retries — log and raise
    log_validation_failure(ticket, MAX_RETRIES, last_error)
    raise RuntimeError(f"Failed after {MAX_RETRIES} attempts. Last error: {last_error}")


def build_system_prompt(last_error: str = None, last_output: str = None) -> str:
    """
    On retry, include the previous error to guide Claude's correction.
    """
    base_prompt = f"""Analyze support tickets. Return JSON matching this schema:
{json.dumps(ANALYSIS_SCHEMA, indent=2)}
Return ONLY valid JSON. No other text."""

    if last_error:
        # Feed the error back to Claude — this is key to successful retries
        base_prompt += f"""

IMPORTANT: Your previous response had an error.
Error: {last_error}
Previous output: {last_output}

Fix the error and return valid JSON this time."""

    return base_prompt


def log_validation_failure(ticket: str, attempt: int, error: str):
    """
    Log all validation failures — patterns reveal prompt weaknesses.
    Exam knowledge: always log failures, analyze patterns.
    """
    print(f"Validation failure | Attempt {attempt} | Error: {error}")
    # In production: send to your observability platform
```

### Retry Rules

| Scenario | Action |
|----------|--------|
| JSON parse error | Retry with error context |
| Schema validation failure | Retry with error context |
| API timeout | Retry with exponential backoff |
| Max retries reached | Raise exception, log for analysis |
| Successful on retry | Log which attempt succeeded (reveals prompt quality) |

---

## 4.4 Batch vs. Synchronous API

### Critical Decision — Exam Specific

**Anti-Pattern 6**: Using synchronous API for batch jobs. This is explicitly tested.

| API Type | Use For | Cost | Latency |
|----------|---------|------|---------|
| **Synchronous** | Real-time, user-facing requests | Standard | Immediate |
| **Batch** | Background processing, bulk jobs | Significantly cheaper | Async (poll for results) |

### Synchronous API — Real-Time

```python
# Use when: user is waiting for the response right now
def handle_user_request(user_message: str) -> str:
    """Real-time: user is waiting."""
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": user_message}]
    )
    return response.content[0].text
```

### Batch API — Background Processing

```python
# Use when: processing large volumes, not time-sensitive
# The Batch API is async — submit, then poll for results

def submit_batch_analysis(tickets: list[str]) -> str:
    """
    Submit a batch of tickets for background analysis.
    Returns a batch_id to poll for results later.
    """
    requests = []
    for i, ticket in enumerate(tickets):
        requests.append({
            "custom_id": f"ticket_{i}",
            "params": {
                "model": "claude-opus-4-6",
                "max_tokens": 512,
                "system": SYSTEM_PROMPT,
                "messages": [{"role": "user", "content": ticket}]
            }
        })

    # Submit the batch
    batch = client.beta.messages.batches.create(requests=requests)
    return batch.id   # ← Store this to poll later


def poll_batch_results(batch_id: str) -> list[dict]:
    """
    Poll the batch until complete. In production, use a scheduler
    (e.g., Celery, APScheduler) rather than blocking.
    """
    while True:
        batch = client.beta.messages.batches.retrieve(batch_id)

        if batch.processing_status == "ended":
            # Collect results
            results = []
            for result in client.beta.messages.batches.results(batch_id):
                if result.result.type == "succeeded":
                    text = result.result.message.content[0].text
                    results.append({
                        "id": result.custom_id,
                        "result": json.loads(text)
                    })
                else:
                    results.append({
                        "id": result.custom_id,
                        "error": result.result.error
                    })
            return results

        time.sleep(30)  # Poll every 30 seconds


# Production use cases for Batch API:
# ✓ Nightly analysis of all support tickets from the day
# ✓ Bulk document processing (1000+ documents)
# ✓ Dataset generation for fine-tuning
# ✓ Bulk content moderation
# ✓ Monthly report generation

# Production use cases for Synchronous API:
# ✓ Chat interface — user is waiting
# ✓ Real-time code completion
# ✓ API endpoint that returns a response immediately
# ✓ Interactive search queries
```

---

## 4.5 Precision in Instructions

### The Calibration Problem

**Anti-Pattern 3**: Vague calibration instructions. The exam tests this heavily.

Claude cannot act on vague directives because it has no way to calibrate what "conservative"
or "high-confidence" means in your specific context.

```python
# WRONG — vague, Claude cannot calibrate
vague_instructions = """
System: Review code comments for accuracy. Be conservative.
Only flag high-confidence issues.
"""
# What is "conservative"? What is "high-confidence"?
# Claude will interpret this differently every run


# CORRECT — specific, measurable, unambiguous
precise_instructions = """
System: Review code comments for accuracy.

Flag a comment as INACCURATE if and only if:
1. The documented parameter name differs from the actual parameter name in the function signature
2. The documented return type differs from the actual return type annotation
3. The comment says the function does X but the code clearly does Y

Do NOT flag:
- Style differences (e.g., abbreviated vs. full parameter names)
- Missing documentation (only flag INCORRECT documentation)
- Comments that are incomplete but not wrong

For each finding, output:
{"file": "...", "line": N, "issue": "...", "evidence": "..."}
"""
```

### Bad vs. Good Instruction Examples (Exam Pattern)

| Bad (Vague) | Good (Specific) |
|-------------|-----------------|
| "Be conservative" | "Flag only when the claimed behavior directly contradicts the code behavior" |
| "Check that comments are accurate" | "Flag comments where the documented parameter name differs from the actual parameter name" |
| "Only report high-confidence findings" | "Flag only when confidence ≥ 0.90 based on explicit code evidence" |
| "Handle edge cases carefully" | "If input is null, return {\"result\": null, \"error\": \"INPUT_NULL\"}" |
| "Summarize the key points" | "Return exactly 3 bullet points, each under 20 words, covering the main action, impact, and next step" |

### Writing Testable Instructions

Instructions are good when they are **testable** — you can determine if Claude followed them or not.

```python
# How to test if your instruction is precise enough:
# Ask yourself: "Could two developers disagree on whether Claude followed this?"

# Testable (precise):
# "Flag only when the parameter name in the docstring differs from the signature"
# → Two developers would agree: either the names match or they don't

# Not testable (vague):
# "Flag issues that seem significant"
# → Two developers would disagree on what "significant" means

# Exam pattern: "Be conservative" is always the wrong answer choice
# Always pick the specific, measurable alternative
```

---

## Domain 4 — Concept Check Questions

1. When should you use few-shot prompting? How many examples minimum?
2. What happens if your few-shot examples have only some of the output categories?
3. What is the difference between a required field and a nullable optional field in JSON schema?
4. What should you include in the retry prompt when a validation fails?
5. What is the maximum number of retries recommended?
6. Name 3 use cases for the Batch API and 3 for the Synchronous API.
7. Why does "be conservative" fail as a Claude instruction?
8. What makes an instruction "testable"?

---

## Domain 4 Summary

| Rule | Detail |
|------|--------|
| Few-shot: 2–3 examples | Cover ALL categories and edge cases |
| Bad examples = worse than none | Wrong examples teach wrong patterns |
| JSON schema for structured output | Define required, optional, nullable fields |
| Always validate output | Check against schema before using |
| Retry with error context | Feed the error message back to Claude |
| Max 3 retries | Prevent infinite retry loops |
| Log all failures | Failure patterns reveal prompt weaknesses |
| Batch API for bulk jobs | Significantly cheaper, async |
| Sync API for real-time | User is waiting |
| Specific > vague instructions | "be conservative" never works |
| Instructions must be testable | Could two devs agree Claude followed this? |
