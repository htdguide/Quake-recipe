# QW/server/pr_exec.c

> The game-logic interpreter: three-address operations over a flat slot pool, with a runaway bound and a call stack.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`progs.h`](progs.h.md) · [`pr_comp.h`](pr_comp.h.md) · [`pr_cmds.c`](pr_cmds.c.md)
**Used by** — every part of the server that calls into the game logic
**Tier floor** — none

## Purpose

Read [`pr_exec.c`](../../WinQuake/pr_exec.c.md) in full: the operation set, the flat global pool, entity references as byte
offsets, the locals saved and restored as runs of globals, the runaway instruction bound and the hard-coded animation interval are
all unchanged and are the load-bearing content.

## State

As [`pr_exec.c`](../../WinQuake/pr_exec.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The profiling and tracing facilities are extended**, with per-function instruction counts reported on demand. That is how the
measurement in [`profile.txt`](profile.txt.md) attributes cost to game logic at all.

**The error reporting names the server's own error path** rather than the host's, and a runtime error drops the offending client
where possible instead of killing the server. On a public server a modification's bug must not end the session for twenty players,
which is a real operational difference from the original's behaviour.

**The interpreter is compiled into only one program.** In the original both client and server link it because the engine is one
process; here it is the server's alone, which is why the client directory has no copy.

**Notes** — the measured cost is about six percent of the server under load with roughly a hundred thousand program executions per
sample ([`profile.txt`](profile.txt.md)). That is the number that licenses a rebuild to keep a simple interpreter: **the game
logic is not the bottleneck and does not need compiling.**
