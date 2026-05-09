---
description: Turn a high-level project change into a fully-decomposed Jira ticket structure (epic → stories → tasks split FE/BE) via repo crawl + business-first interview
argument-hint: <high-level-change-description> [<frontend-repo-path>] [<backend-repo-path>]
---

# Jira Spec Builder

Take a high-level description of what needs to change, crawl the codebase, interview the user about business outcomes and user flows, and produce a decomposed ticket structure (epic → frontend stories → backend stories → tasks). On approval, create the tickets in Jira.

This is a **planning skill** — no code is written or modified. The output is tickets that any engineer can pick up.

## Inputs

- `$1` — high-level description of the change (one paragraph or more)
- `$2` (optional) — frontend repo path; defaults to current working directory
- `$3` (optional) — backend repo path
- Jira MCP must be configured (`mcp__atlassian__*` tools available)

If `$1` is missing, ask the user for the change description.

## Workflow

### 0. Pre-flight

- Verify Jira MCP is available: try `mcp__atlassian__getMyself`. If it fails, abort with a clear error.
- Get cloudId via `mcp__atlassian__getAccessibleAtlassianResources`
- Confirm the target Jira project key with the user (e.g. `APP`, `WEB`)

### 1. Parse the change

Read the user's description. Restate it back: "I'm hearing you want to: ..." Get explicit confirmation before proceeding. If you misunderstood, the rest of the skill misfires.

### 2. Map the affected area in the frontend repo

- Identify which existing modules / features are touched by this change (use grep, file search, semantic exploration)
- For each touched module: read the main file(s), understand the current UX
- Build a map: "This change touches modules A, B, C. Module A does X. Module B does Y. Module C does Z."

Print the map to the user. Ask: "Did I miss anything? Is anything in here NOT actually affected?" Let the user correct.

### 3. (Optional) Map the backend repo

If `$3` is provided, repeat step 2 for backend. Identify which API endpoints, services, or schemas are touched.

If backend repo is not provided, ask the user: "Are backend changes needed? If yes, describe them at a high level."

### 4. Business + user-flow interview

This step captures the **business and user-experience layer**. Technical implementation details (error codes, retry logic, race conditions, schema migrations) are NOT discussed here — they belong inside individual tasks during decomposition, where engineers picking up the work add the concrete detail.

Cover these dimensions:

- **Goal** — what's the desired business state after this ships? One sentence.
- **Primary users** — who is this for? Role, context.
- **Primary user flows** — the 1–2 most common scenarios. Step-by-step from the user's perspective.
- **Alternative paths** — secondary flows worth supporting in this iteration.
- **Out of scope** — what explicitly is NOT in this iteration. Keeps scope honest.
- **User-visible states** — empty, no-results, success confirmation, error messaging at the UX level (NOT HTTP codes or specific failure modes).
- **Permissions / gating** — who can access this (business-level: role, paid tier, ownership), NOT auth implementation.
- **Acceptance criteria** — what does "done" look like for the PM / business owner.

Ask in batches of 2–3 questions per round. Hard cap: **3 rounds total**. The user can answer "skip" or "default" for any dimension.

Track the answers as you go.

**Do NOT interview about technical edge cases here.** Error codes, race conditions, retry logic, schema migrations, network failure handling — these are implementation concerns. They get noted in tasks during decomposition, but engineers fill in the concrete detail when they pick the ticket up.

### 5. Draft ticket structure

Produce a markdown structure following this hierarchy:

- **Epic** — one-line business goal (the *why*).
- **Stories** — user-facing slices or business capabilities. Written as "As [user], I can [action] so that [outcome]" or as a clear business capability statement. Each story carries acceptance criteria from the interview. No technical specifics.
- **Tasks** — concrete dev work under each story. **This is where technical detail lives**: API endpoints, screens, tables, validation rules, library / integration names, file paths if needed. Tasks split FE / BE.

```
EPIC: <business goal in one line>

  STORY (FE): As an admin, I can revoke an active session token from the user-detail page
    Acceptance:
      - Admin sees a "Revoke" action on every active session row
      - Confirming the action removes the session within seconds
      - Revoked sessions disappear from the active list and appear in audit history
    User-visible states:
      - Empty list when the user has no active sessions
      - Toast confirmation on success
      - Inline error message on failure
    Out of scope: bulk revocation, scheduled revocation

    Task (FE): Add Revoke action button to session row in user-detail page
    Task (FE): Wire confirm modal + optimistic update + error toast
    Task (BE): Endpoint POST /sessions/:id/revoke; update session.status, write audit row

  STORY (BE): Audit log captures revocation events
    Acceptance:
      - Every revocation creates an audit_log row with actor, target user, timestamp, reason
      - Audit rows are visible in the existing audit dashboard

    Task (BE): Extend audit_log schema with action_type='session_revoke'
    Task (BE): Emit event from revoke endpoint; ensure idempotency on retries
```

**Story descriptions stay business-focused** — what the user can do, why, what they see. No code, no file paths.

**Task descriptions can carry technical specifics** — file paths, library names, endpoint paths, schema changes. That's the level engineers want. Still no code blocks; that's their job to write.

Show the structure to the user.

### 6. Approval loop

User options:

- **Approve as-is** → proceed to step 7
- **Edit** — user calls out specific changes; apply and re-show
- **Add a story / task** — append and re-show
- **Drop a story / task** — remove and re-show
- **Restart** — go back to step 4

Loop until approved.

### 7. Create tickets in Jira

Using Jira MCP:

1. Create the epic via `mcp__atlassian__createJiraIssue` (issueType: Epic)
2. For each story: create with `parent` linked to the epic's key
3. For each task: create with `parent` linked to the story's key
4. Set assignee on the **epic** to the user (`mcp__atlassian__getMyself` → accountId). Leave stories and tasks unassigned by default.
5. Print the URLs of all created tickets

### 8. Recap

Print a summary:

- N tickets created
- Links to the epic and top-level stories
- Reminder: stories and tasks are unassigned, the user should triage / assign in Jira

## Rules

- **Planning only.** Never write or modify code in this skill. The output is tickets, period.
- **Stories are business, tasks are technical.** Stories describe user scenarios or business capabilities, no technical specifics. Tasks carry the concrete dev detail (file paths, library names, schema changes are fair game inside tasks).
- **No technical interrogation at the interview stage.** Don't ask about error codes, race conditions, retry behavior, network failures, or schema migrations during step 4. Those belong inside task descriptions or get added by engineers when they pick up the work.
- **Confirm understanding before crawling.** Step 1 (restate the change) is non-skippable. Misunderstanding the change cascades through the whole flow.
- **Cap interview rounds.** Hard limit: 3 rounds, 2–3 questions per round.
- **One epic per skill run.** If the change is too big for a single epic, ask the user to split it before running this skill.
- **Approval before Jira write.** Never call `mcp__atlassian__createJiraIssue` until the user has approved the full structure.
- **Stop on Jira failure.** If any ticket creation fails mid-run, stop and report what was created vs what failed. Don't try to roll back automatically.
- **No code blocks in tickets.** Engineers write code. The skill describes what needs to be built.
