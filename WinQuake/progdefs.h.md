# WinQuake/progdefs.h

> Selects which generated game-interface layout this build uses.

**Needs** — [`progdefs.q1`](progdefs.q1.md) or [`progdefs.q2`](progdefs.q2.md)
**Used by** — [`progs.h`](progs.h.md)
**Tier floor** — none

## Purpose

Two generated layouts exist in the tree, one per game, and only one can be compiled in
because the engine overlays a native record on interpreter memory. This file is the
switch.

## State

Stateless.

## Contract

**Contract** — a build switch selects the sequel's layout; otherwise the original
game's. Nothing else.

**Notes** — the choice is made at build time and cannot be made at run time, which is
the whole reason the file exists as its own compilation unit rather than as two
includes at the use site. A rebuild that resolves fields by name at load time
([`progdefs.q1`](progdefs.q1.md#why-the-file-is-generated)) can support both layouts in
one binary and delete this file.
