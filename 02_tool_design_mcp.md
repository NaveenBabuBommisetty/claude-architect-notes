# Domain 2: Tool Design & MCP Integration
## Weight: 18%

---

## What This Domain Tests

Your ability to **design reliable, well-scoped tools** and connect Claude to external systems
using the **Model Context Protocol (MCP)** — Anthropic's standard for external integrations.

---

## 2.1 MCP Core Primitives

MCP defines exactly three building blocks. The exam distinguishes between them.

### The 3 Primitives

| Primitive | Type | What It Is | Example |
|-----------|------|------------|---------|
| **Tools** | Executable | Functions Claude can call | `search_database`, `send_email` |
| **Resources** | Read-only | Data Claude can access | Files, database records, documents |
| **Prompts** | Templates | Reusable context injectors | A standard analysis template |

### Primitive Decision Tree

```
Does Claude need to DO something (write/call/execute)?
  YES → Tool

Does Claude need to READ something (data/file/record)?
  YES → Resource

Does Claude need a reusable context template injected?
  YES → Prompt
```

### Code Example — Basic MCP Server with All 3 Primitives

```python
# mcp_server.py — MCP server using Python SDK
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types
import json

app = Server("my-production-server")

# ── TOOL: Executable function ────────────────────────────────────────
@app.list_tools()
async def list_tools():
    return [
        types.Tool(
            name="search_customer",
            description=(
                "Search the customer database by email or customer ID. "
                "Returns customer profile including name, plan, and account status. "
                "Use this when you need to look up customer information. "
                "Example input: {'email': 'john@example.com'} "
                "Returns error_code 'NOT_FOUND' if customer does not exist."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "email": {
                        "type": "string",
                        "description": "Customer email address"
                    },
                    "customer_id": {
                        "type": "string",
                        "description": "Customer ID (alternative to email)"
                    }
                },
                "oneOf": [
                    {"required": ["email"]},
                    {"required": ["customer_id"]}
                ]
            }
        ),
        types.Tool(
            name="update_customer_plan",
            description=(
                "Update a customer's subscription plan. "
                "SIDE EFFECT: This modifies the customer record in the database. "
                "Only call this after explicit confirmation from the user. "
                "Valid plans: 'basic', 'professional', 'enterprise'. "
                "Returns error_code 'INVALID_PLAN' for unknown plan names."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "customer_id": {"type": "string"},
                    "new_plan": {
                        "type": "string",
                        "enum": ["basic", "professional", "enterprise"]
                    }
                },
                "required": ["customer_id", "new_plan"]
            }
        )
    ]


@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "search_customer":
        result = db_search_customer(arguments)
        return [types.TextContent(type="text", text=json.dumps(result))]

    elif name == "update_customer_plan":
        result = db_update_plan(arguments)
        if result.get("error"):
            # Use isError for error signaling
            return [types.TextContent(
                type="text",
                text=json.dumps({
                    "error_code": result["error_code"],
                    "message": result["message"],
                    "retry_eligible": result.get("retry_eligible", False)
                })
            )]
        return [types.TextContent(type="text", text=json.dumps(result))]


# ── RESOURCE: Read-only data ─────────────────────────────────────────
@app.list_resources()
async def list_resources():
    return [
        types.Resource(
            uri="db://customers/schema",
            name="Customer Database Schema",
            description="Schema definition for the customers table",
            mimeType="application/json"
        )
    ]


@app.read_resource()
async def read_resource(uri: str):
    if uri == "db://customers/schema":
        schema = load_customer_schema()
        return types.ReadResourceResult(
            contents=[types.TextContent(type="text", text=json.dumps(schema))]
        )


# ── PROMPT: Reusable template ────────────────────────────────────────
@app.list_prompts()
async def list_prompts():
    return [
        types.Prompt(
            name="customer_analysis",
            description="Standard template for analyzing a customer account",
            arguments=[
                types.PromptArgument(
                    name="customer_id",
                    description="Customer ID to analyze",
                    required=True
                )
            ]
        )
    ]


@app.get_prompt()
async def get_prompt(name: str, arguments: dict):
    if name == "customer_analysis":
        cid = arguments["customer_id"]
        return types.GetPromptResult(
            messages=[{
                "role": "user",
                "content": f"Analyze customer {cid}: review their plan, usage, and support history."
            }]
        )


if __name__ == "__main__":
    import asyncio
    asyncio.run(stdio_server(app))
```

