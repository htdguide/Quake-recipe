# QW/client/cmd.c

> The command interpreter: a text buffer executed a line at a time, aliases, variable substitution, and the table of registered commands.

**Needs** — [`quakedef.h`](quakedef.h.md) or [`qwsvdef.h`](../server/qwsvdef.h.md) · [`cvar.h`](cvar.h.md) · [`common.h`](common.h.md)
**Used by** — every subsystem registers commands with it; both programs
**Tier floor** — none

## Purpose

Read [`cmd.c`](../../WinQuake/cmd.c.md) for the whole substance: the deferred text buffer, the line splitting that respects quoting,
the alias expansion, the settings fallthrough, and the reason a command's effect is deferred to a known point in the frame.

## State

As [`cmd.c`](../../WinQuake/cmd.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**A command carries a flag saying where it may be issued from** — the local console, a server instruction, or a client's text
command. So a command reachable by a remote party is marked as such, and the same table serves the console and the network.
***That flag is the boundary that stops a client's text command reaching an engine command it should not*** — see
[`sv_user.c`](../server/sv_user.c.md), which dispatches through this table, and
[`cl_main.c`](cl_main.c.md), which executes commands a server sends. A rebuild must have the equivalent; without it, either the
network can run anything or the console cannot run everything.

**A command's arguments can be recovered as one unquoted string** as well as tokenized, which the chat commands need
([`sv_user.c`](../server/sv_user.c.md)) because a chat line must survive tokenization unchanged.

**The buffer can be inserted at the front as well as appended**, so a command that must run before whatever is already queued can be.
Used by the level-change sequence.

**Invariants** — the deferred buffer is still the key idea: **a command's effects happen at one known point in the frame**, never
inside whatever code issued it. That is what makes it safe for a network packet to carry a command at all.

**Notes** — the interpreter is shared by both programs and by the network, which makes it the engine's widest trust surface. The
source-flag mechanism is the only thing narrowing it, and it is worth reading carefully before copying.
