# WinQuake/resource.h

> Numeric identifiers for the Windows build's embedded resources: the application icon and the dialogue shown while the engine starts.

**Needs** — nothing
**Used by** — [`vid_win.c`](vid_win.c.md) · [`gl_vidnt.c`](gl_vidnt.c.md) · [`sys_win.c`](sys_win.c.md) · [`winquake.rc`](winquake.rc.md)
**Tier floor** — none

## Purpose

Generated identifiers, kept in the tree because the resource script and the code must agree on them.

## State

```text
CONSTANT an identifier for the application icon
CONSTANT identifiers for the startup dialogue and its controls
```

**Notes** — data, not design. The one fact worth carrying is that the Windows build shows a small dialogue while it loads,
which is what [`sys_win.c`](sys_win.c.md) replaces with the game window once initialization finishes — a loading indicator
for the period before the engine can draw one itself ([`gl_screen.c`](gl_screen.c.md) can only draw once video is up).