---

## 2.2 Writing Effective Tool Descriptions

**This is where most developers lose exam marks.** Tool descriptions are how Claude decides:
1. **Which tool to call**
2. **How to call it correctly**
3. **When NOT to call it**

### The 5 Elements of a Great Tool Description

```
1. WHAT it does (one sentence)
2. WHEN to use it (trigger condition)
3. INPUT examples (concrete example values)
4. SIDE EFFECTS (if it writes/sends/deletes — say so explicitly)
5. ERROR conditions (what error codes or messages to expect)
```

### Bad vs. Good Tool Descriptions

```python
# BAD — vague, no examples, no side effects, no errors
bad_tool = {
    "name": "send_message",
    "description": "Sends a message to a user.",
    "inputSchema": {
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "message": {"type": "string"}
        }
    }
}

# GOOD — explicit, with examples, side effects, and error conditions
good_tool = {
    "name": "send_message",
    "description": (
        "Send a notification message to a user via their registered email. "
        "SIDE EFFECT: This sends an actual email — only call this after the user "
        "has explicitly confirmed they want to receive the message. "
        "Example input: {'user_id': 'usr_12345', 'message': 'Your order has shipped'} "
        "Returns: {'sent': true, 'message_id': 'msg_xxx'} on success. "
        "Error codes: 'USER_NOT_FOUND' if user_id is invalid, "
        "'EMAIL_UNVERIFIED' if user has not verified their email."
    ),
    "inputSchema": {
        "type": "object",
        "properties": {
            "user_id": {
                "type": "string",
                "description": "The user's ID (format: usr_xxxxx)"
            },
            "message": {
                "type": "string",
                "description": "The message body to send (max 500 characters)"
            }
        },
        "required": ["user_id", "message"]
    }
}
```

---

## 2.3 Tool Boundary Design

### One Tool = One Responsibility

This is **Anti-Pattern 4** from the exam. Do not build multi-purpose tools.

```python
# WRONG — multi-purpose tool (Anti-Pattern 4)
bad_multi_tool = {
    "name": "search_and_update_customer",
    "description": "Search for a customer and optionally update their plan.",
    # Problems:
    # 1. Ambiguous — when does it search? When does it update?
    # 2. Risk of unintended updates when searching
    # 3. Hard for Claude to reason about when to use it
}

# CORRECT — separate tools with clear boundaries
search_tool = {
    "name": "search_customer",
    "description": "Search the customer database. READ-ONLY. No side effects.",
}

update_tool = {
    "name": "update_customer_plan",
    "description": "Update plan. WRITE operation. Requires confirmed customer_id.",
}
```

### Scope Tools to Agent Role

```python
# Customer Support Agent — read-only tools only
support_agent_tools = [
    "search_customer",      # Read: find customer
    "read_support_tickets", # Read: view tickets
    "add_ticket_note",      # Write: add notes (limited scope)
    # NOT included: "delete_customer", "refund_payment"
]

# Billing Agent — write tools scoped to billing only
billing_agent_tools = [
    "search_customer",      # Read: find customer
    "process_refund",       # Write: billing-scoped
    "update_billing_info",  # Write: billing-scoped
    # NOT included: "delete_customer", "send_marketing_email"
]
```

### Structured Error Response

```python
# Production-quality error response from a tool
def tool_error_response(
    error_code: str,
    message: str,
    retry_eligible: bool,
    details: dict = None
) -> dict:
    """
    Always return structured errors — not raw exceptions.
    Claude uses these to decide whether to retry or escalate.
    """
    return {
        "error_code": error_code,          # Machine-readable code
        "message": message,                 # Human-readable message
        "retry_eligible": retry_eligible,   # Can Claude retry this?
        "details": details or {}            # Additional context
    }


# Usage examples
# Transient error (retry eligible)
timeout_error = tool_error_response(
    error_code="TIMEOUT",
    message="Database connection timed out after 30 seconds",
    retry_eligible=True,
    details={"timeout_seconds": 30, "attempt": 2}
)

# Terminal error (do NOT retry)
permission_error = tool_error_response(
    error_code="PERMISSION_DENIED",
    message="User does not have access to customer ID 99999",
    retry_eligible=False,
    details={"required_role": "admin", "user_role": "viewer"}
)

# Valid empty result (NOT an error — Anti-Pattern 2)
empty_result = {
    "results": [],
    "count": 0,
    "message": "No customers found matching the search criteria"
    # NOTE: No error_code — this is a valid response, not an error
}
```

