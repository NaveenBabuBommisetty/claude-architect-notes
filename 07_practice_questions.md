# CCA Foundations — 60 Practice Questions
## Complete Mock Exam with Answers & Explanations

**Instructions**: Answer all questions before reading the answers.
Set a 60-minute timer to simulate real exam conditions.
Pass score: 72/100 equivalent (43/60 correct).

---

## Domain 1: Agentic Architecture & Orchestration (Questions 1–16)

**Q1.** An agentic loop runs for 200 iterations without completing. What is the most likely root cause?

A) The model has too many tools and cannot choose
B) The max_iterations guard is missing or set too high
C) The stop_reason is always "end_turn"
D) The session state was not stored externally

**Answer: B**
*Missing max_iterations guard is the primary cause of infinite agentic loops in production.*

---

**Q2.** What does `stop_reason: "tool_use"` indicate?

A) Claude has finished the task
B) Claude hit the maximum token limit
C) Claude wants to call an external tool and is waiting for the result
D) Claude encountered an error in tool execution

**Answer: C**
*`tool_use` means Claude is mid-reasoning and wants to invoke a tool. Your code must execute the tool and return the result.*

---

**Q3.** In a hub-and-spoke architecture, a subagent is given the task "Analyze Q3 sales data." What context should it receive?

A) The full coordinator conversation history, plus the task
B) Only the Q3 sales data and the specific analysis instructions
C) The full system prompt, all tool definitions, and conversation history
D) No context — the subagent should request what it needs

**Answer: B**
*Anti-Pattern 1: subagents receive ONLY the context needed for their specific subtask.*

---

**Q4.** A long-running agent is processing 500 documents when the server crashes at document 237. What enables it to resume from document 238?

A) Claude's context window retains progress
B) Externally stored session state with checkpointing
C) The agentic loop's stop_reason
D) The hub agent's conversation history

**Answer: B**
*External state management with checkpointing is the only way to resume after a crash.*

---

**Q5.** Which of the following is a valid trigger for human-in-the-loop escalation?

A) A tool returns an empty result after the first call
B) The agent completes a task with 0.75 confidence
C) A tool call fails with a non-retriable error after retry exhaustion
D) The task takes longer than 30 seconds to complete

**Answer: C**
*The 3 valid escalation triggers: (1) non-retriable error after retry exhaustion, (2) confidence below threshold on sensitive action, (3) task outside authorized scope.*

---

**Q6.** Two subtasks are: "Fetch competitor prices" and "Fetch our current inventory." What is the correct execution strategy?

A) Sequential — one must complete before the other starts
B) Parallel — they are independent and can run simultaneously
C) Hub-only — the hub should execute both directly
D) Sequential — to avoid rate limiting

**Answer: B**
*Independent subtasks (no data dependency between them) run in parallel for efficiency.*

---

**Q7.** A lifecycle pre-execution hook should do all of the following EXCEPT:

A) Validate the task inputs
B) Check user permissions
C) Synthesize results from multiple subagents
D) Log task initiation for audit

**Answer: C**
*Synthesis is a post-execution concern. Pre-execution hooks: validate, check permissions, log.*

---

**Q8.** What is the purpose of the `stop_reason: "max_tokens"` value?

A) Claude finished the task naturally
B) Claude wants to call a tool
C) Claude's response was truncated by hitting the token limit
D) Claude encountered a stop sequence defined in the API call

**Answer: C**
*`max_tokens` means the response was cut off — your application must handle truncation.*

---

**Q9.** An agent confidence score of 0.45 is returned for a financial transaction. What should happen?

A) Proceed with the transaction — 0.45 is above zero
B) Retry the entire task from the beginning
C) Escalate to a human operator
D) Log a warning and proceed

**Answer: C**
*Confidence below threshold on a sensitive action (financial transaction) is an escalation trigger.*

---

**Q10.** Which describes the correct purpose of a hub agent in hub-and-spoke?

A) Execute all subtasks directly to minimize API calls
B) Receive the main task, decompose it, delegate to subagents, and synthesize results
C) Monitor subagents for errors and restart failed ones
D) Provide the same context to all subagents for consistency

**Answer: B**
*The hub's role: decompose → delegate → synthesize. It does not execute subtasks itself.*

---

**Q11.** A session ID is used in long-running agent architectures to:

A) Authenticate the user with the Claude API
B) Track which Claude model version processed the request
C) Resume an interrupted task from the last checkpoint
D) Limit the number of API calls per user

