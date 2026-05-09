# routine-skills

Personal collection of [Claude Code](https://claude.com/claude-code) skills I use across projects to automate routine engineering work.

Not a framework. Not a library. Just working `SKILL.md` files I drop into projects and adapt. Sharing publicly in case the patterns are useful as a starting point.

## What's here

| Skill | What it does |
|---|---|
| [`adopt-code`](./adopt-code/) | Audit a PR against project conventions, produce a refactor plan, optionally apply (sequential or via parallel teammates) |
| [`bugfix`](./bugfix/) | Batch-process Jira bug tickets — fetch, fix, branch, PR |
| [`jira-spec-builder`](./jira-spec-builder/) | Turn a high-level change into a decomposed Jira ticket structure (epic → stories → tasks) via repo crawl + business interview |
| [`pr-review-fix`](./pr-review-fix/) | Read PR review comments, apply fixes, reply to reviewers, resolve threads |

## Layout

Each skill lives in its own directory:

```
<skill-name>/
└── SKILL.md
```

Supporting files (examples, scripts, screenshots) can live alongside `SKILL.md` as the skill grows.

## How to use

**As Claude Code skills (recommended).** Copy the skill directory into your project's `.claude/skills/` (or the global `~/.claude/skills/`). The model picks the relevant skill based on context and the `description` in its frontmatter.

**As slash commands.** Copy `<skill>/SKILL.md` to `.claude/commands/<skill>.md` (or the global equivalent), then invoke with `/<skill> <args>`. Each skill's frontmatter has an `argument-hint` showing what's expected.

## Notes on portability

The skills reference specific tooling: Jira via Atlassian MCP, GitHub via `gh`, project conventions documented in `CLAUDE.md` / `AGENTS.md`, and stack-specific commands like typecheck and lint. Adapt the wiring to your stack — the workflow logic generalizes.

## Status

Active. New skills get added when a real recurring task earns extraction.

## License

MIT — see [LICENSE](./LICENSE).
