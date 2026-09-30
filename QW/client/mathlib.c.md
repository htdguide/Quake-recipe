# QW/client/mathlib.c

> Vector and angle arithmetic, and the box-against-plane test the collision code leans on.

**Needs** — [`mathlib.h`](mathlib.h.md)
**Used by** — every part of both programs
**Tier floor** — none

## Purpose

Read [`mathlib.c`](../../WinQuake/mathlib.c.md) for the substance: the angle conventions and their negations, the basis construction,
the sign-pattern box test, and the portable twins of the hand-written routines.

## State

As [`mathlib.c`](../../WinQuake/mathlib.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Two additions: an **angle interpolation that takes the shorter way around**, needed by the spectator camera
([`cl_cam.c`](cl_cam.c.md)) and the angle interpolation in
[`cl_ents.c`](cl_ents.c.md); and a helper that rounds a value to the wire's precision, used by the command quantization
([`cl_input.c`](cl_input.c.md)).

**Invariants** — the shorter-way-around rule is the classic angle bug's fix and it belongs here rather than in each caller.

The angle conventions — which axis is negated, in what order rotations compose — are unchanged and remain a contract with the
protocol, the model format and the game logic alike.
