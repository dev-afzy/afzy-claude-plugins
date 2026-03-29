---
name: debugging
description: "Use this skill whenever the user needs help debugging, troubleshooting, or fixing a bug. Triggers include: any mention of 'debug', 'error', 'bug', 'fix', 'broken', 'not working', 'failing', 'crash', 'exception', 'stack trace', 'traceback', 'unexpected behavior', 'issue', or when the user pastes an error message, log output, or stack trace. Also trigger when the user says things like 'something is wrong with...', 'this used to work but now...', 'I'm getting a weird...', 'help me figure out why...', 'it works locally but not in...', 'intermittent failure', or 'flaky test'. Use this skill even for vague descriptions like 'my app is slow' or 'the page is blank' — these are debugging problems. Do NOT use for writing new features from scratch, code reviews without a specific issue, or general 'how does X work' questions unless they stem from a debugging context."
---

# Debugging & Troubleshooting

## Philosophy

Debugging is detective work. The goal is not to guess a fix — it's to **understand the cause** first and then fix it with confidence. Resist the urge to jump to solutions. A wrong fix applied quickly costs more time than a slow, correct diagnosis.

The debugging process follows a loop:

```
Observe → Hypothesize → Test → Narrow → Fix → Verify
```

Every step below serves this loop.

## Step 1: Gather Evidence

Before forming any hypothesis, collect everything available. The quality of the diagnosis depends entirely on the quality of the evidence.

### What to ask for (if not already provided)

Ask for the **most useful missing piece** first — don't overwhelm with a checklist. Prioritize based on what's likely to crack the case fastest.

| Priority | Evidence | Why it matters |
|----------|----------|----------------|
| 🔴 High | **The exact error message or stack trace** | Often points directly to the failing line and error type |
| 🔴 High | **Steps to reproduce** | If you can't reproduce it, you're guessing |
| 🟡 Medium | **What changed recently** | Most bugs are caused by recent changes — deploys, config updates, dependency bumps, data changes |
| 🟡 Medium | **Expected vs actual behavior** | Clarifies whether it's a crash, wrong output, or performance issue |
| 🟡 Medium | **Environment details** | OS, runtime version, local vs staging vs prod, browser if frontend |
| 🟢 Low | **Relevant logs** | Broader context around the error — look for warnings before the crash |
| 🟢 Low | **Has it ever worked?** | Distinguishes regressions from never-worked bugs |

### Reading error messages

Error messages are the single most valuable debugging artifact. Read them carefully — most developers skim them and miss critical details.

**Anatomy of a useful error message:**
- **Error type/class** — `TypeError`, `ConnectionRefusedError`, `OOMKilled` — this categorizes the problem
- **Message text** — the human-readable description, often contains the "what"
- **Location** — file path + line number, the "where"
- **Stack trace** — the call chain that led to the error, read **bottom-up** for the origin, **top-down** for the trigger

**Common patterns to recognize immediately:**

| Pattern | Likely cause |
|---------|-------------|
| `undefined is not a function` / `Cannot read property X of undefined` | Null reference — something expected to exist doesn't |
| `Connection refused` / `ECONNREFUSED` | Target service is down or wrong host/port |
| `Timeout` / `ETIMEDOUT` | Network issue, slow query, or deadlock |
| `Permission denied` / `403` / `EACCES` | Auth/IAM/file permission misconfiguration |
| `Out of memory` / `OOMKilled` | Memory leak, oversized payload, or insufficient allocation |
| `Syntax error` / `Unexpected token` | Malformed code, JSON, YAML, or config file |
| `Module not found` / `ImportError` | Missing dependency, wrong path, or version mismatch |
| `Constraint violation` / `duplicate key` | Database schema conflict or race condition |
| `CORS` / `Access-Control-Allow-Origin` | Frontend calling a backend that doesn't allow the origin |

## Step 2: Form Hypotheses

Once you have evidence, generate **ranked hypotheses** — most likely cause first. Explain your reasoning so the user can follow along and correct you if your assumptions are wrong.

### Reasoning framework