---

## 2.4 MCP Transport Mechanisms

### Two Transport Types

| Transport | Use Case | Protocol |
|-----------|----------|----------|
| **stdio** | Local process communication | Standard input/output |
| **SSE** (Server-Sent Events) | Remote MCP servers over HTTP | HTTP with streaming |

```python
# stdio transport — for Claude Code, local agents
# Run server as a subprocess that communicates via stdin/stdout

# In claude_desktop_config.json or .mcp.json:
{
    "mcpServers": {
        "my-local-server": {
            "command": "python3",
            "args": ["/path/to/mcp_server.py"],
            "transport": "stdio"   # ← stdio for local
        }
    }
}


# SSE transport — for remote servers
# The MCP server runs as an HTTP server

from mcp.server.sse import SseServerTransport
from starlette.applications import Starlette
from starlette.routing import Route

async def handle_sse(request):
    transport = SseServerTransport("/messages")
    async with transport.connect_sse(
        request.scope, request.receive, request._send
    ) as streams:
        await app.run(streams[0], streams[1], app.create_initialization_options())

starlette_app = Starlette(routes=[
    Route("/sse", endpoint=handle_sse)
])
# Deploy this as a web service; clients connect via HTTP SSE
```

### Implementation Language Choice

- **Python**: Use `mcp` Python package — best for data pipelines, ML integrations
- **TypeScript**: Use `@modelcontextprotocol/sdk` — best for Node.js web services

---

## 2.5 Preventing Reasoning Overload

### The Problem

When Claude has too many tools, it spends its "reasoning budget" picking the right tool
instead of actually solving the problem. This is called **reasoning overload**.

```
5 tools: Claude picks the right tool in 1-2 reasoning steps
15 tools: Claude spends 4-6 reasoning steps just on tool selection
25 tools: Significant performance degradation — wrong tools chosen frequently
```

### Solutions

```python
# Solution 1: Limit tools to 5-8 per agent
# Don't give every agent every tool

# Solution 2: Tool namespacing — group related tools visually
tools_with_namespacing = [
    # Customer tools — grouped
    {"name": "customer__search",  "description": "..."},
    {"name": "customer__update",  "description": "..."},
    {"name": "customer__delete",  "description": "..."},

    # Order tools — grouped
    {"name": "order__list",       "description": "..."},
    {"name": "order__cancel",     "description": "..."},
]
# The __ namespace helps Claude reason about tool families

# Solution 3: Dynamic tool loading — inject only relevant tools
def get_tools_for_context(task_type: str) -> list:
    """Load only the tools needed for this specific task type."""
    tool_registry = {
        "customer_support": [search_customer_tool, add_note_tool],
        "billing": [search_customer_tool, process_refund_tool],
        "technical": [read_logs_tool, restart_service_tool],
    }
    return tool_registry.get(task_type, [])
```

---

## Domain 2 — Concept Check Questions

1. What are the 3 MCP primitives? What is the difference between a Tool and a Resource?
2. What are the 5 elements of a great tool description?
3. Why should a tool NOT do multiple things?
4. What is the difference between stdio and SSE transport?
5. What is reasoning overload? How do you prevent it?
6. When should a tool return `retry_eligible: false`?
7. "No results found" — is this an error or a valid response? What should the tool return?

---

## Domain 2 Summary

| Rule | Detail |
|------|--------|
| Tools = executable | Resources = read-only, Prompts = templates |
| 5 description elements | What, when, example, side effects, errors |
| One tool = one job | No multi-purpose tools |
| Scope to agent role | Billing agent gets billing tools only |
| 5–8 tools max | More tools → reasoning overload |
| Structured errors | error_code + message + retry_eligible |
| Empty ≠ error | `{"results": [], "count": 0}` is valid |
| stdio for local | SSE for remote HTTP servers |
