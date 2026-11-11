# Code Review Practices

## Introduction

Code review is the practice of having other developers examine your code before it is merged. It is one of the most effective quality assurance practices in software engineering — catching bugs, sharing knowledge, and maintaining consistency across a codebase.

**Why Code Review Matters:**

- **Bug detection**: Reviews catch 60-75% of defects in PRs under 400 lines

- **Knowledge sharing**: Reviewers learn the codebase; authors learn better patterns

- **Consistency**: The team converges on shared conventions and standards

- **Security**: A second pair of eyes catches vulnerabilities that automated tools miss

- **Mentorship**: Junior developers learn from senior feedback, and seniors stay grounded

**The 2026 Context:** With AI tools generating 41% of commits (DORA 2025), human review is more important than ever. AI writes syntactically correct code, but humans verify business logic, security, and architectural fit.

------------------------------------------------------------------------

## PR Sizing

The single most impactful practice for effective code review is **keeping PRs small**.

### The Data

| PR Size        | Defect Detection Rate | Review Time   | Merge Cycle |
|----------------|-----------------------|---------------|-------------|
| \< 200 lines   | ~80%                  | 15-30 min     | Same day    |
| 200-400 lines  | ~75%                  | 30-60 min     | 1 day       |
| 400-1000 lines | ~50%                  | 1-2 hours     | 2-3 days    |
| \> 1000 lines  | ~31%                  | Often skimmed | 3+ days     |

### How to Keep PRs Small

1.  **One concern per PR**: A PR should do one thing (add a feature, fix a bug, refactor code)

2.  **Stack PRs**: Break large features into a sequence of small, mergeable PRs

3.  **Separate refactoring from feature work**: Don’t mix code cleanup with new functionality

4.  **Extract infrastructure changes**: Database migrations, CI changes, and dependency updates should be separate PRs

### What Belongs in One PR

```
GOOD (focused):
- "Add JWT authentication endpoint"
- "Fix pagination offset bug in ticket list"
- "Refactor database session management to async"

BAD (mixed concerns):
- "Add auth, fix pagination, update deps, and refactor DB"
- "Sprint 3 work" (all changes from a 2-week sprint)
```

------------------------------------------------------------------------

## Review Targets

| Metric | Target | Why |
|----|----|----|
| **Time to first review** | \< 6 hours | Author stays in context; not blocked for days |
| **Review cycle time** | \< 24 hours | Keep the PR from going stale |
| **Number of review rounds** | 1-2 | If more, the PR is too large or requirements were unclear |
| **Reviewer count** | 1-2 reviewers | More reviewers = diminishing returns |

------------------------------------------------------------------------

## The Review Checklist

Focus on **5 categories** (not 30+ items). Automated tools handle formatting and style.

### 1. Correctness

- Does the code do what the PR description says?

- Are edge cases handled (empty lists, null values, boundary conditions)?

- Is error handling appropriate (not swallowed, not too generic)?

- Do tests actually test the behavior described?

### 2. Security

- Is user input validated before use?

- Are SQL queries parameterized (no string concatenation)?

- Are secrets stored in environment variables (not hardcoded)?

- Does the endpoint check both authentication AND authorization?

- Are error messages safe (no stack traces, no internal details)?

### 3. Performance

- Are there N+1 query patterns? (ORM lazy loading)

- Are list endpoints paginated?

- Are expensive operations cached where appropriate?

- Are database queries using indexes?

### 4. Readability

- Are function and variable names clear and descriptive?

- Is the code self-documenting (no need for comments explaining *what*)?

- Are functions focused (\< 50 lines) and files cohesive (\< 800 lines)?

- Is nesting depth reasonable (\< 4 levels)?

### 5. Test Coverage

- Are new features covered by tests?

- Do tests cover the happy path AND error paths?

- Are tests independent (no shared mutable state)?

- Is test coverage at least 80%?

------------------------------------------------------------------------

## How to Write Good Review Comments

### Severity Levels

Use prefixes to communicate urgency:

| Prefix | Meaning | Action |
|----|----|----|
| **`blocker:`** | Security vulnerability or correctness bug | Must fix before merge |
| **`concern:`** | Significant quality or performance issue | Should fix before merge |
| **`suggestion:`** | Improvement idea | Consider, but optional |
| **`nit:`** | Style or minor preference | Truly optional |
| **`question:`** | Need clarification | Explain, no code change needed |

### Examples

```
blocker: This query concatenates user input directly into SQL.
Use parameterized queries to prevent SQL injection.

concern: This endpoint returns all user fields including `hashed_password`.
Use a response_model that excludes sensitive fields.

suggestion: Consider extracting this validation into a shared utility —
the same logic appears in tickets.py and bookings.py.

nit: Prefer `user_count` over `cnt` for readability.

question: Why is this endpoint not behind authentication?
Is it intentionally public?
```

### Good vs Bad Comments

```
BAD: "This is wrong."
GOOD: "blocker: This returns 200 for failed deletions.
Return 404 when the ticket doesn't exist."

BAD: "Use a different approach."
GOOD: "suggestion: Consider using `joinedload` here to avoid
N+1 queries — this will execute 1 query instead of N+1."

BAD: "LGTM" (on a 500-line PR)
GOOD: "LGTM — verified auth flow works correctly.
One suggestion on line 42 for error handling."
```

------------------------------------------------------------------------

