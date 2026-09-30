# WinQuake/in_dos.c

> The DOS input backend: the mouse read through a firmware call, a joystick read by timing how long its axis capacitors take to discharge, and a hook for an external control device.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`input.h`](input.h.md) · [`dosisms.h`](dosisms.h.md) · [`client.h`](client.h.md) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`cl_input.c`](cl_input.c.md) · [`host.c`](host.c.md)
**Tier floor** — T1: the joystick is read by counting a timing loop against a hardware port

## Purpose

The same interface as [`in_win.c`](in_win.c.md) — read that twin for the accumulate-then-consume rule, the pitch clamp, the
filter and the axis mapping, all of which are the same. Two things here are specific.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What is worth recording

**The joystick is read by timing.** The hardware reports each axis as the time a monostable takes to fire, so reading it
means starting the timer and polling a port until each axis's bit clears, counting iterations. The count therefore depends
on the processor's speed and must be calibrated, and the read **blocks for up to a millisecond**.

**Invariants** — a blocking read of that length inside the frame is acceptable only because it happens once per frame. A
rebuild will never meet this hardware, but the lesson transfers: **an input device whose read costs real time must be read
exactly once per frame, at a known point**, which is also why the mouse is accumulated rather than polled.

Calibration is a startup step that waits for a button press, because the count is meaningless until the range is known.

**The mouse is a firmware call reporting relative counts**, so there is no pointer, no confinement and no acceleration to
disable — the whole apparatus of [`in_win.c`](in_win.c.md) is unnecessary. The comparison is instructive: a platform that
reports what the engine actually wants needs almost no input layer.

**An external device hook** lets a third-party controller supply motion and button state through a documented interface,
which is added to the mouse and joystick contributions. It is a **pluggable** seam in miniature, and the reason it exists
is that head-tracking and specialist controllers were sold for this game.

**Notes** — the auxiliary-look toggle is a separate control here, kept because the device hook could supply a view
independent of the mouse.
