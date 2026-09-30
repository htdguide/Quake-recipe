# WinQuake/nonintel.c

> Empty implementations of the three self-modifying-code patch routines, for builds with no assembly.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`r_local.h`](r_local.h.md)
**Used by** — [`r_main.c`](r_main.c.md) and [`r_edge.c`](r_edge.c.md) call them unconditionally
**Tier floor** — none

## Purpose

The accelerated renderer patches constants into its own instruction stream
([`surf8.s`](surf8.s.md), [`d_polysa.s`](d_polysa.s.md)) and the callers invoke the patch routines without a
compile-time test. This file supplies do-nothing versions so the portable build links.

## State

Stateless.

## Contract

**Contract** — three routines — the eight-bit surface patch, the sixteen-bit surface patch, and the surface
pool base patch — each doing nothing.

**Notes** — the file's name records where its content was expected to go: the portable counterparts of the
x86-specific arithmetic. One such routine, the fixed-point reciprocal, ended up in
[`mathlib.c`](mathlib.c.md#invert24to16) instead, with a comment saying it belongs here.

A rebuild that passes the patched values as arguments deletes this file and the calls.
