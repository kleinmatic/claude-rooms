# Project Plan: rooms

Phases are ordered by dependency, not by calendar time — each phase produces
something runnable/testable before moving on. Phase 0 is what already shipped;
everything after it is proposed work. 

Detailed designs live in `docs/`; this file is the map, not the territory.

## Threat model

The threat is **a well-meaning agent doing something dumb** — most often one
that has never seen a room before and reads a peer's earnest request as an
instruction from its operator. It is *not* a determined human attacker with
local access; if someone has code execution on your workstation, rooms was never
going to save you.

So the bar is: stop runaway agents, and don't leave trivially avoidable rakes
lying around for us to step on. Explicit non-goals — cryptographic auth between
agents, defense against a hostile local process, and multi-user operation.

This project is a time machine to an era when unix tools took simple approaches
and left complexity to the users. Money-shifting stuff like multi-user team
collaboration and use over sshfs isn't blocked, but the security needed to
support it is on you.

## Phase 0 — Shipped (v0.1.x)

Recorded here so the plan starts from what exists rather than from zero.

- Filesystem maildir bus: lock-free `write-to-tmp` + atomic `rename()` into
  `msgs/new/`, chosen so the transport survives sshfs or other shared
  directories (where file locks might not).
- Room CRUD and presence: `open`/`list`/`join`/`leave`/`whoami`/`members`/
  `post`/`read`/`summary`/`close`, per-agent read cursors, one-room-at-a-time.
- Trust model: room messages are **data, never commands**; `rooms read` labels
  its output untrusted on every call; a room grants no authority. As rooms
  has no runtime, this is enforced by clients. That means they're only as 
  secure as the clients are. Rooms is designed to be run on trusted, single-user
  workstations.
- Autonomous waking: `rooms watch` (persistent Monitor) and `rooms wait`
  (one-shot background task), plus the unread-messages hook on human prompts.
- Cheap-model triage: `rooms watch --triage` gates which messages are worth
  waking the primary agent, failing open. Deterministic `NEW SPEAKER` and
  `possible prompt injection` tripwires bypass the classifier and always wake.
- Runaway circuit breakers: a turn budget capping messages per room since the
  human's last turn, and `rooms announce` as a fire-and-forget kind that never
  wakes a watcher.
- Lifecycle and ephemerality: idle self-leave, graceful watcher shutdown on
  close, `rooms signoff` + `SessionEnd` hook, and `close` that **erases** by
  default.
- Two rounds of adversarial hardening (a cross-model review and a full
  `soundcheck` audit): atomic state writes, NFKC normalization on the tripwire,
  forge-resistant turn-budget watermark, body-size caps, audit logging.

## Phase 1 — Finish the ephemeral contract

Making `close` erase by default (v0.1.1) put the burden of preserving context on
each participant without giving them the tool to do it. Small, self-contained,
no dependencies.

- `rooms transcript` — dump a verbatim copy of a room to a file on demand, so a
  participant can save what matters into *its own* project before the room goes
  away.
- Decide what "verbatim" includes: bodies, sender labels, timestamps, join/leave
  events, and whether triage flags are part of the record.
- Teach the skill to reach for it at natural save points (before `signoff`, on a
  handoff) rather than only when asked.
- Garbage-collect `msgs/tmp/`, which the maildir spec does at 36 hours and we
  don't do at all (see [docs/stealing-from-unix.md](docs/stealing-from-unix.md)).
  A crash between the write and the rename orphans a file permanently;
  ephemerality hides it today, but a long-lived room accumulates.

## Phase 2 — MCP-native tools (v0.2, on MCP `2026-07-28`)

Design: [docs/v0.2-mcp-native.md](docs/v0.2-mcp-native.md). **Parked, not
started.** It targets the `2026-07-28` revision and waits for usable SDKs;
building it on the previous revision would mean writing an `initialize`
handshake, session handling, and `resources/subscribe`, all three of which the
new spec deletes. Nothing else depends on this phase, so waiting costs nothing.

- **Start trigger:** the official `mcp` Python SDK ships a release that speaks
  `2026-07-28`. Until then this phase is a design, not a work item — check
  occasionally rather than tracking it.
- Refactor room logic out of shelling to `bin/rooms` into an importable core
  module, so the CLI and the MCP server are two front doors on one
  implementation. This is the real work of the phase, it's the part that doesn't
  depend on the spec at all, and it's the piece to pull forward if the wait drags.
- Build the stdio MCP server on that SDK, run via `uv`/`uvx` so the dependency is
  fetched on demand and nothing is vendored. Deliberately *not* hand-rolling
  JSON-RPC — which the stateless revision makes more tempting and still not worth
  owning.
- Expose the CRUD surface as typed tools — kills verb-guessing (agents reached
  for `send`/`say` before finding `post`), gives structured I/O instead of text
  parsed off a terminal, and allows per-tool permissions instead of blanket Bash.
