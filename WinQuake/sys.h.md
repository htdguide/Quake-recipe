# WinQuake/sys.h

> The complete list of what the engine asks of an operating system: fifteen calls, no more.

**Needs** — nothing; this is a leaf
**Used by** — every module; and implemented once per platform by [`sys_win.c`](sys_win.c.md), [`sys_linux.c`](sys_linux.c.md), [`sys_dos.c`](sys_dos.c.md), [`sys_sun.c`](sys_sun.c.md), [`sys_wind.c`](sys_wind.c.md) and [`sys_null.c`](sys_null.c.md)
**Tier floor** — none; this is the seam, not an implementation of it

## Purpose

This file is the whole of [Seam: Operating system
services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services), written down. Its
value to a rebuilder is its shortness: an engine of this size depends on fifteen
platform calls, and a rebuild that provides these fifteen can bring the rest of the
engine up unchanged.

## State

Stateless.

## File access

**Contract** — a blocking byte-stream interface over small integer handles.
`Sys_FileOpenRead` returns the file's length and fills in a handle, or returns −1 and
sets the handle to −1 when the file is absent. `Sys_FileOpenWrite` truncates or
creates and returns a handle. Seeking is to an absolute offset from the start. Reads
and writes return the count transferred. `Sys_FileTime` returns a modification
timestamp, or −1 when the file does not exist — the value is compared against another
of its own kind and is never interpreted, so any monotone encoding works.
`Sys_mkdir` creates one directory and reports nothing, not even failure.

**Invariants** — files must be opened in binary mode: the engine reads structures and
any newline translation corrupts them. Handles are small integers because the virtual
filesystem compares them for equality to recognize the shared archive handle
([`common.c`](common.c.md#com_openfile-com_fopenfile-com_closefile)) — so a rebuild may use any comparable
handle type, but it must be comparable.

**Notes** — there is no directory enumeration anywhere, which is why the archive
numbering in [`common.c`](common.c.md#com_addgamedirectory) probes `pak0`, `pak1` and
so on rather than listing a directory, and why the save-game menu probes twelve fixed
filenames. A rebuild that adds enumeration can simplify both.

There is no failure signal from most of these. A read that fails short is
indistinguishable from a short file. The engine survives because every read is of a
length it obtained from the same layer.

## `Sys_MakeCodeWriteable`

**Contract** — takes an address range and makes it writable for execution. Used
because two platform backends patch their own assembly at startup — the software
renderer's span loops write a shift count into an immediate operand.

**Notes** — self-modifying code is the most incidental thing in the engine. What
survives is the *problem*: one inner loop needs a shift count that is constant for a
whole frame but not at compile time. A rebuild passes it in a register, precomputes
per-width variants, or lets the compiler hoist it, and deletes this call.

## Console and process

**Contract** — `Sys_Printf` writes a formatted line somewhere the user can see it,
which must work even while the graphics surface owns the screen. `Sys_ConsoleInput`
returns a complete line if one is available and nothing otherwise, never blocking;
only the dedicated server calls it. `Sys_Error` releases the graphics surface, shows a
message, and terminates — it never returns, and the engine calls it from a hundred
places as its only failure mode. `Sys_Quit` terminates cleanly. `Sys_DebugLog` appends
a formatted line to a named file. `Sys_Sleep` yields briefly so a paused or
debugging process does not spin.

**Invariants** — `Sys_Error` not returning is depended on everywhere; callers do not
guard the code after it. The "release the graphics surface first" part is what makes
the message readable at all, and a rebuild in a windowed environment must still do the
equivalent or the message goes nowhere.

## `Sys_FloatTime`

**Contract** — returns a monotonically increasing count of seconds as a
double-precision value, from an arbitrary origin. Resolution better than a
millisecond. This is the engine's only clock: frame timing, network timeouts, sound
synchronization and the benchmark all read differences of it.

**Invariants** — must be monotonic. Several backends contain workarounds for clocks
that were not, and the engine's frame-time clamp
([`host.c`](host.c.md#host_filtertime)) is partly a defence against the same. A
rebuild on a monotonic clock should delete the workarounds rather than port them.

## `Sys_SendKeyEvents`

**Contract** — drains the platform's input queue, calling the engine's key handler for
each event, until empty. Called once per frame. On a platform whose window messages
must be pumped, this is also where that happens, which is why it is a *system* call
rather than an input one.

## Floating-point control

**Contract** — `Sys_LowFPPrecision`, `Sys_HighFPPrecision` and `Sys_SetFPCW` set the
hardware's rounding precision. The renderer switches to single precision around its
inner loops, where a shorter mantissa makes division faster, and back for code that
needs the full one.

**Notes** — this is a 1996 x86 concern and on most platforms the three are empty. What
a rebuild must take from it is that the software rasterizer's arithmetic was *tuned*
at single precision, so a rebuild computing its span coordinates in double precision
will get slightly different pixels. See
[`mathlib.c`](mathlib.c.md#invert24to16) for the one place the precision is explicitly
part of the contract.
