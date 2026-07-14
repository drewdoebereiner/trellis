---
name: research-backlog
description: "Use when asked to research backlog tickets, enrich Linear issues with context, add research comments to backlog items, or prepare tickets for developers before implementation. Triggers on: research backlog, enrich tickets, add context to issues, research each ticket."
---

# Research Backlog

## Overview

For each backlog ticket in your Linear team, dispatch a parallel research subagent that investigates the ticket deeply and posts a structured comment with findings. The goal is to give the dev agent concrete resources, implementation approaches, and gotchas before coding begins.

**Always uses `$LINEAR_API_KEY` via curl. Never uses MCP Linear tools.**

---

## Cron behavior

This sub-command is cron-safe. Before doing anything, run the startup preamble in [`cron-contract.md`](./cron-contract.md): resolve non-interactive mode, bootstrap env, open the run log, and acquire the `research-backlog` lock. Required env vars: `LINEAR_API_KEY`, `LINEAR_TEAM_NAME`.

Key non-interactive rules for this skill:
- Missing `LINEAR_API_KEY` or `LINEAR_TEAM_NAME` → **fail fast** (exit non-zero), never prompt.
- A ticket that can't be researched (empty title and description) → **skip and log**, continue the batch.
- Idempotency: skip any ticket that already carries a `<!-- trellis:research-backlog v1 -->` comment (see Step 2b).

---

## Pre-flight: Confirm API Key

Do this before anything else:

```bash
echo $LINEAR_API_KEY
```

Per the cron contract, resolve the key from the environment or `$PROJECT_ROOT/.env`. If it is still empty: in non-interactive mode, fail fast and exit non-zero; only when running interactively may you ask the user to provide it.

---

## CRITICAL: curl only, never MCP

MCP tool calls are token-heavy and bloat context — each invocation carries significant overhead. A targeted GraphQL query via curl returns exactly what you ask for and nothing more, keeping context lean across a skill that may dispatch many parallel agents.

| Rationalization | Reality |
|----------------|---------|
| "MCP is easier" | `$LINEAR_API_KEY` + curl is equally straightforward. |
| "MCP handles errors better" | curl returns the same JSON. Check the `errors` field. |
| "MCP tools are already configured" | Does not matter. MCP overhead is not justified here. |

If you are about to call any `mcp__claude_ai_Linear__*` tool: stop. Use curl instead.

---

## Step 1: Identify the Target Team

`$LINEAR_TEAM_NAME` selects the team without a human in the loop. Filter for it:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ teams { nodes { id name } } }"}' \
  | jq --arg name "$LINEAR_TEAM_NAME" '.data.teams.nodes[] | select(.name | ascii_downcase | contains($name | ascii_downcase))'
```

Save the `id` as `TEAM_ID`.

If `$LINEAR_TEAM_NAME` is unset or matches no team:
- **Non-interactive:** fail fast (exit non-zero) — a cron run cannot pick a team from a list. The one exception: if the workspace has exactly one team, use it.
- **Interactive:** list all teams (`jq '.data.teams.nodes'`) and ask the user which to target.

---

## Step 2: Fetch Backlog Issues

Fetch all issues in Backlog or Todo state. Use pagination if count > 50.

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query BacklogIssues($teamId: String!) { issues(filter: { team: { id: { eq: $teamId } }, state: { type: { in: [\"backlog\", \"unstarted\"] } } }, first: 100) { nodes { id identifier title description url priority labels { nodes { name } } } } }",
    "variables": { "teamId": "TEAM_ID" }
  }' | jq '.data.issues.nodes'
```

