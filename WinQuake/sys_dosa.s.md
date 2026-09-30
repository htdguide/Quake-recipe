# WinQuake/sys_dosa.s

> Floating-point control-word manipulation for the DOS build.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — [`sys_dos.c`](sys_dos.c.md)
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

**Contract** — as [`sys.h`](sys.h.md#floating-point-control): set the hardware's rounding precision to
single or to full, save and restore the control word around a region, and mask or unmask the arithmetic
exceptions.

**Invariants** — masking exceptions is what lets the renderer divide by very small numbers without
trapping, and the single-precision setting is what the rasterizer's arithmetic was tuned at
([`mathlib.c`](mathlib.c.md#invert24to16)).

**Notes** — a rebuild on a platform whose floating point is always IEEE single or double with masked
exceptions implements all seven as nothing, which is what every other backend in the tree does.
