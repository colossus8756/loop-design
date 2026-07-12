# loop-design

Turn a recurring intention into a safe, self-expiring unattended automation loop for headless
Claude/Codex sessions.

This is a [Claude Code](https://docs.claude.com/en/docs/claude-code) skill. It gives you a repeatable
design protocol for building automation loops that run while you sleep — and that fail loudly, expire on
their own, and can never quietly wreck your repo.

## What is a Claude Code skill?

A skill is a small folder of Markdown instructions that Claude Code loads on demand. When your request
matches the skill's description, Claude follows its guidance automatically — or you can invoke it by name
with a slash command. Think of it as a reusable playbook you teach the assistant once and reuse forever.

## What problem this solves

Ad-hoc cron jobs that call an AI break silently. The prompt was unbounded so one run went off the rails;
the cron environment had no `PATH` so `claude` was never found; the "temporary" job outlived its purpose by
six months; a half-finished commit landed on `main`. You only find out days later — if at all.

`loop-design` replaces guesswork with a protocol. Every loop goes through the same pipeline:

**mechanism choice → wrapper script → unattended prompt → schedule → safety layers.**

The result is a loop that is bounded (a per-round cap), gated (commit only when tests pass), isolated (a
dedicated git branch that is never pushed), and self-expiring (a date guard that shuts it off on a cutoff
date). Plus the hard-won operational lessons that only show up at 3am on the first night live.

## Install

Clone this repo into your Claude Code skills directory:

```bash
git clone <this-repo-url> ~/.claude/skills/loop-design
```

It's then available as a slash command:

```
/loop-design
```

Codex users can share the same skill by symlinking it into their skills directory:

```bash
ln -s ~/.claude/skills/loop-design ~/.codex/skills/loop-design
```

## How to use

Invoke it with what you want to automate, and it will walk you through classifying the task, picking a
mechanism, writing the prompt, and adding the safety layers:

```
/loop-design fix issues automatically every night
```

```
/loop-design help me figure out why last night's loop did not run
```

```
/loop-design what mechanism should this task use
```

You can also just describe a recurring intention in plain language ("scan these RSS feeds every morning and
append new items", "advance this refactor overnight") and the skill triggers on its own.

## What's inside

| File | What it is |
|---|---|
| `SKILL.md` | The core protocol: the 8-step run protocol, five loop types, four mechanisms, the eight prompt rules, three safety layers, and wrap-up/exit design. |
| `references/checklist.md` | A pre-launch checklist to tick before shipping a loop, plus a "common failure quick reference" for debugging one after launch. |
| `references/templates.md` | Ready-to-copy templates: a wrapper script, an unattended prompt, a smoke-test prompt, a crontab block, and a scheduling cheat sheet. |

## Platform notes

The operational lessons are macOS-flavored (launchd, TCC Full Disk Access, iCloud "Optimize Mac Storage"
eviction). But the underlying principles apply to any OS:

- **Date guards** — every temporary automation needs an expiry date.
- **Git isolation** — a dedicated branch that is never pushed, so the worst case is deleting a branch.
- **Locks** — prevent overlapping runs and clean up stale locks.
- **PATH in cron** — the cron environment is bare; export `PATH` in the wrapper.

Linux users use plain `cron` instead of `launchd`, and can ignore the TCC/iCloud sections — but everything
about bounding prompts, gating commits, staggering the schedule, and self-expiry applies unchanged.

## Safety first

Unattended automation is powerful and dangerous. A headless session runs with permission prompts disabled,
so **you** are responsible for its guardrails. Always use the three safety layers:

1. **Prompt prohibitions** — an explicit no-go checklist in the prompt itself.
2. **A dedicated git branch that is never pushed** — commit-only, never touch `main`.
3. **A date guard** — the loop shuts itself off on a cutoff date.

And the golden rule: **never let a headless session take outward actions** — no sending email, no posting
tweets, no pushing to a remote. Outward actions are always left to a human.

## License

MIT — see [LICENSE](LICENSE).