If 0 issues are returned, also try fetching without the state filter and look for states named "Backlog" explicitly:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { workflowStates(filter: { team: { id: { eq: \"TEAM_ID\" } } }) { nodes { id name type } } }"
  }' | jq '.data.workflowStates.nodes'
```

Then re-query filtering by the correct state IDs.

---

## Step 2b: Skip Already-Researched Tickets (idempotency)

Trellis marks every research comment with `<!-- trellis:research-backlog v1 -->` on its first line. Before dispatching agents, fetch each ticket's comments and drop any ticket that already has one — this is what makes re-runs (and cron) safe.

```bash
# Returns the marker if this ticket was already researched by Trellis
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query IssueComments($id: String!) { issue(id: $id) { comments { nodes { body } } } }",
    "variables": { "id": "ISSUE_ID" }
  }' | jq -r '.data.issue.comments.nodes[].body' | grep -q "trellis:research-backlog" && echo "ALREADY_RESEARCHED"
```

Log skipped tickets as "already researched" in the run log and exclude them from Step 3. Only research tickets without the marker.

---

## Step 3: Dispatch Parallel Research Agents

For each issue returned, dispatch ONE subagent in parallel. Pass the full issue context in the prompt. Do NOT batch multiple tickets into one agent.

**Agent prompt template:**

```
You are a research agent preparing context for a developer who will implement this Linear ticket.

Ticket: [IDENTIFIER] - [TITLE]
Description:
[DESCRIPTION or "(no description provided)"]
Labels: [LABELS]
Priority: [PRIORITY]

Your job:
1. Read the ticket carefully. Infer the technical domain (auth, UI, API, data pipeline, etc.).
2. Read the repo's README, CLAUDE.md, and any docs/ directory to understand the tech stack before searching.
3. Search the web for:
   - Official documentation for relevant frameworks, APIs, or libraries
   - Prior art: open source implementations, GitHub repos, blog posts that solve this or something similar
   - Best practices, common pitfalls, and gotchas for this type of work
   - Competing approaches and trade-offs
4. Synthesize findings into a structured research comment (see format below).

Research comment format -- return ONLY the markdown comment body:

## Research Notes

### What This Ticket Is Asking For
[1-2 sentence plain-English summary of the work]

### Recommended Approach
[The implementation path that makes the most sense given the stack and constraints. Be specific.]

### Key Resources
- [Resource name](URL) -- one-line description of why this is useful
- (3-6 resources minimum; prefer official docs, then high-quality articles/repos)

### Implementation Considerations
- [Bullet list of non-obvious decisions, trade-offs, or design questions the developer should resolve before starting]

### Potential Gotchas
- [Bullet list of known failure modes, edge cases, or things that commonly trip up developers on this type of work]

### Estimated Complexity
[Simple / Medium / Complex] -- [one-line rationale]

Return only the markdown. No preamble, no explanation.
```

---

## Step 4: Post Comment to Each Ticket

After each research agent returns its markdown, post it as a comment on the ticket. **Prepend the idempotency marker as the very first line of the body** so re-runs and Step 2b can detect it:

```
<!-- trellis:research-backlog v1 -->

## Research Notes
...
```

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateComment($input: CommentCreateInput!) { commentCreate(input: $input) { success comment { id url } } }",
    "variables": {
      "input": {
        "issueId": "ISSUE_ID",
        "body": "RESEARCH_MARKDOWN"
      }
    }
  }'
```

Check `success: true` in the response. If false, log the error and continue to the next ticket -- do not halt the entire run.

**Escape the markdown body correctly:** Use a bash variable to avoid escaping issues:

```bash
BODY=$(cat <<'EOF'
[paste research markdown here]
EOF
)

jq -n --arg body "$BODY" --arg issueId "ISSUE_ID" \
  '{query: "mutation CreateComment($input: CommentCreateInput!) { commentCreate(input: $input) { success comment { id url } } }", variables: {input: {issueId: $issueId, body: $body}}}' \
  | curl -s -X POST https://api.linear.app/graphql \
    -H "Authorization: $LINEAR_API_KEY" \
    -H "Content-Type: application/json" \
    -d @-
```

---

## Step 5: Report

After all tickets are processed, write the summary to both the run log (`$RUN_LOG`) and stdout, then end with a `RESULT:` line per the cron contract (e.g. `RESULT: ok — 4 researched, 2 skipped (already done), 1 skipped (no content)`). Exit 0 even if some tickets were skipped; only env/auth/team failures exit non-zero.

Summary table:

| Ticket | Title | Comment Posted | Notes |
|--------|-------|---------------|-------|
| TEAM-42 | Add webhook support | Yes | https://linear.app/... |
| TEAM-38 | Fix auth redirect | Yes | https://linear.app/... |
| TEAM-31 | (no description) | Skipped | No description to research |

Skip tickets with no title AND no description -- they cannot be researched meaningfully. Log them as skipped.

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using MCP tools instead of curl | Stop. Re-read the CRITICAL section. Use curl. |
| Posting all research in one batch call | Each ticket needs its own curl mutation. |
| Not escaping markdown in JSON body | Use `jq -n --arg body "$BODY"` pattern. |
| Skipping tickets with no description | Try -- title alone is often enough for research. Only skip if both title and description are empty. |
| Running agents sequentially | All research agents should be dispatched in parallel. |
| Hallucinating resources | Research agents must use WebSearch and WebFetch. No fabricated URLs. |
| Hardcoding the tech stack in the agent prompt | The agent should read the repo's README and docs to discover the stack. |