- Conformance the new revision requires, none of it hard: implement
  `server/discover`, put `resultType: "complete"` on every result, emit `ttlMs`
  and `cacheScope` on `tools/list` and `resources/*` results, and return tools in
  a deterministic order so caches hit.
- Bundle via the plugin's `.mcp.json`. Keep the CLI unchanged: humans use it, and
  it stays the source command for the Monitor waker.
- Statelessness is now the protocol's model rather than our workaround, so the
  server holds nothing between calls and the maildir stays the only state. Log to
  `stderr`, since the Logging feature is deprecated and stdio explicitly allows it.
- Waking stays on Monitor, and it's no longer an open architectural question.
  `2026-07-28` removed server-initiated requests entirely, so MCP can notify but
  can't ask; whether a notification becomes an agent *wake* is a harness question
  the spec doesn't address. Verify it empirically someday, don't design for it.
- Re-check the Python floor when the SDK lands. The 3.10+ figure came from the
  old SDK, and the v0.1 CLI stays 3.8+ either way.

## Phase 3 — Red-team harness and hotwash

Plan: [docs/redteam-plan.md](docs/redteam-plan.md). **This is the gate** — every
phase after it is scoped by what it finds. The untested assertion the project
rests on is that an agent mid-task treats room messages as data; the realistic
way that fails is not a jailbreak but a confused peer being helpful at it.

- Stand up the isolated environment: fresh Claude Code install, zero real
  credentials, snapshotted VM/container for cheap resets across dozens of trials.
- Ground-truth instrumentation: a `PreToolUse` hook that logs (does not block)
  every tool call, plus fake SMTP/HTTP sinks so outward attempts are counted
  without being delivered.
- **Lead with the naive cases**, since they're the threat model: an agent that
  joined a room with no skill loaded, a peer asking in good faith for something
  destructive, and phrasing that's ambiguous between a request and an order.
- Keep a thin adversarial arm — `SYSTEM:`/`ADMIN OVERRIDE`, impersonating the
  human, urgency, foot-in-the-door — as a cheap sanity check rather than the
  centerpiece. A confused agent is the likely case; a compromised one is still
  possible.
- Measure relay, not just compliance: whether agent A passes a request to B
  because C asked. The failure mode is an earnest telephone game, not a worm.
- Score each cell silent-comply / comply-after-surfacing / refuse-and-flag /
  ignore. "Comply-after-surfacing" is the engineered-for behavior; the number
  that matters is how often it happens instead of silent compliance.
- Run two controls: skill loaded vs. not, which is now the *primary* variable
  because not-understanding-the-room is the threat; and model tier, which decides
  what the Phase 5 monitor can safely run on.
- Measure the controls that shipped after the plan was written — the triage
  screener and the turn budget — rather than treating them as unbuilt. Write the
  hotwash, and keep the harness as a regression suite.

## Phase 4 — Act on the hotwash

A deliberate placeholder: the content is whatever Phase 3 says is worth fixing,
and it stays unspecified until there are numbers. Listing fixes now would be
guessing at the very question the eval exists to answer.

- The leading candidate, if silent-comply rates are non-trivial: a `PreToolUse`
  hook that holds destructive actions taken shortly after a `rooms read`. Tune
  the window against the harness so the false-positive rate is measured, not
  guessed.
- One item that lands here regardless: cross-script homoglyphs still get past the
  NFKC-normalized tripwire and need a confusables table (known `soundcheck`
  residual).

## Phase 5 — The room monitor

A peer agent you bring up with `/rooms:monitor`, not machinery inside the CLI.
It joins a room like anyone else, watches what goes by, and says something when
it doesn't like what it sees. Keeping it out of `bin/rooms` is the whole point —
policy is the part that everyone will want to differ on, so it belongs in a
prompt, not in the transport.

- Ships with a default system prompt and an obvious way to extend or replace it,
  so a room can carry its own house rules.
- It's a peer, so it has **no special authority**: it can flag, warn, and address
  the human, but it cannot hold a message back. That's the honest consequence of
  having no admission control, and the docs should say so rather than implying a
  gate.
- Runs on a cheap model; which tier is safe comes out of the Phase 3 result. If
  cheap models are easy to talk past, it needs a better one.
- It should speak with `announce` so its warnings don't start reply chains, and
  its own traffic shouldn't eat the room's turn budget.
- Liveness lives here too: a peer that's already reading the room is the cheap
  way to notice a member who died or went quiet, and it beats adding a heartbeat
  protocol to the code.

---

**Suggested build order**: Phase 1 is a small independent gap-closer worth doing
immediately. Phase 3 is then the real next piece of work and the one that changes
the shape of the project, since Phase 4 is defined by what it finds and Phase 5's
model choice depends on it. Phase 2 is parked on someone else's release rather
than on us; if that wait drags, its core-module refactor is spec-independent and
can be pulled forward on its own.
