# WinQuake/progdefs.q2

> Generated: the same engine-to-game layout as the original, extended by fifteen fields for the sequel's game logic, with its own checksum.

**Needs** — [`mathlib.h`](mathlib.h.md) · [`pr_comp.h`](pr_comp.h.md)
**Used by** — [`progdefs.h`](progdefs.h.md), when the sequel build switch is set
**Tier floor** — as [`progdefs.q1`](progdefs.q1.md)

## Purpose

An alternative layout, selected by a build switch, for a sequel-era game whose logic
declares more shared fields. It is never compiled in this build. Its value to a rebuilder
is what the differences reveal about which responsibilities were moving from the game
into the engine.

## State

Identical to [`progdefs.q1`](progdefs.q1.md) except for the additions below and a
different checksum.

## Added globals

```text
startspot : string        # which spawn point a level-change arrival should use,
                          # so a hub level can be re-entered from several doors
```

## Added entity fields

```text
null             : string   # a placeholder holding the field order stable
basevelocity     : vec3     # velocity contributed by whatever this entity is
                            # standing on, kept separate from its own velocity
drawPercent      : real
gravity          : real     # a per-entity gravity multiplier
mass             : real
light_level      : real     # how brightly lit this entity is
items2           : real     # a second 32-bit item set; the first ran out of bits
pitch_speed      : real     # the pitch counterpart of the original's yaw_speed
dmg              : real
dmgtime          : real
air_finished     : real     # when drowning begins
pain_finished    : real     # when the pain animation may be interrupted
radsuit_finished : real
speed            : real
```

## What the additions mean

**Separated base velocity** is the substantial one. In the original, an entity standing
on a moving platform has the platform's motion folded into its own velocity, which makes
"how fast am I actually moving" ambiguous and produces the well-known bugs with conveyor
belts and moving trains. Splitting it lets the engine add the carrier's motion each frame
without corrupting the entity's own. A rebuild implementing moving-platform physics should
adopt the split whichever layout it targets.

**Per-entity gravity and mass** move two constants from the game's arithmetic into the
engine's physics, so that a low-gravity entity no longer has to reimplement falling in
the interpreted language.

**A light level** hands the game the result of a lookup only the engine can do —
sampling the level's baked lighting at an entity's position — which is what monster
stealth behaviour needs.

**A second item set** is the direct consequence of the original's item bits reaching bit
31 ([`quakedef.h`](quakedef.h.md)) with three mission packs already reassigning them.

**Four "finished" timestamps** replace what the original game logic tracked in its own
private fields, moving drowning, pain interruption and powerup expiry into the shared
layout so the engine can participate.

**The placeholder string field** exists to keep the surrounding offsets where a
hand-written engine expected them. It is the clearest evidence that this layout was
edited by hand rather than purely generated, despite the header's claim.

## Checksum

```text
CONSTANT progheader_crc = 31586
```

**Invariants** — a different value from the original's, so the two layouts' compiled game
logic are mutually unloadable, by design.

**Notes** — this build never selects this file; the sequel engine is not in this tree.
A rebuild should treat it as documentation of the layout's evolution and implement the
original.
