# QW/Makefile.Linux

> Data: the build for every QuakeWorld variant on Unix — and therefore the authoritative statement of which files make the client, which make the server, and which are shared.

**Needs** — nothing
**Used by** — nothing in the engine; it is the build
**Tier floor** — none

## Purpose

The definitive answer to "what is the client and what is the server", and the clearest single statement of the QuakeWorld architecture.
Read it beside [`server/makefile`](server/makefile.md) and [`client/makefile.svgalib`](client/makefile.svgalib.md), which are narrower views
of the same thing.

## State

Data; no run-time state.

## What it records

**Three programs from two directories:**

| Program | Renderer | Video backend |
|---|---|---|
| Console software client | the span rasterizer | [`client/vid_svgalib.c`](client/vid_svgalib.c.md) |
| Windowed software client | the span rasterizer | [`client/vid_x.c`](client/vid_x.c.md) |
| Hardware client | a graphics library | [`client/gl_vidlinux_x11.c`](client/gl_vidlinux_x11.c.md) or [`client/gl_vidlinux_svga.c`](client/gl_vidlinux_svga.c.md) |
| Dedicated server | none | none |

**The shared object list** — the files that appear in *both* the client's and the server's link:

- the channel ([`client/net_chan.c`](client/net_chan.c.md)) and the transport
  ([`client/net_udp.c`](client/net_udp.c.md))
- **the movement model and its collision queries** ([`client/pmove.c`](client/pmove.c.md),
  [`client/pmovetst.c`](client/pmovetst.c.md)) — the single most important line in the file, because it is what makes prediction possible
- the foundation: the byte encoders and file system, the command interpreter, the settings, the allocators, the checksum, the vector maths,
  the digest

**Invariants** —

- **The renderer substitution is the same rule as the original's**
  ([`../WinQuake/Makefile.linuxi386`](../WinQuake/Makefile.linuxi386.md)): the hardware variant *replaces* the software renderer's objects
  rather than adding to them, and everything else is shared verbatim. That substitution list is the renderer boundary.
- **A build flag distinguishes the server**, and shared files test it to omit client-only paths
  ([`client/bothdefs.h`](client/bothdefs.h.md)). That conditional compilation is how one file serves two programs — and it is the part a
  rebuild should replace with an explicit interface rather than copy.
- **The portable inner loops are selected, not the hand-written ones**
  ([Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)), which is again the evidence that the assembly is
  optional.
- Debug and release configurations differ only in optimization and symbols.
- Packaging targets build installable archives from the templates beside this file
  ([`qwcl.spec.sh`](qwcl.spec.sh.md) and siblings), which record the installation layout the search path depends on.

**Notes** — absolute paths to the author's own machine are embedded throughout; they are not decisions. The two object lists and the shared
list are the content.
