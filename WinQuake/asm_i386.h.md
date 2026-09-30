# WinQuake/asm_i386.h

> The assembly's copy of the layouts shared between the server, the sound mixer and their hand-written inner loops, plus the name-decoration macro that lets one source serve three assemblers.

**Needs** — nothing; a list of constants
**Used by** — [`worlda.s`](worlda.s.md) · [`snd_mixa.s`](snd_mixa.s.md) · [`math.s`](math.s.md) · [`sys_wina.s`](sys_wina.s.md) · and every other `.s`/`.asm` file indirectly through [`quakeasm.h`](quakeasm.h.md)
**Tier floor** — T0 as written; a rebuild does not need the file

## Purpose

Two jobs. It restates five C record layouts as byte offsets so the assembly can reach their fields, and
it defines the symbol-decoration macro that reconciles three assemblers' different conventions for how
a C name appears to assembly.

Both are pure translation. What survives into a recipe is the **inventory of frozen layouts**, because
it names exactly which records a rebuild cannot rearrange without touching the accelerated build — and
one of the warnings here is more consequential than the rest.

## State

Constants only.

## Name decoration

```text
# On one toolchain a C symbol appears to assembly unchanged; on the others it
# gains a leading underscore. One macro hides the difference.
MACRO C(label) = label            WHEN building for the ELF toolchain
                 "_" + label      OTHERWISE
```

**Notes** — the same problem the build-time translator ([`gas2masm/`](gas2masm/README.md)) exists to
solve at a larger scale. A rebuild with one toolchain deletes it.

## The frozen layouts

```text
# The in-memory plane. The strongest warning in the tree attaches here:
# changing its SIZE breaks the indexed lookup in the collision point query.
plane.normal = 0 ;  .dist = 12 ;  .type = 16 ;  .signbits = 17 ;  .pad = 18
      size = 20

# A collision tree.
hull.clipnodes = 0 ;  .planes = 4 ;  .firstclipnode = 8 ;  .lastclipnode = 12
    .clip_mins = 16 ;  .clip_maxs = 28 ;  size = 40

# An on-disk BSP node — read directly by the renderer's culling assembly.
dnode.planenum = 0 ;  .children = 4 ;  .mins = 8 ;  .maxs = 20
     .firstface = 32 ;  .numfaces = 36 ;  size = 40

# A decoded sound.
sfxcache.length = 0 ;  .loopstart = 4 ;  .speed = 8 ;  .width = 12
        .stereo = 16 ;  .data = 20

# A playing sound channel.
channel.sfx = 0 ;  .leftvol = 4 ;  .rightvol = 8 ;  .end = 12 ;  .pos = 16
       .looping = 20 ;  .entnum = 24 ;  .entchannel = 28 ;  .origin = 32
       .dist_mult = 44 ;  .master_vol = 48 ;  size = 52

# One stereo sample in the mixer's accumulator.
samplepair.left = 0 ;  .right = 4 ;  size = 8
```

**Invariants** — the plane's **size of 20 bytes** is the load-bearing number, and the comment says why:
the collision point query indexes an array of planes by scaling the index by that constant
([`worlda.s`](worlda.s.md)). The record is 19 bytes of content padded to 20 — three floats, a byte, a
byte, and two pad bytes — so the padding is not alignment, it is a *chosen* stride. A rebuild that
widens the plane must change the assembly's scale factor, and a rebuild that narrows it to 19 breaks
alignment on the three floats.

The on-disk node appearing here is the one place the assembly reads a *file* layout rather than a
memory one, because the collision trees are used unconverted
([`model.c`](model.c.md#mod_loadclipnodes)).

The channel and sound layouts are here because the **audio mixer's inner loop is also assembly**
([`snd_mixa.s`](snd_mixa.s.md)) — the only non-graphics use of the assembly seam.
