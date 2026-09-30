# WinQuake/spritegn.h

> The sprite format: palette-indexed bitmaps with an origin offset and one of five orientation rules that decide how each faces the camera.

**Needs** — [`mathlib.h`](mathlib.h.md)
**Used by** — [`model.c`](model.c.md) and [`gl_model.c`](gl_model.c.md) (the loader) · [`r_sprite.c`](r_sprite.c.md) and [`gl_rmain.c`](gl_rmain.c.md) (the renderers)
**Tier floor** — none; a byte layout

## Purpose

Explosions, bubbles, flames and the lightning bolt are sprites: flat bitmaps that turn
to face the viewer. The format is the simplest in the engine, and almost all of its
content is one field — the orientation rule — which decides *how* a sprite turns, and
whose five settings each exist for a specific effect in the game.

## State

A file format. No runtime state.

## The file

```text
RECORD SpriteHeader
  ident          : int      # the four bytes "IDSP", little-endian
  version        : int      # must be 1
  type           : int      # the orientation rule; see below
  boundingradius : real
  width, height  : int      # the largest frame's dimensions
  numframes      : int
  beamlength     : real     # how far the sprite extends along its axis;
                            # meaningful only for the oriented rules
  synctype       : int      # 0 synchronized to the global clock, 1 randomized

# Then `numframes` entries, each either single or a group:
RECORD FrameType { type : int }         # 0 single, 1 group
#   single: a Frame header, then width*height bytes of palette indices
#   group:  a count, then that many intervals (one real each),
#           then that many (Frame header + bitmap) blocks

RECORD Frame
  origin        : int[2]    # the offset from the sprite's world position to
                            # its top-left corner, in pixels
  width, height : int       # this frame's own size
```

## The orientation rules

```text
ENUM SpriteType
  vp_parallel_upright    = 0  # faces the view plane, but stays vertical:
                              # it yaws to the camera and never pitches
  facing_upright         = 1  # rotates to face the camera's POSITION rather
                              # than its plane, and stays vertical
  vp_parallel            = 2  # faces the view plane exactly: the usual
                              # billboard, tilting with the camera
  oriented               = 3  # fixed in the world by the entity's own angles;
                              # does not turn at all
  vp_parallel_oriented   = 4  # faces the view plane, then rolls by the
                              # entity's own roll angle
```

**Invariants** — the difference between rule 0 and rule 2 is whether the sprite pitches
with the camera. A tall flame must not, or it leans over when the player looks up; an
explosion must, or it reads as a flat card. The difference between rule 0 and rule 1 is
view-plane-parallel versus view-point-facing, which only shows at the screen edges
under a wide field of view.

Rule 3 is not a billboard at all — it is a flat quad placed by the entity's angles, and
it is how the lightning bolt and a few decals are drawn. Rule 4 is rule 2 plus a
per-entity roll, used to make a spinning effect out of a single frame.

The five rules are the format's reason to exist, and a rebuild that implements only
rule 2 will get most sprites subtly and some sprites grossly wrong. Each is a basis
construction, given in [`r_sprite.c`](r_sprite.c.md#r_getspriteframe).

## Invariants a rebuild must honour

**Per-frame size, per-file maximum.** The header's dimensions are the largest frame's;
each frame carries its own. The software renderer allocates working space from the
header's and draws at the frame's.

**The origin is a pixel offset, and it is usually negative.** It gives where the
bitmap's top-left corner sits relative to the entity's world position, so a
conventionally centred sprite has an origin of about minus half its width and plus
half its height. Getting the sign wrong displaces every effect in the game by its own
size.

**Index 255 is transparent.** The format has no alpha channel and no transparency flag;
the last palette entry is the convention, shared with every other palette-indexed
asset in the engine.

**Frame groups time the same way animated models do.** One interval per frame, giving
cumulative end times from the group's start.

## Notes

The header's bounding radius and beam length are read by the loader and used only for
culling and for the oriented rules respectively.

As with the model format, there is no length validation: the loader must check the
frame count and dimensions against the file size, and the original does not.

The file's own comment describes the layout as a repetition with a conditional inside,
which is exactly how it must be parsed — the frame table is not an array of fixed-size
records and cannot be indexed without walking it. The loader therefore builds an index
at load time.
