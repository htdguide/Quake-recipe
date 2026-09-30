# QW/qwchangelog.txt

> Data: the release-by-release change history — the order in which the networking's problems were actually discovered and fixed.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The chronology. [`server/newnet.txt`](server/newnet.txt.md) says what was designed; this says what had to be repaired afterwards, in
order. For a rebuilder that order is valuable in itself: **it is the list of problems a networked engine encounters, ranked by when they
bite.**

## State

Data; no run-time state.

## What it records

The recurring themes across the releases, each mapping onto a page in this chapter:

- **Reliable-stream overflows dropping players**, repeatedly, until the spillover queue was added
  ([`server/sv_nchan.c`](server/sv_nchan.c.md)). The most-fixed problem in the history.
- **Prediction disagreeing with the server** — the grid snap, the command quantization, the split threshold matching, the movement
  tunables travelling over the wire. Each was a separate release
  ([`client/pmove.c`](client/pmove.c.md), [`client/cl_input.c`](client/cl_input.c.md), [`client/cl_pred.c`](client/cl_pred.c.md)).
- **Address translators breaking connections mid-game**, answered by the connection identifier
  ([`client/net_chan.c`](client/net_chan.c.md)).
- **Bandwidth settings** being wrong in both directions, and the clamps that resulted
  ([`server/sv_user.c`](server/sv_user.c.md)).
- **Spectator support** arriving in stages and its interactions with visibility filtering
  ([`client/cl_cam.c`](client/cl_cam.c.md), [`server/sv_ents.c`](server/sv_ents.c.md)).
- **Content downloading** and the path checks around it
  ([`client/cl_parse.c`](client/cl_parse.c.md), [`client/skin.c`](client/skin.c.md)).
- **Flood protection and administrative controls**, added once servers were public
  ([`server/sv_ccmds.c`](server/sv_ccmds.c.md)).
- **Protocol version bumps**, each breaking compatibility deliberately.

**Notes** — the ordering is the lesson. A rebuild will meet these in roughly the same sequence, and knowing that the reliable-stream
overflow and the prediction divergence are the first two saves real time: **build the spillover queue and the quantization before you need
them.**
