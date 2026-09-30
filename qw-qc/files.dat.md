# qw-qc/files.dat

> Data: the list of every content file the game requires, produced by the compiler from the precache calls.

**Needs** — nothing
**Used by** — the packaging process
**Tier floor** — none

## Purpose

A by-product of compilation: the compiler records every file name passed to a precache operation and writes the list here. It is
therefore the **authoritative inventory of the game's assets**, derived rather than maintained.

## State

Data; no run-time state.

## What it records

Every model, sound and image file the game logic names — several hundred entries — each exactly as it will be requested at run time
([`world.qc`](world.qc.md) makes most of the calls).

**Invariants** —

- **The list is derived from the code, so it cannot drift.** Adding a precache call adds an entry; an asset nobody precaches does not
  appear, and an entry with no file is a missing asset the packaging step catches. That is the right direction for the dependency, and it
  is worth copying: **let the manifest fall out of the code rather than maintaining it beside it.**
- The names are the same strings the client downloads by
  ([`cl_parse.c`](../QW/client/cl_parse.c.md)), so this list is also what a server may be asked to send.

**Notes** — a rebuild should generate an equivalent, because the alternative — a hand-maintained asset list — is wrong within a week.
