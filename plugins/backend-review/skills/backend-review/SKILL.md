---
name: backend-review
description: "Reviews backend code for security issues and compliance mishaps. Use when reviewing server-side code, APIs, database queries, authentication/authorization logic, middleware, or any code that handles sensitive data, user input, or system operations. Triggers include: any mention of 'security review', 'check for vulnerabilities', 'review my API', 'is this secure', 'review backend code', 'compliance check', 'audit this code', 'check for injection', 'review auth logic', 'review middleware', 'check my endpoint', or when the user shares backend code (Node.js, Python, Java, Go, C#, etc.) and asks for a security-focused review. Also trigger for: 'OWASP check', 'pen test prep', 'data protection review', 'PII handling', 'review my database queries', 'check my webhook', 'review this Lambda', or 'is this API safe'. This skill complements the general pr-review skill by focusing specifically on security vulnerabilities, data protection, and compliance concerns. Do NOT use for frontend accessibility/UX reviews (use frontend-review), general code quality without security context (use pr-review), or debugging runtime errors (use debugger)."
---

# Backend Code Review: Security & Compliance

## Purpose

Review backend code to catch security vulnerabilities and compliance issues before they reach production. Security bugs are uniquely expensive — a single missed injection vulnerability or exposed secret can result in data breaches, regulatory fines, and loss of user trust. This review is the last line of defense before deployment.

## Review Approach

Think like an attacker, review like a defender:

1. **Identify the attack surface** — what inputs does this code accept? What systems does it talk to? What data does it handle?
2. **Trace data flow** — follow untrusted input from entry to storage/output. Every transformation and decision point along the way is a potential vulnerability.
3. **Check the boundaries** — authentication, authorization, validation, and error handling at every system boundary.

## Security Review Checklist

### 1. Injection

The most consistently dangerous vulnerability class. Anywhere user input is concatenated into a command or query is a potential injection point.

**SQL Injection**
```
[Blocker] Raw string interpolation in SQL query — this is injectable.

  const query = `SELECT * FROM users WHERE email = '${req.body.email}'`;

An attacker can send: ' OR '1'='1' -- to dump the entire users table,
or worse, use UNION-based injection to read other tables.

Fix — use parameterized queries:
  const query = 'SELECT * FROM users WHERE email = $1';
  const result = await db.query(query, [req.body.email]);

For ORMs (Prisma, Sequelize, TypeORM): use the built-in query builders,
never pass raw interpolated strings to .raw() or .$queryRaw().
```

**NoSQL Injection** — MongoDB queries with user input can be injected via object payloads:
```javascript
// Vulnerable: req.body.password could be { "$gt": "" }
db.users.find({ email: req.body.email, password: req.body.password });

// Fix: explicitly cast to string or validate type
db.users.find({ email: String(req.body.email), password: String(req.body.password) });
```

**Command Injection** — any use of `exec()`, `spawn()`, `system()`, or shell commands with user input:
```
[Blocker] User input passed directly to shell command.

  exec(`convert ${req.body.filename} output.png`);

An attacker can send: "; rm -rf /" as the filename.

Fix: Use execFile() with arguments array (no shell), or validate
the filename against a strict allowlist pattern.
```

**Other injection vectors to check:** LDAP queries, XML parsers (XXE), template engines (SSTI), log statements (log injection/forging), HTTP headers (header injection), regex with user input (ReDoS).

### 2. Authentication

- **Password storage** — passwords must be hashed with bcrypt, scrypt, or Argon2. Flag: MD5, SHA-1, SHA-256 without salt, plain text storage, reversible encryption.

- **Session management** — session tokens should be: cryptographically random, sufficiently long (128+ bits), stored server-side or in signed/encrypted cookies, invalidated on logout.

- **JWT handling** — check for:
  - Algorithm confusion: is the `alg` header validated? Accepting `none` is a critical vulnerability.
  - Secret strength: is the signing key a hardcoded weak string?
  - Expiry: do tokens have reasonable `exp` claims?
  - Sensitive data in payload: JWTs are base64-encoded, NOT encrypted. Don't store PII in them.

- **Rate limiting** — login endpoints need rate limiting or account lockout to prevent brute force. Check for: per-IP limits, per-account limits, CAPTCHA after N failures.

