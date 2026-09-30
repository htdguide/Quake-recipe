# WinQuake/gl_vidlinuxglx.c

> The X11 hardware-video backend: a window with a rendering context, plus the mouse and keyboard, including the pointer grab that makes relative mouse look possible.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`vid.h`](vid.h.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`host.c`](host.c.md) · [`gl_screen.c`](gl_screen.c.md)
**Tier floor** — none

## Purpose

The same job as [`gl_vidnt.c`](gl_vidnt.c.md) — read that twin for the extension survey, the frame bracket and the
gamma decision, which are identical — over X11. Two things are worth their own treatment because they are where this
platform is genuinely different, and both are **input**, not video.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs

**Video and input are one file.** The window system delivers both through one event stream, so the input backend cannot
be a separate file the way [`in_win.c`](in_win.c.md) is. A rebuild should expect its platform to dictate this split, not
choose it.

**The mouse must be grabbed and the pointer hidden to get relative motion.** The engine wants "how far did the mouse
move", and a windowing system reports "where is the pointer". So:

```text
FUNCTION activate_mouse()
  hide the pointer by setting an empty cursor
  grab the pointer to this window, so motion outside it still arrives
  grab the keyboard, so the window manager's shortcuts do not steal keys
FUNCTION handle_motion(event)
  delta = event position - the window centre
  IF delta is nonzero  warp the pointer back to the centre
  accumulate delta
```

**Invariants** —

- **The pointer is warped back to the centre after every motion event**, which is what turns absolute positions into
  unbounded relative motion. The warp itself generates a motion event, which must be recognized and ignored or the
  motion doubles and oscillates. That recognition — a motion event landing exactly at the centre is the warp's own — is
  the subtle part, and getting it wrong produces a view that drifts or judders. A rebuild on a platform with a real
  relative-motion mode should use it and delete all of this.
- The **keyboard is grabbed too**, because otherwise the window manager consumes keys the game wants. Grabbing the
  keyboard is hostile to the desktop, so it must be released whenever the mouse is released — on losing focus, on
  opening the console, on pausing.
- An empty cursor must be **created**, since the platform offers no hide operation.

**The keyboard map is a translation function, not a table**, because the platform's key symbols are not dense. It maps
the printable range through directly and names the rest.

**Signal handlers are installed for the fatal signals**, so that a crash restores the display and releases the grabs
before dying. Without them a crash leaves the user with no pointer and no keyboard. This is the same reasoning as the
interrupt-vector restoration in [`net_comx.c`](net_comx.c.md), and it generalizes: **anything you grab from the system
must be released on the crash path, not only the exit path.**

**Notes** — mode switching is not attempted: the window is created at the requested size and the display mode is left
alone. So there is no fullscreen and no confirmation dance. That is a reasonable simplification and a rebuild may make
the same one.
