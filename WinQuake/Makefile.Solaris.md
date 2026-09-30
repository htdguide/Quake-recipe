# WinQuake/Makefile.Solaris

> Data: the build description for the workstation port, naming its own video, input and system backends.

**Needs** — nothing
**Used by** — nothing in the engine; it is the build
**Tier floor** — none

## Purpose

Read [`Makefile.linuxi386.md`](Makefile.linuxi386.md) for the structure and for the renderer substitution rule. This file
records only that the workstation port is the software renderer plus
[`vid_sunx.c`](vid_sunx.c.md) or [`vid_sunxil.c`](vid_sunxil.c.md), [`in_sun.c`](in_sun.c.md),
[`sys_sun.c`](sys_sun.c.md), [`snd_sun.c`](snd_sun.c.md) and [`cd_null.c`](cd_null.c.md) — so that port has no music and
no hardware renderer.

**Invariants** — it builds for a **big-endian** processor, which is the only place in the tree that exercises the byte-order
handling in [`common.c`](common.c.md). That makes this build the conformance test for every wire and file format in the
recipe, and it is worth saying so: **if a rebuild is byte-order-correct, this is the configuration that proves it.**

**Notes** — see [`README.Solaris`](README.Solaris.md) for what the port does not support.
## State

Data; no run-time state.

