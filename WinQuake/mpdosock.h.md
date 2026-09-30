# WinQuake/mpdosock.h

> Declarations for the third-party matchmaking library: its socket types, address forms, error codes and queue-node layout.

**Needs** — nothing
**Used by** — [`mplib.c`](mplib.c.md) · [`mplpc.c`](mplpc.c.md) · [`net_mp.c`](net_mp.c.md)
**Tier floor** — T1: it fixes record layouts that a resident library reads directly

## Purpose

A vendor's header, carried in the tree because the library it describes was not on the machine. It is a **given**, not
a decision: the recipe records its existence and its role, and a rebuild that drops the MPath transport drops this
with it.

## State

```text
The declarations fall into four groups:
  socket handle type, address record, and the option and family constants
  the error code set the facade must translate from
  the queue-node record shared with the library, whose LAYOUT IS FIXED
  the entry-point signatures the loader fills in
```

**Invariants** — the queue-node layout is a **binary contract with code the engine does not compile**, the same class
of constraint as the assembly layout headers ([`asm_i386.h`](asm_i386.h.md)). Nothing may reorder or resize a field.

**Notes** — recorded for completeness. Nothing here is rebuildable or worth rebuilding; see
[`net_mp.c`](net_mp.c.md) for the one durable idea in this group of files.
