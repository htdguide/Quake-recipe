# QW/server/makefile

> Data: the dedicated server's build — and therefore the definitive list of which files the server consists of.

**Needs** — nothing
**Used by** — nothing in the engine; it is the build
**Tier floor** — none

## Purpose

The answer to "what is the server, exactly". Its object list is short and its composition is the interesting fact.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## What it records

**The server is built from three sources:** this directory, a handful of files reached from the client's directory, and nothing
else.

From the client's directory it takes exactly the files that are **genuinely shared between the two programs**:

- the channel ([`net_chan.c`](../client/net_chan.c.md)) and the transport ([`net_udp.c`](../client/net_udp.c.md))
- the movement model and its collision queries
  ([`pmove.c`](../client/pmove.c.md), [`pmovetst.c`](../client/pmovetst.c.md))
- the byte encoders, file system, command interpreter, settings, allocators, checksum and vector maths
  ([`common.c`](../client/common.c.md), [`cmd.c`](../client/cmd.c.md), [`cvar.c`](../client/cvar.c.md),
  [`zone.c`](../client/zone.c.md), [`crc.c`](../client/crc.c.md), [`mathlib.c`](../client/mathlib.c.md))
- the digest ([`md4.c`](../client/md4.c.md))

**Invariants** —

- **The movement model is compiled into both programs from one source.** That is the fact the whole QuakeWorld design rests on,
  and this file is where it is stated. A rebuild that lets the two copies drift has broken prediction and will not know why.
- A flag distinguishes the server build, and shared files test it to omit client-only paths
  ([`bothdefs.h`](../client/bothdefs.h.md)). That conditional compilation is how one file serves two programs, and it is the
  part a rebuild should replace with an explicit interface rather than copy.
- The server links **no renderer, no sound, no input, no interface** — compare the null backends in the original
  ([`vid_null.c`](../../WinQuake/vid_null.c.md)), which exist only because that engine cannot omit them at the build level.
  QuakeWorld's server simply does not compile them, which is cleaner and is the better model.
- There is **no assembly** in the server build at all.

**Notes** — the shared-file list is the recipe's authoritative answer to which parts of the engine are renderer-independent and
client-independent. A rebuilder planning to ship a headless server should start from it.
