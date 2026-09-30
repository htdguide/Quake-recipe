# QW/Makefile.Solaris

> Data: the build for the workstation port of the QuakeWorld client and server.

**Needs** — nothing
**Used by** — nothing in the engine; it is the build
**Tier floor** — none

## Purpose

Read [`Makefile.Linux`](Makefile.Linux.md) for the structure and the shared object list. This file records only that the workstation port
builds the software client and the server, with that platform's own video, input, system and sound backends, and no hardware renderer.

## State

Data; no run-time state.

## What it records

**It builds for a big-endian processor**, which makes it the one configuration that exercises the byte-order handling in
[`client/common.c`](client/common.c.md). So this build is the conformance test for every wire and file format in the chapter: **if a
rebuild is byte-order-correct, this is the configuration that proves it.** Same observation as
[`../WinQuake/Makefile.Solaris`](../WinQuake/Makefile.Solaris.md).

**Notes** — the port has no music backend and no hardware renderer, which is the same minimum viable engine
[`../WinQuake/README.Solaris`](../WinQuake/README.Solaris.md) describes.
