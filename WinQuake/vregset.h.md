# WinQuake/vregset.h

> The shape of a display register table: groups of index-and-value pairs, each group naming the ports it is written through.

**Needs** — nothing
**Used by** — [`vregset.c`](vregset.c.md) · [`vgamodes.h`](vgamodes.h.md) · [`vid_vga.c`](vid_vga.c.md)
**Tier floor** — T0

## Purpose

Declares the record the mode tables are written in.

## State

```text
RECORD RegisterGroup
  index_port, data_port : port address     # one of them may be absent
  count : int
  pairs : (index, value)[]                 # written in order
RECORD RegisterTable = a list of groups, terminated
```

**Invariants** — a group with no index port writes each value **directly to the port named by its index field**, which is how
one record serves both indexed and direct registers. Order is significant ([`vregset.c`](vregset.c.md)).

**Notes** — the data description for [`vgamodes.h`](vgamodes.h.md).
