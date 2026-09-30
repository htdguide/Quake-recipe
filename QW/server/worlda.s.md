# QW/server/worlda.s

> A hand-written version of the hottest collision helper: adding an entity to a list of things a sweep must test.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md)
**Used by** — linked in place of the portable version in [`world.c`](world.c.md)
**Tier floor** — T1: hand-written processor code, replacing a portable equivalent
[Seam: Vectorized inner loops](../../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)

## Purpose

Read [`worlda.s`](../../WinQuake/worlda.s.md) for what the routine does. It exists because the collision code is the server's
largest measured cost and this is its innermost step
([`profile.txt`](profile.txt.md)).

## State

As [`worlda.s`](../../WinQuake/worlda.s.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Adjusted for the trace's added owner-exclusion field ([`world.h`](world.h.md)), which changes the record's offsets.

**Notes** — the useful content is not the code but its existence: **of the entire server, exactly one function was worth hand
writing**, and the profile says which. A rebuild should optimize the same function and should measure before assuming any other.
