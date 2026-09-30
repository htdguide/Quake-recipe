# qw-qc/progdefs.h

> Data, generated: the engine-side view of the shared globals and entity fields, emitted by the game-logic compiler from `defs.qc`.

**Needs** — nothing
**Used by** — the engine, which compiles it in ([`progdefs.h`](../QW/server/progdefs.h.md) is the same file in the engine's tree)
**Tier floor** — T1: a record layout shared between two separately compiled programs

## Purpose

The generated half of the contract. The compiler reads the section of
[`defs.qc`](defs.qc.md) before each marker and writes this; the engine compiles it and thereby agrees with the game logic about every
offset.

Read [`progdefs.h`](../QW/server/progdefs.h.md) for the content — it is the same file, and that twin describes the globals, the trace
results, the persistent parameters and the callback list.

## State

Data; no run-time state.

## What it records

```text
a record of the shared globals, in declaration order, after a reserved run
a record of the shared entity fields, in declaration order
a version number and a checksum of the layout
```

**Invariants** —

- **It is generated and must never be edited.** Editing it without editing
  [`defs.qc`](defs.qc.md) makes the engine and the game logic disagree about offsets, and the symptom is arbitrary memory corruption
  rather than a clear failure.
- **The checksum is what catches the disagreement**, compared at load
  ([`pr_edict.c`](../QW/server/pr_edict.c.md)), which is why game logic compiled for the original engine is refused cleanly.
- **The same file exists in two places in the tree** — here beside the game logic, and in the engine's directory — and they must be the
  same. Two copies of a generated file is a hazard; a rebuild generates it once into one place.

**Notes** — the presence of this file in the game logic's directory is what makes the chapter self-contained: the game logic carries the
interface it was compiled against.
