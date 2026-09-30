# WinQuake/d_vars.c

> Defines the rasterizer's shared globals in one contiguous block, for the portable build only.

**Needs** — [`quakedef.h`](quakedef.h.md)
**Used by** — the portable build; the accelerated build defines the same names in [`d_varsa.s`](d_varsa.s.md) instead
**Tier floor** — none

## Purpose

Nineteen definitions and a comment explaining the file's existence: **the globals are collected into one
contiguous block to avoid cache conflicts.** On a machine with a small, low-associativity cache, variables
read together in an inner loop but scattered across memory can repeatedly evict one another; placing them
adjacently avoids it.

## State

```text
# The nine span-gradient values, the texture offset and bounds, the cached
# surface's address and width, the view buffer, and the depth buffer's address
# and strides. Enumerated in quakeasm.h.
```

**Invariants** — exactly one of this file and [`d_varsa.s`](d_varsa.s.md) is compiled, selected by the
architecture. Compiling both is a duplicate-definition error, and compiling neither leaves the renderer
unlinked.

**Notes** — the file's own comments ask for the globals to be gathered into one record, and separately for
the renderer's and the rasterizer's to be split apart. Both are the right refactors and a rebuild should
do them; the cache-locality motive survives as "keep the inner loop's state together", which a record
achieves.
