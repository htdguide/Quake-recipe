# QW/server/qwsvdef.h

> The server program's single umbrella header: everything the server compiles against, in dependency order, plus the startup parameters and the two functions the server implements in place of the client's.

**Needs** — [`bothdefs.h`](../client/bothdefs.h.md) · [`common.h`](../client/common.h.md) · [`bspfile.h`](../client/bspfile.h.md) · [`sys.h`](sys.h.md) · [`zone.h`](../client/zone.h.md) · [`mathlib.h`](../client/mathlib.h.md) · [`cvar.h`](../client/cvar.h.md) · [`net.h`](../client/net.h.md) · [`protocol.h`](../client/protocol.h.md) · [`cmd.h`](../client/cmd.h.md) · [`model.h`](../client/model.h.md) · [`crc.h`](../client/crc.h.md) · [`progs.h`](progs.h.md) · [`server.h`](server.h.md) · [`world.h`](world.h.md) · [`pmove.h`](../client/pmove.h.md)
**Used by** — every `.c` file in this directory
**Tier floor** — none

## Purpose

The server's counterpart to [`quakedef.h`](../client/quakedef.h.md), and the clearest statement of **what the dedicated server
is**: exactly this list of headers and nothing else. Compare the client's umbrella header and the difference is the entire
renderer, sound, input and interface.

## State

```text
RECORD QuakeParms                   # what the platform hands the program
  basedir   : text                  # where content lives
  cachedir  : text                  # an optional faster copy of it
  argc, argv                        # the command line
  membase, memsize                  # the single heap
VARIABLE host_parms
VARIABLE host_initialized, host_frametime, realtime
FUNCTION SV_Error, SV_Init, Con_Printf, Con_DPrintf
```

**Invariants** —

- **The include order is the dependency order** and is load-bearing, because these headers declare records in terms of each
  other without forward declarations. The order is also a reading order for this chapter.
- **The server includes the movement interface** ([`pmove.h`](../client/pmove.h.md)), which is the whole point of
  QuakeWorld's split: the server runs the same movement code the client predicts with.
- **The server implements the printing functions itself** ([`sv_send.c`](sv_send.c.md)), declared here, so that code shared with
  the client prints to the server's console and its redirect. Two programs, one set of names, chosen at link time. A rebuild
  should make this an explicit interface rather than a naming coincidence.
- The parameters record is identical to the client's, which is what lets the file-system and allocator code be shared
  unchanged.
- **The cache directory exists to copy content to faster storage** at startup. It is a development convenience of its era and a
  rebuild can drop it; it is recorded because the file-system code checks it on every open
  ([`common.c`](../client/common.c.md)).