Think through these dimensions for each hypothesis:

1. **Does the evidence support it?** — A hypothesis must explain ALL the observed symptoms, not just some.
2. **Is it consistent with what changed?** — Recent changes are the most likely culprit. If nothing changed, look at environmental factors (data, load, time-based triggers).
3. **Occam's Razor** — prefer the simplest explanation. A typo in a config file is more likely than a framework bug.
4. **Frequency** — common mistakes first. Off-by-one errors, null references, missing environment variables, and wrong file paths cause the vast majority of bugs.

### Present hypotheses clearly

Format hypotheses so the user can quickly evaluate them:

```
Based on the error and context, here are the most likely causes:

1. **[Most likely] Missing environment variable `DATABASE_URL`**
   - The error `ConnectionRefusedError: connect ECONNREFUSED 127.0.0.1:5432` suggests the app is falling back to localhost because the DB connection string isn't set.
   - Evidence: This started after the recent deploy, and the .env file isn't committed to the repo.
   - To verify: Run `echo $DATABASE_URL` in the container.

2. **[Possible] Database service is down**
   - Same error pattern, but less likely because other services using the same DB are working.
   - To verify: Try connecting directly with `psql -h <host> -U <user>`.

3. **[Unlikely] Firewall rule change**
   - Could explain connection refused, but nothing in the deploy notes suggests infra changes.
   - To verify: Check security group / firewall rules for port 5432.
```

This structure lets the user skip straight to verification without re-reading the analysis.

## Step 3: Isolate and Verify

Help the user narrow down from hypothesis to confirmed cause. The goal is to **eliminate possibilities systematically**, not try random fixes.

### Isolation techniques

**Binary search / bisect** — for "it used to work" regressions:
- Use `git bisect` to find the exact commit that introduced the bug
- Or manually check: does the bug exist at the midpoint between "last known good" and "first known bad"?

**Minimal reproduction** — strip away everything that isn't necessary:
- Remove unrelated code/config until you have the smallest case that still reproduces the bug
- This is the single most powerful debugging technique. If you can reproduce it in 10 lines, you can fix it in minutes.

**Variable isolation** — change one thing at a time:
- If you suspect environment, test with hardcoded values
- If you suspect data, test with known-good data
- If you suspect timing, add deliberate delays or remove async behavior

**Divide by layer** — for full-stack issues:
- Is the problem in the frontend, API, service, or database?
- Test each layer independently: curl the API directly, query the DB directly, check the browser console

**Logging and instrumentation** — when the error doesn't give enough context:
- Add targeted console.log / print statements at decision points (not everywhere)
- Log **inputs and outputs** of the suspected function, not just "reached here"
- For async issues, log timestamps to detect ordering problems

## Step 4: Fix

Once the cause is confirmed, propose a fix. A good fix addresses the **root cause**, not just the symptom.

### Fix quality checklist

Before presenting a fix, verify:

- [ ] **Addresses root cause** — not just masking the symptom (e.g., don't just add a try/catch around a null reference — fix why it's null)
- [ ] **Doesn't break other things** — consider side effects, especially for shared utilities or database schema changes
- [ ] **Includes a test** — if the codebase has tests, suggest a test that would have caught this bug. Regressions hate tests.
- [ ] **Is minimal** — the fix should change the least amount of code necessary. Don't refactor while fixing a bug — that's a separate PR.

### Present the fix clearly

Show the fix as a before/after diff when possible:

```
The issue: `calculateTotal()` uses `items.length` but `items` can be `undefined`
when the cart is empty because the API returns `null` instead of `[]`.

Fix — add a default value:

- const total = items.length * pricePerItem;
+ const total = (items ?? []).length * pricePerItem;

This handles the null case without changing the API contract.
A better long-term fix would be to normalize the API response,
but this addresses the immediate crash.
```

### When a fix involves trade-offs

Be upfront about them:

- "This fixes the immediate crash but introduces a performance cost because..."
- "This is a quick fix. A proper solution would involve X, which I'd recommend as a follow-up."
- "There are two approaches: A is simpler but less robust, B is more work but handles edge cases. Here's both..."

