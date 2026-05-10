---
name: dev-backlog
description: "Use when asked to work the dev backlog, implement Linear tickets, or run a nightly coding pass on your team's issues. Triggers on: work the backlog, implement tickets, dev backlog run, code up tickets, nightly dev run."
---

# Dev Backlog

## Overview

For each Todo ticket in your Linear team, implement the code changes, open a PR targeting dev, and move the ticket to In Review. Maximum 5 tickets per run. Stop cleanly if blockers are hit.

**Always uses `$LINEAR_API_KEY` via curl. Never uses MCP Linear tools.**

---

## Configuration

Set these before running:

```bash
# Absolute path to the repo on this machine
export PROJECT_ROOT=/path/to/your/repo

# Your Linear team name (used to look up the team ID)
export LINEAR_TEAM_NAME="your-team-name"
```

---

## Pre-flight: Confirm API Key

```bash
if [ -z "$LINEAR_API_KEY" ]; then
  export LINEAR_API_KEY=$(grep '^LINEAR_API_KEY=' "$PROJECT_ROOT/.env" 2>/dev/null | cut -d= -f2-)
fi
echo $LINEAR_API_KEY
```

If still empty, stop and ask the user. Do not proceed without a confirmed key.

---

## CRITICAL: curl for Linear, gh CLI for GitHub — never MCP for either

**Linear:** use `curl` with `$LINEAR_API_KEY`. **GitHub:** use `gh` CLI with `$GH_TOKEN`. Never use MCP tools for either.

```bash
# gh uses GH_TOKEN automatically — no login needed in remote environments
export GH_TOKEN=$GH_TOKEN
```

| Rationalization | Reality |
|----------------|---------|
| "MCP is easier" | The user explicitly requires API key auth. |
| "MCP handles errors better" | curl / gh return the same data. Check the response. |
| "MCP tools are already configured" | Does not matter. Use curl / gh. |
| "gh isn't available" | Fall back to `curl` against the GitHub REST API with `GH_TOKEN`. Never MCP. |

If you are about to call any `mcp__claude_ai_Linear__*` or GitHub MCP tool: stop. Use curl / gh instead.

---

## Step 1: Sync the Repo

Run this before touching any ticket. Refuse to proceed if it fails.

```bash
REPO=$PROJECT_ROOT

# Fail if working tree is dirty
cd "$REPO"
if ! git diff --quiet || ! git diff --cached --quiet; then
  echo "ERROR: working tree is dirty. Aborting." && exit 1
fi

git fetch origin

# Switch to dev or create it from origin/main
if git rev-parse --verify dev >/dev/null 2>&1; then
  git checkout dev
else
  git checkout -b dev origin/main
fi

# Fast-forward only -- refuse to proceed if diverged
if ! git merge --ff-only origin/dev 2>/dev/null && ! git merge --ff-only origin/main 2>/dev/null; then
  echo "ERROR: cannot fast-forward dev to remote baseline. Aborting." && exit 1
fi

echo "Repo synced. HEAD: $(git rev-parse --short HEAD)"
```

If any command exits non-zero, stop the entire run and report the error.

---

## Step 2: Fetch Team ID

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ teams { nodes { id name } } }"}' \
  | jq --arg name "$LINEAR_TEAM_NAME" '.data.teams.nodes[] | select(.name | ascii_downcase | contains($name | ascii_downcase))'
```

Save the `id` as `TEAM_ID`.

---

## Step 3: Fetch Todo Tickets (up to 5)

Fetch unstarted issues ordered by priority. Cap at 5.

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query TodoIssues($teamId: String!) { issues(filter: { team: { id: { eq: $teamId } }, state: { type: { eq: \"unstarted\" } } }, first: 5, orderBy: priority) { nodes { id identifier title description url priority labels { nodes { name } } } } }",
    "variables": { "teamId": "TEAM_ID" }
  }' | jq '.data.issues.nodes'
```

