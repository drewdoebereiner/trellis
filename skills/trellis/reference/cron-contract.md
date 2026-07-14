---
name: cron-contract
description: "Shared execution contract every Trellis sub-command follows when run unattended (cron, CI, scheduled agent). Covers non-interactive behavior, env bootstrap, run logging, concurrency locks, idempotency markers, and exit signaling. Not invoked directly — referenced by the other sub-commands."
---

# Trellis Cron Contract

Every Trellis sub-command that runs on a schedule follows this contract. It is what makes the pipeline safe to wire into cron: no command blocks on a human, every run is recoverable from a log, overlapping runs cannot corrupt state, and re-runs never duplicate work.

**Cron-safe sub-commands:** `research-backlog`, `dev-backlog`, `bulk-pr-review`, `fix-pr-comments`.

**Not cron-safe:** `vision-roadmap` is interactive by design (it asks the human where the product should go) — it fails fast in non-interactive mode instead of guessing. `write-unit-tests` is a sub-skill invoked *by* `dev-backlog`, not a standalone scheduled job.

---

## 1. Non-interactive mandate

A scheduled run has no human to answer questions. Determine the mode once, at the start:

```bash
# Non-interactive if explicitly set, or if there is no terminal attached (cron/CI)
if [ "${TRELLIS_NONINTERACTIVE:-}" = "1" ] || [ ! -t 0 ]; then
  NONINTERACTIVE=1
else
  NONINTERACTIVE=0
fi
```

**When `NONINTERACTIVE=1`, you must NEVER stop and ask the user anything.** Every decision point resolves one of two ways:

- **Fail fast** — for a missing prerequisite that makes the whole run impossible (no API key, dirty tree, no team). Write the reason to the log and exit non-zero.
- **Skip and log** — for a single work item that can't be handled (ambiguous ticket, diverged branch, missing migration). Record it in the run log and continue to the next item.

Never guess at something a human was supposed to decide. Skip it and move on.

When `NONINTERACTIVE=0`, the skill may fall back to asking — but the default assumption for Trellis is unattended.

---

## 2. Environment bootstrap

Cron shells are minimal and do not source your profile. Resolve every required variable in this order, then verify — do not assume the caller exported anything:

```bash
# 1. PROJECT_ROOT: explicit env, else the git repo root, else cwd
PROJECT_ROOT="${PROJECT_ROOT:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"

# 2. Load a project .env if present (does not override already-set vars)
if [ -f "$PROJECT_ROOT/.env" ]; then
  set -a
  # shellcheck disable=SC1090
  . "$PROJECT_ROOT/.env"
  set +a
fi

# 3. Verify each required var for THIS sub-command (see table below). Example:
: "${LINEAR_API_KEY:?LINEAR_API_KEY is required — set it in the environment or $PROJECT_ROOT/.env}"
```

In non-interactive mode a missing required variable is a **fail fast**: log it and exit non-zero. Never prompt.

| Sub-command | Required env vars |
|---|---|
| `research-backlog` | `LINEAR_API_KEY`, `LINEAR_TEAM_NAME` |
| `dev-backlog` | `LINEAR_API_KEY`, `LINEAR_TEAM_NAME`, `GH_TOKEN` |
| `bulk-pr-review` | `GH_TOKEN` |
| `fix-pr-comments` | `GH_TOKEN` |

`LINEAR_TEAM_NAME` is required for cron because there is no human to pick a team from a list.

---

## 3. Run log (durable output)

Nobody watches a cron run live. Every run writes a structured summary that survives it. Print the same summary to stdout so it also lands in cron mail / CI logs.

```bash
TRELLIS_HOME="${TRELLIS_HOME:-$PROJECT_ROOT/.trellis}"
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)"
LOG_DIR="$TRELLIS_HOME/runs"
mkdir -p "$LOG_DIR"
RUN_LOG="$LOG_DIR/<sub-command>-$RUN_ID.md"
```

