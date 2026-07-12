---
name: loop-design
version: "1.1.0"
description: >
  Loop engineering skill: turn a recurring intent into a deployed, safe, self-expiring
  automation loop (headless Claude/Codex sessions on a schedule). Use for ANY of:
  designing recurring automation ("design a loop", "run X automatically every night",
  "scheduled job", "advance a project overnight", "burn quota", "automate this workflow");
  choosing the mechanism ("cron or /loop", "session-only or persistent", "cloud routine or local");
  writing unattended prompts ("how do I write a headless prompt", "unattended", "is auto-commit safe");
  scheduling ("how do I schedule it", "avoid rate limits", "run while I sleep");
  reviewing/debugging an existing loop ("the loop didn't run", "the morning report is low quality",
  "it deadlocked", "review the loop's output").
  Also triggers on: cron, crontab, launchd, nightly, scheduled, polling, unattended, headless.
allowed-tools: Bash, Read, Write, Edit, AskUserQuestion
user-invocable: true
argument-hint: 'loop-design fix issues automatically every night | loop-design help me figure out why last night''s loop did not run | loop-design what mechanism should this task use'
---

# Loop Design — Unattended Automation Design Skill

> Reference implementation: `~/automation/` (a nightly loop system:
> run-loop.sh + prompts/ + logs/ + reports/). Keep runners and prompts outside
> iCloud-synced folders (see §6 for why). When designing a new loop, prefer reusing
> this pattern over building from scratch.

## 1. What This Skill Does

Turn "I want an AI to do X regularly/repeatedly" into a loop system that is **bounded, gated,
self-expiring, and has a clear exit**. It produces four artifacts: a mechanism decision → a wrapper
script → one prompt file per loop → a schedule (crontab or equivalent).

## 2. Run Protocol

```
1. Classify   — Which kind of use is this? (§3 five types). Split mixed cases into separate loops.
2. Pick a mechanism — Choose one of the four in §4; the wrong mechanism is the most expensive mistake.
3. Map the clock    — First mark the user's unavailable windows (sleep/work); loops only fill dead
                      time. Leave live time for the user's own interactive work (or you fight them for quota).
4. Write the prompt — One standalone prompt file per loop; self-check against each of the 8 rules in §5.
5. Add safety layers — date guard + one lock per loop (with stale-lock cleanup) + git isolation (§6).
6. Schedule   — Interval ≥ one round's duration; avoid the top/half of the hour; put wrap-up loops last,
                so their human-readable report lands before the user's next live window starts.
7. Smoke      — Validate the whole chain with a dummy prompt that only replies once
                (PATH/logging/lock/headless invocation). Only ship the real task after it passes.
                Template in references/templates.md.
8. Acceptance — After the first cycle, read the morning report/logs by hand. Feed lessons back (see §7).
```

Templates and checklist: [references/templates.md](references/templates.md) · [references/checklist.md](references/checklist.md)

## 3. Five Types of Use (classify before designing)

| Type | Does what | Precondition | Example |
|---|---|---|---|
| Advance | Chews through a backlog, 1-2 items per round | A task queue exists (issue dir / failing tests / TODOs) | Work through an issue backlog |
| Collect | Periodically scans the outside world, appends signals (append-only) | A watchlist exists; never overwrite history | Scan RSS/API signal sources |
| Maintain | Fights entropy: stale docs / drift / cleanup | A clear "desired state" to compare against | Rewrite drifted docs |
| Distill | Reads other loops' output, extracts lessons back into skills/docs | Other loops already produce structured reports | Distill last night's output |
| Guard | Short-interval polling of state; acts/notifies on change | State can be judged programmatically | Watch CI / deploy / a price |

**Composition rule**: a healthy loop system = a few Advance loops + one Distill loop to wrap up. Advance
without Distill and you lose every lesson; Collect without Advance and you're hoarding intel, not working.

## 4. Four Mechanisms (selection decision table)

| Mechanism | Lifetime | Choose it if and only if |
|---|---|---|
| OS cron/launchd + `claude -p`/`codex exec` | Machine-level, persistent | Runs overnight/across days, unattended. **Default choice.** |
| `/loop` (in-session self-loop) | Current session | The user is present; short-cycle repetition until some condition is met |
| CronCreate (in-session cron) | Current session, 7-day cap | Only "do X periodically while this session happens to be open" |
| Cloud routine (`/schedule`) | Anthropic cloud | A pure network task that never touches local files |

Distinguish: **Monitor is not a loop** — use Monitor to wait for an event (event-driven); use a loop only
to do something periodically (time-driven).

