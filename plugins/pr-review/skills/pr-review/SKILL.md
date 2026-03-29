---
name: pr-review
description: "Reviews pull requests for code quality. Use when reviewing PRs or checking code changes. Triggers include: any mention of 'review this PR', 'code review', 'review my code', 'check this diff', 'review these changes', 'look at my PR', 'PR feedback', 'what do you think of this code', or when the user shares a diff, code changes, or a GitHub PR link for review purposes. Also trigger when the user says 'is this code okay', 'any issues with this', 'spot check this', or 'sanity check my changes'. This skill focuses on general code quality — correctness, readability, maintainability, and best practices. For frontend-specific accessibility/UX concerns use the frontend-review skill. For backend-specific security/compliance concerns use the backend-review skill. Do NOT use for writing PR descriptions (use pr-description skill), debugging existing bugs (use debugger skill), or writing new code from scratch."
---

# PR Code Review

## Purpose

Perform thorough, constructive code reviews that catch real issues while respecting the author's intent. A good review improves code quality without becoming a gatekeeping exercise. Focus on things that matter — correctness, maintainability, and clarity — not style preferences that linters should handle.

## Review Mindset

Before diving into line-by-line feedback, understand the PR's purpose. Read the description (if available), scan the file list to grasp scope, then read the diff with intent.

**Ask yourself:**
- What is this PR trying to accomplish?
- Does the approach make sense for the goal?
- Would I understand this code in 6 months without the PR context?

**Tone principles:**
- Phrase feedback as suggestions, not commands: "Consider..." / "Have you thought about..." / "This might be clearer if..."
- Distinguish blocking issues from nits — label them explicitly: `[Blocker]`, `[Suggestion]`, `[Nit]`, `[Question]`
- Acknowledge good work — if something is well done, say so. Reviews shouldn't be all negative.
- Assume the author had reasons — ask "why" before assuming something is wrong.

## Review Checklist

Work through these areas in order. Not every area applies to every PR — skip sections that aren't relevant.

### 1. Correctness

The most important dimension. Does the code do what it's supposed to do?

- **Logic errors** — off-by-one, wrong comparison operator, inverted conditions, missing negation
- **Edge cases** — empty inputs, null/undefined, boundary values, concurrent access, zero-length arrays, very large inputs
- **Error handling** — are errors caught? Are they handled meaningfully or silently swallowed? Are error messages helpful?
- **State management** — is mutable state modified safely? Are there race conditions? Is shared state protected?
- **Data flow** — do inputs flow correctly through transformations to outputs? Are types consistent?
- **Return values** — does every code path return the expected type? Are early returns handled correctly?

**Example finding:**
```
[Blocker] The `findUser()` function returns `null` when the user isn't found,
but the caller on line 42 doesn't handle the null case — this will crash
with `TypeError: Cannot read property 'email' of null` when looking up
a non-existent user.

Suggestion:
+ const user = findUser(id);
+ if (!user) {
+   throw new NotFoundError(`User ${id} not found`);
+ }
```

### 2. Naming and Readability

Code is read far more often than it's written.

- **Variable/function names** — do they describe what the thing IS or DOES? `data`, `temp`, `result`, `handle` are red flags unless the scope is tiny.
- **Function length** — functions over ~30 lines often do too much. Can they be broken into named steps?
- **Nesting depth** — more than 3 levels of nesting hurts readability. Suggest early returns, guard clauses, or extraction into helper functions.
- **Comments** — are they explaining "why" (good) or "what" (usually redundant)? Is there complex logic that lacks a comment?
- **Magic numbers/strings** — are literal values used inline that should be named constants?
- **Consistency** — does the new code follow the conventions of the surrounding codebase?

### 3. Architecture and Design

Zoom out from individual lines to the overall approach.

- **Right level of abstraction** — is the code too clever or too verbose? Does it introduce unnecessary indirection?
- **Single Responsibility** — does each function/class/module have one clear job?
- **Coupling** — does this change create tight coupling between modules that should be independent?
- **Duplication** — is logic copy-pasted that should be extracted into a shared function? (But note: premature DRY is also a problem — duplication is fine if the use cases might diverge.)
- **API design** — if the PR introduces or modifies an API: are the names intuitive? Is the contract clear? Is it backwards-compatible?
- **Over-engineering** — is this solving tomorrow's problem instead of today's? YAGNI applies.

