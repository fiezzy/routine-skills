# routine-skills

Personal collection of [Claude Code](https://claude.com/claude-code) slash-commands I use across projects to automate routine engineering work.

Not a framework. Not a library. Just working `.md` files I drop into projects and adapt. Sharing publicly in case the patterns are useful as a starting point.

## What's here

| Skill | What it does |
|---|---|
| [`adopt-code.md`](./adopt-code.md) | Audit a PR against project conventions, produce a refactor plan, optionally apply (sequential or via parallel teammates) |
| [`bugfix.md`](./bugfix.md) | Batch-process Jira bug tickets — fetch, fix, branch, PR |
| [`jira-spec-builder.md`](./jira-spec-builder.md) | Turn a high-level change into a decomposed Jira ticket structure (epic → stories → tasks) via repo crawl + business interview |
| [`pr-review-fix.md`](./pr-review-fix.md) | Read PR review comments, apply fixes, reply to reviewers, resolve threads |

## How to use

Drop the file into your project's `.claude/commands/` directory (or the global `~/.claude/commands/`). Invoke with `/<skill-name> <args>`. Each file has frontmatter with `description` and `argument-hint`.

## Notes on portability

The skills reference specific tooling: Jira via Atlassian MCP, GitHub via `gh`, project conventions documented in `CLAUDE.md` / `AGENTS.md`, and stack-specific commands like typecheck and lint. Adapt the wiring to your stack — the workflow logic generalizes.

## Status

Active. New skills get added when a real recurring task earns extraction.

## License

MIT — see [LICENSE](./LICENSE).
