# QW/server/sys_unix.c

> The dedicated server's Unix system layer and entry point: file operations, the clock, non-blocking console input, and a frame loop that waits on the socket and the console together.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`sys.h`](sys.h.md) · [Seam: Operating system services](../../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — the server program; owns the entry point
**Tier floor** — none

## Purpose

The smallest system layer in the recipe, and the best illustration of how little the server needs. Read
[`sys_win.c`](../../WinQuake/sys_win.c.md) for the interface's shape; this twin records only what the dedicated server does
differently, which is one thing and it matters.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What is worth recording

**The frame loop waits on the socket and the console at once**, with a timeout, instead of polling or sleeping blindly:

```text
FUNCTION main()
  obtain the heap ;  parse the command line ;  initialize the server
  LOOP forever
    wait until either the socket has a packet or the console has a line,
      or a short timeout expires
    now = clock ;  frame(now - oldtime) ;  oldtime = now
```

**Invariants** — this is the correction to what [`sys_wind.c`](../../WinQuake/sys_wind.c.md) does, and the difference is
visible: waiting on the console alone makes a packet's latency depend on the timeout, whereas waiting on both makes the server
respond to a packet the moment it arrives while still costing nothing when idle. **A rebuild should do this and not the other
thing** — the readiness wait is the whole of the technique.

The timeout still exists, because the server must run a frame on schedule even with no input at all: entities move, timers
fire, and clients must be timed out.

**There is no floating-point control, no code patching, no display and no input**, so those operations are absent rather than
empty. The file is the platform seam reduced to: a clock, a heap, files, printing, and a readiness wait.

**The console is read non-blocking** and lines are handed to the command interpreter, which is the only local interface the
server has.

**Notes** — worth reading alongside [`sys_null.c`](../../WinQuake/sys_null.c.md): between them they bracket the platform
requirement from below and from the side.