**Answer: C**
*Session IDs enable task resumption after interruption or failure.*

---

**Q12.** When should subtasks be executed sequentially (not in parallel)?

A) Always — parallel execution is unreliable
B) When the subtasks are independent of each other
C) When the output of one subtask is needed as input for the next
D) When the tasks take longer than 10 seconds each

**Answer: C**
*Sequential execution is required only when there is a data dependency between tasks.*

---

**Q13.** What is context isolation in hub-and-spoke, and why is it critical?

A) Encrypting communication between agents for security
B) Ensuring each subagent only receives the context it needs for its subtask
C) Preventing agents from accessing the internet
D) Limiting the number of tools available to each agent

**Answer: B**
*Context isolation prevents reasoning degradation, controls token costs, and prevents data leakage.*

---

**Q14.** A post-execution lifecycle hook should do all of the following EXCEPT:

A) Validate the agent's output
B) Trigger downstream agents if needed
C) Validate the agent's input before the task runs
D) Store the result in persistent storage

**Answer: C**
*Input validation is a pre-execution hook concern, not post-execution.*

---

**Q15.** An agent's error hook returns "retry" after a timeout. What should happen next?

A) Immediately retry with the same parameters
B) Escalate to a human operator
C) Retry with exponential backoff
D) Terminate the task

**Answer: C**
*Transient errors (timeouts) should be retried with exponential backoff, not immediately.*

---

**Q16.** Which `stop_reason` would you receive if you included a custom stop sequence in your API call and Claude generated that sequence?

A) `end_turn`
B) `max_tokens`
C) `tool_use`
D) `stop_sequence`

**Answer: D**
*`stop_sequence` indicates Claude hit a custom stop sequence you defined in the request.*

---

## Domain 2: Tool Design & MCP Integration (Questions 17–27)

**Q17.** What is the correct MCP primitive for a function Claude can call to send an email?

A) Resource
B) Prompt
C) Tool
D) Service

**Answer: C**
*Tools are executable functions. Resources are read-only. Prompts are context templates.*

---

**Q18.** A database schema that Claude should read (not modify) should be implemented as which MCP primitive?

A) Tool
B) Resource
C) Prompt
D) Function

**Answer: B**
*Read-only data (files, schemas, records) are Resources, not Tools.*

---

**Q19.** Which element is MOST critical to include in a tool description for production reliability?

A) The tool's implementation language
B) The server's IP address
C) Side effects (especially writes, sends, deletes)
D) The developer's name

**Answer: C**
*Side effects must be documented so Claude knows when NOT to call a tool accidentally.*

---

**Q20.** What does "reasoning overload" mean in the context of tool design?

A) Claude thinks too deeply about a problem and loses the original question
B) Claude has too many tools and wastes tokens deciding which one to use
C) Claude calls the same tool multiple times in a loop
D) Claude generates too-long responses when tools return large results

**Answer: B**
*Too many tools cause Claude to spend its reasoning capacity on tool selection instead of solving the problem.*

---

**Q21.** A search tool returns zero results for a valid query. The tool response should:

A) Return an error with error_code: "NOT_FOUND"
B) Retry the search with different parameters
C) Return {"results": [], "count": 0} — a valid empty response
D) Throw an exception to indicate failure

**Answer: C**
*Anti-Pattern 2: empty results are valid. Do not return errors or trigger retries for empty results.*

---

**Q22.** A production tool fails due to a rate limit (HTTP 429). The `retry_eligible` field should be:

A) false — rate limits are permanent
B) true — rate limits are transient and should be retried
C) null — uncertain whether to retry
D) Not included in the response

**Answer: B**
*Rate limits are transient. Set retry_eligible: true and implement backoff.*

---

**Q23.** A tool fails due to a permission denied error (HTTP 403). The `retry_eligible` field should be:

A) true — retry with different credentials
B) false — permission errors are terminal, escalate to human
C) true — the permission may be granted after waiting
D) null — check with the user first

**Answer: B**
*Permission denied is terminal — no retry will fix it. Escalate to human.*

---

**Q24.** What is the maximum recommended number of tools per agent to prevent reasoning overload?

A) 1–2
B) 5–8
C) 15–20
D) No limit — more tools = more capable

**Answer: B**
*The exam specifies 5–8 tools maximum per agent to maintain reasoning quality.*

---

