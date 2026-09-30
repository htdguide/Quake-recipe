# QW/client/cvar.h

> The settings interface, with the dictionary write-through flags added.

**Needs** — nothing
**Used by** — every subsystem in both programs
**Tier floor** — none

## Purpose

Read [`cvar.h`](../../WinQuake/cvar.h.md) for the interface. The addition is the flags marking a setting as projected into the public
or the client dictionary ([`cvar.c`](cvar.c.md)), and the run-time creation operation.
## State

As [`cvar.h`](../../WinQuake/cvar.h.md); the records are unchanged except where **What differs** says otherwise.

