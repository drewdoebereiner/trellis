---
name: vision-roadmap
description: "Use when the user wants to understand where the product should go next, generate a strategic roadmap, create backlog tickets from a product vision, or thinks about what to build next. Triggers on: roadmap, product vision, what should we build, visionary, where should we take this, improve the product strategically."
---

# Vision Roadmap

## Overview

Act as a product visionary: read the repo and all documentation, ask clarifying questions, then produce a strategic roadmap document and create Linear backlog tickets via the Linear REST/GraphQL API.

The goal is not bug hunting. The goal is imagining what this product needs to become.

## Before You Begin: Check the Linear API Key

**Do this before reading a single file.** You cannot create tickets without it, and discovering a missing key after building the entire roadmap wastes everyone's time.

```bash
echo $LINEAR_API_KEY
```

If the variable is empty or unset, ask the user to provide the key now. Do not proceed to Phase 1 until you have confirmed a working key.

---

## CRITICAL: Linear tickets use curl, never MCP

This skill uses `curl` against the Linear GraphQL API directly, not the Linear MCP server. MCP tool calls are expensive on tokens and context — each MCP invocation carries significant overhead in the conversation window. A targeted GraphQL query via curl returns exactly what you ask for and nothing more, keeping context lean across a skill that already reads a full codebase.

**Rationalizations you will be tempted to use -- and why they are wrong:**

| Rationalization | Reality |
|----------------|---------|
| "MCP handles auth automatically" | `$LINEAR_API_KEY` + curl is equally straightforward. |
| "MCP has better error handling" | curl returns the same JSON errors. Check the `errors` field. |
| "MCP is less error-prone" | A curl command with a wrong field fails loudly. That is fine. |
| "MCP is worth it for complex queries" | The GraphQL queries here are simple and well-defined. MCP overhead is not justified. |
| "The repo has existing Linear tickets so MCP must be set up" | Existing tickets prove nothing about which method to use now. |

If you catch yourself about to call any `mcp__claude_ai_Linear__*` tool: stop. Use curl.

---

## Phase 1: Context Gathering

Read in parallel before saying anything else:

- `README.md` and any top-level docs
- All files under `docs/`, `documentation/`, `wiki/`, or equivalent
- `CLAUDE.md` / `AGENTS.md` for constraints and project values
- `git log --oneline -30` to understand current momentum and recent focus
- Any existing roadmap or planning docs

From this reading, synthesize:
- What is the product's core value proposition?
- Who are the users?
- What has the team been focused on recently?
- What is obviously unfinished or aspirational in the docs?

## Phase 2: Clarifying Questions

**Never skip this phase.** Different answers completely change the roadmap.

Ask the user ALL of the following before writing anything:

1. **Target users** -- who uses this, and what are their biggest unsolved pain points?
2. **Horizon** -- 3 months, 6 months, 1 year, or don't care (focus on impact over timeline)?
3. **Priority axis** -- growth, retention, monetization, developer experience, stability?
4. **Team constraints** -- approximate team size and any hard technical/resource limits?
5. **Off-limits** -- anything explicitly excluded from scope?
6. **Known priorities** -- any in-flight work or committed features to preserve?
7. **Inspiration** -- any competitor products or experiences the user wants to move toward or away from?

Wait for user responses. Do not proceed to Phase 3 without answers.

## Phase 3: Vision Generation

Think expansively. Ask:

- What is the product's 10x version? What would make it irreplaceable?
- What does a user need to do today that requires leaving this product?
- What technical foundations are missing that block future capabilities?
- What would delight a power user vs. a first-time user?
- Where is the product fragile, incomplete, or doing something embarrassing compared to peers?

Group findings into 3-6 **themes** (e.g., "Developer Experience", "Core Workflow", "Integrations", "Performance", "Collaboration", "Discovery").

For each theme, generate 2-5 specific initiatives:

| Field | Description |
|-------|-------------|
| **Title** | Short, active: "Add webhook subscriptions for trade events" |
| **What** | Clear description of the improvement |
| **Why** | Specific user or business value |
| **Priority** | High / Medium / Low |
| **Effort** | Small (days) / Medium (weeks) / Large (months) |