**Q25.** Which transport mechanism should you use for a locally running MCP server in a Claude Code development environment?

A) SSE (Server-Sent Events)
B) WebSocket
C) stdio
D) REST HTTP

**Answer: C**
*stdio is for local process communication. SSE is for remote HTTP servers.*

---

**Q26.** A tool called `search_and_update_customer` violates which principle?

A) Single transport selection
B) Single responsibility — one tool should do one thing
C) Idempotency
D) Provenance tracking

**Answer: B**
*Anti-Pattern 4: multi-purpose tools cause ambiguous descriptions and unintended side effects.*

---

**Q27.** A billing agent should have access to which of the following tools?

A) All company tools — more tools = better service
B) Only billing-scoped tools (process_refund, update_billing_info)
C) All customer-facing tools
D) No tools — billing agents should only read data

**Answer: B**
*Tool scoping: agents receive only the tools relevant to their specific role.*

---

## Domain 3: Claude Code Configuration (Questions 28–39)

**Q28.** What is the primary purpose of CLAUDE.md?

A) Store API keys for Claude Code
B) Define persistent architectural context, rules, and conventions for every session
C) Log Claude Code session history
D) Configure Claude Code's network settings

**Answer: B**
*CLAUDE.md is the "virtual tech lead" that defines project rules for every session.*

---

**Q29.** A project has both a root CLAUDE.md and an `api/CLAUDE.md`. For files in the `api/` directory, which rules apply?

A) Root CLAUDE.md always takes precedence
B) Both files' rules apply, and conflicts are resolved alphabetically
C) The `api/CLAUDE.md` overrides the root CLAUDE.md for files in that directory
D) Neither — subdirectory CLAUDE.md files are ignored

**Answer: C**
*Subdirectory CLAUDE.md files override parent rules for files in that directory.*

---

**Q30.** When should Plan Mode be used instead of Direct Execution?

A) For any change longer than 10 lines of code
B) For multi-file refactoring, architectural changes, or changes with unclear full impact
C) Whenever the user asks a question
D) Only for changes to test files

**Answer: B**
*Plan Mode: large-scale, multi-file, architectural. Direct Execution: single file, clear bug fix.*

---

**Q31.** What does the `--print` (or `-p`) flag do in Claude Code?

A) Prints the current CLAUDE.md contents
B) Makes Claude Code non-interactive for CI/CD automation
C) Prints debug information about API calls
D) Generates a PDF report of the session

**Answer: B**
*`--print` enables non-interactive mode suitable for CI/CD pipelines.*

---

**Q32.** A developer runs Claude Code without creating a CLAUDE.md file first. What is the most likely problem?

A) Claude Code will refuse to run
B) API costs will be higher
C) Every session starts without architectural context, leading to inconsistent code
D) Claude Code will automatically create a CLAUDE.md

**Answer: C**
*Anti-Pattern 5: without CLAUDE.md, sessions start fresh with no project standards.*

---

**Q33.** To combine Claude Code with GitHub Actions for automated PR reviews, which approach is correct?

A) Install Claude desktop app on the GitHub Actions runner
B) Use `claude --print --output-format json` in the workflow step
C) Claude Code cannot be used in CI/CD environments
D) Use `claude --interactive` in the CI pipeline

**Answer: B**
*The `-p` flag and `--output-format json` enable CI/CD integration.*

---

**Q34.** Why should code review be done in a separate Claude Code session from the one that wrote the code?

A) Claude cannot review its own code in the same session
B) Context bias causes the session to overlook issues it reasoned through while writing
C) Separate sessions are required by the API license
D) The first session runs out of context for review

**Answer: B**
*Anti-Pattern 7: context bias makes same-session review less effective.*

---

**Q35.** What is the Explore subagent in Claude Code used for?

A) Searching the internet for documentation
B) Isolating verbose file discovery from the main reasoning session
C) Running automated tests
D) Generating CLAUDE.md files automatically

**Answer: B**
*Explore subagent handles file discovery separately, keeping the main session clean.*

---

**Q36.** In a CLAUDE.md file, where should glob pattern rules be placed?

A) In a separate .globignore file
B) In the CLAUDE.md with the pattern specified before the rule
C) In the .gitignore file
D) In environment variables

**Answer: B**
*Example: `rules/*.ts: All exports must conform to the RuleConfig interface`*

---

**Q37.** A developer wants Claude Code to automatically follow Python PEP 8 style for all Python files. Where should this rule be defined?

