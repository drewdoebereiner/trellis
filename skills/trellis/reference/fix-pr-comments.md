---
name: fix-pr-comments
description: Use when asked to fix open PRs with review comments, address review feedback across multiple branches, resolve review threads, or clear pending review requests. Triggers on: "fix PR comments", "address review feedback", "resolve PR reviews", "fix open PRs".
---

# Fix PR Comments

## Overview

For each open PR that has unresolved review comments or a `CHANGES_REQUESTED` verdict, check out the branch, fix every issue, push the changes, and reply to each thread confirming the fix. One PR at a time -- never parallel, branches can conflict.

---

## CRITICAL: gh CLI only, never GitHub MCP

All GitHub operations use `gh` CLI with `$GH_TOKEN`. Never use GitHub MCP tools.

```bash
# gh uses GH_TOKEN automatically — no login needed in remote environments
export GH_TOKEN=$GH_TOKEN
```

| Rationalization | Reality |
|----------------|---------|
| "GitHub MCP is already configured" | Does not matter. Use `gh`. |
| "MCP is easier than gh CLI" | `gh` returns the same data. |
| "Let me try MCP and fall back to gh" | No. Start with `gh`. There is no fallback. |
| "gh isn't available" | Fall back to `curl` against the GitHub REST API with `GH_TOKEN`. Never MCP. |

If you are about to call any GitHub MCP tool: stop. Use `gh` instead.

---

## Pre-flight: Clean Working Tree

```bash
if ! git diff --quiet || ! git diff --cached --quiet; then
  echo "ERROR: working tree is dirty. Stash or commit first." && exit 1
fi
git fetch origin
```

Do not proceed with a dirty tree. If dirty, report it and stop.

---

## Step 1: Find PRs with Pending Feedback

```bash
# Get all open PRs with their review state
gh pr list --state open --json number,title,headRefName,reviewDecision,url \
  --jq '.[] | select(.reviewDecision == "CHANGES_REQUESTED" or .reviewDecision == null)'
```

Also capture PRs that have inline comments even without a formal CHANGES_REQUESTED decision:

```bash
gh pr list --state open --json number,title,headRefName,url | jq -r '.[].number' | while read n; do
  COUNT=$(gh api repos/:owner/:repo/pulls/$n/comments --jq 'length')
  [ "$COUNT" -gt 0 ] && echo "PR $n has $COUNT inline comments"
done
```

Build a list of PR numbers to process. Skip PRs with no comments and `APPROVED` decision.

---

## Step 2: For Each PR -- Sync Branch

Process PRs sequentially. For each:

```bash
BRANCH=$(gh pr view PR_NUMBER --json headRefName --jq '.headRefName')

git checkout "$BRANCH" 2>/dev/null || git checkout -b "$BRANCH" --track "origin/$BRANCH"

# Fast-forward only -- refuse if diverged
if ! git merge --ff-only "origin/$BRANCH"; then
  echo "ERROR: PR_NUMBER branch diverged from remote. Skipping." && continue
fi
```

If the branch cannot be cleanly synced, skip the PR and log it.

---

## Step 3: Fetch All Review Comments

Run all three fetches for the PR:

```bash
N=PR_NUMBER

# Inline code comments (with file + line context)
gh api repos/:owner/:repo/pulls/$N/comments \
  --jq '[.[] | {id: .id, path: .path, line: .original_line, body: .body, user: .user.login, resolved: .resolved}]'

# Formal review bodies (CHANGES_REQUESTED reviews contain summary feedback)
gh api repos/:owner/:repo/pulls/$N/reviews \
  --jq '[.[] | select(.state == "CHANGES_REQUESTED" or .state == "COMMENTED") | {id: .id, user: .user.login, state: .state, body: .body}]'

# Issue-level comments (general conversation)
gh api repos/:owner/:repo/issues/$N/comments \
  --jq '[.[] | {id: .id, user: .user.login, body: .body}]'
```