- **Multi-factor authentication** — if the system supports MFA, verify the implementation: TOTP codes should be time-limited, backup codes should be single-use, MFA bypass should require additional verification.

### 3. Authorization

The most commonly missed security concern. Authentication answers "who are you?" — authorization answers "are you allowed to do this?"

- **Broken object-level authorization (BOLA/IDOR)** — the #1 API vulnerability. Check that every endpoint verifies the requesting user has access to the specific resource:
  ```
  [Blocker] This endpoint returns any user's data based on the URL parameter
  without checking if the authenticated user is allowed to access it.

  // Vulnerable
  app.get('/api/users/:id', async (req, res) => {
    const user = await User.findById(req.params.id);
    res.json(user);
  });

  // Fix: verify ownership or role
  app.get('/api/users/:id', async (req, res) => {
    const user = await User.findById(req.params.id);
    if (user.id !== req.auth.userId && !req.auth.isAdmin) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    res.json(user);
  });
  ```

- **Privilege escalation** — can a regular user access admin endpoints? Check middleware ordering and route-level authorization.

- **Mass assignment** — does the code blindly spread request body into database models? An attacker can set `role: 'admin'` or `isVerified: true`:
  ```
  // Vulnerable
  await User.update(req.body);  // attacker adds { role: 'admin' }

  // Fix: explicitly pick allowed fields
  const { name, email } = req.body;
  await User.update({ name, email });
  ```

- **Horizontal vs vertical** — horizontal: accessing another user's data at the same privilege level. Vertical: escalating to a higher privilege level. Check for both.

### 4. Input Validation

Never trust client-side validation — it can be bypassed entirely. All validation must be repeated server-side.

- **Type checking** — is the input the expected type? An expected string could arrive as an array or object.

- **Length limits** — are strings bounded? Unbounded input can cause memory exhaustion or storage issues.

- **Format validation** — emails, URLs, dates, IDs should be validated against expected patterns. Use established libraries (validator.js, Joi, Zod, Pydantic) over hand-rolled regex.

