---
description: Audit a PR against project conventions and produce a refactor plan; optionally apply the plan sequentially or via parallel teammate agents
argument-hint: <pr-number> [<repo-owner/repo-name>]
---

# Adopt Code

Given a pull request, audit it against the project's conventions and refactor patterns. By default, produces a refactor plan — a structured list of points where the PR diverges from conventions, with proposed changes and reasoning. The plan is the deliverable: it goes back to the PR author as review comments, or to your future self when you sit down to apply it.

For smaller jobs, the skill can also execute the plan — either sequentially in this session, or by spawning parallel Claude Code teammates each handling one file.

## Inputs

- `$1` — PR number (required)
- `$2` (optional) — repo in `owner/name` format; defaults to the current repo
- Project root must contain `CLAUDE.md` and/or `AGENTS.md` declaring conventions
- Optionally: `docs/refactoring/knowledges/*.md` — refactor heuristic files (the maintainer's mental model for refactoring decisions)

If `$1` is missing, ask the user for the PR number.

## Workflow

### 0. Load context

- Read `CLAUDE.md` and `AGENTS.md` from the project root
- Read all `docs/refactoring/knowledges/*.md` files (if the directory exists)
- Read `tsconfig.json` / `package.json` (or stack equivalent) to understand the toolchain

If neither convention file exists, fall back to inferring style from the codebase. Tell the user this is happening and offer to abort.

### 1. Fetch the PR

```bash
gh pr view $1 --json files,title,body,baseRefName,headRefName
gh pr diff $1
```

- Print PR title and a 1-line summary
- Build the list of changed files with their patches

### 2. Sample the local style

For each changed file, read 2–3 sibling files in the same directory or feature module to understand the local style. This catches "the project allows X in general, but in this corner of the codebase the team does Y" cases.

### 3. Audit each changed file

For every file in the PR, run a layered audit:

1. **Naming** — variables, functions, files conform to conventions?
2. **Types** — `any` usage, missing return types, loose unions?
3. **Decomposition** — is the file at the right layer (FSD or equivalent)? Should it be split?
4. **Reuse** — is bespoke code reimplementing something in `shared/lib`, `shared/ui`, etc.?
5. **Architecture** — does the file follow patterns from refactor-knowledge files (state management, side effects, error handling)?

For each violation, record: file path, line, current state, proposed change, why (which convention or knowledge file backs the change).

### 4. Produce the plan

Print a structured refactor plan:

```
PR #1234 — refactor plan

Files audited: 7
Files conformant: 3 (no action)
Files with findings: 4
Total points: 14

────────────────────────────────────────
src/features/user-profile/UserProfile.tsx
────────────────────────────────────────
[Naming] L42: variable `data` should be `userProfile`
  Why: CLAUDE.md → naming → no generic identifiers in feature components

[Types] L67: parameter `e: any` should be `e: ChangeEvent<HTMLInputElement>`
  Why: AGENTS.md → typescript → forbid `any` in event handlers

[Reuse] L88–95: bespoke `formatDate` reimplements shared/lib/date/format.ts
  Why: standing rule — search shared/lib before writing utilities

────────────────────────────────────────
src/entities/user/api.ts
────────────────────────────────────────
[Architecture] L20: side-effect inside reducer
  Why: knowledges/state-side-effects.md → reducers must be pure
```

The plan is the primary output. Save it to a temp file (`/tmp/adopt-code-plan-<pr-number>.md`) so the user can read it later or paste into PR review comments.

### 5. Ask the user what's next

Offer three modes:

- **Plan only (default)** — exit here. The plan is the deliverable. The user takes it and either pastes points as PR review comments, edits manually, or returns later.
- **Apply sequentially** — apply all points in this session, file by file, layer by layer, with typecheck between layers.
- **Apply via parallel teammates** — for small PRs only. Spawn one Claude Code teammate per file via the Task tool (or worktree-isolated Agent calls), each handling its own file's points. Good for low-conflict refactors where files are independent.

Eligibility for parallel mode (all must hold):

- ≤ 5 files changed
- No point in the plan touches more than one file
- No `Decomposition` points (those create new files; risky to parallelize)
- No `Architecture` points that span multiple files

If the user requests parallel mode but the conditions aren't met, fall back to sequential and explain why.

### 6. Apply — sequential mode

Group points by file. For each file:

- Read the file fresh
- Apply changes layer by layer, in the order naming → types → decomposition → reuse → architecture
- Run typecheck after each layer (`npm run typecheck` or stack equivalent)
- If typecheck fails, fix and retry; if you can't fix it in 2 attempts, revert that layer for this file and surface to the user

Never apply two layers in one pass. Each layer must be independently revertible.

### 7. Apply — parallel teammates mode

Dispatch one agent per file via the Task tool (or Agent with `isolation: "worktree"` if cross-file isolation is needed). Each agent's brief:

- The file path it owns
- The list of points for that file (from the plan)
- The conventions context: `CLAUDE.md`, `AGENTS.md`, refactor-knowledge files
- Hard rule: only edit this one file. No new files. No edits to other files.
- Output: diff for its file + typecheck-pass confirmation

After all agents report back:

- Collect the diffs
- Apply them to the working tree (sequentially is fine; they don't conflict)
- Run a final whole-project typecheck
- If anything fails, surface to the user with the failing file

### 8. Verify

- Run the project linter if configured
- Run unit tests for affected files (if any exist)
- Show the full diff (`git diff`)

### 9. Recap

Print:

- Total points addressed vs deferred
- Files changed
- "Diff is staged, review and commit in your normal flow"

Do NOT commit automatically. The user owns the commit.

## Rules

- **Plan is the default.** The skill exits at step 5 unless the user explicitly asks to apply.
- **Never change behavior, only structure.** If a refactor would change what the code does, surface it as a separate finding, don't apply it silently.
- **Read before edit.** Always read the file fresh before each layer's edits.
- **One layer at a time** in sequential mode. Don't combine layers.
- **No new abstractions** unless the refactor-knowledge files explicitly call for one.
- **Skip if conformant.** If a file already matches conventions, list it as PASS in the plan and don't audit deeper.
- **Parallel mode has hard preconditions.** Don't bend them. If conditions aren't met, fall back to sequential and say why.
- **Surface inferred conventions.** If you fall back to inferring style (no `CLAUDE.md` / `AGENTS.md`), name what you inferred so the user can correct.
- **Plan persists.** Always save the plan to `/tmp/adopt-code-plan-<pr-number>.md` even when applying — so the user has a record of what was changed and why.
