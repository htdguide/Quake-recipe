# WinQuake/in_null.c

> An empty input backend: no mouse, no joystick, for a build with no player at the keyboard.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`input.h`](input.h.md)
**Used by** — the dedicated-server build
**Tier floor** — none

## Purpose

Every operation of the input interface ([`input.h`](input.h.md)) present and doing nothing. A dedicated server links it and
takes its commands from the console instead.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## The operations

**Contract** — initialize, shut down, report button changes, and contribute to the command being built: all do nothing.

**Invariants** — the interface is small enough that this file is four empty bodies, which is itself the useful measurement:
**the engine's input seam is four operations wide** ([Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)).