A) In the Python files as comments
B) In the root CLAUDE.md under a "Code Style" section
C) In each individual Claude Code prompt
D) In a requirements.txt file

**Answer: B**
*Global project conventions belong in the root CLAUDE.md.*

---

**Q38.** Plan Mode workflow: Claude proposes a plan, you review it, and then:

A) Claude automatically executes immediately without further confirmation
B) You approve the plan, and then Claude executes step by step
C) You must implement the plan manually
D) Claude restarts the session before executing

**Answer: B**
*Plan Mode: propose → review → approve → execute. You control when execution begins.*

---

**Q39.** For CLAUDE.md to be effective in a CI/CD automated run, it MUST include:

A) The developer's email address
B) Test command, lint command, and what constitutes a failure
C) Git branch naming conventions only
D) The project's logo as ASCII art

**Answer: B**
*CI runs have no human to ask — CLAUDE.md must define test and lint commands explicitly.*

---

## Domain 4: Prompt Engineering & Structured Output (Questions 40–51)

**Q40.** You are building a ticket classifier with 5 categories. How many few-shot examples should you provide at minimum?

A) 0 — examples are optional
B) 1 — one example is sufficient
C) 2–3 with at least one example per category
D) 10+ for reliable classification

**Answer: C**
*2–3 examples covering all categories. Missing a category in examples = unreliable classification for that category.*

---

**Q41.** In a JSON schema for structured output, how should an optional string field be typed?

A) `"type": "string"` with `"required": false`
B) `"type": ["string", "null"]`
C) `"type": "optional_string"`
D) Omit it from the schema

**Answer: B**
*Nullable optional fields use a type array: `"type": ["string", "null"]`*

---

**Q42.** Claude returns JSON with a missing required field. The production system should:

A) Proceed with the missing field set to null
B) Log the error and crash
C) Validate, identify the specific error, feed the error back to Claude, and retry
D) Ask the user to try again

**Answer: C**
*Validation failure: identify the specific error, include it in the retry prompt with the original output.*

---

**Q43.** What is the recommended maximum number of retries for a validation-retry loop?

A) 1
B) 3
C) 10
D) Unlimited — keep retrying until success

**Answer: B**
*3 retries is the standard. Beyond that, log the failure and raise an exception.*

---

**Q44.** Which API type should be used for a nightly job that analyzes 50,000 customer reviews?

A) Synchronous API — for reliable immediate results
B) Streaming API — for real-time output
C) Batch API — for non-urgent bulk processing at lower cost
D) WebSocket API — for persistent connections

**Answer: C**
*Anti-Pattern 6: batch jobs use the Batch API. It is async, polled, and significantly cheaper.*

---

**Q45.** A developer writes: "Analyze code for issues. Be thorough but avoid false positives." This instruction is problematic because:

A) It's too long
B) "Be thorough" and "avoid false positives" are vague and cannot be calibrated by Claude
C) Code analysis requires a special API endpoint
D) It should use a tool instead of a system prompt

**Answer: B**
*Anti-Pattern 3: vague instructions like "be thorough" and "avoid false positives" cannot be calibrated.*

---

**Q46.** Few-shot examples should be chosen to:

A) Use the shortest possible examples to save tokens
B) Cover all output categories and include edge cases
C) Only show the most common case
D) Be taken directly from the training data

**Answer: B**
*Good few-shot sets cover ALL categories and important edge cases.*

---

**Q47.** A validation retry prompt should include:

A) Only the original task
B) The error message AND the original (incorrect) output
C) Only an apology for the format issue
D) A completely different prompt

**Answer: B**
*The retry prompt must include both the error AND what Claude actually returned, so it understands what went wrong.*

---

**Q48.** Why is `"additionalProperties": false` useful in a JSON schema?

A) It prevents Claude from thinking about the problem
B) It rejects output that contains unexpected fields
C) It speeds up JSON parsing
D) It limits the length of string values

**Answer: B**
*`additionalProperties: false` ensures Claude only returns the fields you defined.*

---

**Q49.** Which instruction is more likely to produce consistent, reliable output?

A) "Flag security issues that seem important"
B) "Flag only SQL injection risks where user input is directly concatenated into a query string without parameterization"

**Answer: B**
*Specific, testable instructions produce consistent results. Vague instructions produce variable results.*

---

**Q50.** What is the correct API to use for real-time chat interactions where the user is waiting?

