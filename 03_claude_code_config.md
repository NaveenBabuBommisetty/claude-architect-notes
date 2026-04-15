# Domain 3: Claude Code Configuration
## Weight: 20%

---

## What This Domain Tests

Your ability to **configure Claude Code** for production development workflows — including
project context files, execution modes, CI/CD integration, and subagent patterns.

---

## 3.1 CLAUDE.md — The Project Tech Lead

### What Is CLAUDE.md?

`CLAUDE.md` is a **persistent context file** that tells Claude Code:
- The architectural rules of your project
- Naming conventions
- Forbidden patterns
- Test standards
- File structure expectations

Think of it as a **virtual tech lead** that Claude Code reads at the start of every session.
Without it, every session starts from zero — Claude has no knowledge of your project's standards.

### File Hierarchy — Global vs. Module-Specific

```
project-root/
├── CLAUDE.md              ← Global rules (applies to ALL files)
├── src/
│   ├── CLAUDE.md          ← src-specific rules (override global for src/)
│   └── components/
│       ├── CLAUDE.md      ← Component rules (override parent rules)
│       └── Button.tsx
├── api/
│   ├── CLAUDE.md          ← API-specific rules
│   └── routes/
└── tests/
    └── CLAUDE.md          ← Test-specific rules
```

**Rule precedence:** Subdirectory CLAUDE.md overrides parent CLAUDE.md for files in that directory.

### Complete CLAUDE.md Example

```markdown
# Project: E-Commerce Platform
# Last Updated: 2026-03-28

## Tech Stack
- Frontend: Next.js 14, TypeScript, Tailwind CSS
- Backend: FastAPI (Python 3.12), PostgreSQL
- Testing: pytest (backend), Vitest (frontend)
- State: Zustand (frontend), Redis (session)

## Architectural Rules
1. All API routes MUST be in `api/routes/` — never in `api/main.py` directly
2. Database models go in `api/models/` — never inline in routes
3. All database queries MUST use the ORM (SQLAlchemy) — no raw SQL strings
4. Authentication logic lives ONLY in `api/auth/` — never duplicated

## File Naming Conventions
- React components: PascalCase.tsx (e.g., ProductCard.tsx)
- API routes: snake_case.py (e.g., product_routes.py)
- Utility functions: camelCase.ts (e.g., formatPrice.ts)
- Test files: test_<filename>.py or <filename>.test.tsx

## Forbidden Patterns
- NEVER use `any` type in TypeScript — use proper types or `unknown`
- NEVER commit API keys or secrets — use environment variables
- NEVER use `console.log` in production code — use the logger service
- NEVER write business logic in React components — use hooks or services

## Test Standards
- Every API endpoint MUST have at least one happy-path test and one error test
- Mock external services in tests — never call real APIs in tests
- Test files MUST be in the same directory as the file being tested

## Glob Pattern Rules
# rules for TypeScript files in the rules folder only
rules/*.ts: Follow the rule schema interface exactly, no deviations allowed

## Environment
- Local dev: docker-compose up
- Run tests: pytest api/ && npx vitest run
- Lint: ruff check api/ && npx eslint src/
```

### Key CLAUDE.md Rules for Exam

```markdown
# Good pattern — specific and actionable
## Forbidden Patterns
- NEVER use raw SQL strings — use SQLAlchemy ORM only
  ✗ db.execute("SELECT * FROM users WHERE id = " + user_id)
  ✓ db.query(User).filter(User.id == user_id).first()

# Good pattern — glob-scoped rules
# These rules apply ONLY to files matching the glob
rules/*.ts: All exported functions must conform to the RuleConfig interface
api/routes/*.py: All route handlers must use the @require_auth decorator
```

### Subdirectory Override Example

```markdown
# Root CLAUDE.md
Test standards: Use pytest for all tests, mock all external services

# api/integrations/CLAUDE.md  ← This OVERRIDES the root rule for this folder
Test standards: Integration tests in this folder SHOULD call real external
APIs using test credentials from .env.test — mocking defeats the purpose here.
```

---

## 3.2 Plan Mode vs. Direct Execution

### The Two Execution Modes

This is a common exam question — choosing the wrong mode is explicitly called out as an error.

| Mode | Use When | How It Works |
|------|----------|-------------|
| **Plan Mode** | Large changes, architectural decisions, multi-file refactoring | Claude proposes a plan → you review → Claude executes |
| **Direct Execution** | Single-file changes, clear stack traces, simple additions | Claude makes the change immediately |

### Decision Framework

```
Is this change across multiple files?
  YES → Plan Mode

Is this a major architectural change?
  YES → Plan Mode

Are you unsure of the full impact of the change?
  YES → Plan Mode

Is this a simple bug fix with a clear stack trace?
  YES → Direct Execution

Are you adding a single new function to one file?
  YES → Direct Execution

Are you changing a single clear error in one file?
  YES → Direct Execution
```

### Plan Mode Workflow

```
1. You type: "Refactor the authentication system to use JWT tokens"
         ↓
2. Claude enters Plan Mode:
   - Reads all relevant files
   - Identifies every file that needs to change
   - Produces a numbered plan with rationale

3. Claude shows you the plan:
   Plan:
   1. Create api/auth/jwt_handler.py
   2. Modify api/auth/middleware.py — replace session auth with JWT
   3. Update api/routes/user_routes.py — add JWT decorator
   4. Update tests/test_auth.py — add JWT test cases
   5. Remove api/auth/session_handler.py (deprecated)

4. YOU REVIEW the plan — check for missing steps or wrong assumptions

5. You approve: "Looks good, proceed"
         ↓
6. Claude executes each step
```