## Phase 4: Roadmap Document

Write to `ROADMAP-<YYYY-MM>.md`. Place it somewhere sensible for the project — `docs/`, `documentation/roadmap/`, or the repo root are all fine. Create the directory if it does not exist.

```markdown
# Product Roadmap -- [Month Year]

## Vision Statement
[1-2 sentences: where the product is going and why it matters]

## Themes

### [Theme Name]
[What this theme is about and why it matters now]

**Initiatives:**
- [Initiative title] -- [one-line why]
- ...

## Prioritized Initiative List

| Initiative | Theme | Priority | Effort | Value |
|------------|-------|----------|--------|-------|
| ...        | ...   | High     | Medium | ...   |

## Now (Next 3 Months)
Top 5-8 initiatives with brief rationale for why these come first.

## Next (3-12 Months)
Strategic bets, larger investments, and foundations for the future.

## Later / Explicitly Excluded
Things left out and the reasoning -- this prevents re-litigating them.
```

## Phase 5: Linear Ticket Creation

Create one issue per initiative using the Linear GraphQL API directly via curl. The API key was confirmed at the start; use `$LINEAR_API_KEY` throughout.

**Step 1 -- Get team ID:**
```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ teams { nodes { id name } } }"}' | jq '.data.teams.nodes'
```

**Step 2 -- Get backlog state ID for the team:**
```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ workflowStates(filter: { team: { id: { eq: \"TEAM_ID\" } } }) { nodes { id name type } } }"}' \
  | jq '.data.workflowStates.nodes[] | select(.type == "backlog" or (.name | ascii_downcase | contains("backlog")))'
```

**Step 3 -- Create each issue:**
```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateIssue($input: IssueCreateInput!) { issueCreate(input: $input) { success issue { id identifier url } } }",
    "variables": {
      "input": {
        "teamId": "TEAM_ID",
        "stateId": "BACKLOG_STATE_ID",
        "title": "TITLE",
        "description": "DESCRIPTION_MARKDOWN",
        "priority": PRIORITY_INT
      }
    }
  }'
```

**Priority mapping:**
| Roadmap priority | Linear integer |
|-----------------|---------------|
| High | 2 |
| Medium | 3 |
| Low | 4 |
| No priority | 0 |

**Issue description template (markdown):**
```
## What
[clear description]

## Why
[user or business value]

## Effort estimate
[Small / Medium / Large]

## Theme
[theme name from roadmap]
```

Create issues one at a time. Log each identifier and URL as you go.

## Phase 6: Summary Report

After all tickets are created, report to the user:

- Path to the roadmap document
- Table of created Linear tickets: identifier, title, priority, URL
- Count by theme
- Any issues that failed to create and why

## Common Mistakes

**Generating a feature list, not a vision.** The roadmap must tell a story about where the product is going. Themes with rationale beat a flat list of features.

**Skipping clarifying questions.** Phase 2 is not optional. A 3-month vs. 1-year horizon produces completely different roadmaps. The same product with a "developer experience" priority vs. "growth" priority produces completely different roadmaps.

**Using the Linear MCP server.** MCP calls are token-heavy and bloat context. Always use `curl` against the GraphQL API with `$LINEAR_API_KEY` — it returns exactly what you need and nothing more.

**Creating too many tickets.** 10 focused, well-described tickets beat 40 vague ones. If the roadmap has more than 20 initiatives, consolidate before ticketing.

**Assuming the API key exists.** The key must be confirmed at the very start -- before reading any files. Discovering it is missing after the roadmap is built means repeating Phase 5 from scratch.

**Ignoring the git log.** Recent commit history reveals the team's actual velocity and current focus. A roadmap that ignores in-flight work will feel disconnected.

## Red Flags -- STOP if you notice these

- About to write the roadmap without asking clarifying questions first
- Using the Linear MCP tools instead of `curl`
- Creating issues without a backlog state ID (they will land in the wrong column)
- Writing the roadmap document after creating tickets (doc should come first)
