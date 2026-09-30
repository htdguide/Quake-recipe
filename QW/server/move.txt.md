# QW/server/move.txt

> Data: the author's notes on the movement model's state machine — the four probe heights and what each combination implies.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The design sketch behind [`pmove.c`](../client/pmove.c.md)'s position classification. It names the probes and lays out the
decision as a table, which the shipped code expresses as nested conditionals — so this file is the clearer statement of the same
thing.

## State

Data; no run-time state.

## What it records

**Four probes, each with a name that says what it decides:**

```text
floor  -- a test under the feet          -> is the player on ground?
feet   -- a test just above the feet     -> is the player wading?
waist  -- a test at the midpoint         -> is the player swimming?
head   -- a test at eye height           -> is the player submerged?
```

Those are exactly the four samples [`pmove.c`](../client/pmove.c.md#pm_catagorizeposition) takes, and the immersion levels zero
through three are the count of the last three that came back wet.

**The decision laid out by what is under the feet**: solid ground gives walking with friction scaled by how deep the player is;
liquid under the feet gives swimming or treading depending on the waist and head probes; air gives falling. The shipped code
follows this structure, with friction chosen by immersion level
([`pmove.c`](../client/pmove.c.md#pm_friction)) and the water mover selected at waist depth.

**A note that a dead player takes no input**, which is the death flag in the movement record
([`pmove.h`](../client/pmove.h.md)) and the early return in the acceleration functions.

**Notes** — the file is short and it is the right way to understand the classification: **the player's relationship to the world
is four boolean probes, and every movement mode is a combination of them.** A rebuild that writes the table first and the
conditionals second will get the edge cases — wading, treading, stepping out of water — right the first time.
