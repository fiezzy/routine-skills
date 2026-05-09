---
description: Read PR review comments and fix code / reply to reviewers
argument-hint: <pr-number> [repo-owner/repo-name]
---

# PR Review Fix

Apply code fixes and respond to review comments on a GitHub pull request.

## Inputs

- `$1` — PR number (required)
- `$2` — Repository in `owner/name` format (optional, defaults to current repo)

## Workflow

### 1. Fetch PR context

```
REPO=$2 (or detect from git remote)
```

- Get PR metadata: title, branch, base branch, state
- Get all review comments (including pending): file, line, body, thread ID
- Group comments by file for efficient processing

### 2. Read all affected files

Read every file that has review comments. Understand the full context before making any changes.

### 3. Classify each comment

For each review comment, decide the action:

| Category | Action | Example |
|----------|--------|---------|
| **Code fix** | Make the change | "вынести в shared", "переименовать", "добавить типы" |
| **Disagree** | Reply with reasoning | "useMemo здесь лишний потому что..." |
| **Question** | Ask user before acting | Ambiguous or architectural comments |

When disagreeing — always provide a clear technical argument, not just "не нужно".

### 4. Apply code fixes

- Follow project conventions from CLAUDE.md and docs/CONTRIBUTING.md
- Check if similar utilities already exist in `shared/` before creating new ones
- Run `npm run typecheck` after all changes to verify nothing is broken
- If typecheck fails — fix the errors before proceeding

### 5. Reply to comments where needed

For comments where I disagree or want to explain the approach:

```bash
gh api repos/{owner}/{repo}/pulls/comments/{comment_id}/replies \
  -f body='<explanation>

*— via Claude Code*'
```

Every reply MUST end with `*— via Claude Code*` signature.

### 6. Resolve all threads

Use GraphQL API to resolve all review threads:

```bash
# Get unresolved thread IDs
gh api graphql -f query='{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr_number}) {
      reviewThreads(first: 50) {
        nodes {
          id
          isResolved
          comments(first: 1) {
            nodes { body }
          }
        }
      }
    }
  }
}'

# Resolve each thread
gh api graphql -f query='mutation {
  resolveReviewThread(input: {threadId: "{thread_id}"}) {
    thread { isResolved }
  }
}'
```

### 7. Commit and push

```bash
git add <changed-files>
git commit -m "<type>: address PR review — <summary>

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
git push
```

### 8. Verify

Confirm:
- [ ] All threads resolved (unresolved count = 0)
- [ ] Typecheck passes
- [ ] Changes pushed to remote

## Rules

- **Read before edit** — never modify a file without reading it first
- **Check shared/** — before creating any utility, check if it already exists in `shared/lib/`
- **Minimal changes** — only change what the review comment asks for, no scope creep
- **Signature** — all PR replies must have `*— via Claude Code*`
- **Ask when unsure** — if a comment is ambiguous or requires an architectural decision, ask the user
- **Pending reviews block comments** — if the reviewer has a pending (unsubmitted) review, notify the user to submit it first before I can reply
