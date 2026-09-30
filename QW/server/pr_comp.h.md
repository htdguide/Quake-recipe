# QW/server/pr_comp.h

> The compiled game language's instruction set and file format, unchanged from the original.

**Needs** — nothing
**Used by** — [`pr_exec.c`](pr_exec.c.md) · [`pr_edict.c`](pr_edict.c.md)
**Tier floor** — none

## Purpose

Byte-for-byte identical to [`pr_comp.h`](../../WinQuake/pr_comp.h.md) — read that twin for the sixty-two operations, the type
set and the compiled file's sections.

**Invariants** — **the instruction set did not change between the two engines.** So a rebuild implementing the interpreter
implements it once, and the language is a fixed target. What changed is the *field set* the programs declare
([`progdefs.h`](progdefs.h.md)) and the *engine operations* they may call ([`pr_cmds.c`](pr_cmds.c.md)) — the language itself is
stable.
## State

As [`pr_comp.h`](../../WinQuake/pr_comp.h.md); the records are unchanged except where **What differs** says otherwise.