Common mismatches: an overnight task on a session-only mechanism (close the terminal and it all dies);
a task that edits local code on a cloud routine (it can't touch your files).

## 5. The Eight Prompt Rules (self-check each one)

A headless session has no memory, no user, and cannot ask questions. The prompt must stand in for the
user's presence:

1. **Declare the situation** — Open with: "You are an unattended headless session. You cannot ask
   questions. Skip and log anything that needs a user decision."
2. **Bound each round** — "At most N per round." An unbounded loop runs away for a round, burns quota,
   and produces half-finished work. Prefer many small rounds.
3. **Set a gate** — "Commit only when tests are all green; if you can't get to green, revert and log why."
   Auto-commit with no gate = dumping garbage into the repo.
4. **Idempotent and re-entrant** — Assume the previous round may not have finished: "switch to the branch
   if it exists, else create it," "append to the report if it exists," "inspect leftover WIP before
   touching it."
5. **Draw the no-go zones** — Write them as a checklist (don't push / don't touch main / don't install
   dependencies / no placeholder features). Don't count on it to "get the spirit."
6. **Files are memory** — Every round must write a report / update the handoff; the next session starts
   fresh, so the baton is a file, not a conversation.
7. **Report first** — Step one is to create today's report file and append as you go; never "write the
   report all at once at the end." When a session is force-killed or interrupted (see TCC in §6), any
   report written late is lost — the only remaining record is the wrapper log.
8. **No wrap-up-time background fan-out** — In headless `-p` mode, background tasks are force-killed after
   600s by default ("Background tasks still running after 600s; terminating"), losing all results. Require
   audit-type subtasks to run synchronously, or set `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS` in the wrapper.

Soft rule: give it permission to "exit if there's nothing to do" ("don't force output if nothing is
genuinely worth keeping"), or it will manufacture busywork.

**Interruption self-rescue clause (standard in every prompt)**: "If you lose file access or get cut off by
a rate limit mid-round, immediately write done / not-done / workspace-leftover-risks to
`/tmp/<date>-<loop>-draft.md` and to project memory, then exit." This is what lets a session that gets cut
off mid-task leave a flawed, half-finished diff marked as a scene someone can safely pick up.

## 6. Safety Layers (three fallbacks, all required)

Headless must run with `--dangerously-skip-permissions`; safety is restored elsewhere:

- **Prompt no-go checklist** (layer 1, stops most things)
- **Git isolation**: a dedicated `claude/*` branch per project, commit-only, never push, never touch main —
  worst case is deleting one branch (layer 2)
- **Date guard**: first line of the script `[[ "$(date +%Y%m%d)" -le <cutoff> ]] || exit 0` — any
  "temporary" automation needs an expiry date, because temporary things most easily become permanent (layer 3)

Plus runtime protections: one `mkdir` lock per loop to prevent overlapping runs + stale-lock cleanup on
timeout (4h); the cron environment is bare, so the script must `export PATH` itself.

Environment-level protections (hard-won operational lessons):

- **Runners outside iCloud**: `~/Documents` is governed by "Optimize Mac Storage"; when disk is tight, a
  whole directory can be evicted overnight into `.icloud` placeholders (this can wipe out an entire night's
  runs). Put runners/prompts in a HOME-level directory like `~/automation`; if a project working directory
  lives inside iCloud, materialize it with `brctl download` before starting as a fallback.
- **Pre-authorize TCC**: a headless session has nobody to click the authorization dialog, and macOS TCC can
  revoke access to `~/Documents` mid-run (EPERM across the whole process tree). Before going live, grant
  Full Disk Access to both the shell that runs cron and the `claude` binary.
- **Cron login state**: `claude` working in an interactive shell ≠ the cron environment having credentials
  ("Not logged in · Please run /login"). The smoke test must run once from a real cron entry (or `env -i`).

## 7. Exit and Wrap-up (decide at design time)

- **Human-readable exit**: every loop system must have one periodic report meant for a human (morning/weekly),
  scheduled on the wrap-up shift, landing before the user's next live window. A loop system with no exit =
  it runs silently for three days and you don't know if it ran.
- **Separate log from report**: the log is raw output (for debugging, per-round with timestamps); the report
  is structured output (for humans and Distill loops to read).
- **Wrap-up actions**: after the date guard expires, delete the schedule entries and review/merge/delete the
  loop branches; the last Distill loop's prompt should include "remind the user to wrap up on the cutoff date."
- **Catch-up runs stay staggered too**: when manually re-running a missed shift, run serially at the original
  crontab's staggered interval — don't fire up several headless sessions at once (concurrent catch-up runs
  amplify rate-limit and resource-contention risk, and can trigger TCC revocation).
- **Feed lessons back**: pits you discover after going live (prompt not specific enough, a missing gate, a
  scheduling collision) go back into this skill's references/ or the relevant prompt file — the loop system
  itself must also be distilled.

## 8. Prohibitions

- Don't write unbounded loops (an Advance prompt with no per-round cap is not acceptable)
- Don't schedule loops in the user's live time unless the user says so explicitly
- Don't let a headless session take outward actions (send email / post a tweet / push to a remote) —
  outward actions are always left to a human
- Don't skip the smoke test and ship directly
- Don't put "temporary" schedules into crontab without a date guard
- Collect-type loops don't overwrite historical signals, only append
- Don't gloss over anything in reports: rate limits, skipped locks, red tests — write them all honestly
  into the human-readable exit

## 9. Clarifying Questions

When the request is vague, ask first:
1. "Is this an Advance / Collect / Maintain / Distill / Guard task? Where's the task queue or watchlist?"
2. "What are your unavailable windows (sleep/work)? What times should the loop run?"
3. "What level of write permission do the automated changes get — read-only reports / commit to a branch /
   something else?"
4. "When should this loop die? (a cutoff date or an exit condition)"
