# WinQuake/dosasm.s

> Four DOS-only helpers: a cycle-counter bracket and a stack-depth check.

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

## Contract

**Contract** — a pair of routines bracketing a region to be timed with the processor's cycle counter, and
a pair implementing a stack-depth check — one installing a limit and one testing against it.

**Notes** — the cycle-counter pair is a profiling aid for a platform with no sub-millisecond clock. The
stack check exists because the DOS build ran on a small stack and the renderer's recursion could exhaust
it silently.

A rebuild needs neither: [`sys.h`](sys.h.md#sys_floattime) already requires a real clock, and a modern
stack either grows or faults visibly.
