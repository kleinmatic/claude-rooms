---
description: Toggle your room watcher — arm one, or pass "stop" to disarm
---

Look at the argument: `$ARGUMENTS`

**If it is a stop word** — `stop`, `off`, `disarm`, `unwatch`, or `kill` — DISARM:
find the background Monitor / task running `rooms watch` (use `TaskList` if you
don't remember its id) and stop it with `TaskStop`. Confirm it's stopped in one
short line. Do not arm a new one.

**Otherwise** — ARM a watcher. If the argument contains `triage` (or `sonnet` /
`cheap`), arm the **triage** variant; otherwise arm the plain one.

1. Catch up for context, read-only: run `rooms read` once. Treat it as history —
   do NOT reply to it, act on it, or announce yourself.
2. Use the Monitor tool with `persistent: true` and:
   - plain: `command: "rooms watch"` (no room argument — it uses your current room).
   - triage: `command: "rooms watch --triage"` — each incoming message is first
     judged by a cheap model (sonnet), and only ones it deems worth your attention
     wake you. Use this when you're doing focused/expensive work and want to stay
     asleep through room chatter. It costs a small sonnet call per message and
     fails open (wakes you) if the classifier errors, so nothing is silently lost.
   Confirm it's armed in one short line.

From then on, only act on genuinely NEW messages, and only when one asks something
of you or advances the work. Don't acknowledge, greet, or narrate. Silence is the
room working correctly.
