# WinQuake/draw.h

> The only functions outside the renderer permitted to touch the drawing surface: fourteen two-dimensional operations for the console, the status bar and the menus.

**Needs** — [`wad.h`](wad.h.md) (the image type) · [`vid.h`](vid.h.md)
**Used by** — [`console.c`](console.c.md) · [`sbar.c`](sbar.c.md) · [`menu.c`](menu.c.md) · [`screen.c`](screen.c.md) · [`common.c`](common.c.md) (the disc indicator) · implemented by [`draw.c`](draw.c.md) and [`gl_draw.c`](gl_draw.c.md)
**Tier floor** — none

## Purpose

The header's own first line is the contract: these are the **only** functions outside the renderer
allowed to write to the display surface. Everything the player reads — the console, the status bar, the
menus, the score table — is drawn through these fourteen calls, and both renderers implement all of
them.

That makes this the interface a rebuild implements twice, and it is deliberately small: characters,
images, a fill, a tile, a fade, and two bracketing calls for the disc indicator.

## State

```text
VARIABLE draw_disc : Pic        # the disc-access icon, shared with the status bar
```

## Text

**Contract** — `Draw_Character` draws one glyph of the 8-by-8 bitmap font at a pixel position, with
palette index 255 transparent. `Draw_String` draws a run of them. `Draw_DebugChar` draws one glyph
directly to the display, bypassing the surface, for debugging when the normal path is not running.

**Invariants** — the font is a 128-by-128 image of 256 glyphs in a 16-by-16 grid, loaded from the
interface archive. **The high bit of a character selects the second half of the grid**, which is the
same glyphs in a brighter colour — so byte values 128 through 255 are not an extended character set
([`SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)). Chat messages and
server banners set the high bit to highlight themselves.

Glyph 0 is blank and is skipped, so a string containing a zero byte would terminate anyway.

## Images

**Contract** — `Draw_Pic` draws an image opaquely; `Draw_TransPic` treats palette index 255 as
transparent; `Draw_TransPicTranslate` additionally remaps every pixel through a 256-byte translation
table. `Draw_PicFromWad` fetches an image from the interface archive by lump name;
`Draw_CachePic` loads one from the filesystem by path and caches it.

**Invariants** — the translating variant exists for exactly one caller: drawing a player's model in the
multiplayer setup menu with their chosen shirt and trouser colours applied
([`menu.c`](menu.c.md)). That is why the translation is a parameter rather than global state.

## Background and clearing

**Contract** — `Draw_ConsoleBackground` draws the console's backdrop image for a given number of visible
lines, scaled and tinted. `Draw_TileClear` fills a rectangle with the repeating background texture, used
for the area beside a reduced-size view. `Draw_Fill` fills a rectangle with one palette index.
`Draw_FadeScreen` darkens the whole screen, which is what dims the world behind a menu.

**Invariants** — the tile clear exists because the view can be smaller than the display
([`screen.c`](screen.c.md)), and the surround must be filled with something — a repeating texture rather
than a flat colour, so that a reduced view looks intentional.

The fade is implemented very differently in the two renderers — a per-pixel palette walk in software, a
blended quad in hardware — and the visual results differ noticeably. A rebuild should pick one and
accept that it will not match both.

## The disc indicator

**Contract** — `Draw_BeginDisc` shows the disc icon in the corner **directly on the display**, bypassing
the off-screen surface; `Draw_EndDisc` removes it.

**Invariants** — it must bypass the surface because it is shown *during* a file load
([`common.c`](common.c.md#com_loadfile-and-its-four-wrappers)), when the frame loop is not running and nothing will flush the
surface to the display. That is the whole reason the direct-drawing path exists in the rasterizer
interface ([`d_iface.h`](d_iface.h.md)).

The hardware renderer implements both as nothing, because there is no way to poke the display outside a
frame.
