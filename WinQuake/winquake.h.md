# WinQuake/winquake.h

> The shared declarations of the Windows platform layer: the window and its state, the sound and input device handles, and the flags the backends use to coordinate focus, mode changes and pausing.

**Needs** — nothing in the recipe; it pulls in the platform's own headers
**Used by** — [`vid_win.c`](vid_win.c.md) · [`gl_vidnt.c`](gl_vidnt.c.md) · [`in_win.c`](in_win.c.md) · [`snd_win.c`](snd_win.c.md) · [`cd_win.c`](cd_win.c.md) · [`sys_win.c`](sys_win.c.md) · [`net_wins.c`](net_wins.c.md) · [`net_wipx.c`](net_wipx.c.md)
**Tier floor** — none

## Purpose

The Windows backends are several files that must agree about one window and one focus state, so those facts live here.
Reading it tells you exactly how much shared mutable state a platform layer needs, which is the only reason it is in the
recipe.

## State

```text
VARIABLE mainwindow, the window class, the instance
VARIABLE ActiveApp, Minimized                 # focus, shared by every backend
VARIABLE WinNT                                # which platform generation
VARIABLE the sound device handles and buffers
VARIABLE the mouse and joystick device handles and saved parameters
VARIABLE the display contexts, the palette, and the saved gamma ramp
VARIABLE window_x, window_y, window_width, window_height, window_rect
VARIABLE the mode-change-in-progress guard
FUNCTION VID_SetDefaultMode, VID_ForceLockState, VID_ForceUnlockedAndReturnState,
         IN_ShowMouse, IN_HideMouse, IN_UpdateClipCursor, S_BlockSound, ...
```

**Invariants** —

- **Focus is one shared boolean, written by the window handler and read by video, sound, input and the host.** Losing focus
  must pause the game, silence the mixer, release the mouse and release every held key, and those four live in four files.
  A rebuild is better served by a focus event the subsystems subscribe to; the recipe records that the original uses a
  shared flag and that all four consequences must happen.
- **The window rectangle is shared** because the input layer confines the pointer to it and must be told when it changes
  ([`in_win.c`](in_win.c.md)).
- The mode-change guard is shared because the window handler must ignore events while a change is in flight
  ([`vid_win.c`](vid_win.c.md)).

**Notes** — everything declared here is platform-specific and none of it crosses into the engine proper: no file above the
platform layer includes it. That separation is the load-bearing fact, and a rebuild should preserve it — the engine's
interfaces are [`vid.h`](vid.h.md), [`input.h`](input.h.md), [`sound.h`](sound.h.md) and [`sys.h`](sys.h.md), and nothing
else may leak.
