# QW/client/r_aliasa.asm

> The same hand-written routines as [`r_aliasa.s`](../../WinQuake/r_aliasa.s.md), transcribed into the other assembler's syntax.

**Needs** — as [`r_aliasa.s`](../../WinQuake/r_aliasa.s.md)
**Used by** — the Windows build, in place of [`r_aliasa.s`](../../WinQuake/r_aliasa.s.md)
**Tier floor** — T1: hand-written processor code, replacing a portable equivalent
[Seam: Vectorized inner loops](../../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)

## Purpose

Read [`r_aliasa.s`](../../WinQuake/r_aliasa.s.md) for what these routines compute and for their portable twins. This file is the same code in
the syntax the Windows assembler accepts, checked into the tree rather than generated at build time.

## State

As [`r_aliasa.s`](../../WinQuake/r_aliasa.s.md) — the same routines, differing only in syntax.

## What differs

Nothing computational. The differences are mechanical and are exactly the ones
[`gas2masm.c`](../../WinQuake/gas2masm/gas2masm.c.md) performs: **operand order is reversed**, operand size moves from the
mnemonic to the operand, memory references are rewritten, and symbol decoration changes.

**Invariants** — **the two copies must agree.** The original engine keeps one copy and translates at build time
([`gas2masm.c`](../../WinQuake/gas2masm/gas2masm.c.md)); this build keeps both, so a change to one must be made to the other by
hand. That is a real maintenance hazard and it is the argument for the original's approach — or, better, for a rebuild using the
portable twins and keeping neither.

**Notes** — a rebuild ignores this file. It is recorded because the recipe mirrors the tree, and because the checked-in duplication
is a fact a reader comparing the two directories will notice and wonder about.