A) Batch API
B) Streaming API or Synchronous API
C) Background processing API
D) Webhook API

**Answer: B**
*Real-time, user-waiting interactions use synchronous or streaming API.*

---

**Q51.** All validation failures in a production system should be:

A) Silently ignored if they eventually succeed on retry
B) Logged — patterns in failures reveal weaknesses in your prompts
C) Reported to the end user immediately
D) Treated as permanent errors

**Answer: B**
*Always log validation failures. Patterns tell you where to improve your prompts.*

---

## Domain 5: Context & Reliability (Questions 52–60)

**Q52.** Where should session state for a long-running agent be stored?

A) In Claude's conversation context window
B) In the agent's local variables
C) In an external store (Redis, database)
D) In environment variables

**Answer: C**
*External stores (Redis, DB) persist across crashes and can be accessed by any process.*

---

**Q53.** A tool call fails with a "permission denied" error. It should be classified as:

A) Transient — retry after waiting
B) Terminal — escalate to human, do not retry
C) Valid empty result — return empty data
D) Unknown — apply exponential backoff

**Answer: B**
*Permission denied = terminal error. Retrying won't fix a permissions problem.*

---

**Q54.** A search tool returns "No results found for query." How should the agent handle this?

A) Retry the search with different parameters
B) Raise an exception
C) Return it as a valid empty result and let the application handle it
D) Escalate to a human

**Answer: C**
*Anti-Pattern 2: empty results are valid application states, not errors.*

---

**Q55.** What is the primary benefit of idempotent tool calls?

A) They run faster
B) They can be safely retried without causing duplicate side effects
C) They cost fewer tokens
D) They do not require authentication

**Answer: B**
*Idempotency allows safe retries — calling the same tool twice produces the same result.*

---

**Q56.** Provenance tracking in a multi-agent system records:

A) The user's identity and session duration
B) Which agent and tool call produced each piece of information
C) The cost of each API call
D) The number of tokens used per agent

**Answer: B**
*Provenance = source agent ID + tool call ID + timestamp for each data item.*

---

**Q57.** Which content type benefits MOST from prompt caching?

A) User input messages (different every request)
B) A large system prompt used identically in every request
C) The model's output
D) Tool result payloads

**Answer: B**
*Large, stable system prompts are the ideal caching candidate — same content, every request.*

---

**Q58.** A database query times out after 30 seconds. `retry_eligible` should be set to:

A) false — the data might not exist
B) true — timeouts are transient, retry with backoff
C) null — uncertain
D) false — the timeout should be reported to the user

**Answer: B**
*Timeouts are transient errors — retry with exponential backoff.*

---

**Q59.** When should an agent checkpoint its progress?

A) Only at the very end of all subtasks
B) After every completed subtask
C) Only when an error occurs
D) Once per minute regardless of progress

**Answer: B**
*Checkpoint after EVERY subtask so any failure can be resumed from the last completed step.*

---

**Q60.** The `cache_control: {"type": "ephemeral"}` parameter should be added to:

A) User messages that change every request
B) Large, stable content blocks like system prompts and reference documents
C) Tool result payloads
D) Every API call regardless of content

**Answer: B**
*Cache stable, repeated content. Not user input, not tool results.*

---

## Score Card

| Domain | Questions | Your Correct | Score |
|--------|-----------|--------------|-------|
| Domain 1: Agentic Architecture | Q1–Q16 | __/16 | __% |
| Domain 2: Tool Design & MCP | Q17–Q27 | __/11 | __% |
| Domain 3: Claude Code Config | Q28–Q39 | __/12 | __% |
| Domain 4: Prompt Engineering | Q40–Q51 | __/12 | __% |
| Domain 5: Context & Reliability | Q52–Q60 | __/9 | __% |
| **Total** | **Q1–Q60** | **__/60** | **__%** |

**Pass: 43/60 (72%)**

---

## Where to Focus Based on Your Score

| Score on Domain | Action |
|----------------|--------|
| < 60% | Re-read the full domain tutorial, all code examples |
| 60–75% | Re-read the summary tables and anti-patterns section |
| 75–90% | Review the concept check questions at the end of each tutorial |
| > 90% | Move on — you've mastered this domain |

---

## Last 24 Hours Before Exam

1. Read `06_anti_patterns.md` — the quick reference table
2. Re-read all domain summary tables
3. Answer 10 random questions from this file without looking at answers
4. Sleep well — cognitive performance is more important than last-minute cramming
