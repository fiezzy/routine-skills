---
description: Serial Jira ticket intake — the user dictates short raw context, you draft a pickup-ready ticket, they approve, you create it, repeat
argument-hint: [<project-key>] [<sprint>]
---

# Jira Ticket Intake

Run a ticket-filing session. The user dictates raw context in one or two sentences; you turn each one into a ticket an engineer can pick up without asking questions, show it as a draft, and create it in Jira **only** after an explicit go-ahead. Then the next one. Many tickets per session, one at a time.

Core principle: **the draft is the deliverable until the user says "create it".**

This is an **intake skill** — no code is written or modified. Reading the codebase is allowed (and expected) to make tickets precise.

## Inputs

- `$1` (optional) — Jira project key (e.g. `PROJ`); ask once if missing
- `$2` (optional) — target sprint; resolve the current open sprint yourself if missing
- Jira MCP must be configured (`mcp__atlassian__*` tools available)
- Repo paths: current working directory, plus any repos the user names during the session

## Workflow

### 0. Pre-flight (once per session)

- Verify Jira MCP: try `mcp__atlassian__getMyself`. If unavailable, say so and switch to paste mode (step 6).
- Get cloudId via `mcp__atlassian__getAccessibleAtlassianResources`.
- Confirm the project key, the ticket language, and the default parent (epic or standalone).
- Read the project's issue types via `mcp__atlassian__getJiraProjectIssueTypesMetadata`: which standalone types exist, which subtask types exist (many boards have per-discipline subtasks like `Frontend` / `Backend` / `Design`), and their hierarchy levels.
- Resolve the current sprint: JQL `project = <KEY> AND sprint in openSprints()` requesting the sprint field (commonly `customfield_10020`) to read its numeric id. Don't block the session on this — put the result in the draft header and let the user correct it in one line.
- Assignee stays **empty** unless the user says otherwise.

Recap the settings in two lines and start taking context.

### 1. Take one piece of context

The user speaks in fragments: a symptom, a screenshot, a half-sentence about a screen. Treat each message as one ticket unless it obviously splits into a story plus per-discipline subtasks.

Do not ask clarifying questions yet.

### 2. Duplicate check — always, one search

JQL-search the key words of the request. If an existing ticket already covers it, name its key, type and status, and recommend expanding that ticket instead of creating a twin. Wait for the user's decision — never silently duplicate.

### 3. Read the code when the request touches implementation

Goal: name the real screen, component, service and call chain, and decide the split (frontend only / backend only / both).

- **Stop condition:** stop as soon as you can name the mechanism, even without a proven root cause. Hand over the chain plus 2–3 concrete spots to verify live, and say that's what you're doing. Do not keep opening files chasing certainty.
- **If the code shows no clear defect:** write the chain you traced, mark it unverified, state that a live repro is needed. Never invent a plausible-sounding root cause to fill the section.
- **If the user asks which side the work belongs to:** answer with the reason ("generation already exists on the backend, so the bulk is frontend plus a small backend guard"), then propose the ticket structure that follows.

### 4. Draft the ticket

Print a header line, then the description. Paste-ready, no preamble.

```
Project · Type · Parent · Sprint · Assignee
```

Never omit a header field — use `—` for empty ones.

Sections, in this order, skipping what does not apply:

- **Problem / Why** — the user's pain in plain words and why it matters. Business-first.
- **What we do** — numbered, concrete, decisions already made.
- **Likely cause (verify, not gospel)** — bugs only, when you read the code: the mechanism and the 2–3 places it breaks.
- **Steps to reproduce** + expected / actual — bugs.
- **Scope** — what is explicitly out.
- **Dependencies** — related keys with one line of real status each ("done, Jira status not updated").
- **Acceptance criteria** — checkable statements, not intentions. Include the project's gates (typecheck and lint for frontend, service tests for backend).
- **Affected places (orientation)** — optional, only in per-discipline subtasks and only when you read the code: bullet list of file paths you actually opened. Never guessed paths, never in the business-level story.
- **Screenshot** — one line "the reporter will attach it" when the user sent an image; attachments cannot be uploaded through Jira MCP.

Then ask questions — only the ones that change the work (which screen, which epic, which of two behaviours) — and end with a single "create it?"

### 5. Approval loop

- Only an explicit go word creates anything: "create it", "заводи", "go ahead". Approving the *wording* ("looks good", "нормально") is **not** approval to create.
- On a correction: apply it, re-print **only** what changed, ask again.
- One go-ahead covers only the tickets just discussed. Never create extras you thought of.

### 6. Create in Jira

Using Jira MCP, with `contentFormat: "markdown"` for descriptions:

1. Create the parent-level ticket first (`mcp__atlassian__createJiraIssue`), setting the sprint field to the numeric sprint id.
2. Create per-discipline subtasks with `parent` set to the ticket's key. **Do not send the sprint field on subtasks** — Jira rejects it; subtasks inherit the parent's sprint.
3. If the board has automation that prefixes summaries by discipline (`[FE]` / `[BE]`), do not type the prefix yourself — you'll get it twice.
4. Leave assignee empty unless told otherwise.
5. Verify what landed (parent, sprint, assignee) with one read call.

No Jira tool available? Print the finished ticket as markdown plus the field values so the user can paste it. Never guess credentials or invent an API token.

### 7. Report and continue

Per ticket: key, browse URL, one line of what you verified. Then invite the next piece of context.

Keep a running list of unresolved questions (which epic, which screen, whether to expand an existing ticket) and re-surface it at the end of each turn until the user answers or drops it.

## Rules

- **Intake only.** Never write or modify code in this skill. The output is tickets.
- **No Jira write before an explicit go-ahead.** Applies to creation, edits, comments, links, and status transitions.
- **Never touch tickets you did not create in this session.** No status changes, no reassignment. Offer, and let the user decide.
- **Don't invent technical detail the user didn't give.** No API shapes, endpoint names, schemas, field names or library choices. Mechanisms and file paths you verified in the code are fine; guesses are not.
- **Business voice in stories, technical detail in subtasks.** Same split the engineers rely on when they pick the work up.
- **Acceptance criteria must be checkable.** "Works correctly" is not a criterion; "publishing succeeds without entering a slug" is.
- **Don't block the session on metadata.** Resolve sprint and parent yourself where possible, state the assumption, let the user correct it.
- **No estimates, story points or due dates** unless the user names them.
- **Stop on Jira failure.** Report what was created and what failed; don't roll back automatically.
- **No code blocks in tickets.** Engineers write the code; the ticket says what must be true.