**CodeRabbit special handling:** If any comment is from a user matching `coderabbit`, look for a `<details>` block titled "Prompt for AI Agents" in the review body. Use that block as the authoritative fix list for CodeRabbit feedback -- it is more precise than the inline summaries.

Filter out already-resolved threads. Only fix open feedback.

---

## Step 4: Read Affected Files Before Touching Anything

Before writing a single edit, read every file mentioned in the comments. Confirm each issue still exists in the current code. Issues may already be fixed from a previous push.

```bash
# For each file mentioned in comments:
cat path/to/file.py  # use Read tool, not cat
```

If an issue is already fixed, mark it as skipped in your notes. Do not touch it.

---

## Step 5: Fix All Issues

Work through every open comment in order:

- Apply the minimal change to satisfy the feedback
- Follow project conventions (CLAUDE.md or AGENTS.md if present)
- Do not refactor surrounding code that was not flagged
- Do not fix issues in files you have not confirmed exist
- For schema-related changes: all required migration files must be applied, or skip and log it

Commit after all issues for this PR are fixed:

```bash
git add <specific files only -- never git add -A>
git commit -m "$(cat <<'EOF'
fix(PR_NUMBER): address review feedback

- brief bullet per issue fixed
EOF
)"
```

---

## Step 6: Push

Push to the org remote (`upstream`), never to a personal fork (`origin`):

```bash
git remote -v | grep upstream  # confirm upstream is the org remote
git push upstream "$BRANCH"
```

---

## Step 7: Reply to Each Comment Thread

After pushing, reply to each comment you fixed so reviewers know which commit addressed it:

```bash
COMMIT=$(git rev-parse --short HEAD)

# Reply to an inline comment thread
gh api repos/:owner/:repo/pulls/$N/comments \
  --method POST \
  --field body="Fixed in $COMMIT." \
  --field in_reply_to=COMMENT_ID
```

For review-level feedback (not inline), post a general PR comment:

```bash
gh pr comment $N --body "Addressed all review feedback in $COMMIT. See individual thread replies for details."
```

If a thread can be resolved via GraphQL:

```bash
gh api graphql -f query='
  mutation ResolveThread($threadId: ID!) {
    resolveReviewThread(input: {threadId: $threadId}) {
      thread { isResolved }
    }
  }
' -f threadId="THREAD_NODE_ID"
```

Thread node IDs are available in the GraphQL PR review threads query:

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $number) {
        reviewThreads(first: 50) {
          nodes { id isResolved comments(first: 1) { nodes { body } } }
        }
      }
    }
  }
' -f owner=OWNER -f repo=REPO -F number=PR_NUMBER \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false) | .id'
```

Resolve each unresolved thread after replying.

---

## Step 8: Report

After all PRs are processed:

| PR | Branch | Issues Fixed | Issues Skipped | Commit | Status |
|----|--------|-------------|----------------|--------|--------|
| #42 | feat/qwy-42-webhooks | 3 | 1 (already fixed) | a3f9c1d | Pushed, threads replied |
| #38 | bug/qwy-38-auth | 2 | 0 | b81e20f | Pushed, threads replied |
| #31 | feat/qwy-31-schema | 0 | -- | -- | Skipped: schema change, no migration |

---

## Stopping Rules

- Dirty working tree before starting: stop, report, do not proceed
- Branch cannot fast-forward: skip that PR, continue to next
- Comment is from a bot other than CodeRabbit and is ambiguous: skip that comment, note it
- Schema change required but migration missing: skip the PR, note it
- Push fails: log the error, do not retry, continue to next PR

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Fixing issues that are already resolved | Read the file first. Confirm the issue still exists. |
| Pushing to origin instead of upstream | Check `git remote -v`. Push to upstream only. |
| Using `git add -A` | Add specific files. Avoid staging .env or secrets. |
| Replying to comments without pushing first | Push first, then reply with the commit SHA. |
| Fixing CodeRabbit inline summaries without reading the AI Agents block | Find the `<details>` block. That is the authoritative fix list. |
| Starting next PR before current one is pushed | One PR fully done (push + replies) before moving on. |
