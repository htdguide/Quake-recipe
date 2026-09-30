# WinQuake/adivtab.h

> A table of quotients and remainders for every pair of small integers, so the model clipper can divide without a divide instruction.

**Needs** — nothing; a table of literals
**Used by** — [`r_aliasa.s`](r_aliasa.s.md) and the model rasterizer's assembly
**Tier floor** — none; a rebuild deletes it

## Purpose

Dividing one small integer by another, on a 1996 x86, cost tens of cycles. The model rasterizer needs
exactly that division per triangle edge — a screen-space slope — and the operands are always small. So
the results are tabulated.

## State

```text
CONSTANT adivtab : (quotient, remainder)[32 * 32]
    # indexed by (numerator + 15) * 32 + (denominator + 15),
    # for numerators and denominators in -15 .. +16
```

**Invariants** — the range is −15 through +16 on both axes, 32 by 32, and the division is **floor-based**
rather than truncating — the remainder is the one that makes `quotient * denominator + remainder` equal
the numerator with the remainder taking the denominator's sign convention. That matters because the
rasterizer steps with the remainder as an error term
([`mathlib.c`](mathlib.c.md#floordivmod) implements the same semantics for the general case).

Division by zero is present in the table and holds whatever the generator produced; the caller never
reaches it.

The operand range is a claim about the geometry: a model triangle's screen-space edge, after the
subdivision threshold has sent distant models down the point-plotting path, spans at most sixteen
scanlines. A rebuild must not assume the same bound.

**Notes** — the entire file is an optimization for a machine whose integer divide is slow. Every current
processor divides small integers in a few cycles. A rebuild should divide, and should keep the
floor-based semantics.
