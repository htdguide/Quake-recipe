# WinQuake/vregset.c

> Programs a run of display controller registers from a table, and issues a firmware call through a register block.

**Needs** — [`vregset.h`](vregset.h.md) · [`dosisms.h`](dosisms.h.md)
**Used by** — [`vid_vga.c`](vid_vga.c.md) · [`vid_ext.c`](vid_ext.c.md) · [`vgamodes.h`](vgamodes.h.md)
**Tier floor** — T0: it writes display controller registers directly

## Purpose

The mechanism behind the mode tables in [`vgamodes.h`](vgamodes.h.md): a display mode is a list of register addresses and
values, and this walks the list writing each one.

## State

Stateless.

## `VideoRegisterSet`, `loutportb`

**Contract** — takes a table of register groups; for each group, writes each value to its register through the group's index
port, or directly when the register is not indexed. The helper writes one byte to one port.

```text
FUNCTION video_register_set(table)
  FOR EACH group in the table
    FOR EACH (index, value) pair in the group
      IF the group has an index port
        write index to the index port ;  write value to the data port
      ELSE
        write value to the port named by index
```

**Invariants** — **the order within the table is load-bearing.** Some registers must be written before others for the
controller to accept them, and the tables in [`vgamodes.h`](vgamodes.h.md) encode a working order discovered by experiment.
Reordering them produces a mode that does not display.

**Notes** — pure hardware, a **given**. Recorded because the mode tables are data twins that point here for their meaning.
