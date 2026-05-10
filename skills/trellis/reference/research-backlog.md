---
name: research-backlog
description: "Use when asked to research backlog tickets, enrich Linear issues with context, add research comments to backlog items, or prepare tickets for developers before implementation. Triggers on: research backlog, enrich tickets, add context to issues, research each ticket."
---

# Research Backlog

## Overview

For each backlog ticket in your Linear team, dispatch a parallel research subagent that investigates the ticket deeply and posts a structured comment with findings. The goal is to give the dev agent concrete resources, implementation approaches, and gotchas before coding begins.

**Always uses `$LINEAR_API_KEY` via curl. Never uses MCP Linear tools.**

---

## Pre-flight: Confirm API Key

Do this before anything else:

```bash
echo $LINEAR_API_KEY
```

If empty, ask the user to provide it. Do not proceed without a confirmed key.

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

List all teams and ask the user which one to target if `$LINEAR_TEAM_NAME` is not set:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ teams { nodes { id name } } }"}' \
  | jq '.data.teams.nodes'
```

If `$LINEAR_TEAM_NAME` is set, filter automatically:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ teams { nodes { id name } } }"}' \
  | jq --arg name "$LINEAR_TEAM_NAME" '.data.teams.nodes[] | select(.name | ascii_downcase | contains($name | ascii_downcase))'
```

Save the `id` as `TEAM_ID`.

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

After each research agent returns its markdown, post it as a comment on the ticket.

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

## Step 5: Report to User

After all tickets are processed, print a summary table:

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
