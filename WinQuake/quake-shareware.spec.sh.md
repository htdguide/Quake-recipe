# WinQuake/quake-shareware.spec.sh

> Data: a package description for the freely distributable subset of the game's content.

**Needs** — nothing
**Used by** — [`Makefile.linuxi386`](Makefile.linuxi386.md) generates a package from it
**Tier floor** — none

## Purpose

The fourth packaging template beside [`quake.spec.sh`](quake.spec.sh.md),
[`quake-data.spec.sh`](quake-data.spec.sh.md) and the two expansions — read any of those twins for the format and the layout rule.

## State

Data; no run-time state.

## What it differs in

**It packages only the freely distributable content**, which is a subset of the full set: one episode's maps and the assets they need.

**Invariants** — the engine treats the reduced set as a **mode**, not as a different game: a flag is set when the full content is absent,
and the game logic and the menus check it ([`host.c`](host.c.md), [`menu.c`](menu.c.md), and the registration check in
[`../qw-qc/triggers.qc`](../qw-qc/triggers.qc.md)). So the content split reaches into the code, and a rebuild that only ever sees one
content set should know the flag exists and what it gates.

**Notes** — the layout rule is the same as the other templates': a base directory plus per-expansion directories searched first
([`common.c`](common.c.md)).
