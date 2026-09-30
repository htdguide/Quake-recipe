# QW/gas2masm/gas2masm.c

> The assembly syntax translator, unchanged.

**Needs** — nothing beyond standard input and output
**Used by** — the Windows build, though this tree also checks in the translated output
**Tier floor** — none

## Purpose

The same program as [`gas2masm.c`](../../WinQuake/gas2masm/gas2masm.c.md) — read that twin for the tables, the reversed operand order,
the size-in-the-mnemonic rule and the specially handled non-commutative operations.

## State

As [`gas2masm.c`](../../WinQuake/gas2masm/gas2masm.c.md) — the token buffers, the segment being emitted, and the input line number.

## What differs

Nothing of substance.

**Notes** — its presence here is slightly odd, because this tree *also* checks in the translated assembly
([`d_draw.asm`](../client/d_draw.asm.md) and its siblings). So the translator and its output are both in the tree and the output is
maintained by hand. That is the two-copy hazard named in those twins, and this file is the tool that would have prevented it.