### 4. Testing

- **Are there tests?** — if not, should there be? New logic and bug fixes especially benefit from tests.
- **Test quality** — do tests verify behavior (good) or implementation details (brittle)?
- **Test coverage** — are the happy path AND error/edge cases covered?
- **Test naming** — can you understand what's being tested from the test name alone?
- **Test isolation** — do tests depend on each other, shared state, or external services they shouldn't?

### 5. Performance

Only flag performance issues when they're likely to matter in practice. Premature optimization is the root of all evil.

- **Algorithmic complexity** — O(n²) in a hot path, nested loops over large datasets, repeated linear scans that should be lookups
- **Database queries** — N+1 queries, missing indexes for new query patterns, unbounded `SELECT *`
- **Memory** — loading entire files/datasets into memory when streaming would work, creating large objects in loops
- **Network** — unnecessary API calls, missing caching for expensive operations, synchronous calls that could be parallel

### 6. Security (Lightweight)

Basic security checks — the backend-review skill covers this in depth.

- **User input** — is external input validated/sanitized before use?
- **Secrets** — are API keys, passwords, or tokens hardcoded or logged?
- **SQL/NoSQL injection** — are queries parameterized or using raw string concatenation?
- **Access control** — does the change respect authorization boundaries?

### 7. Operational Concerns

- **Logging** — are important operations logged? Is sensitive data excluded from logs?
- **Configuration** — are new settings configurable or hardcoded? Are defaults reasonable?
- **Migrations** — are database migrations reversible? Do they require downtime?
- **Dependencies** — are new dependencies justified? Are they maintained and trustworthy?
- **Backwards compatibility** — does this break existing callers, API consumers, or stored data?

## Structuring Review Feedback

Organize feedback by severity so the author knows what to address first.

### Output format

```markdown
## PR Review: [Brief title or PR reference]

### Overview
<!-- 2-3 sentences: Your understanding of what this PR does and your overall impression. -->

### Blockers
<!-- Issues that must be fixed before merging. These are correctness bugs, security holes, or data loss risks. -->

1. **[File:Line]** — Description of the issue, why it matters, and suggested fix.

### Suggestions
<!-- Improvements that would make the code better but aren't strictly required. -->

1. **[File:Line]** — Description and reasoning.

### Nits
<!-- Style or preference items. Low priority. -->

1. **[File:Line]** — Description.

### What's Good
<!-- Call out things done well. This matters for morale and reinforcement. -->

- Specific praise for good patterns, thorough tests, clean abstractions, etc.
```

### Severity classification guide

| Severity | Criteria | Merge without fixing? |
|----------|----------|----------------------|
| **Blocker** | Correctness bug, security vulnerability, data loss risk, breaks existing functionality | No |
| **Suggestion** | Readability improvement, better pattern, missing test, performance concern | Yes, but should be addressed |
| **Nit** | Style preference, minor naming, formatting not caught by linter | Yes |

## Adapting to PR Size

**Small PRs (1–5 files):** Give detailed line-level feedback. These are easy to review well.

**Medium PRs (5–20 files):** Focus on architecture + correctness. Call out the most important 3–5 findings rather than commenting on everything.

**Large PRs (20+ files):** Start with an architectural review. Flag if the PR should be split. Focus blockers on the core logic — don't try to review every file at equal depth. Suggest which areas need the closest attention.

## Common Anti-Patterns to Flag

These recur across codebases and are worth calling out when spotted:

- **Catching and ignoring errors** — `catch (e) {}` or `catch (e) { console.log(e) }` with no recovery
- **Boolean parameters** — `createUser(name, true, false)` is unreadable; suggest an options object
- **Stringly-typed code** — using strings where enums or constants belong
- **Premature abstraction** — creating interfaces/base classes for a single implementation
- **God functions/classes** — one function/class doing everything
- **Commented-out code** — if it's not needed, delete it; that's what version control is for
- **TODO without context** — `// TODO: fix this` with no ticket reference or explanation of what needs fixing