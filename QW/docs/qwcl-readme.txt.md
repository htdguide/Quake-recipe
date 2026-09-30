# QW/docs/qwcl-readme.txt

> Data: the long-form client documentation — the full console command and setting reference, the connection troubleshooting, and the protocol's user-visible behaviour.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The most complete description of the shipped system from outside the code. For a rebuilder its value is as a **specification of
observable behaviour**: everything in it is something a player can check, which makes it the closest thing the recipe has to an
acceptance test for the client.

## State

Data; no run-time state.

## What it records

**The complete setting and command reference.** Every name, its default and its effect. That is the authoritative list of the client's
configurable surface, and it is longer than any header — a rebuild wanting to accept existing configuration files should take it from
here.

**The troubleshooting guidance**, which is a list of observable symptoms mapped to causes:

- jerky movement of other players → extrapolation disabled, or a link with high jitter
  ([`cl_ents.c`](../client/cl_ents.c.md))
- one's own movement rubber-banding → prediction disagreeing with the server, usually a rate set too high
  ([`cl_pred.c`](../client/cl_pred.c.md))
- entities freezing while the world continues → the delta reference aged out, so absolute snapshots are being sent
  ([`cl_parse.c`](../client/cl_parse.c.md))
- a long pause on joining → the download sequence
  ([`cl_parse.c`](../client/cl_parse.c.md))

**The spectator commands and what a spectator can and cannot do**
([`cl_cam.c`](../client/cl_cam.c.md), [`sv_user.c`](../server/sv_user.c.md)).

**The recording and playback commands**, and the note that a recording is self-contained
([`cl_demo.c`](../client/cl_demo.c.md)).

**Invariants** — the symptom-to-cause table is the useful part and it is worth reproducing in a rebuild's own terms, because **every
one of those four symptoms is a distinct failure of the networking and they look similar to a player.** A rebuild that cannot
distinguish them in its diagnostics will not be able to support its users.

**Notes** — the file also documents behaviour the code does not implement as described, which is normal for release notes; where the
two disagree the code is what shipped, and [`docs.txt`](../client/docs.txt.md) lists the known gaps.
