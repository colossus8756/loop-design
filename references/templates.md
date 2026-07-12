# Loop Template Library

Reference implementation: `~/automation/` (copy and adapt beats writing from scratch). Keep it in a
HOME-level directory, not inside an iCloud-synced folder like ~/Documents (see the wrapper comments below).

## 1. Wrapper Script Template (run-loop.sh)

The key structure — five parts, none optional:

```zsh
#!/bin/zsh
set -euo pipefail
export PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:$PATH"   # (1) the cron environment is bare

DIR="$HOME/automation"                           # must be outside iCloud sync range; don't put it in ~/Documents
LOOP="${1:?usage: run-loop.sh <loop-name>}"
LOG="$DIR/logs/$(date '+%Y-%m-%d')-${LOOP}.log"

[[ "$(date +%Y%m%d)" -le 20260712 ]] || exit 0   # (2) date guard: change to your cutoff

PROMPT_FILE="$DIR/prompts/${LOOP}.md"
[[ -f "$PROMPT_FILE" ]] || { echo "[$(date '+%F %T')] ERROR: no prompt $PROMPT_FILE" >> "$LOG"; exit 1; }

LOCK="$DIR/.lock-${LOOP}"                         # (3) one lock per loop + stale-lock cleanup
if [[ -d "$LOCK" && -n "$(find "$LOCK" -maxdepth 0 -mmin +240 2>/dev/null)" ]]; then
  rmdir "$LOCK" 2>/dev/null || true
  echo "[$(date '+%F %T')] stale lock cleared" >> "$LOG"
fi
mkdir "$LOCK" 2>/dev/null || { echo "[$(date '+%F %T')] skipped: still running" >> "$LOG"; exit 0; }
trap 'rmdir "$LOCK" 2>/dev/null || true' EXIT

case "$LOOP" in                                  # (4) start each loop in the correct project dir
  myproject) WORKDIR="$HOME/path/to/project" ;;
  *)         WORKDIR="$HOME/Documents/research" ;;
esac
if [[ "$WORKDIR" == "$HOME/Documents/"* ]]; then # iCloud materialize fallback: guard against overnight "Optimize Storage" eviction
  brctl download "$WORKDIR" 2>/dev/null || true
fi
cd "$WORKDIR"

# export CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS=1800000   # optional: headless -p force-kills background tasks at 600s by default

echo "[$(date '+%F %T')] $LOOP start (workdir=$WORKDIR)" >> "$LOG"   # (5) per-round, timestamped log
if claude -p "$(cat "$PROMPT_FILE")" --dangerously-skip-permissions >> "$LOG" 2>&1; then
  echo "[$(date '+%F %T')] $LOOP done" >> "$LOG"
else
  echo "[$(date '+%F %T')] $LOOP exited non-zero (rate limit or error, see above)" >> "$LOG"
fi
```

The engine is swappable: `codex exec --skip-git-repo-check "$PROMPT"` (see a dual-engine wrapper for the pattern).

## 2. Unattended Prompt Template (mapped to the eight rules, section by section)

```markdown
You are a headless session running unattended [overnight / early morning]. You cannot ask the user
questions; skip and log anything that needs a user decision. The current working directory is <project path>.   ← Rule 1: declare the situation

Task: <one sentence>. Complete at most <N> <units> this round.                                                  ← Rule 2: bounded

Steps:
1. First create a report file named with today's date under <reports path> (append if it exists),
   then append progress after each step — don't wait until the end to write.                                    ← Rule 7: report first
2. Read <handoff/queue file> to understand the current state.                                                   ← Rule 6: files are memory (reader side)
3. git: switch to branch <claude/xxx> if it exists, else create it from HEAD;
   inspect any leftover uncommitted changes first, and if reasonable commit them as "wip: leftover from last round". ← Rule 4: idempotent, re-entrant
4. <core work>. Run all audit/check-type subtasks synchronously; do not dispatch background agents.             ← Rule 8: no background fan-out
5. Per unit: complete → run <test command>; commit only when all green (one commit per unit);
   if you can't get to green, revert that change and log the reason in the report.                              ← Rule 3: gate
6. Update the handoff/status file per project convention, also committed to this branch.

No-go: never push, don't touch main, don't install new dependencies, <project-specific no-go>.                  ← Rule 5: no-go checklist
If there's nothing genuinely worth doing, just exit — don't manufacture busywork.                               ← soft permission
If you lose file access / get cut off by a rate limit mid-round: immediately write done / not-done /
workspace-leftover-risks to /tmp/<today's date>-<loop name>-draft.md and to project memory, then exit.          ← interruption self-rescue

Before finishing: confirm the report file contains what you did, commit hashes, test status, and pitfalls.      ← Rule 6: files are memory (writer side)
Your final reply only needs to be a one-line summary.
```

## 3. Smoke Prompt (must run before launch)

```markdown
This is a smoke test. Reply with exactly one line: "smoke ok, headless claude is working." Do nothing else.
```

Verify: `zsh run-loop.sh smoke && cat logs/$(date +%F)-smoke.log` — seeing the reply plus the two
start/done timestamps means it passed.

## 4. Crontab Block Template

```cron
# === <system name> loops (date-guard expires on <cutoff>, auto-disables; delete this block afterward) ===
7  0 * * *   /abs/path/run-loop.sh <AdvanceA>     # avoid the top of the hour: 07/09/11/23…
9  2 * * *   /abs/path/run-loop.sh <AdvanceB>     # interval ≥ one round's duration (rule of thumb: 2h)
50 6 * * *   /abs/path/run-loop.sh <Distill>      # wrap-up shift; report lands before the user's live window
# === <system name> loops end ===
```

Install with an idempotent script (grep for a marker first to prevent re-installing; see install-cron.sh).
Note: in auto mode Claude can't modify its own crontab or launch a skip-permissions process — after
generating the install script, have the user run `zsh install-cron.sh` by hand.

## 5. Scheduling Cheat Sheet

- Dead time first: sleep 00-08, work hours; don't schedule in live time
- Spread across the 5h rate-limit window: stagger adjacent shifts ≥ 2h apart
- Avoid :00 / :30 in the minute field
- Put the Distill loop on the last shift; run Collect loops at low frequency (if the signal is on a 30-day scale, don't scan it daily)
- macOS prerequisite: confirm `sleep 0` via `pmset -g` (no sleep), or cron won't wake overnight
