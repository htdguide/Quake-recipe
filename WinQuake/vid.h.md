# WinQuake/vid.h

> The video seam: an eight-call interface over a palette-indexed framebuffer, and the record through which the rest of the engine learns its shape.

**Needs** — nothing; this is a leaf
**Used by** — every drawing module: [`draw.c`](draw.c.md) · [`screen.c`](screen.c.md) · [`d_init.c`](d_init.c.md) and the whole software renderer · [`console.c`](console.c.md) · [`sbar.c`](sbar.c.md) · [`menu.c`](menu.c.md) · and implemented once per platform by [`vid_win.c`](vid_win.c.md), [`vid_dos.c`](vid_dos.c.md), [`vid_svgalib.c`](vid_svgalib.c.md), [`vid_x.c`](vid_x.c.md), [`vid_vga.c`](vid_vga.c.md), [`vid_sunx.c`](vid_sunx.c.md), [`vid_sunxil.c`](vid_sunxil.c.md), [`vid_null.c`](vid_null.c.md) and the hardware backends
**Tier floor** — none; this is the seam

## Purpose

This file is [Seam: Framebuffer
surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface) written down. Nine
implementations exist in the tree and the interface is eight calls and one record,
which is the useful fact: bringing this engine up on a new display system is a day's
work, not a port.

The record is more interesting than the calls. Everything downstream — the renderer's
working buffers, the console's size, the status bar's placement, the sky warp's cost —
is derived from fields the backend fills in, so a backend can hand the engine an
unusual surface (a window narrower than the console, a 16-bit destination, a
direct-to-display pointer) and the engine adapts.

## State

```text
CONSTANT vid_cbits  = 6
CONSTANT vid_grades = 64        # shading levels in the lighting table

TYPE pixel = byte               # one palette index

RECORD Rect                     # a rectangle, and a list of them
  x, y, width, height : int
  next : optional<Rect>

RECORD VidDef                   # global; the backend fills it, everyone reads it
  buffer        : pixel[]       # the off-screen surface the engine draws into
  colormap      : bytes         # 256 * 64: palette index by shade level
  colormap16    : list<int (16-bit)>   # the same, for a 16-bit destination
  fullbright    : int           # the first palette index that ignores lighting
  rowbytes      : int           # may exceed width, when the surface is a window
                                # into something larger
  width, height : int
  aspect        : real          # width divided by height; below zero means
                                # taller than wide
  numpages      : int           # 1 for single-buffered, 2 for double
  recalc_refdef : int           # set by the backend to make the renderer
                                # recompute everything derived from this record
  conbuffer     : pixel[]       # the surface the console draws into
  conrowbytes   : int
  conwidth      : int           # the console's own dimensions, which need not
  conheight     : int           # match the view's
  maxwarpwidth  : int           # the largest sky-warp working buffer the
  maxwarpheight : int           # backend will allow

VARIABLE vid           : VidDef
VARIABLE d_8to16table  : list<int (16-bit)>[256]   # palette to 16-bit colour
VARIABLE d_8to24table  : list<int (32-bit)>[256]   # palette to 32-bit colour
VARIABLE vid_menudrawfn : handler                  # the backend's own menu page
VARIABLE vid_menukeyfn  : handler
```

**Invariants** — `rowbytes` rather than `width` is the stride, always. Every drawing
loop in the engine advances by it, and a backend whose surface is a window into a
larger buffer sets the two differently. A rebuild that conflates them works until the
first windowed mode.

The **console surface may differ from the view surface**. At high resolutions the
console is drawn at a lower one and scaled, because a 640-pixel-wide console font is
unreadably small. So there are two sizes, two strides and two buffers, and code that
draws interface elements must pick the right pair.

The **recalculation flag** is how a backend tells the renderer that the record has
changed: everything from the field of view to the surface cache size is derived from
it, and the derivation is deferred to the start of the next frame rather than performed
inside the backend.

`numpages` being 1 or 2 changes the engine's damage tracking: with one page the engine
must redraw everything it dirtied; with two it must redraw what it dirtied in *either*
of the last two frames. That is why the status bar and console track their own dirty
counts.

## The lighting table

**Contract** — a 256-by-64 table mapping a palette index and a shading level to a
palette index. The software renderer's entire lighting model is a lookup in it: a
surface's light value picks a row, and every texel's index picks a column.

**Invariants** — 64 shading levels, from the constant. The table is loaded from an
asset, not computed, so it is a *given* for a rebuild that wants to look like the
original. Indices at or above the fullbright threshold are exempt: those palette
entries are light sources in the artwork and are copied through unshaded.

The 16-bit variant exists for backends whose surface is 16-bit; its entries are the
same shading applied in a packed colour space.

**Notes** — replacing this table with real arithmetic is the single largest visual
change a rebuild can make, and it is why hardware-rendered Quake does not look like
software-rendered Quake. The table is not a linear ramp — it is hand-authored per
palette row.

## `VID_Init`

**Contract** — takes 256 RGB triples; brings up a surface and fills in the record. The
palette data is **not retained**: a backend that needs it later must copy it.

## `VID_Shutdown`

**Contract** — releases the surface and restores the display. Must be safe to call
after a failed initialization, because the fatal-error path calls it.

## `VID_SetPalette`

**Contract** — takes 256 RGB triples and installs them. Called at startup and after any
gamma change.

## `VID_ShiftPalette`

**Contract** — takes 256 RGB triples and installs them as a transient variation. Used
for damage and pickup flashes and for the underwater tint.

**Notes** — the two palette calls are separate so that a backend can implement the
transient one more cheaply — with hardware gamma, say — while the permanent one
rebuilds derived tables. Several backends make them the same call.

## `VID_Update`

**Contract** — takes a list of rectangles; copies exactly those regions of the
off-screen surface to the display. The list is the engine's damage report, and honouring
it rather than the whole surface is where a software renderer's frame time goes.

**Invariants** — the rectangles may overlap and are not sorted. A backend may ignore the
list and push everything; correctness does not depend on the list, only speed.

## `VID_SetMode`

**Contract** — takes a mode number and a palette; switches to that mode, refilling the
record, and returns whether it succeeded. Failure must leave a working surface — the
engine's own use of this call is to *fall back* to the base mode after an allocation
failure at a larger one.

## `VID_HandlePause`

**Contract** — takes a paused flag; lets a backend release an exclusive input grab
while the game is paused.

**Notes** — the header's comment says only one platform needs it. A rebuild should keep
it, because every modern windowed environment has the same problem.

## `VID_LockBuffer`, `VID_UnlockBuffer`

**Contract** — bracket a run of writes to the surface. Declared in
[`quakedef.h`](quakedef.h.md) rather than here, and empty on every platform but one.

**Invariants** — the brackets appear around every drawing operation in the engine,
nested. A backend whose surface must be locked has to tolerate nesting or count.

## The backend's menu page

**Contract** — two handler variables a backend fills in to add its own page to the
options menu: one to draw it, one to handle a key. A backend that sets neither gets no
page.

**Notes** — this is the one place the seam runs backwards, with the platform code
contributing interface rather than just servicing calls. It exists because resolution
selection is irreducibly platform-specific. A rebuild should keep the inversion; the
alternative is a lowest-common-denominator mode list.
