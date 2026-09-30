# WinQuake/sys_wina.s

> Floating-point control-word manipulation for the Win32 build.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — [`sys_win.c`](sys_win.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Identical in contract to [`sys_dosa.s`](sys_dosa.s.md); the two differ only in assembler syntax and
symbol decoration.

**Notes** — that two files exist for one contract is what the build-time translator
([`gas2masm/`](gas2masm/README.md)) was written to avoid, and this pair is evidence it was not applied
everywhere.