## Step 5: Verify the Fix

Don't stop at writing the fix — help the user confirm it works.

- Suggest specific verification steps: "Run the failing test again", "Hit the endpoint with the same payload that caused the error", "Check the logs for the error pattern over the next hour"
- If the bug was intermittent, discuss how to gain confidence it's truly fixed (run it N times, monitor in staging, check metrics)
- If there's a related test suite, run it to check for regressions

## Debugging by Category

Different categories of bugs have different diagnostic playbooks. Jump to the relevant section based on the symptoms.

### Runtime Crashes / Exceptions
1. Read the stack trace bottom-up to find the origin
2. Check the failing line — is any variable potentially null/undefined?
3. Check the inputs to the crashing function — are they what you expect?
4. Look at recent changes to that file or its callers

### Wrong Output (no crash)
1. Identify where the output diverges from expected — add logging at key transformation points
2. Work backwards from the wrong output to find where correct data becomes incorrect
3. Check edge cases: empty inputs, boundary values, special characters, timezone differences

### Performance / Slowness
1. **Measure before guessing** — use profiling tools, not intuition
2. Common culprain: N+1 queries, missing indexes, unbounded loops, large payloads, memory leaks
3. Check if it's consistently slow or only under load / with certain data
4. For DB slowness: run `EXPLAIN ANALYZE` on the suspect query

### Intermittent / Flaky Failures
1. These are almost always caused by: race conditions, timing dependencies, shared mutable state, or external service instability
2. Look for: missing `await`, unguarded concurrent access, test pollution (shared state between test cases), clock/timezone sensitivity
3. Try to make them deterministic: add locks, increase timeouts (to confirm it's timing), or seed random values

### Environment-Specific ("works on my machine")
1. Compare environments systematically: runtime version, env vars, OS, dependencies, config files
2. Common culprits: different Node/Python/Java versions, missing env vars in CI/prod, case-sensitive file systems (Linux vs macOS), line ending differences (CRLF vs LF)
3. Use Docker or Nix to reproduce the other environment locally

### Build / Compilation Errors
1. Read the **first** error — subsequent errors are often cascading failures
2. Check for: version mismatches in dependencies, missing type definitions, circular imports, stale caches (`node_modules`, `.next`, `__pycache__`, `target/`)
3. Try a clean build: delete cache/build artifacts and rebuild

### Deployment / Infrastructure Failures
1. Check: are environment variables set? Are secrets mounted? Is the service reachable?
2. Compare the deploy diff — what changed between last working deploy and this one?
3. Check resource limits: CPU, memory, disk, connection pool exhaustion
4. Review health checks and startup logs — is the app even starting?

## Anti-Patterns to Avoid

These patterns waste time and create new problems:

- **Shotgun debugging** — changing multiple things at once hoping something works. You won't know what fixed it, and you might introduce new bugs.
- **Fix the symptom, ignore the cause** — wrapping everything in try/catch, adding null checks everywhere without understanding why things are null. This hides bugs instead of fixing them.
- **Debugging by rewriting** — "let me just rewrite this function" before understanding why it broke. The rewrite often has the same bug because the root cause wasn't understood.
- **Assuming the framework/library is broken** — it almost never is. Check your code, your config, and your data first.
- **Debugging without reproduction** — if you can't reproduce it, you can't verify your fix. Invest time in getting a repro first.

## Output Format

Structure your debugging response to match where the user is in the process:

- **User provides an error**: Start with Step 1–2. Read the error, present hypotheses, suggest verification steps.
- **User is stuck mid-debug**: Jump to Step 3. Help isolate. Ask what they've already tried so you don't repeat work.
- **User knows the cause, needs a fix**: Jump to Step 4–5. Propose a fix with before/after and verification.
- **User provides code + "it's not working"**: Start by asking what "not working" means — crash? wrong output? slow? Then proceed from Step 1.

Always be specific to their codebase, language, and framework. Generic advice like "check your code for bugs" is useless. Reference their actual file names, function names, and error messages.