If 0 issues returned, confirm state names by querying workflow states:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query { workflowStates(filter: { team: { id: { eq: \"TEAM_ID\" } } }) { nodes { id name type } } }"}' \
  | jq '.data.workflowStates.nodes'
```

Then re-query using the correct state IDs for the "Todo" equivalent.

Also fetch the "In Review" state ID now -- you will need it in Step 9:

```bash
IN_REVIEW_STATE_ID=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query { workflowStates(filter: { team: { id: { eq: \"TEAM_ID\" } } }) { nodes { id name } } }"}' \
  | jq -r '.data.workflowStates.nodes[] | select(.name | test("review"; "i")) | .id' | head -1)
```

---

## Step 4: Read Full Ticket Details

For each ticket, fetch the full description and any existing comments before starting work:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query Issue($id: String!) { issue(id: $id) { id identifier title description comments { nodes { body createdAt } } } }",
    "variables": { "id": "ISSUE_ID" }
  }' | jq '.data.issue'
```

Read any research comments (posted by the research-backlog skill) -- they contain implementation approaches and gotchas worth following.

---

## Step 5: Reference Repo Docs

Before writing a single line of code, read the relevant context files:

```bash
# Always read if present
cat "$PROJECT_ROOT/CLAUDE.md" 2>/dev/null || true
cat "$PROJECT_ROOT/AGENTS.md" 2>/dev/null || true

# Explore based on ticket domain
ls "$PROJECT_ROOT/"
```

Also check for a `tasks/lessons.md` file and read it -- it may contain patterns to avoid.

Infer which files need changing before branching. If the ticket is ambiguous, stop and ask rather than guessing.

---

## Step 6: Create Feature Branch

Branch from the freshly synced local dev branch. Never branch from main or an existing feature branch.

```bash
REPO=$PROJECT_ROOT
cd "$REPO"

# Derive slug from ticket title: lowercase, spaces to hyphens, strip special chars
SLUG=$(echo "TICKET_TITLE" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9 -]//g' | tr ' ' '-' | sed 's/--*/-/g' | cut -c1-40)

# Use feat/ prefix for features, bug/ for bugs
BRANCH="feat/IDENTIFIER-$SLUG"
# (use bug/ if the ticket is a bug fix)

git checkout dev
git checkout -b "$BRANCH"
```

**Never use the raw `branchName` field from the Linear API** -- it may contain garbage characters. Always derive the slug from the title.

---

## Step 7: Implement the Ticket

Work sequentially -- one ticket at a time. Do not start the next ticket until the current one has a merged-ready PR.

Implementation checklist for each ticket:
- [ ] Read all files you will touch before editing
- [ ] Make the minimal change to satisfy the ticket
- [ ] Use the **write-unit-tests** skill to write or update tests for the changed code
- [ ] Discover and run the project's test command (check `package.json`, `pytest.ini`, `Makefile`, or project docs)
- [ ] Verify no obvious regressions (read diff before committing)
- [ ] Do not add features, refactor, or introduce abstractions beyond what the ticket requires
- [ ] Do not browser test -- rely on unit/integration tests

Commit when the implementation is complete:

```bash
cd $PROJECT_ROOT
git add <specific files -- never git add -A>
git commit -m "$(cat <<'EOF'
feat(IDENTIFIER): short description of what changed

Why: one sentence on the motivation.
EOF
)"
```

Never include "Co-Authored-By: Claude" in commit messages.

---

## Step 8: Push and Open PR

Push to the org remote (`upstream`), never to a personal fork (`origin`):

```bash
cd $PROJECT_ROOT

# Confirm upstream points to your org remote
git remote -v | grep upstream

git push upstream "$BRANCH"
```

Open PR targeting dev, never main:

```bash
gh pr create \
  --title "IDENTIFIER: TICKET_TITLE" \
  --body "$(cat <<'EOF'
## Summary
- BULLET_SUMMARY

## Linear
TICKET_URL

## Test plan
- [ ] Unit tests pass
- [ ] Manual smoke test steps if any
EOF
)" \
  --base dev \
  --head "$BRANCH"
```

Save the PR URL.

---

## Step 9: Move Ticket to In Review

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation UpdateIssue($id: String!, $stateId: String!) { issueUpdate(id: $id, input: { stateId: $stateId }) { success issue { id state { name } } } }",
    "variables": { "id": "ISSUE_ID", "stateId": "IN_REVIEW_STATE_ID" }
  }' | jq '.data.issueUpdate'
```

Check `success: true`. If false, log and continue -- do not halt the run.

---

## Step 10: Report to User

After all tickets are processed (or run is stopped), print a summary table:

| Ticket | Title | Branch | PR | Status |
|--------|-------|--------|----|--------|
| PROJ-42 | Add webhook support | feat/proj-42-add-webhook-support | https://github.com/... | In Review |
| PROJ-38 | Fix auth redirect | bug/proj-38-fix-auth-redirect | https://github.com/... | In Review |
| PROJ-31 | Ambiguous spec | -- | -- | Skipped: description unclear |

---

## Run Limits and Stopping Rules

- **Maximum 5 tickets per run.** Stop after 5 even if more remain.
- Stop early if:
  - Repo sync fails (dirty tree or diverged branch)
  - A ticket's scope is unclear and cannot be resolved by reading docs
  - Tests are failing before your changes (record the failure, skip the ticket)
  - Rate limits are hit on Linear or GitHub APIs
- Never force-merge or skip test failures to hit the ticket quota.
- If the ticket involves schema or migration changes, all required migration files must be present -- skip and note it rather than doing a partial migration.

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using MCP tools instead of curl | Stop. Re-read the CRITICAL section. Use curl. |
| Branching from main instead of dev | Always `git checkout dev` before creating a branch. |
| Pushing to a personal fork | Push to the org upstream only. Check `git remote -v`. |
| Using the Linear `branchName` field | Derive slug from ticket title. |
| Opening PR against main | `--base dev` always. |
| Starting ticket 6 | Hard stop at 5. |
| Merging the PR | Never merge. Open and leave for review. |
| Using `git add -A` | Add specific files only. Avoid accidentally staging secrets. |
| Skipping tests | Run tests for every ticket. No exceptions. |
| Partial migrations | Apply all required migration files or skip the ticket. |
