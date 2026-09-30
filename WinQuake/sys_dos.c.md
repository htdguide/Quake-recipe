# WinQuake/sys_dos.c

> The DOS system layer: the same interface over firmware and interrupt services, plus a stack-overflow guard, exception handlers that restore the display before dying, and a startup that must find and lock its own memory.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`sys.h`](sys.h.md) · [`dosisms.h`](dosisms.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — everything, through [`sys.h`](sys.h.md); owns the entry point
**Tier floor** — T1: memory must be explicitly locked, the stack is guarded by hand, and exception handlers run with the display in an unusable state

## Purpose

The original platform layer. Read [`sys_win.c`](sys_win.c.md) for the shared shape — the handle table, the clock, the
floating-point control, the outer loop — and this twin for what a platform with no operating system to speak of forces.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What is worth recording

**Memory must be found, requested and locked.** The engine's single heap comes from asking the memory manager how much it
can give, taking it, and **locking it so it is never paged**. The renderer's timing assumes no page faults, and the
interrupt handlers ([`net_comx.c`](net_comx.c.md), [`snd_dos.c`](snd_dos.c.md)) cannot tolerate one at all. A rebuild on a
platform with real-time requirements meets the same need under a different name.

**The stack is guarded by hand.** A pattern is written below the stack at startup and checked periodically; a change means
the stack overflowed. There is no other protection. This is worth recording because the engine's recursive tree walks
([`world.c`](world.c.md), [`r_bsp.c`](r_bsp.c.md)) are the things that would overflow it, and their depth is bounded by the
map compiler — an unwritten contract that this check is the only enforcement of.

**Exception handlers restore the display and then print.** A fault, a floating-point exception or a break must put the
display back into text mode before saying anything ([`vid_dos.c`](vid_dos.c.md)), and must release the interrupt vectors
the serial and sound drivers claimed. The exit handlers are registered as a list so each subsystem can add its own. Same
rule as everywhere: **release on the crash path, not only the exit path.**

**A key must be trapped at the interrupt level** to break out of a hung frame, because there is no task manager. That is
the platform's only escape hatch.

**The clock comes from the timer interrupt and wraps.** The wrap must be detected and accumulated, which is the same
monotonicity requirement as [`sys_win.c`](sys_win.c.md) with a different cause.

**It detects whether it is running under a multitasking system** and adjusts, because yielding is possible there and not
otherwise.

**Notes** — everything in this file is a **given** that no modern platform exposes, with two exceptions worth carrying: the
single locked heap, and the discipline that every claimed system resource is released from the fault handler.
