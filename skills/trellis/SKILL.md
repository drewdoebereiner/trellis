---
name: trellis
description: "Agentic SDLC automation for the full development lifecycle: roadmap vision, backlog research, autonomous implementation, unit test generation, PR review, and PR comment resolution. Invoke as /trellis <sub-command>. Sub-commands: vision-roadmap, research-backlog, dev-backlog, write-unit-tests, fix-pr-comments, bulk-pr-review."
---

# Trellis

Trellis automates the full software development lifecycle with six composable sub-commands. Invoke any sub-command as `/trellis <sub-command>`.

## Sub-commands

| Sub-command | What it does |
|---|---|
| `vision-roadmap` | Read the repo, ask clarifying questions, produce a strategic roadmap doc, create Linear backlog tickets |
| `research-backlog` | Enrich each Linear backlog ticket with parallel research comments before implementation |
| `dev-backlog` | Fetch up to 5 Todo tickets, implement each on its own branch, open PRs targeting dev, move to In Review |
| `write-unit-tests` | Discover test framework from codebase, write pattern-matched tests for staged changes |
| `fix-pr-comments` | Find all open PRs with CHANGES_REQUESTED or unresolved comments, fix every issue, reply to threads |
| `bulk-pr-review` | Map PR dependency layers, dispatch parallel review subagents per layer, post verdicts to GitHub |

## Routing

When invoked as `/trellis <sub-command>`, load and execute the matching reference file:

- `vision-roadmap` -> `reference/vision-roadmap.md`
- `research-backlog` -> `reference/research-backlog.md`
- `dev-backlog` -> `reference/dev-backlog.md`
- `write-unit-tests` -> `reference/write-unit-tests.md`
- `fix-pr-comments` -> `reference/fix-pr-comments.md`
- `bulk-pr-review` -> `reference/bulk-pr-review.md`

If invoked as `/trellis` with no sub-command, list the table above and ask the user which sub-command to run.
