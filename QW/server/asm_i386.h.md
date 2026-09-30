# QW/server/asm_i386.h

> The record layouts shared between the assembly and the rest of the engine, carried into the server's directory unchanged.

**Needs** — nothing
**Used by** — [`math.s`](math.s.md) · [`worlda.s`](worlda.s.md)
**Tier floor** — T1: it freezes record field offsets that hand-written code depends on

## Purpose

Byte-for-byte identical to [`asm_i386.h`](../../WinQuake/asm_i386.h.md) — read that twin. It is present here because the two
assembly files in this directory include it.

**Invariants** — **only the offsets the server's two assembly files need are actually used**: the plane record and the trace
result. The rest of the file describes renderer records the server does not have. So the file's real content, for the server, is
much smaller than it appears, and a rebuild that drops the assembly ([`math.s`](math.s.md)) drops the whole file.
## State

As [`asm_i386.h`](../../WinQuake/asm_i386.h.md); the records are unchanged except where **What differs** says otherwise.

