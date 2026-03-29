---
name: git-stacked-prs
description: >
  Split uncommitted or working-branch changes into stacked git branches for easier PR review.
  Use this skill whenever the user wants to break up a large set of changes into multiple
  reviewable PRs, mentions "stacked branches", "stacked PRs", "split my changes into PRs",
  "create stacked git branches", or asks how to organise commits into separate pull requests
  for review. Also trigger when a user has a single branch with many changed files and wants
  to submit them as multiple focused PRs.
---

# Git Stacked PRs

Split a large set of uncommitted (or working-branch) changes into cleanly stacked git branches,
each representing a single reviewable PR.

---

## Workflow

### Step 1 — Save point

Commit **all** current working-tree changes to the current branch so nothing is lost:

```bash
git add -A
git commit -m "wip: save point before splitting into stacked PRs"
```

Record the commit SHA:

```bash
git rev-parse HEAD   # → <save-point-sha>
```

Also stash any untracked files that shouldn't be committed yet.

---

### Step 2 — Propose groupings (wait for confirmation)

Inspect the changed files:

```bash
git diff HEAD~1 --name-only        # files in the save-point commit
# or, if already on a feature branch:
git diff main...HEAD --name-only
```

Propose **3–5 logical PR groups**, for example:

| PR | Suggested scope | Example file patterns |
|----|----------------|----------------------|
| 1  | Schema / migrations | `db/`, `*.sql`, `migrations/` |
| 2  | Types / interfaces  | `types/`, `*.d.ts`, `interfaces/` |
| 3  | Core logic          | `lib/`, `src/core/` |
| 4  | Services / API      | `services/`, `api/`, `routes/` |
| 5  | Tests               | `*.test.*`, `*.spec.*`, `__tests__/` |

Adapt these to the actual file list. Present the grouping to the user and **wait for
confirmation before proceeding**.

---

### Step 3 — Handle local-only files

For any file that should not be in a PR (`.env`, local dev scripts, personal config):

```bash
echo "path/to/local-file" >> .gitignore
git add .gitignore
git commit --amend --no-edit
```

---

### Step 4 — Create stacked branches

Once the user confirms the groupings, create branches in order. Each branch builds on
the previous one:

```bash
git checkout master
git checkout -b pr/1-<slug>
git checkout <save-point-sha> -- path/to/file1 path/to/file2
git add -A
git commit -m "feat(<scope>): <description>"

git checkout -b pr/2-<slug>
git checkout <save-point-sha> -- path/to/file3 ...
git add -A
git commit -m "feat(<scope>): ..."
```

Naming convention: `pr/<number>-<short-slug>` (e.g. `pr/1-schema`, `pr/2-types`).
Commit messages follow **Conventional Commits**.

---

### Step 5 — Return to working branch

```bash
git checkout <original-working-branch>
git stash pop
```

---

### Step 6 — Report the final structure

```
Stacked branch structure
────────────────────────
master
  └── pr/1-schema       ← PR 1: Add database schema
        └── pr/2-types  ← PR 2: Add TypeScript interfaces
              └── pr/3-logic   ← PR 3: Implement core logic
                    └── pr/4-services ← PR 4: Wire up services
                          └── pr/5-tests ← PR 5: Add test coverage
```

Use `git log --oneline --graph --all` to show it visually.

---

## Rules

- **Never push** — the user will push and open PRs manually.
- Each PR branch must contain only the files relevant to its scope.
- Local-only files go into `.gitignore`, never into a PR.
- Every commit message must clearly describe that layer's single responsibility.
- If two files are tightly coupled and cannot be separated cleanly, keep them together and note this.
- If the user wants to rebase/update a lower branch, remind them to rebase all downstream branches (`git rebase --onto`).
