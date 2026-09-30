# QW/qw2do.txt

> Data: the working to-do list for a QuakeWorld release, with the finished items marked — so it records both what was done and what was left.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The most candid page in the recipe. It is a developer's list with completed items struck through, which makes it two documents at once: a
changelog of the release's real changes, and a list of known defects that survived it.

Read it beside [`client/docs.txt`](client/docs.txt.md) and
[`server/newnet.txt`](server/newnet.txt.md); together the three say where the shipped code does not match the intent.

## State

Data; no run-time state.

## What it records — finished

Each of these is a decision the recipe documents elsewhere, and this file is where the *reason* is stated in the author's own words:

- **The spillover queue in front of the reliable stream** — the entry proposes exactly what shipped: rotate buffers automatically, route
  every reliable write through typed functions, and *"no more overflows"*
  ([`server/sv_nchan.c`](server/sv_nchan.c.md)).
- **Delta-compressed movement commands**, explicitly *to reduce client-to-server bandwidth*
  ([`client/common.c`](client/common.c.md), [`client/cl_input.c`](client/cl_input.c.md)).
- **A fix for player drift in the initial snapped position** — the grid-snap problem
  ([`client/pmove.c`](client/pmove.c.md)).
- **The through-the-eyes spectator camera**, noted as *broken in recording playback*
  ([`client/cl_cam.c`](client/cl_cam.c.md)) — a defect shipped knowingly.
- **A restored earlier movement model** with the newer gravity handling, friction unchanged, with two specific map locations named as the
  things to check. That is the closest the tree comes to a regression test for movement: *two stairs on two named maps*
  ([`client/pmove.c`](client/pmove.c.md)).

## What it records — unfinished

- **Name changes are not broadcast on a second change**, and should count against flood protection
  ([`server/sv_main.c`](server/sv_main.c.md)'s rate-limited name handling is the partial fix).
- **Spectator name changes are not broadcast at all.**
- **Keys bound to press-and-release controls do not work with the console open** in the original engine
  ([`WinQuake/keys.c`](../WinQuake/keys.c.md)).
- Two renderer settings that were wanted and not added.

**Notes** — the pairing of *what was done* with *why* is what makes this worth a page. The movement entry in particular is the only place
in the whole source that names a concrete acceptance check for the mover, and a rebuild reproducing this game's movement should use it:
**walk the named stairs and compare.**
