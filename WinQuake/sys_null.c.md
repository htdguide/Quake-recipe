# WinQuake/sys_null.c

> A skeleton system layer: every operation present, most of them empty, as the starting point for a new platform.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`sys.h`](sys.h.md)
**Used by** — nothing; it is a template
**Tier floor** — none

## Purpose

The checklist of what a new platform must supply, expressed as compilable stubs. Its value to a rebuilder is exactly that:
**this file is the platform interface, enumerated.** Read it beside [`sys.h`](sys.h.md) to see the full surface and no
more.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## The operations

**Contract** — file open, close, seek, read, write, length and modification time over a fixed handle table; directory
creation; making code writable; the fatal error and the clean exit; printing; the clock; console input; the input pump;
the yield; the floating-point precision pair; and an entry point that initializes the host and loops.

**Invariants** — the file operations are implemented (they are portable); everything platform-specific is empty. So the
list of empty bodies is the list of genuinely platform-dependent capabilities, and it is short: clock, print, exit, input
pump, sleep, code-writable, floating-point mode. That is the whole seam
([Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)).

**Notes** — the clock stub returning a value that increments by a fixed step per call is a hint worth taking: a rebuild
can be tested deterministically by driving the engine from a fake clock, which is the same idea as
[`net_vcr.c`](net_vcr.c.md) applied to time.
