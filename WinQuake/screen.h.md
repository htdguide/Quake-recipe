# WinQuake/screen.h

> The frame composer: one call that draws everything, plus the damage-tracking flags that decide how much of the display must be refreshed.

**Needs** — [`vid.h`](vid.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — [`host.c`](host.c.md) calls the update every frame; [`console.c`](console.c.md), [`sbar.c`](sbar.c.md), [`menu.c`](menu.c.md), [`view.c`](view.c.md), [`cl_parse.c`](cl_parse.c.md), [`host_cmd.c`](host_cmd.c.md) and every video backend set its flags; implemented by [`screen.c`](screen.c.md) and [`gl_screen.c`](gl_screen.c.md)
**Tier floor** — none

## Purpose

One function composes a frame: the world, then the weapon, then the status bar, then the console or the
menu, then any overlay. Everything else here is about **how much to refresh** — because a software
renderer that redraws the whole display every frame wastes most of its time, and the engine instead
tracks what changed.

## State

```text
VARIABLE scr_con_current : real      # how many console lines are visible now
VARIABLE scr_conlines    : real      # how many it is sliding toward
VARIABLE scr_fullupdate  : int       # set to 0 to force a complete redraw
VARIABLE sb_lines        : int       # how many scanlines the status bar occupies
VARIABLE clearnotify     : int       # cleared when notification text is drawn
VARIABLE scr_disabled_for_loading : bool
VARIABLE scr_skipupdate  : bool
VARIABLE scr_copytop     : int       # only the top portion changed
VARIABLE scr_copyeverything : int    # the whole display changed
VARIABLE block_drawing   : bool
VARIABLE scr_viewsize    : Cvar      # the view's size as a percentage
```

**Invariants** — the two copy flags and the full-update counter together are the damage model, and it
interacts with the surface's page count ([`vid.h`](vid.h.md)): with two pages, anything drawn must be
drawn again next frame, so the counter is *set to the page count* and decremented rather than being a
boolean. That is the pattern every interface element uses — the status bar, the console and the menus
each keep their own such counter.

The console's visible-line count is **animated**: a current value chases a target, which is what makes
the console slide. Both are floats because the animation is in fractional lines.

`scr_disabled_for_loading` suppresses updates during a level load so that partial frames are not shown —
and it is cleared by the error path ([`host.c`](host.c.md#host_error)) precisely so that an error during
a load is visible.

## Entry points

**Contract** — `SCR_Init` registers the screen's variables and loads its images. `SCR_UpdateScreen`
composes and presents one frame; it is the only caller of the renderer.
`SCR_UpdateWholeScreen` forces a complete refresh. `SCR_SizeUp` and `SCR_SizeDown` step the view size.
`SCR_BringDownConsole` forces the console shut. `SCR_CenterPrint` shows a message in the middle of the
screen for a few seconds. `SCR_BeginLoadingPlaque` and `SCR_EndLoadingPlaque` bracket a load with a
"loading" overlay. `SCR_ModalMessage` shows a message and **blocks** until a key is pressed, returning
whether it was affirmative.

**Invariants** — the modal message is the only blocking input wait in the engine, and it pumps the
platform's event queue itself. It is used for the quit confirmation on a dedicated build and for network
errors. A rebuild with a real event loop should make it asynchronous, but must then handle the caller's
expectation of a synchronous answer.

The loading overlay pair is not symmetric in effect: beginning it also forces a full update and disables
further updates, and ending it re-enables them. So a load that fails without ending it leaves the screen
frozen — which is why the error path calls the end explicitly.
