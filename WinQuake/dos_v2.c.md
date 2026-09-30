# WinQuake/dos_v2.c

> The implementation of the DOS platform primitives against a particular memory extender's services.

**Needs** — [`dosisms.h`](dosisms.h.md) · [`quakedef.h`](quakedef.h.md)
**Used by** — every DOS backend, through [`dosisms.h`](dosisms.h.md)
**Tier floor** — T0: it issues the firmware calls that map, lock and address physical memory

## Purpose

Fills [`dosisms.h`](dosisms.h.md). Read that twin for what the primitives mean; this one exists to record that the
implementation is a set of firmware calls and that two of them have a subtlety.

## State

Stateless.

## The operations

**Contract** — read and write hardware ports; enable and disable an interrupt line by masking it at the controller; convert
between the program's addresses and the device-visible ones; map a region of low memory into the program's address space;
allocate and free device-reachable memory; report the remaining heap; and lock and unlock a memory range.

**Invariants** —

- **Masking an interrupt is a read-modify-write of a controller register**, so it is not safe to do from two places at
  once; the drivers that use it ([`net_comx.c`](net_comx.c.md), [`snd_dos.c`](snd_dos.c.md)) each own their line.
- **Mapping low memory must happen before any address conversion involving it**, which is why the mapping is done once at
  startup and the converters then assume it.
- Locking takes a base and a length and both must be **page-aligned outward**; locking a sub-page range silently locks
  less than asked. That is the kind of off-by-a-page bug that shows up as a rare fault under load.

**Notes** — the numeric service codes are trivia. The alignment rule is the one thing a reader should take away, because its
modern counterparts — pinning, mapping, cache maintenance — all share it.
