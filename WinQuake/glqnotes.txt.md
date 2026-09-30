# WinQuake/glqnotes.txt

> Data: the hardware renderer's release note — the driver requirements, the console switches worth changing, and the known failure modes.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The document that explains *why* the hardware renderer has thirty console switches ([`glquake.h`](glquake.h.md)). It is a
troubleshooting guide, and read as a recipe page it is a list of the ways hardware of the era failed and which switch works
around each.

## State

Data; no run-time state.

## What it records

- The **minimum driver capability**: a full implementation, with a depth buffer, at a usable speed; a partial one is
  explicitly not supported.
- Which switches to change for which symptom — texture sorting off for transparency order, the blob lighting for frame rate,
  culling off for a driver that culls wrongly, the alternating depth scheme off when the screen tears, the maximum texture
  size down for limited texture memory.
- That **the software renderer remains the fallback** on hardware the driver does not serve. Two renderers shipping side by
  side was a requirement, not a luxury.
- The performance expectation, and that the frame rate is bounded by fill rate rather than by the engine.

**Invariants** — the guidance that **transparency ordering requires texture sorting to be off** is the user-visible face of
the conflict recorded in [`gl_rsurf.c`](gl_rsurf.c.md): sorting by texture destroys the front-to-back order that blending
needs. A rebuild resolves it by sorting opaque geometry and drawing blended geometry separately, and then needs no switch.

**Notes** — for a rebuilder, the file's value is the confirmation that every switch corresponds to a real observed failure.
None of them is speculative, and a rebuild on known hardware can delete them all.
