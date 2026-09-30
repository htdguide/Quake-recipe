# WinQuake/conproc.h

> The four command codes of the external console protocol.

**Needs** — nothing
**Used by** — [`conproc.c`](conproc.c.md) · [`sys_wind.c`](sys_wind.c.md)
**Tier floor** — none

## Purpose

Declares the four commands an external program may send to a dedicated server over the shared-memory channel
([`conproc.c`](conproc.c.md)). It is the whole of that protocol, and it is worth one page because it is the engine's only
machine-readable control interface.

## State

```text
CONSTANT ccom_write_text    = 0x2
CONSTANT ccom_get_text      = 0x3
CONSTANT ccom_get_scr_lines = 0x4
CONSTANT ccom_set_scr_lines = 0x5
```

**Invariants** — the numbering starts at 2, so values 0 and 1 are reserved or were removed. A rebuild inventing
its own protocol need not preserve them, since both ends are in this tree.

## Contract

**Contract** — declares the eight operations [`conproc.c`](conproc.c.md) implements, plus the four codes above.
The semantics are in that file.
