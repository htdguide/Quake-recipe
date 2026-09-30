# WinQuake/input.h

> The non-keyboard input seam: five calls, of which the interesting one adds mouse and joystick motion on top of the keyboard's contribution.

**Needs** — [`client.h`](client.h.md) (the movement command)
**Used by** — [`host.c`](host.c.md) · [`cl_input.c`](cl_input.c.md) · implemented per platform by [`in_win.c`](in_win.c.md), [`in_dos.c`](in_dos.c.md), [`in_sun.c`](in_sun.c.md) and [`in_null.c`](in_null.c.md)
**Tier floor** — none; this is the seam

## Purpose

Part of [Seam: Keyboard and mouse
input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input) — the *analogue* part. Keys arrive
through a different path entirely ([`keys.h`](keys.h.md), driven by
[`sys.h`](sys.h.md)'s event pump); this file covers relative mouse motion and joystick axes.

The design worth noting is in one word of a comment: `IN_Move` adds movement **on top of** the keyboard's
command. So the client builds a command from keys first and the device backend then modifies it, rather
than the two being merged by a third party. That ordering is why mouse look and keyboard turning compose
correctly.

## State

Stateless as declared; each backend owns its own.

## Entry points

**Contract** — `IN_Init` opens the devices; `IN_Shutdown` releases them. `IN_Commands` gives a device
the opportunity to push console commands into the queue, once per frame.
`IN_Move` takes the movement command already built from the keyboard and adds the device's
contribution. `IN_ClearStates` resets every button and axis to neutral.

**Invariants** — `IN_Commands` is how a **joystick button** becomes an action: the backend synthesizes
the same key event a keyboard would, so joystick buttons can be bound like keys. That is the right
factoring and a rebuild should keep it.

`IN_ClearStates` exists for the moment focus is lost or a level is loading, so that a held button does
not remain held across the gap.

`IN_Move` receiving the command by reference and modifying it is what allows a backend to *replace*
rather than add — the mouse-look case overwrites the view angles rather than adding to the movement —
and each backend chooses per axis.
