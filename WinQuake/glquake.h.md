# WinQuake/glquake.h

> The hardware renderer's private vocabulary: the texture upload interface, the frame-bracket seam, the extension pointers looked up at run time, and the switches that expose every rendering decision to the console.

**Needs** — [`gl_model.h`](gl_model.h.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)
**Used by** — every `gl_*` twin in this chapter
**Tier floor** — none

## Purpose

The counterpart to [`r_local.h`](r_local.h.md): where that header declares the internals of the span rasterizer, this
one declares the internals of the version that hands triangles to a graphics library. Reading both side by side is the
clearest statement of what the hardware port changed and what it left alone — and the answer is that it changed the
renderer entirely and the rest of the engine not at all.

A rebuilder choosing the hardware path reads this chapter instead of the software one
([`r_main.c`](r_main.c.md) onward), keeps everything else, and gets the same game.

## State

```text
# The frame bracket -- the only two calls the platform must provide
FUNCTION begin_rendering(out x, y, width, height)   # make the context current
FUNCTION end_rendering()                            # present

# Texture management
FUNCTION load_texture(identifier, width, height, indexed_pixels, mipmap, alpha) -> handle
FUNCTION find_texture(identifier) -> handle
FUNCTION upload_indexed(pixels, width, height, mipmap, alpha)
FUNCTION upload_true_colour(pixels, width, height, mipmap, alpha)
FUNCTION bind(handle)

RECORD Vertex           # position, texture coordinate, colour
  x, y, z : real ;  s, t : real ;  r, g, b : real

VARIABLE glx, gly, glwidth, glheight     # the viewport inside the window
VARIABLE gldepthmin, gldepthmax          # the depth range currently in force
VARIABLE texture_extension_number        # the next handle to hand out
VARIABLE currenttexture                  # what is bound, to avoid redundant binds
VARIABLE particletexture, playertextures, mirrortexturenum, skytexturenum
VARIABLE multitexture entry points, and whether they were found
VARIABLE vendor, renderer, version and extension strings as reported
```

**Invariants** —

- **Texture handles are allocated by the engine, not requested from the library.** A counter hands out the next
  number and the engine tells the library to use it. That is legal in the interface of the day and it means the engine
  can compute a handle for a player skin from the player's slot number. A rebuild on an interface that *returns*
  handles needs a table from slot to handle, and must keep the reserved ranges below distinct.
- **Handles are reserved in ranges**: the fixed textures first, then a run of one per player for translated skins,
  then everything the map loads. The ranges are what make "player 3's skin" addressable without a lookup.
- **Extension entry points are resolved at run time and their absence is a supported configuration.** Multitexture,
  the vertex-array calls and the subimage upload are each used only if found, with a slower path behind each. That is
  the same degrade-not-fail rule as [`net_wins.c`](net_wins.c.md), applied to the graphics library.
- The **currently bound texture is cached** and a redundant bind is skipped, because a bind was the expensive call on
  the hardware of the day. A rebuild should keep the check; it costs a comparison.
- The depth range is a **variable, not a constant**, because the view model is drawn into a compressed slice of it
  ([`gl_rmain.c`](gl_rmain.c.md#r_drawviewmodel)) so it cannot poke through walls.

## Constants

```text
CONSTANT alias_base_size_ratio = 1/11     # normalizes model triangle size
CONSTANT tile_size   = 128                # generated tiling surface
CONSTANT sky_shift   = 7 ;  sky_size = 128 ;  sky_mask = 127
CONSTANT backface_epsilon = 0.01
CONSTANT multitexture unit selectors
```

**Invariants** — the sky size being a power of two with a matching mask is load-bearing: the scrolling sky
([`gl_warp.c`](gl_warp.c.md)) wraps its coordinates by masking, not by a modulus.

## The console switches

**Contract** — roughly thirty named values, each readable and writable from the console, controlling: whether the
world, entities and view model are drawn at all; texture sorting; culling; polygon versus triangle-fan submission;
smooth versus flat model shading; affine versus perspective model texturing; the screen-tint blend; mirror and water
transparency; dynamic lighting either as a real light or as a cheap blended blob; shadows; the maximum texture size;
the player-skin detail reduction; whether to keep or remove shared edges; and the visibility override.

**Invariants** — that every one of these is a **run-time switch rather than a build option** is the single most
important thing this header records. The hardware of 1996 varied enormously, so the renderer ships every trade-off as
a dial and lets the player find the combination their card survives. A rebuild targeting varied hardware should copy
the practice; one targeting a known baseline can hard-wire its choices and delete the dials, and this list then reads
as the menu of decisions it must make.

**Notes** — the surface-cache record and the surface-drawing record are declared here and **unused**: they are
leftovers from the software renderer's header, kept so shared files compile. Do not implement them in a hardware
rebuild; the lightmap-block scheme in [`gl_rsurf.c`](gl_rsurf.c.md) replaces them entirely.

The particle record is duplicated here and in [`r_local.h`](r_local.h.md), with a comment warning that a third copy
exists in the assembly layout header ([`d_ifacea.h`](d_ifacea.h.md)). Three hand-synchronized copies of one record is
a hazard the source names and does not fix; a rebuild has one.
