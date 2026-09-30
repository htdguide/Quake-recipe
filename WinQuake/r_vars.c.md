# WinQuake/r_vars.c

> Defines the renderer's shared globals in one contiguous block, for the portable build only.

**Needs** — [`quakedef.h`](quakedef.h.md)
**Used by** — the portable build; [`r_varsa.s`](r_varsa.s.md) defines the same names for the accelerated one
**Tier floor** — none

## Purpose

The renderer's counterpart to [`d_vars.c`](d_vars.c.md), with the same motivation — the comment says the
variables are gathered to avoid cache conflicts — and the same build exclusivity: exactly one of this file and
its assembly twin is compiled.

## State

The sub-model-active flag and the small set of renderer globals the assembly reads.

**Notes** — the file's own comments ask for the globals to become one record and for the renderer's and the
rasterizer's to be separated. A rebuild should do both; the cache-locality reason survives as "keep an inner
loop's state together".
