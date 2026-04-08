---
description: Batch bugfix workflow — fetch Jira tickets, fix each one, push branch and create PR to dev
argument-hint: <TICKET-1> [notes...], <TICKET-2> [notes...], ...
---

# Bugfix Batch

Fix one or more Jira bugs in a single branch with a PR to dev.

## Inputs

Arguments are comma-separated entries. Each entry is a Jira ticket ID optionally followed by free-text notes:

```
/bugfix APP-1234 кнопка не работает на мобилке, APP-5678, APP-9999 проверь только тёмную тему
```

- Ticket ID is required (e.g. `APP-1234`)
- Notes after the ID are optional context/hints from the user
- If no arguments — ask the user for ticket IDs

## Workflow

### 0. Setup

- Ensure you are on the `dev` branch and it is up to date: `git checkout dev && git pull`
- Parse arguments into a list of `{ ticketId, notes }` entries
- Create a working branch: `git checkout -b fix/<short-summary>`
  - Name the branch after the first ticket or a common theme if multiple tickets share one

### 1. Process each ticket sequentially

For each ticket:

#### 1a. Fetch ticket from Jira

- Use `mcp__atlassian__getJiraIssue` with `cloudId` from `mcp__atlassian__getAccessibleAtlassianResources`
- Extract: summary, description, attachments, comments
- Print a short summary of the ticket to the user
- **Assign ticket and update status** (run in parallel):
  - Use `mcp__atlassian__editJiraIssue` to set assignee to Maks Zhers (accountId: `712020:65332aee-84f4-4fe1-9775-0a8886e43490`)
  - Use `mcp__atlassian__transitionJiraIssue` with transition id `571` ("Development In Progress")

#### 1b. Investigate the codebase

- Use the ticket description, user notes, and attachments to understand the bug
- Search the codebase for relevant files (Grep, Glob, Read)
- Understand the root cause before writing any code

#### 1c. Ask questions if needed

- If the bug is ambiguous or has multiple possible fixes — ask the user ONE question at a time
- If the user provided notes that clarify the fix — proceed without asking
- Never guess when the fix could break other functionality

#### 1d. Implement the fix

- Follow all rules from CLAUDE.md (FSD layers, naming conventions, code reuse protocol)
- Before creating any new utility/component/type — search if it already exists
- Keep changes minimal — only fix what the ticket requires
- Run `npm run typecheck` after each ticket's fix to catch errors early

#### 1e. Confirm with user

- Show what was changed and why
- If the user wants adjustments — apply them before moving to the next ticket

### 2. Finalize

After ALL tickets are processed:

#### 2a. Final verification

```bash
npm run typecheck   # Must pass
npm run lint        # Must pass
```

Fix any errors before proceeding.

#### 2b. Commit

- Stage only the changed files (no `git add .`)
- Write a single commit message covering all fixes:

```
fix: <concise summary of all fixes>

- <TICKET-1>: <what was fixed>
- <TICKET-2>: <what was fixed>
...

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
```

#### 2c. Push and create PR

```bash
git push -u origin <branch-name>
```

Create PR to `dev` using `gh pr create`:

```
Title: fix: <summary>
Base: dev

Body:
## Summary
- **<TICKET-1>**: <what was fixed and why>
- **<TICKET-2>**: <what was fixed and why>

## Test plan
- [ ] <verification step per ticket>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

#### 2d. Return to dev

```bash
git checkout dev
```

Print the PR URL.

## Rules

- **One branch, one PR** — all tickets go into a single branch and PR
- **Sequential processing** — fix tickets one by one, verify each before moving on
- **Read before edit** — never modify a file without reading it first
- **Minimal changes** — only fix what the ticket asks for, no refactoring or scope creep
- **Ask when unsure** — ambiguous tickets get clarified with the user, not guessed
- **Typecheck after each fix** — catch regressions early, not at the end
- **Never force push** — always regular push
- **Check shared code** — before creating utilities, check `shared/lib/`, `shared/ui/`, entity layers
