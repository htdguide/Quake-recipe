# WinQuake/draw.c

> The two-dimensional drawing operations for the software renderer: the bitmap font, images with and without transparency, the console backdrop, the tiled surround, and the screen fade.

**Needs** — [`draw.h`](draw.h.md) · [`vid.h`](vid.h.md) · [`wad.h`](wad.h.md) · [`d_iface.h`](d_iface.h.md) · [`zone.h`](zone.h.md) · [`common.h`](common.h.md) · [`console.h`](console.h.md) · [`client.h`](client.h.md)
**Used by** — [`console.c`](console.c.md) · [`sbar.c`](sbar.c.md) · [`menu.c`](menu.c.md) · [`screen.c`](screen.c.md) · [`common.c`](common.c.md)
**Tier floor** — none

## Purpose

The software implementation of [`draw.h`](draw.h.md). Mostly clipped byte copies; three parts are worth noting —
the image cache, the console backdrop's construction, and the fade.

## State

```text
CONSTANT max_cached_pics = 128
VARIABLE menu_cachepics : (name, Pic)[128]      # loaded-from-file images
VARIABLE draw_chars : bytes                     # the 128 by 128 font
VARIABLE draw_disc, draw_backtile : Pic
```

## `Draw_PicFromWad`, `Draw_CachePic`

**Contract** — fetch an image from the interface archive by lump name; or load one from the filesystem by path,
returning a previously loaded copy if there is one. A full cache is a fatal error, as is a missing file.

**Invariants** — two sources for images: the archive for the interface graphics, the filesystem for anything a
map or a modification adds. The cache is keyed by path and never evicts, which is why it is bounded at 128.

## `Draw_Init`

**Contract** — loads the font, the disc icon and the background tile from the archive.

**Invariants** — the font is a 128-by-128 image of 256 glyphs in a 16-by-16 grid, so a glyph is 8 by 8
([`draw.h`](draw.h.md#text)).

## `Draw_Character`, `Draw_String`, `Draw_DebugChar`

**Contract** — draw one glyph at a pixel position with palette index 255 transparent; draw a run of them; draw one
glyph **directly to the display**, bypassing the surface, at a position derived from a rolling counter.

```text
FUNCTION draw_character(x, y, num)
  num = num BITAND 255
  IF num == 32  RETURN                       # space: nothing to draw
  IF y <= -8    RETURN                       # entirely off the top
  clip against the surface's bounds
  source = the font AT ((num BITAND 15)*8, (num SHIFTED RIGHT 4)*8)
  FOR EACH of the 8 rows, and each of the 8 columns
    IF the source byte != 255  write it
```

**Invariants** — the glyph's position in the grid is the character code split into low and high nibbles, so the
high bit selects the second half of the grid — the bright glyph set
([`draw.h`](draw.h.md#text)).

Space is skipped outright, which is the common case in any string.

The debug variant writes to the display through the direct-drawing path
([`d_iface.h`](d_iface.h.md)) so it is visible when the frame loop is not running.

## `Draw_Pic`, `Draw_TransPic`, `Draw_TransPicTranslate`

**Contract** — draw an image opaquely; draw it treating index 255 as transparent; draw it remapping every byte
through a 256-entry table. An image extending outside the surface is a fatal error.

**Invariants** — **drawing outside the surface is fatal rather than clipped**, unlike the character drawing.
Every caller positions its images from the display's dimensions, so the check is an assertion about the
interface's layout rather than about input.

The translating variant serves one caller: the player model preview in the multiplayer setup
([`menu.c`](menu.c.md)).

## `Draw_ConsoleBackground`, `Draw_CharToConback`

**Contract** — draw the console's backdrop image scaled to the display's width for a given number of visible
lines; and, at startup, stamp the engine's version string into the backdrop image itself.

**Invariants** — **the version is drawn into the image once**, not composited per frame, which is why it appears
at a fixed place in the backdrop and scales with it. A small trick that saves per-frame work and is worth noting
as the reason the version text is part of the image.

The backdrop is scaled by a fixed-point step per column rather than resampled, so it is visibly nearest-neighbour
at high resolutions.

## `Draw_TileClear`

**Contract** — fills a rectangle with the repeating background texture, aligned to the display's origin so
adjacent calls tile seamlessly.

**Invariants** — used for the surround when the view is smaller than the display
([`screen.c`](screen.c.md)). Aligning to the display rather than to the rectangle is what makes several calls
produce one continuous pattern.

## `Draw_Fill`

**Contract** — fills a rectangle with one palette index, unclipped.

## `Draw_FadeScreen`

**Contract** — darkens the whole display by replacing every other pixel of every other row with a fixed dark
palette index.

```text
FUNCTION draw_fade_screen()
  FOR EACH row y
    # Offset the pattern by the row's parity, so the result is a checkerboard.
    start at column (y BITAND 1) ;  step by 2
    FOR EACH such pixel  write the dark index
```

**Invariants** — **the fade is a checkerboard of a fixed dark colour, not a blend.** With a palettized surface
there is no way to blend cheaply, so half the pixels are replaced and the eye averages them. That is why the
dimmed background behind a menu in this game has a visible dither pattern, and it is the single most
recognizable difference from the hardware renderer's version — which uses a real blended quad
([`gl_draw.c`](gl_draw.c.md)).

## `Draw_BeginDisc`, `Draw_EndDisc`

**Contract** — draw the disc icon directly to the display through the rasterizer's direct path, and erase it.

**Invariants** — must bypass the surface because it is shown during a load when nothing will flush it
([`common.c`](common.c.md#com_loadfile-and-its-four-wrappers)).

## `R_DrawRect8`, `R_DrawRect16`

**Contract** — copy a source bitmap into a display rectangle, for the two pixel widths, with optional
transparency.

**Invariants** — used by the rasterizer's rectangle path for backends that cannot expose a writable surface
([`d_iface.h`](d_iface.h.md)). Not exercised by any shipped backend.
