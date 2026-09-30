# QW/release233_notes.txt

> Data: the notes for one specific release — a short list of fixes and the compatibility statement.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

A single release's notes, forty lines. Recorded for completeness; its content is a subset of
[`qwchangelog.txt`](qwchangelog.txt.md) and [`qwrlnote.txt`](qwrlnote.txt.md).

## State

Data; no run-time state.

## What it records

The fixes in that release, the settings it added, and whether it is compatible with the previous version's servers.

**Notes** — the only durable observation is the one every release note here makes: **a protocol change is a hard break**, and the engine
chose to break rather than to negotiate. That is a defensible choice for a game whose client and server are distributed together, and the
wrong one for anything with independent deployment — a rebuild should decide which it is before it needs to.