## AI-Assisted Code Review (2026)

AI review tools handle the first pass (mechanical checks), freeing human reviewers for architecture and business logic:

### How to Use AI Review Effectively

| AI Handles (Automate) | Humans Handle (Focus Here) |
|----|----|
| Formatting and style violations | Business logic correctness |
| Simple bug patterns | Architecture and design decisions |
| Security anti-patterns (SQL injection, hardcoded secrets) | Edge cases and domain knowledge |
| Test coverage gaps | User experience implications |
| Dependency vulnerability alerts | Performance under real-world load |

### Workflow: AI + Human Review

``` mermaid
        flowchart LR
    A[Developer opens PR] --> B[CI: Lint + Test + Security Scan]
    B --> C[AI Review: Mechanical checks]
    C --> D[Human Review: Business logic + Architecture]
    D -->|Approved| E[Merge]
    D -->|Changes requested| F[Developer fixes]
    F --> B
```

AI-generated review comments should be treated as suggestions, not gospel. The human reviewer makes the final call on whether an AI flag is a real issue or a false positive.

------------------------------------------------------------------------

## CODEOWNERS

Automatically assign reviewers based on file paths:

```
# .gitlab/CODEOWNERS (or .github/CODEOWNERS)

# Default: team leads review everything
*                       @team-leads

# Domain-specific reviewers
src/auth/               @security-team
src/models/             @backend-team
src/templates/          @frontend-team
tests/                  @qa-team
.gitlab-ci.yml          @devops-team
```

------------------------------------------------------------------------

## Common Review Anti-Patterns

| Anti-Pattern | Problem | Fix |
|----|----|----|
| **Rubber stamping** | Approving without reading | Require comments on PRs \> 100 lines |
| **Bikeshedding** | Arguing about trivial style choices | Enforce style with linters (ruff, black) |
| **Review hoarding** | One person reviews everything | Use CODEOWNERS and rotate reviewers |
| **Stale PRs** | PRs open for days/weeks | Set SLA (\< 24h review cycle) |
| **Massive PRs** | 1000+ line PRs that no one can review | Require PRs \< 400 lines |
| **Drive-by LGTM** | “Looks good” with no actual review | Require at least 1 specific comment |

------------------------------------------------------------------------

## Summary

| Topic | Key Takeaway |
|----|----|
| **PR size** | Keep PRs under 400 lines for 75%+ defect detection |
| **Review speed** | First review within 6 hours, complete within 24 hours |
| **Checklist** | Focus on 5 categories: correctness, security, performance, readability, tests |
| **Comments** | Use severity prefixes (blocker/concern/suggestion/nit/question) |
| **AI review** | Let AI handle mechanical checks; humans focus on business logic |
| **CODEOWNERS** | Auto-assign reviewers by file path |

------------------------------------------------------------------------

## References

1.  [Google Engineering Practices — Code Review](https://google.github.io/eng-practices/review/)

2.  [SmartBear — Best Practices for Code Review](https://smartbear.com/learn/code-review/best-practices-for-peer-code-review/)

3.  [DORA Report 2025 — State of DevOps](https://dora.dev/)

4.  [Conventional Comments](https://conventionalcomments.org/)

## Practice

### Exercise 1: Review a PR

Review this pull request diff and write review comments using severity prefixes:

``` python
# Changes in routers/tickets.py
@router.post("/tickets")
async def create_ticket(content: str, db: Session = Depends(get_db)):
    ticket = Ticket(
        ticket_id=str(uuid4()),
        content=content,
        user_id=1,  # Hardcoded user ID
        status="pending",
    )
    db.add(ticket)
    db.commit()
    return ticket.__dict__

@router.get("/tickets")
async def list_tickets(db: Session = Depends(get_db)):
    return db.query(Ticket).all()  # No pagination, no auth

@router.delete("/tickets/{id}")
async def delete_ticket(id: int, db: Session = Depends(get_db)):
    db.execute(f"DELETE FROM tickets WHERE id = {id}")  # SQL injection!
    db.commit()
    return {"status": "deleted"}
```

**Tasks:**

1.  Write at least 5 review comments with appropriate severity levels

2.  For each blocker, provide the correct code

### Exercise 2: Improve Your Own PR

Take a recent PR you wrote (or a practice assignment) and:

1.  Check it against the 5-category review checklist

2.  Is the PR under 400 lines? If not, how would you split it?

3.  Does it have a clear description explaining *what* and *why*?

4.  Are there tests for the new/changed behavior?

## Review Questions

1.  **Why do PRs over 1,000 lines have only a 31% defect detection rate?**

    - *Hint*: Think about reviewer fatigue and cognitive load.

2.  **What is the difference between a “blocker” and a “suggestion” in code review?**

    - *Hint*: One must be fixed before merge, the other is optional.

3.  **Why should code formatting and style checks be automated rather than reviewed by humans?**

    - *Hint*: Think about consistency, speed, and what humans are uniquely good at.

4.  **A developer says “My PR is 800 lines but it’s all one feature — I can’t split it.” How would you advise them?**

    - *Hint*: Separate infrastructure (models, migrations) from business logic from tests.

5.  **What role should AI play in code review? What should it NOT be trusted for?**

    - *Hint*: AI is good at pattern matching but lacks business context.

6.  **What is the CODEOWNERS file and why is it useful?**

    - *Hint*: Think about ensuring the right people review changes to the right files.
