# WinQuake/d_varsa.s

> Defines the rasterizer's shared gradient and destination globals in the assembly's own data section.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — every rasterizing `.s` file; the C side declares the same names
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

**Contract** — this file *defines* nineteen globals that the C code declares: the nine span-gradient
values, the texture-space offset and clamping bounds, the cached surface's address and width, the view
buffer, and the depth buffer's address and strides.

**Invariants** — they are defined in assembly rather than in C so that the assembly's own alignment
directives apply: several are accessed as pairs and want to share a cache line. That is the only reason,
and it means the accelerated build and the portable build define the same variables in different files —
so exactly one must be compiled.

**Notes** — a rebuild declares them once, in one place, and lets the compiler place them. The list itself
is the useful content and it is enumerated in [`quakeasm.h`](quakeasm.h.md#the-shared-globals).