Write the same summary table the skill would have shown a user to `$RUN_LOG` — plus a first line recording the outcome: `RESULT: ok | partial | failed`, item counts, and the exit code. Results that already go to a durable place (Linear comments, GitHub reviews) still go there; the run log is the local record of what the run *did*, including skips and failures.

Add `.trellis/` to the target repo's `.gitignore` — run logs and locks are local artifacts, not source.

---

## 4. Concurrency lock

A run that overruns its schedule must not collide with the next fire. Take a per-sub-command lock before doing any work:

```bash
LOCK_DIR="$TRELLIS_HOME/locks"
mkdir -p "$LOCK_DIR"
LOCK="$LOCK_DIR/<sub-command>.lock"
LOCK_TTL_SECONDS="${TRELLIS_LOCK_TTL:-21600}"   # 6h default

if [ -f "$LOCK" ]; then
  LOCK_AGE=$(( $(date +%s) - $(stat -f %m "$LOCK" 2>/dev/null || stat -c %Y "$LOCK") ))
  if [ "$LOCK_AGE" -lt "$LOCK_TTL_SECONDS" ]; then
    echo "RESULT: skipped — another <sub-command> run holds the lock (age ${LOCK_AGE}s). Exiting." | tee -a "$RUN_LOG"
    exit 0   # not a failure; the other run is handling it
  fi
  echo "Stale lock (age ${LOCK_AGE}s > TTL). Reclaiming." | tee -a "$RUN_LOG"
fi

echo "pid=$$ started=$RUN_ID" > "$LOCK"
trap 'rm -f "$LOCK"' EXIT
```

A live lock is not an error — exit 0, let the running job finish. Only reclaim locks older than the TTL.

---

## 5. Idempotency markers

Re-running a sub-command must not duplicate work. Every artifact Trellis creates carries a machine-detectable marker on its first line, and every sub-command checks for that marker before acting.

**Marker format** (HTML comment — invisible in rendered Linear/GitHub, greppable in the API response):

```
<!-- trellis:<sub-command> v1 -->
```

Put it as the first line of every Linear comment and every GitHub review/comment body Trellis authors.

**Per-sub-command idempotency guard:**

| Sub-command | Skip an item when… |
|---|---|
| `research-backlog` | the ticket already has a comment starting with `<!-- trellis:research-backlog v1 -->` |
| `dev-backlog` | the ticket is already in a non-`unstarted` state, or an open PR already links its identifier |
| `bulk-pr-review` | the PR already has a Trellis review whose body records the current head SHA |
| `fix-pr-comments` | every review thread is already resolved, or the only open threads were replied to at the current head SHA |

Skipped-as-already-done items are logged (with the reason) and counted, not treated as failures.

---

## 6. Exit signaling

Cron and CI decide success by exit code. Use this convention consistently:

| Exit | Meaning |
|---|---|
| `0` | Clean run. Includes runs where every item was skipped (already done, lock held) — nothing went wrong. |
| non-zero | Hard failure that stopped the run: missing required env var, dirty working tree, unrecoverable API/auth error, or repo sync failure. |

A single work item failing (one ticket, one PR) is **skip-and-log**, not a non-zero exit — it must not abort the batch or fail the whole cron job. Reserve non-zero for whole-run blockers. Always print `RESULT:` as the final line so log scrapers can classify the run without parsing the exit code.

---

## 7. Skill startup checklist

Every cron-safe sub-command runs this preamble before its own steps:

1. Resolve mode (§1) and env (§2); fail fast on missing required vars.
2. Open the run log (§3).
3. Acquire the lock (§4); exit 0 if a live lock is held.
4. Do the work — applying the idempotency guard (§5) to skip already-done items and skip-and-log to survive per-item failures.
5. Write the summary and final `RESULT:` line to the log and stdout (§3, §6); release the lock via the `EXIT` trap.
