# QW/qwrlnote.txt

> Data: the release notes for a QuakeWorld version — the user-visible changes, the new settings, and the compatibility warnings.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The user-facing half of [`qwchangelog.txt`](qwchangelog.txt.md): the same changes described as a player experiences them. Its recipe value
is that it says **which internal changes were visible**, which is a different list from the changes themselves.

## State

Data; no run-time state.

## What it records

- **Protocol incompatibility**, stated plainly: clients and servers of different versions do not interoperate
  ([`client/protocol.h`](client/protocol.h.md)).
- **New settings and what to set them to**, including the connection three
  ([`docs/readme.qwcl`](docs/readme.qwcl.md)).
- **The spectator features** and their commands.
- **The content-download behaviour** and where files land.
- **Movement changes**, described in terms of how the game feels rather than what changed — which is the honest way to describe them, and a
  reminder that the movement model's constants are the game
  ([`client/pmove.c`](client/pmove.c.md)).
- **Known problems**, overlapping with [`qw2do.txt`](qw2do.txt.md)'s unfinished items.

**Notes** — read it for the compatibility warnings and the movement descriptions. The rest duplicates
[`docs/qwcl-readme.txt`](docs/qwcl-readme.txt.md).
