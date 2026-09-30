# WinQuake/d_ifacea.h

> The assembly's copy of the rasterizer interface: every record laid out as named byte offsets, with warnings that each must be kept in step with its C declaration by hand.

**Needs** — nothing; it is a list of constants
**Used by** — every `.s` and `.asm` file in the software renderer
**Tier floor** — T0 as written; a rebuild does not need this file at all

## Purpose

The assembly cannot read a C structure declaration, so this file restates each one as a set of byte
offsets and a size. Every definition carries a comment naming the other file it must match.

There is nothing to rebuild here. What is worth carrying out of it is the **list of layout facts the
assembly depends on**, because that list is the real cost of the assembly seam: nine records' field
orders and sizes are frozen by it, and changing any of them silently breaks the accelerated build while
the portable build keeps working. A rebuild that keeps hand-written inner loops should generate this
file rather than maintain it.

## State

Constants only.

## The duplicated constants

```text
CONSTANT alias_onseam  = 0x0020   # must match r_shared.h
CONSTANT turb_tex_size = 64       # must match d_iface.h
CONSTANT cycle         = 128      # must match d_iface.h
CONSTANT maxheight     = 1024     # must match r_shared.h
CONSTANT cache_size    = 32       # must match quakedef.h
CONSTANT particle_z_clip = 8.0
```

## The frozen layouts

```text
# Particle: the first two fields are the rasterizer's; the rest are not.
particle.org = 0 ;  .color = 12 ;  .next = 16 ;  .vel = 20
         .ramp = 32 ;  .die = 36 ;  .type = 40 ;  size = 44

# A model's final vertex: six fixed-point values then flags.
finalvert.v = 0 ;  .flags = 24 ;  .reserved = 28 ;  size = 32 ;  shift = 5

# A model's texture coordinate.
stvert.onseam = 0 ;  .s = 4 ;  .t = 8 ;  size = 12

# A model's quantized vertex.
trivertx.v = 0 ;  .lightnormalindex = 3 ;  size = 4

# The animated-model mesh descriptor.
affinetridesc.pskin = 0 ;  .pskindesc = 4 ;  .skinwidth = 8 ;  .skinheight = 12
             .ptriangles = 16 ;  .pfinalverts = 20 ;  .numtriangles = 24
             .drawtype = 28 ;  .seamfixupX16 = 32 ;  size = 36
```

**Invariants** — the final vertex's size is 32 bytes and its shift is 5, so indexing a vertex array is
a shift rather than a multiply. That is why the record carries a reserved field: without it the size
would be 28 and not a power of two.

The vertex offset within the final vertex carries the strongest warning in the file — that moving it
breaks code which pushes its address — because the assembly takes the address of the field rather than
of the record.

**Notes** — the only item here that is a *decision* rather than a transcription is the power-of-two
padding of the final vertex. A rebuild indexing an array of six-integer records should consider the
same padding for the same reason, and should measure whether it still matters.
