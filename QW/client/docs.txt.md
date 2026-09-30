# QW/client/docs.txt

> Data: the author's notes on the connection process and the open questions still outstanding when the code shipped.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The companion to [`newnet.txt`](../server/newnet.txt.md), written later: where that file designs the networking, this one documents
the connection sequence as built and lists what was still unresolved. Its value is that the *shipped* code's rough edges are named by
their author.

## State

Data; no run-time state.

## What it records

**The level-counter rule, stated plainly:** each level change increments a counter; every stage of the client's join carries it; a
mismatch restarts the join. That is the mechanism in
[`sv_user.c`](../server/sv_user.c.md) and [`server.h`](../server/server.h.md), and this is the only place it is explained rather than
merely implemented.

**The multi-block level description**, and why several buffers are needed
([`sv_init.c`](../server/sv_init.c.md)).

**A list of open problems**, each of which is a real property of the shipped code:

- *A skin list message* — planned and not present; skins are instead discovered from each player's dictionary
  ([`skin.c`](skin.c.md)), which works but means the client cannot pre-fetch.
- *Putting frag counts in the information dictionary* — not done; scores are their own message
  ([`protocol.h`](protocol.h.md)).
- *A command that both fires and prints a chat line* — an ambiguity in the command interpreter's handling of a bound sequence
  ([`keys.c`](keys.c.md)).
- *No forwarding of commands to the server unless active* — the fallthrough in
  [`keys.c`](keys.c.md) does forward earlier than intended.
- *Level-change and reconnect confusion* — the two paths in
  [`cl_main.c`](cl_main.c.md) overlap, which is a known source of a client getting stuck between levels.
- *Synchronizing a forced view angle with the first update* — a forced angle travels unreliably and can be lost, so a teleport can
  leave the player facing the wrong way. The note proposes sending it reliably and it was not done.

**Notes** — read this beside the code it describes. **A list of known defects written by the author is worth more to a rebuilder than
any amount of inference**, because each item marks a place where the shipped behaviour is not the intended behaviour — and a rebuild
should implement the intention, not the code.