- **File uploads** — check for: file type validation (don't trust Content-Type header — check magic bytes), file size limits, filename sanitization (path traversal via `../../etc/passwd`), storage location (never store in a web-accessible directory with execute permissions).

- **Numeric bounds** — are numbers validated for reasonable ranges? Negative quantities, zero-division, integer overflow.

### 5. Data Protection

- **PII exposure** — is personally identifiable information logged, returned in error messages, or stored unnecessarily? Check: names, emails, phone numbers, addresses, SSNs, payment details in logs, API responses, and error outputs.
  ```
  [Blocker] User's full credit card number is logged on line 84.

  console.log(`Processing payment for card: ${cardNumber}`);

  Fix: Log only last 4 digits or a tokenized reference.
  console.log(`Processing payment for card ending in ${cardNumber.slice(-4)}`);
  ```

- **Sensitive data in URLs** — query parameters are logged in server access logs, browser history, and referrer headers. Tokens, passwords, and PII should never be in URLs.

- **Encryption at rest** — are sensitive fields encrypted in the database? Check for: payment data, health records, authentication credentials, API keys.

- **Encryption in transit** — are internal service calls using TLS? Are webhook payloads signed (HMAC)?

- **Data retention** — does the code store data longer than needed? Are there cleanup mechanisms?

### 6. Secrets Management

- **Hardcoded secrets** — API keys, database passwords, JWT secrets, encryption keys must NOT be in source code. Check: source files, config files committed to the repo, Docker files, CI/CD configs.
  ```
  [Blocker] Database password hardcoded in source file.

  const db = new Client({ password: 'pr0d_p@ssw0rd!' });

  Fix: Use environment variables or a secrets manager.
  const db = new Client({ password: process.env.DB_PASSWORD });
  ```

- **`.env` files** — should be in `.gitignore`. If committed, flag immediately — the secrets are now in git history even if the file is later removed.

- **Logging secrets** — check that `console.log`, `logger.info`, or debug output doesn't dump request headers (which may contain auth tokens), environment variables, or connection strings.

### 7. API Security

- **Rate limiting** — are endpoints rate-limited? Especially: auth endpoints, search/query endpoints, resource-creation endpoints, webhooks.

- **CORS** — is the Access-Control-Allow-Origin overly permissive? `*` in production is almost always wrong. Check that credentials aren't sent with wildcard origins.

- **Request size limits** — is there a body size limit? Without one, an attacker can send arbitrarily large payloads to exhaust memory.

- **HTTP methods** — are endpoints restricted to the intended methods? A `GET` endpoint that accepts `DELETE` via method override is dangerous.

- **Webhook security** — inbound webhooks should verify signatures (HMAC-SHA256). Without verification, anyone can send fake webhook payloads.
  ```
  // Good: HMAC verification for incoming webhooks
  const signature = req.headers['x-webhook-signature'];
  const expected = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(JSON.stringify(req.body))
    .digest('hex');
  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    return res.status(401).json({ error: 'Invalid signature' });
  }
  ```

### 8. Error Handling and Information Disclosure

- **Stack traces in production** — error responses should never include stack traces, file paths, or internal details in production. These reveal the technology stack, file structure, and sometimes source code.

- **Verbose error messages** — "User not found" vs "Invalid credentials" — the first tells an attacker the username exists. Use generic messages for auth failures.

- **Error codes** — different error responses for different failure modes can leak information. Login should return the same error for "user not found" and "wrong password".

- **Debug endpoints** — check for exposed debug routes, health check endpoints that leak system info, or metrics endpoints without authentication.

### 9. Dependency and Infrastructure

- **Known vulnerabilities** — are dependencies up to date? Check for known CVEs in the dependency tree (`npm audit`, `pip audit`, `snyk`).

- **Dependency scope** — are dev dependencies accidentally included in production builds?

- **Docker security** — running as root? Using `latest` tag? Exposing unnecessary ports? Including source code or `.git` directory?

- **Environment separation** — can dev/staging data or config leak into production? Are test fixtures or seed data accessible?

## Structuring Review Feedback

```markdown
## Security Review: [Endpoint, service, or PR reference]

### Attack Surface
<!-- Brief summary: what inputs does this code handle, what systems does it connect to, what data does it process? -->

### Critical Vulnerabilities
<!-- Issues that could be exploited to breach data, bypass auth, or take control of systems. Must be fixed before merge. -->

1. **[File:Line] [OWASP Category]** — Description, exploitation scenario, fix.

### Security Concerns
<!-- Issues that weaken the security posture but aren't immediately exploitable, or require specific conditions. -->

1. **[File:Line]** — Description, risk level, recommendation.

### Compliance Notes
<!-- Data handling, PII, logging, retention, or regulatory concerns. -->

1. **[File:Line]** — What's affected and what needs to change.

### Good Practices Observed
<!-- Acknowledge security-conscious patterns to reinforce them. -->
```

### Severity guide

| Severity | Criteria | Examples |
|----------|----------|---------|
| **Critical** | Directly exploitable, leads to data breach or system compromise | SQL injection, hardcoded secrets, broken auth, IDOR |
| **High** | Exploitable under certain conditions or weakens security significantly | Missing rate limiting, overly permissive CORS, PII in logs |
| **Medium** | Defense-in-depth concern, not immediately exploitable | Missing input validation, verbose errors, weak password policy |
| **Low** | Best practice deviation, minimal immediate risk | Missing security headers, dependency not at latest version |

## Quick Reference: OWASP Top 10 Mapping

When flagging issues, reference the relevant OWASP category so the team can research further:

| # | Category | What to look for |
|---|----------|-----------------|
| A01 | Broken Access Control | IDOR, missing auth checks, privilege escalation, CORS misconfiguration |
| A02 | Cryptographic Failures | Weak hashing, missing encryption, secrets in code, PII exposure |
| A03 | Injection | SQL, NoSQL, command, LDAP, template injection |
| A04 | Insecure Design | Missing rate limits, no fraud controls, trust boundary violations |
| A05 | Security Misconfiguration | Debug mode in prod, default credentials, verbose errors, missing headers |
| A06 | Vulnerable Components | Outdated dependencies with known CVEs |
| A07 | Auth Failures | Weak passwords, missing MFA, credential stuffing, session fixation |
| A08 | Data Integrity Failures | Unsigned updates, untrusted deserialization, missing CI/CD controls |
| A09 | Logging Failures | Missing audit logs, sensitive data in logs, no alerting |
| A10 | SSRF | User-controlled URLs in server-side requests without validation |