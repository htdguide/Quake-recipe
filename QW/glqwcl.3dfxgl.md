# QW/glqwcl.3dfxgl

> Data: a nine-line launcher that points the hardware client at one vendor's graphics library before starting it.

**Needs** — nothing
**Used by** — run instead of the client, on that hardware
**Tier floor** — none

## Purpose

A shell wrapper that sets the library search path so that the vendor's implementation is loaded in place of the system's, then runs the
client.

## State

Data; no run-time state.

**Notes** — the recipe-relevant fact is that it needs to exist at all: **the graphics library is selected at load time by name**, so one
executable serves several hardware families
([`WinQuake/3dfx.txt`](../WinQuake/3dfx.txt.md) records the same arrangement on the other platform). That is the seam
([Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)) being exercised in the crudest possible way, and it
worked.
