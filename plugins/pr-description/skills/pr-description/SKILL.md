---
name: pr-description
description: "Use this skill whenever the user wants to create, draft, or improve a GitHub Pull Request description. Triggers include: any mention of 'PR description', 'pull request description', 'PR summary', 'summarize changes', 'summarize the diff', 'write a PR', 'create a PR', 'PR template', or when the user asks to describe, explain, or document code changes for a pull request. Also trigger when the user provides a diff, a list of changed files, commit messages, or a branch name and wants a summary suitable for a PR. Use this skill even if the user says something casual like 'help me with my PR', 'what should I put in the PR', or 'describe these changes'. Do NOT use for git operations themselves (rebasing, merging, cherry-picking) or for commit message writing — only for the PR description body."
---

# GitHub PR Description Generator

## Purpose

Generate clear, well-structured pull request descriptions that help reviewers understand **what** changed, **why** it changed, and **how** to verify it. A good PR description reduces review time, prevents misunderstandings, and serves as future documentation.

## Gathering Context

Before writing, collect as much context as possible. The more context you have, the better the description. Use what's available — you rarely have everything.

### Sources of context (in order of richness)

1. **The diff itself** — the ground truth of what changed. If the user provides a diff or you can access one, read it carefully. Look at file names, function signatures, config changes, test additions, and deleted code.
2. **Commit messages** — often contain the "why" behind individual changes.
3. **Jira / Linear / ticket references** — the user may mention a ticket ID. Include it as a link if provided.
4. **User's verbal explanation** — the user often knows the "why" better than the code shows. Ask if unclear.
5. **Changed file list** — even without a full diff, file paths reveal scope (e.g., `src/billing/` vs `tests/` vs `migrations/`).

### What to ask if context is thin

If the user just says "write a PR description" without providing changes, ask:

- What files or areas of the codebase were changed?
- What was the motivation — a bug fix, feature, refactor, or chore?
- Is there a ticket or issue number?
- Any special reviewer instructions (e.g., "focus on the migration logic")?

Don't block on getting all answers — work with what you have and fill in `[TODO]` placeholders for anything missing.

## PR Description Structure

Use the following template as the default structure. Adapt section depth to PR size — a 5-line config change doesn't need the same scaffolding as a 40-file feature.

```markdown
## Summary

<!-- 2-4 sentences: What does this PR do and why? Lead with the user-facing or system-level impact. -->

## Changes

<!-- Grouped list of what was changed. Organize by logical area, not by file. Use sub-bullets for detail when helpful. -->

- **Area or component** — brief description of what changed and why
  - specific detail if needed
- **Area or component** — brief description
- ...

## How to Test

<!-- Step-by-step instructions a reviewer can follow to verify the change works. Include setup steps, test commands, or manual verification flows. -->

1. ...
2. ...

## Notes for Reviewers

<!-- Optional. Anything that helps the reviewer: areas to focus on, known limitations, follow-up items, screenshots, links to related PRs. Remove this section if not needed. -->
```

### Section guidelines

**Summary**
- Lead with the impact: what problem is solved or what capability is added.
- Mention the ticket/issue if one exists: `Resolves MBRS-1224` or `Related to #412`.
- Keep it to 2–4 sentences. If you need more, the PR might be too large.

**Changes**
- Group by logical concern, not by file path. For example, "Billing calculation logic" is better than "src/billing/calculator.ts".
- Each bullet should answer: what changed AND why (even briefly). "Added `retryCount` field to webhook payload — needed for idempotency tracking" is better than "Added `retryCount` field".
- For large PRs, use sub-groups with bold headers.
- Call out deletions and deprecations explicitly — reviewers often miss removed code.

**How to Test**
- Be specific. "Run the tests" is not helpful. "Run `npm test -- --grep 'reconciliation'` and verify the new edge case passes" is.
- For UI changes, describe the manual flow: navigate to X, click Y, expect Z.
- For API changes, include example curl commands or reference a Postman collection.
- If no automated tests were added, explain why or flag it.

**Notes for Reviewers**
- Use this for: migration instructions, deployment order dependencies, screenshots/recordings, links to design docs or ADRs, known trade-offs, follow-up tickets.
- Remove the section entirely if there's nothing to add — empty sections are noise.

## Adapting to PR Size

### Small PRs (1–5 files, single concern)
Keep it lean. Summary + a short Changes list is often enough. How to Test can be a single line. Skip Notes for Reviewers unless there's something non-obvious.

```markdown
## Summary
Fix off-by-one error in pagination that caused the last page to show duplicate items. Resolves #318.

## Changes
- **Pagination logic** — adjusted offset calculation in `getPaginatedResults` to use `(page - 1) * pageSize` instead of `page * pageSize`
- **Tests** — added edge case for last-page boundary

## How to Test
Run `npm test -- --grep 'pagination'` — the new test case should pass.
```

### Medium PRs (5–20 files, one feature or refactor)
Use the full template. Group changes into 3–6 logical bullets. Include How to Test with 2–4 steps.

### Large PRs (20+ files, multi-concern)
Consider whether the PR should be split. If it can't be, use the full template with sub-grouped Changes, a thorough How to Test, and use Notes for Reviewers to guide attention: "Start with `src/billing/` — that's the core logic. The rest is mechanical renaming."

## Tone and Style

- **Be direct.** Reviewers are busy. Don't pad with filler like "This PR aims to enhance the overall experience by..."
- **Use present tense.** "Adds retry logic" not "Added retry logic" — the description describes the PR's current state.
- **Technical but readable.** Assume the reviewer knows the codebase but not your head. Avoid acronyms without context unless they're team-standard (e.g., if the team always says "TPF" for Third-Party Funding, that's fine).
- **No commit-log dumps.** A PR description is not a list of commit messages. Synthesize them into a coherent narrative.

## Handling Special Cases

### Breaking changes
Add a prominent callout at the top of the Summary:

```markdown
## Summary
⚠️ **Breaking Change**: The `/api/v1/invoices` response shape has changed — `lineItems` is now nested under `billing`.

...rest of summary...
```

### Database migrations
Always mention migrations in Notes for Reviewers with:
- Whether the migration is reversible
- Whether it requires downtime
- Run order if there are multiple migrations

### Dependency updates
List what was updated and why. If it's a security patch, say so. If it's a major version bump, note any breaking changes from the dependency's changelog.

### Draft / WIP PRs
If the user is creating a draft PR, adjust the template:
- Summary should state what's done and what's still in progress
- Add a "Remaining Work" section or checklist with `- [ ]` items

```markdown
## Summary
[WIP] Adds webhook retry mechanism for failed PeopleSoft callbacks. Core retry logic is implemented; exponential backoff and dead-letter queue are pending.

## Remaining Work
- [ ] Implement exponential backoff with jitter
- [ ] Add dead-letter queue for permanently failed webhooks
- [ ] Write integration tests
```

## Output Format

- Output the PR description as a Markdown code block so the user can copy-paste it directly into GitHub.
- If the user's context is incomplete, use `[TODO: ...]` placeholders and tell the user what's missing.
- If the user provides a diff or file list, do NOT just echo it back — synthesize it into the structured format above.