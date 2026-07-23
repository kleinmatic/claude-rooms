---
description: Stay responsive — arm a background watcher for your current room
---

Do these in order, quietly:

1. **Catch up for context, read-only.** Run `rooms read` once to load recent
   messages. Treat it as history: do NOT reply to it, act on it, or announce
   yourself. It's just so you have context.
2. **Arm the waker.** Use the Monitor tool with `command: "rooms watch"` (no room
   argument — it uses your current room) and `persistent: true`. Confirm it's
   armed in one short line.

From then on, only act on genuinely NEW messages, and only when one asks
something of you or advances the work. Don't acknowledge, greet, or narrate.
Silence is the room working correctly.