### Keyboard Shortcuts (Exam May Test These)

```bash
# Enter Plan Mode
Shift+Tab  # or type /plan in the Claude Code prompt

# In Plan Mode, to approve and execute:
# Type "go", "proceed", or "yes" after reviewing the plan
```

---

## 3.3 The -p Flag for CI/CD

### Non-Interactive Mode

The `--print` (or `-p`) flag makes Claude Code run without user interaction — critical for automation.

```bash
# Basic non-interactive execution
claude --print "Run the test suite and report any failures"

# With JSON output for machine parsing
claude --print --output-format json "Check for security vulnerabilities in the API"

# Pipe output to a file
claude --print "Generate a summary of recent code changes" > changes_summary.txt

# In a CI/CD pipeline script
claude --print "Run linting checks" && echo "Lint passed" || echo "Lint failed"
```

### GitHub Actions Integration Example

```yaml
# .github/workflows/claude_review.yml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  claude-review:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run Claude Code Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # Non-interactive review using -p flag
          claude --print \
            --output-format json \
            "Review the changed files in this PR for:
             1. Security vulnerabilities
             2. Violations of CLAUDE.md architectural rules
             3. Missing test coverage
             Return a JSON report with findings." \
          > review_report.json

      - name: Post Review Comment
        uses: actions/github-script@v6
        with:
          script: |
            const report = require('./review_report.json');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Claude Code Review\n${JSON.stringify(report, null, 2)}`
            });
```

### What Must Be in CLAUDE.md for CI/CD Runs

When Claude runs in CI without human interaction, it needs all context from CLAUDE.md:

```markdown
# CLAUDE.md — CI-critical sections

## Test Standards (Required for CI)
- Test command: pytest api/ -v --tb=short
- Frontend tests: npx vitest run --reporter=json
- Coverage threshold: 80% minimum

## Lint Standards (Required for CI)
- Python: ruff check api/ --output-format json
- TypeScript: npx eslint src/ --format json

## What Counts as a Failure
- Any test failure = block the PR
- Any security vulnerability of severity HIGH or CRITICAL = block the PR
- Missing tests for new API routes = flag but do not block
```

---

## 3.4 Subagent Patterns in Claude Code

### The Explore Subagent

The **Explore subagent** is Claude Code's built-in pattern for isolating verbose file discovery
from the main reasoning session.

```
Without Explore subagent:
  Main session: "Find all files that import the auth module"
  → Claude lists 87 files in main context → context bloated with file paths
  → Main session has less context space for actual reasoning

With Explore subagent:
  Explore subagent: discovers and filters files independently
  → Returns only relevant summary to main session
  → Main session context stays clean for architecture decisions
```

### The Review Isolation Pattern (Anti-Pattern 7 Prevention)

```bash
# WRONG — reviewing in the same session that wrote the code
# The session has context bias — it knows why it made each decision
# It is less likely to spot issues

claude> "Write a function to parse user input"
# ... Claude writes the function ...
claude> "Now review this function for security issues"
# ← Context bias: Claude "remembers" its own reasoning, less likely to find flaws


# CORRECT — use a fresh session for review
# Session 1: Write the code
claude> "Write a function to parse user input"
# Save the file, end session

# Session 2: Review (no context of why it was written)
claude> "Review src/utils/input_parser.py for security vulnerabilities and edge cases"
# ← No context bias — Claude evaluates the code purely on its merits
```

### Pattern: Plan → Implement → Review (Three Sessions)

```bash
# Session 1: Planning (Plan Mode)
claude --plan "Design the authentication system architecture for a multi-tenant SaaS"
# Review plan, approve, save plan to PLANNING.md

# Session 2: Implementation (Direct Execution)
claude "Implement the authentication system based on PLANNING.md"
# Claude implements, tests pass

# Session 3: Code Review (Fresh context)
claude "Review the authentication system implementation in api/auth/ for:
1. Security vulnerabilities
2. CLAUDE.md rule violations
3. Missing edge case handling"
```

---

## Domain 3 — Concept Check Questions

1. What is CLAUDE.md? Where should it be placed?
2. When a subdirectory CLAUDE.md conflicts with the root CLAUDE.md, which wins?
3. What is the difference between Plan Mode and Direct Execution?
4. Name 3 scenarios where you should use Plan Mode.
5. What does the `-p` flag do? Why is it important for CI/CD?
6. Why should code review happen in a different session than code writing?
7. What is the Explore subagent and what problem does it solve?

---

## Domain 3 Summary

| Rule | Detail |
|------|--------|
| CLAUDE.md = virtual tech lead | Defines rules, naming, constraints, test standards |
| Subdirectory overrides parent | Closest CLAUDE.md wins |
| Plan Mode for big changes | Multi-file refactors, architecture, unclear impact |
| Direct Execution for simple | Single file, clear stack trace |
| `-p` for CI/CD | Non-interactive, use `--output-format json` |
| Explore for file discovery | Isolates verbose results from main session |
| Review in fresh session | Prevents context bias (Anti-Pattern 7) |
| CLAUDE.md must define test standards | CI runs have no human to ask |
