# WinQuake/sbar.h

> The status bar: a fixed-height strip whose redraw discipline is stated in the header's own two-line comment.

**Needs** — nothing beyond the base types
**Used by** — [`screen.c`](screen.c.md) draws it; [`cl_parse.c`](cl_parse.c.md) and [`view.c`](view.c.md) notify it of changes; implemented by [`sbar.c`](sbar.c.md)
**Tier floor** — none

## Purpose

Four functions, and the header's whole content is the **redraw rule**: the bar is redrawn only when
something on it changed, but when anything does, the entire bar is redrawn for as many frames as the
display has pages.

That rule is the interface. It is stated here rather than in the implementation because every caller that
changes a displayed value has to invoke the notification, and forgetting to leaves a stale number on
screen.

## State

```text
CONSTANT sbar_height = 24            # scanlines, at the base resolution
VARIABLE sb_lines    : int           # how many scanlines are currently drawn
```

**Invariants** — the height is 24 scanlines and is not scaled with resolution in the software renderer:
the bar is drawn at its native size and the view shrinks to make room. The *visible* line count varies
because the player can hide the bar entirely or show an extended version with the score table.

The line count is shared with [`screen.h`](screen.h.md), which subtracts it from the view's height. So
changing the bar's visibility resizes the world view, which is why it forces a full renderer
reconfiguration ([`render.h`](render.h.md#per-frame)).

## Entry points

**Contract** — `Sbar_Init` loads the bar's images and registers its commands.
`Sbar_Changed` must be called whenever any displayed statistic changes; it arms the redraw for the
display's page count. `Sbar_Draw` is called every frame by the screen composer and draws nothing when
the redraw counter has expired. `Sbar_IntermissionOverlay` draws the end-of-level statistics.
`Sbar_FinaleOverlay` draws the end-of-episode text.

**Invariants** — **`Sbar_Draw` being called every frame and usually doing nothing** is the shape the
page-counting rule requires: the decision about whether to draw belongs to the bar, not to its caller,
because only the bar knows whether its counter has expired.
