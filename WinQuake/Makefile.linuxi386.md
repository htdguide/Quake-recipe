# WinQuake/Makefile.linuxi386

> Data: the build description, and therefore the authoritative list of which source files make up each of the three engine variants.

**Needs** — nothing
**Used by** — nothing in the engine; it is the build
**Tier floor** — none

## Purpose

The recipe's most useful build file, because it is the only place in the tree that states **which files belong to which
build**. Three engines are produced from this one source directory, and nothing in the source says so:

| Variant | Renderer | Video backend | Notes |
|---|---|---|---|
| Console software | span rasterizer | [`vid_svgalib.c`](vid_svgalib.c.md) | takes the display exclusively |
| Windowed software | span rasterizer | [`vid_x.c`](vid_x.c.md) | converts indices to display pixels |
| Hardware | graphics library | [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md) | the `gl_*` chapter replaces the renderer |

**Invariants** —

- **The hardware variant substitutes files, it does not add them.** Its object list replaces
  [`r_*.c`](r_main.c.md) and [`d_*.c`](d_scan.c.md) with [`gl_*.c`](gl_rmain.c.md), and
  [`model.c`](model.c.md), [`screen.c`](screen.c.md), [`draw.c`](draw.c.md),
  [`r_light.c`](r_light.c.md), [`r_efrag.c`](r_efrag.c.md) and [`r_misc.c`](r_misc.c.md) with their `gl_` counterparts.
  Everything else — the server, the game interpreter, the network, the sound, the console, the interface — is shared
  verbatim. That substitution list *is* the renderer boundary, and it is the single most valuable fact for a rebuilder
  choosing which renderer to implement.
- **The portable inner loops are used, not the assembly.** This build defines the flag that selects them
  ([`d_scan.c`](d_scan.c.md), [`r_draw.c`](r_draw.c.md)), which is the direct evidence that the assembly is optional
  ([Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)).
- Each variant has a **debug and a release configuration**, differing only in optimization and symbols.
- The compiler flags pin the floating-point behaviour and the structure packing, which matters because the record layouts
  are shared with assembly in other builds ([`asm_i386.h`](asm_i386.h.md)).

## State

Data; no run-time state.

## What else it records

Packaging targets that build installable archives from the four content sets — the base game, its data, and the two
official expansions — using the specification files beside it
([`quake.spec.sh`](quake.spec.sh.md) and siblings). Those record the **file layout an installation must have**, which is
information the engine itself only implies ([`common.c`](common.c.md)'s search-path rules).

**Notes** — absolute paths to the author's own machine are embedded throughout. They are not a decision; a rebuild
parameterizes them. The interesting content is the three object lists and the substitution rule above.
