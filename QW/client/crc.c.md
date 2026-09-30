# QW/client/crc.c

> A sixteen-bit cyclic checksum over a byte sequence.

**Needs** — [`crc.h`](crc.h.md)
**Used by** — [`common.c`](common.c.md) for the movement checksum; [`model.c`](model.c.md) and [`sv_init.c`](../server/sv_init.c.md) for content sums; [`net_ser.c`](../../WinQuake/net_ser.c.md) in the original
**Tier floor** — none

## Purpose

Identical in substance to [`crc.c`](../../WinQuake/crc.c.md) — read that twin for the algorithm, the polynomial and the
initialization value, all of which must be reproduced exactly for the sums to agree between programs.

## State

As [`crc.c`](../../WinQuake/crc.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Only a helper that returns the checksum of a whole block in one call, which the dictionary and checksum paths use.

**Invariants** — the polynomial, the initial value and the final exclusive-or are a wire and file contract
([`common.c`](common.c.md)). Nothing about them may be chosen freely.
