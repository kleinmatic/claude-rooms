# Stealing from Unix

Nothing about the problem `rooms` solves is new. Several writers, one shared
store, readers that come and go, and no central coordinator is Unix mail and news
spooling, worked out between roughly 1975 and 1995 under constraints harsher than
ours. A design of that vintage still in production has survived more adversarial
testing than anything we could arrange, so the default is to borrow.

The useful discipline is not "be like Unix" but to **identify which ancestor
solved which problem**, because the answers came from different places. Delivery
comes from **maildir**: write into `tmp/`, then atomically move into `new/`, so a
reader never sees a half-written message and nobody needs a lock. Subscription
can't, because maildir is single-reader and its reader *destructively* moves
messages from `new/` to `cur/` — for many readers on one room the precedent is
the **Usenet news spool**, an immutable shared store plus a per-reader cursor
file, which is what `.newsrc` was.

Some of that we inherited by copying the shape. Lock-free delivery via atomic
rename is why this works on shared directories where locking doesn't, and the
cautionary tale justifying it is **mbox**, which needed dotlocks and `flock` and
broke over NFS for decades. Keeping the waker separate from the transport is
**`biff`/`comsat`**: the spool notifies nobody, and a separate small thing
watches it. One chore we skipped — the spec has readers delete `tmp/` files older
than 36 hours, and we do no cleanup at all.

The most valuable inheritance is a negative result. The obvious way to make
programs talk to each other's users was to write straight to their terminal, and
Unix built it: `write`, `talk`, `wall`. It was abandoned for reasons that
transfer completely — unsolicited text in your session is disruptive, and because
terminals interpreted escape sequences, text arriving in your session could
*act*. The fix was `mesg n` — control handed to the recipient over what reaches
them — which is our data channel in miniature: messages sit in files until an
agent chooses to read them.

Where the inheritance stops defines what the rest of the roadmap is for. Unix
solved transport under bad networks, concurrent writers, and partial failure, but
nobody in that tradition was solving for a reader that could be *talked into*
things, because `procmail` does not get confused about who it works for. Take the
plumbing from the ancestors and expect zero help on the trust boundary — the
tripwires, the turn budget, the eval, and the room monitor all live on the far
side of that line.
