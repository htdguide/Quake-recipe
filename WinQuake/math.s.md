# WinQuake/math.s

> Three arithmetic primitives: the fixed-point reciprocal, the world-to-view vector transform, and the box-versus-plane predicate.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_i386.h`](asm_i386.h.md) (the plane layout) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`mathlib.c`](mathlib.c.md), [`r_main.c`](r_main.c.md) and [`world.c`](world.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Stateless.

## The routines

**Contract** — each satisfies the same-named contract in [`mathlib.c`](mathlib.c.md) or
[`r_shared.h`](r_shared.h.md): the 8.24-to-16.16 reciprocal with round-half-up and its overflow clamp;
the transform of a world vector into view space; and the bounding-box-versus-plane side test.

**Invariants** — the reciprocal's rounding is part of its contract
([`mathlib.c`](mathlib.c.md#invert24to16)) because the result places texels. The side test's result
encoding — 1, 2 or 3, never 0 — is likewise contractual.

## What differs

**Invariants** — the side test reads the plane's precomputed sign bits and jumps into a table of eight
specialized cases, exactly as the portable version's switch does
([`mathlib.c`](mathlib.c.md#boxonplaneside)). It indexes the plane array with the **20-byte stride**
frozen in [`asm_i386.h`](asm_i386.h.md), which is the one layout fact a rebuild must not disturb in the
accelerated build.
