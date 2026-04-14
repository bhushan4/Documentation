# AI-Powered Backend Engineering Guide

> A structured roadmap to go from 0/10 to 8/10 in AI-assisted backend engineering.
> Primary languages: **Go** and **Java**

---

## Table of Contents

- [How to Use This Guide](#how-to-use-this-guide)
- [Phase 1: Prompt Foundations — Think Before You Type](#phase-1-prompt-foundations--think-before-you-type)
- [Phase 2: New Projects with AI — The Right Setup](#phase-2-new-projects-with-ai--the-right-setup)
- [Phase 3: Legacy Code with AI — Migration and Review](#phase-3-legacy-code-with-ai--migration-and-review)
- [Phase 4: Interview and Live Coding with AI](#phase-4-interview-and-live-coding-with-ai)
- [Phase 5: Stakeholder Demo — Show Don't Tell](#phase-5-stakeholder-demo--show-dont-tell)
- [Prompt Templates Library](#prompt-templates-library)
- [Tools and Setup](#tools-and-setup)
- [Resources](#resources)

---

## How to Use This Guide

This guide is sequential. Each phase builds on the previous one. Don't skip ahead — the prompt foundations in Phase 1 are the single biggest lever for everything that follows.

**Time estimate:** 4–6 weeks of deliberate practice (1–2 hours/day) to reach 8/10.

**What 8/10 looks like:**
- You write prompts that produce production-grade code on the first or second iteration
- You can review, refactor, and migrate legacy code with AI 3–5x faster than without it
- You can build a new feature end-to-end with AI in a live interview setting
- You can demo your AI workflow to stakeholders and articulate the ROI clearly

**What 8/10 does NOT look like:**
- Blindly copying AI output without review
- Using AI as a replacement for understanding your system
- Skipping tests because "the AI wrote it"

---

## Phase 1: Prompt Foundations — Think Before You Type

**Level: 0 → 2/10**
**Duration: Week 1**

This is where 90% of engineers fail. They type vague requests and blame the model. The difference between a hallucination and a perfect output is often just a few lines of well-structured instruction.

### 1.1 The RCTF Framework

Every prompt you write should have four components:

| Component   | What It Does                                      | Example                                                          |
|-------------|---------------------------------------------------|------------------------------------------------------------------|
| **Role**    | Sets the persona and expertise level               | "You are a senior Go engineer specializing in distributed systems" |
| **Context** | Provides background the AI needs                   | "We use Go 1.22, gRPC, PostgreSQL, and deploy on Kubernetes"      |
| **Task**    | The specific thing you want done                   | "Write a middleware that rate-limits API requests per tenant"      |
| **Format**  | How you want the output structured                 | "Return the code with inline comments. Add a test file separately" |

**Bad prompt:**
```
Write a rate limiter in Go
```

**Good prompt:**
```
Role: You are a senior Go engineer building production microservices.

Context: We use Go 1.22, the chi router, and Redis for shared state.
The service handles ~10k req/s across 4 pods behind a load balancer.
Rate limits must be per-tenant (identified by X-Tenant-ID header).

Task: Write a middleware function that enforces rate limiting using
a sliding window algorithm backed by Redis. Return 429 when exceeded.

Format:
- middleware.go — the rate limiter middleware
- middleware_test.go — table-driven tests covering: normal flow,
  limit exceeded, missing tenant header, Redis connection failure
- Use only the standard library + go-redis/redis/v9
```

### 1.2 Use Delimiters to Separate Instructions from Data

When passing code or logs to the AI, wrap them in clear boundaries so the model doesn't confuse your instructions with the data.

```
Review the following Go function for correctness, performance, and idiomatic style.

<code>
func ProcessOrders(ctx context.Context, orders []Order) error {
    for _, o := range orders {
        if err := db.Save(ctx, o); err != nil {
            return err
        }
    }
    return nil
}
</code>

Focus on:
1. Error handling patterns
2. Whether this should use batch operations
3. Context propagation
```

### 1.3 Few-Shot Prompting — Teach by Example

Instead of describing what you want, show the AI 2–3 examples of your desired output. This works exceptionally well for code transformations and review comments.

```
Convert these Java DTOs to Go structs following our conventions.

Example input:
public class UserDTO {
    private String id;
    private String email;
    private Instant createdAt;
}

Example output:
type User struct {
    ID        string    `json:"id" db:"id"`
    Email     string    `json:"email" db:"email"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
}

Now convert this:
public class OrderDTO {
    private UUID id;
    private UUID userId;
    private BigDecimal totalAmount;
    private OrderStatus status;
    private Instant placedAt;
}
```

### 1.4 Chain-of-Thought for Debugging and Design

For complex problems, ask the AI to reason step by step before giving an answer.

```
A user reports that our Go payment service occasionally processes
duplicate charges. Here's the handler code and the database schema.

<handler_code>
// ... paste code here
</handler_code>

<schema>
// ... paste schema here
</schema>

Before suggesting a fix:
1. Trace through the code path for a normal successful payment
2. Identify every point where a failure + retry could cause a duplicate
3. Check if the database schema supports idempotency
4. Then propose a fix with the minimal set of changes needed
```

### 1.5 Iterative Refinement Over Single-Shot

Never accept the first output. Treat AI like a brilliant but literal-minded junior engineer.

**Iteration loop:**
1. Generate → Review → Identify gaps → Refine prompt → Regenerate
2. After generation, ask: "What edge cases does this miss?"
3. Then: "Now add error handling for [specific scenario]"
4. Then: "Write tests that cover the failure modes you identified"

### 1.6 Model-Specific Tips

**Claude** excels with contract-style prompts — explicit success criteria, output format, and constraints. XML tags work well as delimiters.

**General rules across models:**
- Specify library versions explicitly ("use go-redis/redis/v9", "use Spring Boot 3.2")
- Say "use only the standard library" if you don't want external deps
- Tell it what NOT to do: "Do not use deprecated APIs. Do not use global state."
- End with a verification step: "Before finalizing, verify the code compiles and handles all error paths"

### 1.7 Practice Exercises — Phase 1

Complete these before moving on:

- [ ] Rewrite 3 of your recent AI prompts using the RCTF framework. Compare output quality.
- [ ] Take a bug report from your backlog. Write a chain-of-thought debugging prompt.
- [ ] Create a few-shot prompt that converts between two formats you commonly work with.
- [ ] Write a code review prompt for a PR you recently reviewed manually. Compare your review with the AI's.

---

## Phase 2: New Projects with AI — The Right Setup

**Level: 2 → 4/10**
**Duration: Week 2**

The biggest mistake is jumping straight into code generation. The first step is always specification and planning.

### 2.1 Spec-First Development

Before writing any code, use AI to flesh out your requirements.

```
I'm building a new Go microservice for inventory management.

Business context: E-commerce platform, ~500 SKUs, real-time stock
updates from warehouse API, needs to handle flash sales (10x normal
traffic for 30-minute windows).

Ask me questions iteratively until we have a complete spec covering:
- API contract (endpoints, request/response shapes)
- Data model
- Key technical decisions (caching strategy, concurrency model)
- Edge cases and failure modes
- Non-functional requirements (latency, throughput, availability)

Ask one question at a time. Wait for my answer before the next.
```

### 2.2 Harness Engineering — Context Files

Top engineering teams don't paste context into every prompt. They embed it in the repository. Create these files at the root of every project:

**`CLAUDE.md`** (or `AI_CONTEXT.md`):
```markdown
# Project: Inventory Service

## Tech Stack
- Go 1.22
- PostgreSQL 16 with pgx/v5
- Redis 7 for caching
- gRPC for inter-service communication
- Docker + Kubernetes deployment

## Architecture Decisions
- Hexagonal architecture: domain → ports → adapters
- All database queries use sqlc-generated code (do not write raw SQL)
- Errors use custom error types in internal/apperror/
- Context must be propagated through all function signatures

## Naming Conventions
- Package names: lowercase, single word (e.g., inventory, warehouse)
- Interface names: verb-based (e.g., StockChecker, OrderPlacer)
- Test files: *_test.go with table-driven tests
- Config: environment variables via envconfig struct tags

## Patterns to Follow
- Repository pattern for all data access
- Dependency injection via constructor functions (NewService(...))
- Structured logging with slog
- Graceful shutdown with signal handling

## Patterns to Avoid
- No init() functions
- No package-level variables (except errors)
- No direct HTTP handler business logic — always delegate to service layer
- No ORM (we use sqlc)

## Testing
- Table-driven tests for all public functions
- Mocks generated with mockgen
- Integration tests use testcontainers-go
```

**`CONVENTIONS.md`** for code style:
```markdown
# Code Conventions

## Error Handling
- Wrap errors with fmt.Errorf("operation: %w", err)
- Never swallow errors silently
- Use custom error types for domain errors

## API Design
- All endpoints return JSON with envelope: {"data": ..., "error": ...}
- Pagination uses cursor-based approach (not offset)
- Versioning via URL path prefix: /v1/

## Git
- Conventional commits: feat:, fix:, refactor:, test:, docs:
- One logical change per commit
- PR descriptions must include: what, why, how to test
```

### 2.3 The Project Kickoff Sequence

Follow this order when starting any new project with AI:

```
Step 1: SPEC      → Use AI to brainstorm and refine requirements
Step 2: CONTEXT   → Create CLAUDE.md and CONVENTIONS.md
Step 3: SCAFFOLD  → Generate project structure and boilerplate
Step 4: DOMAIN    → Define domain types and interfaces first
Step 5: TESTS     → Write test cases before implementation
Step 6: IMPLEMENT → Generate implementation code with tests as guide
Step 7: REVIEW    → Ask AI to review its own output against the spec
```

**Scaffold prompt example:**
```
Given the spec in CLAUDE.md above, generate the initial project
structure for this Go service. Include:

- Directory layout following hexagonal architecture
- go.mod with the dependencies listed in the spec
- Makefile with targets: build, test, lint, run, docker-build
- Dockerfile (multi-stage, distroless final image)
- docker-compose.yml for local development (app + postgres + redis)
- .golangci.yml with our standard linter config
- Empty placeholder files with package declarations for each package

Do NOT generate any business logic yet. Structure only.
```

### 2.4 Test-Driven AI Development

Write tests first, then ask AI to make them pass. This is the single most reliable way to get correct AI-generated code.

```
Here are the test cases for our inventory service's stock reservation logic.

<test_file>
func TestReserveStock(t *testing.T) {
    tests := []struct {
        name        string
        sku         string
        quantity    int
        available   int
        wantErr     error
    }{
        {"sufficient stock", "SKU-001", 5, 100, nil},
        {"exact stock", "SKU-001", 100, 100, nil},
        {"insufficient stock", "SKU-001", 101, 100, apperror.ErrInsufficientStock},
        {"zero quantity", "SKU-001", 0, 100, apperror.ErrInvalidQuantity},
        {"negative quantity", "SKU-001", -1, 100, apperror.ErrInvalidQuantity},
        {"unknown SKU", "UNKNOWN", 1, 0, apperror.ErrSKUNotFound},
    }
    // ... test body
}
</test_file>

Write the ReserveStock method that makes all these tests pass.
Follow the patterns in CLAUDE.md. Use the repository interface,
not direct database calls.
```

### 2.5 Practice Exercises — Phase 2

- [ ] Pick a small service you need to build. Write a full spec using the iterative questioning prompt.
- [ ] Create CLAUDE.md and CONVENTIONS.md for one of your existing projects.
- [ ] Generate a project scaffold. Compare it to how you'd structure it manually.
- [ ] Write tests first for one feature, then use AI to implement.

---

## Phase 3: Legacy Code with AI — Migration and Review

**Level: 4 → 6/10**
**Duration: Weeks 3–4**

This is the highest-value skill in enterprise engineering. Every company has legacy code. Few engineers can modernize it efficiently.

### 3.1 Code Explanation Prompt

When you encounter unfamiliar legacy code, start here:

```
Role: You are a senior Java engineer performing a codebase audit.

Here is a legacy Java class from our payment processing system.
It was written in 2018 and uses Java 8 patterns.

<code>
// ... paste the legacy class
</code>

Explain:
1. What this class does (high-level purpose in 2-3 sentences)
2. The data flow through its public methods
3. Dependencies it relies on (both explicit imports and implicit assumptions)
4. Code smells or anti-patterns you notice
5. Any potential bugs or race conditions
6. What would break if we modified it carelessly

Keep explanations concrete — reference specific line numbers and method names.
```

### 3.2 Code Review Prompt Template

Use this when reviewing PRs or auditing code:

```
Role: You are a senior backend engineer reviewing code for a team.

<code_to_review>
// ... paste the code
</code_to_review>

<project_context>
- Language: Go 1.22 / Java 21
- This code handles: [what it does]
- It runs in: [environment — k8s, serverless, etc.]
- Traffic: [approximate volume]
</project_context>

Review for:
1. **Correctness** — Logic errors, edge cases, off-by-one, nil/null handling
2. **Security** — SQL injection, auth bypass, data exposure, input validation
3. **Performance** — N+1 queries, unnecessary allocations, missing indexes
4. **Reliability** — Error handling, timeout handling, graceful degradation
5. **Maintainability** — Naming, structure, coupling, test coverage

Format your review as:
- P1 (must fix): Issues that will cause bugs or outages
- P2 (should fix): Issues that will cause pain over time
- P3 (consider): Improvements for code quality

For each issue, include: the specific code location, what's wrong,
why it matters, and a concrete fix.
```

### 3.3 Chunking Large Codebases

AI has a context window limit. For large legacy systems, use this strategy:

```
Approach: Progressive Context Loading

1. START WITH THE MAP
   "Here is the directory structure and package layout of our service.
    Explain the overall architecture based on package names and file names alone."

2. LOAD THE INTERFACES
   "Here are all the interfaces/contracts in our domain layer.
    What capabilities does this system have?"

3. LOAD ONE FLOW AT A TIME
   "I want to understand the order creation flow. Here are the files
    involved, in call order: handler.go → service.go → repository.go.
    Trace the request through these layers."

4. CROSS-REFERENCE
   "Now here's the inventory service that order creation calls.
    Does the interaction between these two services look correct?
    What happens if the inventory call fails after the order is created?"
```

### 3.4 Migration Planning

For modernizing legacy code, use a structured approach:

```
Role: You are a principal engineer planning a migration.

We need to migrate our Java 8 monolith payment module to a
standalone Go microservice.

<current_java_code>
// ... key classes (service layer, repository, DTOs)
</current_java_code>

<current_database_schema>
// ... relevant tables
</current_database_schema>

Create a migration plan that includes:
1. **Inventory**: List every public method/endpoint, its inputs/outputs,
   and which other modules call it
2. **Dependency map**: What does this module depend on? What depends on it?
3. **Risk assessment**: Which parts are high-risk to migrate? Why?
4. **Migration phases**: Break the migration into safe, incremental steps
   where the system works at every intermediate stage
5. **Strangler fig approach**: How to run old and new side-by-side
6. **Verification**: How to confirm the new service is correct
   (shadow traffic, feature flags, comparison testing)

Prioritize safety over speed. Each phase should be independently deployable.
```

### 3.5 Refactoring Workflows

For incremental refactoring within the same language:

```
I want to refactor this Java service class. It's 800 lines and
does too many things.

<code>
// ... paste the service class
</code>

Propose a refactoring plan:
1. Identify the distinct responsibilities in this class
2. Suggest how to split it (new class names and their responsibilities)
3. For each new class, list which methods move there
4. Identify shared state that needs to be resolved
5. Show the dependency graph between the new classes
6. Provide the refactored code for ONE of the new classes as an example

Constraints:
- No changes to the public API of the original class (other callers depend on it)
- Each refactoring step must keep all existing tests passing
- Use constructor injection for new dependencies
```

### 3.6 Practice Exercises — Phase 3

- [ ] Pick a legacy class in your codebase. Use the explanation prompt. Verify accuracy against your own knowledge.
- [ ] Do an AI-assisted code review on a recent PR. Compare with the human review that happened.
- [ ] Take a large file and practice the chunking strategy. See how far the AI can go.
- [ ] Write a migration plan for one module in your system.

---

## Phase 4: Interview and Live Coding with AI

**Level: 6 → 7/10**
**Duration: Week 5**

### 4.1 The Feature Build Approach

When an interviewer or manager says "build X using AI," follow this exact sequence:

```
Step 1: CLARIFY (2 min)
   - Ask clarifying questions about requirements
   - Confirm constraints (language, framework, external deps)
   - Identify acceptance criteria

Step 2: DESIGN (3 min)
   - Sketch the approach in plain language
   - Identify key components and their interactions
   - State your assumptions out loud

Step 3: PROMPT (2 min)
   - Write a structured prompt using RCTF
   - Include the constraints and context from Steps 1-2
   - Specify test requirements in the prompt

Step 4: GENERATE + REVIEW (5-10 min)
   - Generate the code
   - READ the output carefully — don't just accept it
   - Identify what's good and what needs changes
   - Iterate with targeted follow-up prompts

Step 5: TEST + VALIDATE (3-5 min)
   - Run or trace through the tests
   - Ask AI to identify edge cases it missed
   - Add test cases for those edge cases

Step 6: EXPLAIN (2 min)
   - Walk through the final code
   - Explain design decisions
   - Point out where you overrode or corrected the AI
```

**Key message to convey:** You are the architect. The AI is a tool you direct. You should be able to explain every line of the output.

### 4.2 The Code Review Approach

When asked to review code in an interview/work setting:

```
Step 1: READ first — skim the code yourself for 60 seconds
Step 2: Form initial impressions before using AI
Step 3: Feed the code to AI with your review prompt template (Section 3.2)
Step 4: Compare AI findings with your own observations
Step 5: OVERRIDE the AI where it's wrong or irrelevant
Step 6: Add your own insights that the AI missed (domain-specific context)
Step 7: Present a prioritized summary (P1/P2/P3)
```

**What interviewers want to see:**
- You don't blindly trust AI output
- You can identify when the AI is wrong
- You add domain context the AI doesn't have
- You prioritize findings by impact
- You can explain the "why" behind each issue

### 4.3 System Design with AI

For system design interviews where AI use is allowed:

```
Prompt for exploration:
"I'm designing a [system]. Help me think through the architecture.
 Start by asking me about requirements, constraints, and scale.
 After we agree on requirements, propose 2-3 architectural options
 with trade-offs for each."

Prompt for deep-dive:
"We've chosen [approach]. Now let's design the [specific component].
 Show me the data model, API contract, and sequence diagram for
 the [specific flow]."

Prompt for capacity estimation:
"Given [traffic numbers], calculate the storage, compute, and
 bandwidth requirements. Show your math step by step."
```

### 4.4 Phrases That Show AI Maturity

Use these in interviews and conversations:

- "Let me structure the prompt with the right context before generating..."
- "The AI suggested X, but I'd change it to Y because [domain reason]..."
- "I'll write the test cases first, then use AI to implement against them..."
- "This output looks correct syntactically, but let me verify the concurrency model..."
- "The AI doesn't know about our [specific constraint], so I need to add..."

### 4.5 Practice Exercises — Phase 4

- [ ] Set a timer for 25 minutes. Build a CRUD REST API for a resource of your choice using the feature-build approach.
- [ ] Take a Go or Java file you didn't write. Review it using the code-review approach.
- [ ] Practice a system design question (e.g., "Design a URL shortener") using AI as your collaborator.
- [ ] Record yourself walking through the process. Watch it back and refine.

---

## Phase 5: Stakeholder Demo — Show Don't Tell

**Level: 7 → 8/10**
**Duration: Week 6**

### 5.1 The Demo Structure

When presenting AI workflows to leadership or internal teams:

```
1. THE PROBLEM (1 min)
   - "This type of task currently takes X hours / introduces Y bugs"
   - Use a real example from your team's work

2. THE BEFORE (2 min)
   - Show the manual approach briefly
   - Highlight the pain points

3. THE AI-ASSISTED APPROACH (5 min)
   - Walk through your structured workflow live
   - Show the prompt, the generation, and the review
   - Narrate your decision-making as you go

4. THE RESULT (2 min)
   - Show the output side-by-side with the manual approach
   - Highlight quality, completeness, and time saved

5. THE METRICS (2 min)
   - Time saved: "This took 20 minutes vs. the usual 3 hours"
   - Quality: "The AI caught 2 edge cases I would have missed"
   - Consistency: "Every review follows the same checklist now"

6. HANDLING OBJECTIONS (ongoing)
   - "What about hallucinations?" → "That's why the review step exists.
     I verify every output against the tests and the spec."
   - "What about sensitive code?" → "We use [X] which doesn't retain data.
     And the human review catches any leakage."
   - "Will this replace developers?" → "No — it replaces the grunt work.
     I spend more time on architecture and less on boilerplate."
```

### 5.2 Demo Scenarios That Impress

Pick 2-3 of these for your demo:

| Scenario | What to Show | Impact |
|---|---|---|
| Bug investigation | Paste stack trace + relevant code → AI traces root cause | "Found in 5 minutes vs. 2 hours" |
| Code review | Feed a PR → structured P1/P2/P3 review | "Consistent quality every review" |
| Test generation | Point at a module → comprehensive test suite | "80% coverage generated in minutes" |
| Legacy understanding | Paste undocumented code → full explanation + docs | "New team members ramp up 3x faster" |
| API design | Describe requirements → full OpenAPI spec + models | "From idea to contract in 15 minutes" |
| Migration planning | Feed old code → detailed migration plan | "Safe migration path in an hour vs. days" |

### 5.3 Metrics to Track

Keep a log of your AI-assisted work for 2 weeks:

```markdown
## AI Usage Log

| Date | Task | Without AI (est.) | With AI | Quality Notes |
|------|------|-------------------|---------|---------------|
| 4/14 | Code review (PR #234) | 45 min | 15 min | Caught 1 extra race condition |
| 4/14 | Write repo layer tests | 2 hours | 30 min | 12 test cases, all pass |
| 4/15 | Debug timeout issue | 3 hours | 40 min | AI traced to connection pool config |
```

### 5.4 Practice Exercises — Phase 5

- [ ] Record a 10-minute demo of your AI workflow for one real task.
- [ ] Keep the usage log for one full week.
- [ ] Present to one trusted colleague and get feedback.
- [ ] Prepare answers for the 5 most common objections.

---

## Prompt Templates Library

Copy-paste ready templates for daily use.

### Template: New Feature

```
Role: Senior [Go/Java] engineer building production microservices.

Context:
- Language: [Go 1.22 / Java 21 + Spring Boot 3.2]
- Database: [PostgreSQL / MySQL / MongoDB]
- Architecture: [hexagonal / layered / clean]
- Dependencies: [list allowed libraries]
- Constraints: [concurrency model, performance targets, etc.]

Task: Implement [feature description].

Requirements:
- [Requirement 1]
- [Requirement 2]
- [Edge case to handle]

Format:
- Source files with inline comments for non-obvious logic
- Separate test file with table-driven / parameterized tests
- Error handling for all failure modes
- No deprecated APIs
```

### Template: Code Review

```
Review this [Go/Java] code for a production [service type].

<code>
[paste code]
</code>

Context: [what it does, traffic level, deployment environment]

Review criteria (in priority order):
1. Correctness — bugs, logic errors, race conditions
2. Security — injection, auth, data exposure
3. Performance — allocations, queries, caching
4. Reliability — error handling, timeouts, retries
5. Maintainability — naming, structure, testability

Format: P1 (must fix) → P2 (should fix) → P3 (nice to have)
Each item: location, problem, why it matters, concrete fix.
```

### Template: Debug Investigation

```
A [description of the issue] is occurring in our [Go/Java] [service name].

<error_output>
[paste stack trace, logs, or error messages]
</error_output>

<relevant_code>
[paste the code in the execution path]
</relevant_code>

<environment>
[runtime version, OS, deployment info, recent changes]
</environment>

Step through this methodically:
1. Parse the error — what is it actually saying?
2. Trace the code path that leads to this error
3. Identify the root cause (not just the symptom)
4. Propose a fix with minimal blast radius
5. Suggest how to add a test or alert to prevent recurrence
```

### Template: Explain Legacy Code

```
Explain this legacy [Java/Go] code as if briefing a new team member.

<code>
[paste code]
</code>

Cover:
1. Purpose — what does this code do? (2-3 sentences)
2. Flow — trace the main execution path
3. Dependencies — what does it rely on? (explicit + implicit)
4. Risks — what's fragile? What breaks if modified carelessly?
5. Improvement opportunities — what would you change in a modernization?

Reference specific method/function names and line numbers.
```

### Template: Write Tests

```
Write comprehensive tests for the following [Go/Java] code.

<code>
[paste the function/class to test]
</code>

Requirements:
- [Go: table-driven tests / Java: parameterized tests with JUnit 5]
- Cover: happy path, edge cases, error conditions, boundary values
- Mock external dependencies using [Go: mockgen / Java: Mockito]
- Each test name should describe the scenario it validates
- Include setup/teardown if needed
- Aim for full branch coverage

Do NOT test private/internal implementation details.
Focus on the public contract and observable behavior.
```

---

## Tools and Setup

### Recommended Tool Stack (2026)

| Category | Tool | Use Case |
|---|---|---|
| AI coding assistant | Claude Code (CLI) | Agentic coding, repo-aware tasks |
| AI chat | Claude.ai / ChatGPT | Design discussions, explanations, planning |
| IDE integration | Cursor / GitHub Copilot | Inline suggestions, completions |
| Code review AI | In your CI pipeline | Automated PR review |
| Testing AI | AI test generation tools | Test coverage expansion |

### Setup Checklist

- [ ] Install Claude Code or your preferred CLI tool
- [ ] Create `CLAUDE.md` for your top 3 repositories
- [ ] Create `CONVENTIONS.md` for your team's coding standards
- [ ] Set up a prompt templates directory in your dotfiles
- [ ] Configure your editor with AI inline suggestions
- [ ] Set up an AI usage log (spreadsheet or markdown file)

---

## Resources

### Industry Reading
- Addy Osmani — "My LLM Coding Workflow Going Into 2026"
- The Pragmatic Engineer — "AI Tooling for Software Engineers in 2026"
- Red Hat Developer — "Harness Engineering: Structured Workflows for AI-Assisted Development"
- Anthropic — Prompt Engineering documentation (docs.claude.com)

### Key Concepts
- **Harness Engineering**: Designing the environment the AI works in, rather than perfecting individual prompts. Structure in, structure out.
- **Context Engineering**: Managing what information the AI has access to at each step. The LLM is a CPU, the context window is RAM, your job is the OS.
- **Engineer-in-the-Loop**: AI generates, human validates. Every output is reviewed against tests, specs, and domain knowledge.

### Community Benchmarks (2026)
- 95% of professional engineers use AI tools weekly
- 75% use AI for half or more of their engineering work
- 55% regularly use AI agents for complex tasks
- Top engineers use 2–4 AI tools simultaneously

---

## Progress Tracker

| Phase | Status | Date Started | Date Completed | Self-Rating |
|-------|--------|-------------|----------------|-------------|
| Phase 1: Prompt Foundations | ⬜ Not started | | | /10 |
| Phase 2: New Projects | ⬜ Not started | | | /10 |
| Phase 3: Legacy Code | ⬜ Not started | | | /10 |
| Phase 4: Interviews | ⬜ Not started | | | /10 |
| Phase 5: Stakeholder Demo | ⬜ Not started | | | /10 |

---

*Last updated: April 2026*
*Built for backend engineers (Go + Java) leveling up with AI.*