# QW/client/console.c

> The console: a scrolling text buffer, a command line with completion, and the chat line that shares the keyboard with it.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`draw.h`](draw.h.md) · [`keys.h`](keys.h.md) · [`cmd.h`](cmd.h.md)
**Used by** — every subsystem prints through it; [`keys.c`](keys.c.md) routes typing to it
**Tier floor** — none

## Purpose

Read [`console.c`](../../WinQuake/console.c.md) for the substance: the circular text buffer, the reflow on a resolution change, the
notification lines that appear over the game, and the rule that printing is safe from anywhere including an error path.

## State

As [`console.c`](../../WinQuake/console.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**Chat is a separate input mode sharing the same line editor.** A key opens the line in chat mode rather than console mode, and what is
typed is sent as a chat command rather than executed ([`sv_user.c`](../server/sv_user.c.md)). The distinction is one flag and it is
worth noting because it is the only place the client's input is *content* rather than *control*.

**The typing line can be cleared from outside**, which the menu and the chat mode both need.

**The buffer is resizable at run time** rather than only on a mode change, because the console is used while connected and the
resolution can change under it.

**Invariants** — **printing must work before the console exists and after the renderer is gone.** The original's rule, and it matters
more here because a fatal error returns to the console rather than exiting
([`cl_main.c`](cl_main.c.md)) — so the console has to be the thing that survives.

**Notes** — the notification lines over the game are how chat is seen while playing, which makes the console's overlay a gameplay
feature rather than a debugging one. It is drawn with the same importance filter the protocol carries
([`protocol.h`](protocol.h.md)).
