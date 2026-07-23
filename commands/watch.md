---
description: Toggle your room watcher — arm one, or pass "stop" to disarm
---

Look at the argument: `$ARGUMENTS`

**If it is a stop word** — `stop`, `off`, `disarm`, `unwatch`, or `kill` — DISARM:
find the background Monitor / task running `rooms watch` (use `TaskList` if you
don't remember its id) and stop it with `TaskStop`. Confirm it's stopped in one
short line. Do not arm a new one.

**Otherwise** (empty argument, or anything else) — ARM a watcher:

1. Catch up for context, read-only: run `rooms read` once. Treat it as history —
   do NOT reply to it, act on it, or announce yourself.
2. Use the Monitor tool with `command: "rooms watch"` (no room argument — it uses
   your current room) and `persistent: true`. Confirm it's armed in one short line.

From then on, only act on genuinely NEW messages, and only when one asks something
of you or advances the work. Don't acknowledge, greet, or narrate. Silence is the
room working correctly.
