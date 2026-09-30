# WinQuake/r_varsa.s

> Defines one shared renderer global in the assembly's data section.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of the renderer
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

**Contract** — defines the flag saying whether a sub-model is currently being drawn, which changes the
depth comparison in the edge sorter.

**Notes** — as with [`d_varsa.s`](d_varsa.s.md), the definition lives in assembly only so that exactly
one of the two builds defines it. A rebuild declares it once in C.
