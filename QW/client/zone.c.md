# QW/client/zone.c

> The three allocators: a double-ended stack over one heap, a small first-fit heap at its bottom, and an evictable cache in the gap.

**Needs** — [`zone.h`](zone.h.md) · [`common.h`](common.h.md)
**Used by** — every subsystem in both programs
**Tier floor** — T1: it manages one contiguous block by hand, with alignment and eviction

## Purpose

Read [`zone.c`](../../WinQuake/zone.c.md) in full — the double-ended stack, the named allocations, the first-fit small heap, the
least-recently-used evictable cache in the gap, the consistency checks and the reporting are all unchanged and are among the most
transferable things in the recipe.

## State

As [`zone.c`](../../WinQuake/zone.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**Almost nothing.** A few messages and the removal of a reference to the server's own reset point, since the server is a separate
program with its own heap.

**Invariants** — that this file is unchanged across both engines is worth recording: **the memory model survived the rewrite
untouched.** One block from the platform at startup, three allocators over it, nothing returned to the operating system. A rebuild in
a garbage-collected language will replace all of it — see
[`SYSTEM-REQUIREMENTS.md`](../../SYSTEM-REQUIREMENTS.md) — and should read the original's twin for what the eviction policy is
actually protecting before doing so.
