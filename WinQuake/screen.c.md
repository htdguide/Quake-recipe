# WinQuake/screen.c

> Composes each frame: the world, the weapon, the status bar, the console or menu, and the overlays — plus the damage-tracking that decides how much of the display must be refreshed, and the screenshot writer.

**Needs** — [`screen.h`](screen.h.md) · [`client.h`](client.h.md) · [`render.h`](render.h.md) · [`vid.h`](vid.h.md) · [`draw.h`](draw.h.md) · [`console.h`](console.h.md) · [`sbar.h`](sbar.h.md) · [`menu.h`](menu.h.md) · [`view.h`](view.h.md) · [`keys.h`](keys.h.md) · [`sound.h`](sound.h.md) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`host.c`](host.c.md) calls the update once per frame
**Tier floor** — none

## Purpose

One function composes a frame and everything else here supports it. The substantial content is the **damage
model**: a software renderer that refreshed the whole display every frame would waste most of its time, so each
interface element tracks whether it changed and the composer flushes only the rectangles that did.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## `SCR_CalcRefdef`

**Contract** — computes the view's rectangle and field of view from the size setting and the status bar's height,
and notifies the renderer. Called whenever the size, the bar's visibility or the display's shape changes.

```text
FUNCTION scr_calc_refdef()
  scr_fullupdate = 0                      # force a complete redraw
  # The bar's height reduces the view unless the view is full-screen.
  sb_lines = 24 plus the extended rows, OR 0 when the bar is hidden
  r_refdef.fov_x = the field-of-view setting
  r_refdef.fov_y = calc_fov(fov_x, the view's width, its height)
  r_set_vrect(the display's rectangle, r_refdef.vrect, sb_lines)
  r_view_changed(...)
  # The 2D drawing rectangle is the view's, or the whole display when the view
  # is full-screen.
  scr_vrect = ...
```

**Invariants** — the vertical field of view is **derived from the horizontal one and the aspect ratio**, not
configured, so widening the view shows more horizontally rather than stretching.

Changing the view size forces a complete redraw, because the surround must be re-tiled.

## `CalcFov`

**Contract** — takes a horizontal field of view and a rectangle; returns the vertical field of view. A value
outside 1 to 179 degrees is a fatal error.

## `SCR_SizeUp_f`, `SCR_SizeDown_f`

**Contract** — step the view size setting by ten percent and recompute.

## `SCR_CenterPrint`, `SCR_DrawCenterString`, `SCR_CheckDrawCenterString`, `SCR_EraseCenterString`

**Contract** — record a message and a display deadline; draw it centred, line by line; draw it while the deadline
has not passed, counting down; and mark the area it occupied as needing a redraw.

**Invariants** — the erase step exists because the message sits over the world and the world's own redraw does
not cover the area when the view is smaller than the display. With two display pages the erase must happen twice
([`vid.h`](vid.h.md)).

## The overlay indicators

**Contract** — `SCR_DrawRam` shows a symbol when the surface cache is thrashing;
`SCR_DrawTurtle` shows one when the frame rate falls below ten; `SCR_DrawNet` shows one when no message has
arrived recently; `SCR_DrawPause` shows the paused graphic; `SCR_DrawLoading` shows the loading graphic.

**Invariants** — **the thrashing indicator is how a player learns the surface cache is too small**
([`d_surf.c`](d_surf.c.md)), and the network indicator is how they learn packets are being lost. Both are
diagnostics surfaced to the player rather than to a log, which is the right choice for a game and worth
reproducing.

## `SCR_SetUpToDrawConsole`, `SCR_DrawConsole`

**Contract** — advance the console's slide animation toward its target height, forcing it fully open when there is
no world to draw; and draw it, or the notification lines when it is closed.

**Invariants** — the slide is animated in fractional lines
([`screen.h`](screen.h.md)) and the speed is a tunable. Forcing it open with no world is what makes the console
fill the screen at startup.

## `SCR_BeginLoadingPlaque`, `SCR_EndLoadingPlaque`

**Contract** — stop all sounds, force a complete redraw, draw the loading graphic, present it, then suppress
further updates; and re-enable them, forcing another complete redraw.

**Invariants** — the loading graphic is drawn and presented **once**, then updates are suppressed, so the screen
holds that image for the whole load without the frame loop running. Failing to end the plaque leaves the screen
frozen, which is why the error path ends it explicitly
([`host.c`](host.c.md#host_error)).

## `SCR_ModalMessage`, `SCR_DrawNotifyString`

**Contract** — show a message and block until a key is pressed, pumping the platform's event queue and presenting
each frame; return whether the key was affirmative.

**Invariants** — the only blocking input wait in the engine
([`screen.h`](screen.h.md)), and it works by driving the platform's event pump itself.

## `WritePCXfile`, `SCR_ScreenShot_f`

**Contract** — write the display's contents to a numbered file in a run-length-encoded palettized image format,
finding the first unused name among a hundred candidates.

**Invariants** — the format is run-length encoded with its own header and a trailing palette. A rebuild should
write a modern format; nothing depends on this one.

Probing a hundred fixed names rather than listing a directory is a consequence of the platform seam having no
enumeration ([`sys.h`](sys.h.md)).

## `SCR_UpdateScreen`

**Contract** — composes and presents one frame. Returns immediately when updates are suppressed or the display is
blocked. Recomputes the view when anything changed, then draws: the world and weapon through the view, then the
status bar, then whichever of the console, the menu, the loading graphic, the end-of-level overlay or the modal
message applies, then the overlay indicators. Finally flushes the changed rectangles.

```text
FUNCTION scr_update_screen()
  IF updates are suppressed OR the display is blocked  RETURN
  IF the video mode or the view settings changed  scr_calc_refdef()
  IF scr_fullupdate is below the display's page count
    scr_fullupdate = scr_fullupdate + 1 ;  scr_copyeverything = 1

  set up the console's slide
  scr_setuptodrawconsole()
  # Draw the world, unless something opaque covers it.
  IF NOT (the console is fully down OR a menu is up OR at an intermission)
    v_render_view()
  lock the surface
  IF at an intermission
    sbar_intermission_overlay()   # or the finale overlay
  ELSE
    sbar_draw()
    draw the centre-print message if one is pending
    draw the loading, pause, thrash, turtle and net indicators
    draw the console or the notification lines
    m_draw()                       # the menu, over everything
  unlock the surface
  # Flush: everything, the top portion, or just the view.
  IF scr_copyeverything     vid_update(the whole display)
  ELSE IF scr_copytop       vid_update(the view plus the bar)
  ELSE                      vid_update(the view)
```

**Invariants** — five things.

**The world is skipped entirely when something opaque covers it**, which is why the frame rate rises with the
console open and why a menu is cheap.

**The full-update counter is compared against the display's page count and incremented**, rather than being a
boolean, because with two pages anything drawn must be drawn twice
([`vid.h`](vid.h.md)). Every interface element uses the same idiom.

**The flush has three granularities** — everything, the top portion, or the view — and the choice is made from
flags the interface elements set. That is the damage model, and it is the difference between refreshing 64,000
pixels and refreshing 300,000.

**The surface is locked around all two-dimensional drawing** and not around the world, because the renderer
manages its own locking ([`d_init.c`](d_init.c.md)).

**The menu is drawn last**, over everything including the console, which is the intended layering.

## `SCR_Init`

**Contract** — registers the screen's tunables and commands and loads the overlay graphics.
