# WinQuake/sys_win.c

> The Windows system layer and the program entry: file operations, the high-resolution clock, the floating-point control word, the fatal error path, and the outer loop that drives frames only when there is work or a client to serve.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`winquake.h`](winquake.h.md) · [`resource.h`](resource.h.md) · [`sys.h`](sys.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — everything, through [`sys.h`](sys.h.md); it also owns the entry point that calls [`host.c`](host.c.md)
**Tier floor** — T1: the floating-point rounding mode is changed around the renderer's inner loops, and code is made writable so it can patch itself

## Purpose

The shipping platform layer, and the reference for what the engine actually needs from an operating system
([Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)). Read
[`sys.h`](sys.h.md) for the interface. Three parts of this file are decisions rather than plumbing: the clock, the
floating-point mode, and the outer loop.

## State

```text
VARIABLE a small fixed table of open file handles
CONSTANT a minimum required free memory, checked at startup
VARIABLE the heap the engine is given, and its size
VARIABLE the saved and modified floating-point control words
VARIABLE the console window handle, for the dedicated-server build
```

## `Sys_FloatTime`, `Sys_InitFloatTime`

**Contract** — returns seconds as a real number, monotonically increasing, from the platform's highest-resolution counter,
with the first call establishing the origin. Detects and corrects a backward step.

**Invariants** —

- **Time must never go backward and must never jump.** The whole engine's physics, animation and networking read this one
  function, and a backward step produces a negative frame time which propagates into every integration
  ([`host.c`](host.c.md) clamps, but the clamp is a second line of defence). The counter of the day could step backward
  across a processor migration, and the correction — remember the last value and never return less — is the right fix on
  any platform.
- The origin is subtracted so the numbers stay small enough for a real to hold at millisecond resolution over a long
  session. That matters: an absolute counter in a single-precision real loses resolution within hours.

## `Sys_SetFPCW`, `Sys_PushFPCW_SetHigh`, `Sys_PopFPCW`, `MaskExceptions`

**Contract** — record the current floating-point control word and a modified copy; switch to the modified one and restore
it; and disable arithmetic exception traps.

**Invariants** —

- **The renderer's inner loops need a specific rounding and precision mode**, because the span rasterizer converts reals
  to integers by a truncation that must match what the assembly assumes
  ([`asm_i386.h`](asm_i386.h.md), [`d_scan.c`](d_scan.c.md)). A rebuild in a language with defined conversion semantics
  needs none of this — which is worth knowing, because it removes a whole class of platform dependency.
- **Exceptions are masked, not handled.** A division by zero in the renderer must produce an infinity and keep going, not
  trap. That is a deliberate choice: a single bad surface should not end the game.
- The push and pop are paired around the renderer, which is why they are a stack rather than a setter.

## `Sys_MakeCodeWriteable`, `Sys_PageIn`

**Contract** — make a range of the program's own code writable, and touch every page of a region to force it resident.

**Invariants** — the engine **patches its own code**: the span rasterizer writes immediate values into the assembly inner
loops rather than reading them from memory ([`d_scan.c`](d_scan.c.md)). That is the single most aggressive optimization in
the engine and it is why this function exists. A rebuild does not do it — which is precisely why the portable twins of the
assembly routines exist and why [`SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md) marks the vectorized inner loops as
a **pluggable** seam.

Paging in is a load-time courtesy: touch everything once so the first frames do not stall on faults.

## `Sys_Error`, `Sys_Quit`, `Sys_Printf`

**Contract** — report a fatal error and exit; shut down cleanly and exit; and print a message to wherever this build's
console is.

**Invariants** — the error path must **restore the display, the palette and the display mode before it shows anything**
([`vid_win.c`](vid_win.c.md)), or the message appears on a screen the user cannot read. It must also be re-entrant-safe: an
error raised while handling an error exits immediately rather than looping. Both rules apply to every platform.

## `Sys_FileOpenRead`, `Sys_FileOpenWrite`, `Sys_FileClose`, `Sys_FileSeek`, `Sys_FileRead`, `Sys_FileWrite`, `filelength`, `Sys_mkdir`

**Contract** — the file interface as a small table of integer handles: open for reading reporting the length, open for
writing truncating, close, seek from the start, read, write, and create a directory.

**Invariants** — handles are **indices into a fixed table**, not the platform's own, so the rest of the engine never sees a
platform type. The table's size bounds how many files can be open, and the archive layer ([`common.c`](common.c.md)) keeps
one handle per archive for the whole session — so the bound is real and a rebuild should size it from the archive count.

## `Sys_ConsoleInput`, `Sys_SendKeyEvents`, `Sys_Sleep`, `SleepUntilInput`

**Contract** — read a line from the console when this is a dedicated server; pump the window's message queue; yield briefly;
and block until input arrives or a timeout expires.

**Invariants** — **pumping the message queue is what delivers input**, so it must happen once per frame and nowhere else
([`vid_win.c`](vid_win.c.md) handles the messages). A rebuild whose input arrives by callback still needs a fixed point in the
frame where it is drained, because the engine reads accumulated input at one moment
([`cl_input.c`](cl_input.c.md)).

## `WinMain`, `Sys_Init`

**Contract** — the program entry: check free memory, obtain the engine's heap, parse the command line, decide whether this
is a dedicated server, initialize the host, then loop.

```text
FUNCTION main()
  verify enough free memory ;  obtain the heap
  parse the command line into the parameter set
  initialize the host with the heap and the parameters
  oldtime = now
  LOOP forever
    IF this is a dedicated server
      block until console input arrives or a short timeout expires
    ELSE IF the window is not active OR is minimized
      yield briefly                       # do not burn the machine in the background
    now = clock ;  frame_time = now - oldtime
    IF this is a listen server AND frame_time < a minimum  CONTINUE     # cap the rate
    host_frame(frame_time) ;  oldtime = now
```

**Invariants** —

- **A dedicated server blocks rather than spins.** It has no frame to draw, so it sleeps until a packet or a command
  arrives. That is what lets a server run alongside other work.
- **An inactive window yields.** Without it the game consumes the machine while the player is in another application, and
  the game is already paused.
- A minimum frame time is enforced for a listen server so it does not starve the server thread of the machine; the client
  is otherwise uncapped.
- The heap is obtained **once, as one block, at startup** ([`zone.h`](zone.h.md)). The engine never asks the operating
  system for memory again. That single decision is what makes the allocators in [`zone.c`](zone.c.md) possible and is the
  most transferable thing in this file.

**Notes** — the entry point owning the frame loop, rather than the host owning it, is why every platform has its own copy of
the loop above. A rebuild should put the loop in the host and give the platform only the clock, the sleep and the input
pump.
