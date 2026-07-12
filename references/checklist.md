# Pre-Launch Loop Checklist

After the design is done, tick every item. Any ✗ means don't ship.

## Mechanism and Scheduling
- [ ] Mechanism chosen correctly: overnight → OS cron; user present → /loop; pure network → cloud routine; waiting for an event → Monitor (not a loop)
- [ ] Occupies only the user's dead time; no shifts in live time (unless the user says so explicitly)
- [ ] Adjacent shifts are spaced ≥ one round's expected duration apart; minute field avoids :00 / :30
- [ ] The machine doesn't sleep (`pmset -g` → sleep 0) or has a caffeinate/launchd fallback
- [ ] Runner and prompts are outside iCloud's sync scope (a HOME-level dir like ~/automation, not ~/Documents — "Optimize Mac Storage" can evict them into `.icloud` placeholders overnight); if a project working directory is inside iCloud, the wrapper materializes it with `brctl download` before starting
- [ ] The Distill wrap-up shift lands its human-readable report before the user's next live window

## Wrapper Script
- [ ] Exports `PATH` itself (the cron environment is bare)
- [ ] Date guard present, cutoff date correct
- [ ] One `mkdir` lock per loop + stale-lock cleanup (timeout threshold ≥ 2× one round's duration)
- [ ] Changes into the correct project working directory per loop
- [ ] Log is per-round with timestamps; a `claude` failure doesn't blow up the script (wrapped in `if/else`), and failures are still logged
- [ ] `zsh -n` syntax check passes

## Prompt (each of the eight rules)
- [ ] ① Opens by declaring: unattended, cannot ask questions, decisions are skipped and logged
- [ ] ② Per-round cap (at most N)
- [ ] ③ Gate (commit only when tests are all green; on failure, revert + log)
- [ ] ④ Idempotent (switch to the branch if it exists; append to the report if it exists; leftover WIP has handling instructions)
- [ ] ⑤ No-go checklist (don't push / don't touch main / don't install deps / project-specific no-go / no outward actions)
- [ ] ⑥ Must write a report at the end; reader side names the handoff/queue file
- [ ] ⑦ Report first: step one creates today's report file, appending as you go (never "write the report all at once at the end")
- [ ] ⑧ No wrap-up-time background fan-out: audit-type subtasks run synchronously; if you truly need background, set `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS` in the wrapper (headless -p force-kills at 600s by default)
- [ ] Interruption self-rescue clause: on losing file access / hitting a rate limit, write done / not-done / leftover-risks to /tmp/<date>-<loop>-draft.md + project memory, then exit
- [ ] Has a "exit if there's nothing to do" permission
- [ ] All paths in the prompt are absolute or explicit `~` paths; dates are natural language ("name it with today's date"), not `$(date)` (which won't expand when the prompt is `cat`ed in)

## Safety
- [ ] Git isolation: a dedicated `claude/*` branch; confirmed the worst case is recoverable by deleting the branch
- [ ] No outward actions (email / tweet / push to a remote / sending messages)
- [ ] Collect-type is append-only, doesn't overwrite history
- [ ] Sensitive files (e.g. a user profile) can't be read by the loop and then written into a public artifact
- [ ] Pre-authorize TCC: the shell that runs cron and the `claude` binary have Full Disk Access added (TCC can revoke access to ~/Documents mid-run, and a headless session has nobody to click the dialog)

## Verification and Wrap-up
- [ ] Smoke test passes (chain: PATH → lock → headless invocation → log)
- [ ] Cron login state verified: run the smoke with a real cron entry (a one-shot entry two minutes out) or `env -i`, confirming headless `claude` has credentials — passing in an interactive shell ≠ passing in the cron environment
- [ ] Catch-up rule known: manual re-runs of missed shifts go serially at the original staggered interval, not several headless sessions at once
- [ ] The crontab install script is idempotent (grep marker prevents re-installing); existing entries are untouched
- [ ] The user knows where the human-readable exit is (report path) + acceptance is scheduled after the first cycle
- [ ] Wrap-up plan is clear: on the cutoff date delete the schedule block, review the branches; the last Distill loop will remind

## Common Failure Quick Reference (after launch)
| Symptom | Check first |
|---|---|
| No shift ran all night | `pmset -g` sleep; is `crontab -l` still there; script execute permission; **macOS TCC** (see next row) |
| Cron mail reports `can't open input file` (the script clearly exists) | macOS TCC: `/usr/sbin/cron` lacks Full Disk Access and can't read anything under ~/Documents, ~/Downloads, ~/Desktop. Fix: System Settings → Privacy & Security → Full Disk Access → add `/usr/sbin/cron` (⌘⇧G to type the path). **Launch check must include: after installing crontab, test the real cron environment with a one-shot entry two minutes out** — a manual `zsh run-loop.sh smoke` passing ≠ the cron environment passing (a manual run inherits the terminal's TCC permissions!) |
| A shift was skipped | Log says "skipped: still running" → the previous shift ran over; check whether stale-lock cleanup kicked in |
| `claude` command not found | The wrapper's PATH; compare `which claude` against the cron environment |
| One round ran away and changed a pile of things | The prompt lacks a per-round cap or a gate → fill it in against the eight rules |
| Morning report is empty/fabricated | The Distill prompt lacks a concrete standard for "what's worth keeping" + lacks the "don't force output" permission |
| Exited non-zero | Usually a rate limit (5h window); the raw error is above it in the log; the next shift retries automatically |
| Log reports "Not logged in · Please run /login" | The cron environment has no credentials (this happens even when the interactive shell is fine); log in from the interactive shell, then re-test with a real cron entry |
| Script/prompt turned into a `.icloud` placeholder | iCloud "Optimize Mac Storage" evicted it overnight; move the runner to ~/automation, materialize the project dir with `brctl download` |
| Mid-session EPERM on all of ~/Documents | TCC revoked at runtime; grant Full Disk Access to the shell/claude binary; the prompt's interruption self-rescue clause backstops the half-finished scene |
| Background tasks killed ("Background tasks still running after 600s") | headless -p defaults to a 600s cap; run audit subtasks synchronously or set `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS` |
