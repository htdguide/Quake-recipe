# QW/server/quakeasm.h

> The assembly's shared declarations, cut down to what the server's two assembly files need.

**Needs** — nothing
**Used by** — [`math.s`](math.s.md) · [`worlda.s`](worlda.s.md)
**Tier floor** — T1

## Purpose

An eighth the size of [`quakeasm.h`](../../WinQuake/quakeasm.h.md) — read that twin for what the header does. Here it declares
only the symbol-naming convention and the few globals the server's assembly touches.

**Invariants** — **the symbol-decoration convention must match the platform's compiler**, which is the whole reason this header
exists rather than the names being written directly. It is the same problem [`gas2masm.c`](../../WinQuake/gas2masm/gas2masm.c.md)
solves for the syntax.

**Notes** — the shrinkage from the original is a measurement: the server uses two assembly routines and the client uses forty.
## State

As [`quakeasm.h`](../../WinQuake/quakeasm.h.md); the records are unchanged except where **What differs** says otherwise.

