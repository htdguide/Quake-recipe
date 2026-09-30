# QW/server/math.s

> Hand-written vector and matrix routines for the server, a subset of the client's.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md)
**Used by** — linked in place of the portable versions in [`mathlib.c`](../client/mathlib.c.md)
**Tier floor** — T1: hand-written processor code, replacing a portable equivalent
[Seam: Vectorized inner loops](../../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)

## Purpose

Read [`math.s`](../../WinQuake/math.s.md) for what these routines compute and for the general rule: **every assembly routine in
this engine has a portable twin and the build chooses**, so none of it is required.

## State

As [`math.s`](../../WinQuake/math.s.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The renderer's routines are gone**, leaving the general vector operations, the box-on-plane-side test and the
inverse-square-root. The box-on-plane-side test is the one that matters: it is called through the collision code and is measurably
hot ([`profile.txt`](profile.txt.md)).

**Notes** — a rebuild uses [`mathlib.c`](../client/mathlib.c.md) and ignores this file. It is recorded because the *choice* of
which routines were worth hand-writing is itself the profile's conclusion, restated.
