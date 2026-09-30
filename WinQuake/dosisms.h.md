# WinQuake/dosisms.h

> The DOS platform's own interface: port access, interrupt control, address conversion between the two memory models, memory allocation and locking, and the register block used to make firmware calls.

**Needs** — nothing
**Used by** — every DOS backend: [`sys_dos.c`](sys_dos.c.md) · [`vid_dos.c`](vid_dos.c.md) · [`vid_vga.c`](vid_vga.c.md) · [`vid_ext.c`](vid_ext.c.md) · [`in_dos.c`](in_dos.c.md) · [`snd_dos.c`](snd_dos.c.md) · [`cd_audio.c`](cd_audio.c.md) · [`net_ipx.c`](net_ipx.c.md) · [`net_bw.c`](net_bw.c.md) · [`net_comx.c`](net_comx.c.md)
**Tier floor** — T0: it is the declaration of direct hardware access

## Purpose

The declaration of what "no operating system" means in practice. It is a **given** in the strictest sense — no modern
platform offers any of it — but it is worth one page because it enumerates the primitives the whole DOS half of the engine
is built from, and because two of them recur in disguise on modern platforms.

## State

```text
FUNCTION read and write a byte or a word at a hardware port
FUNCTION enable and disable an interrupt line
FUNCTION convert between a pointer and each of the two address forms
FUNCTION obtain and release a block of low memory a device can reach
FUNCTION lock a memory range so it is never paged, and unlock it
FUNCTION make a firmware call with a register block
RECORD the register block
CONSTANT the interrupt numbers the drivers use
```

**Invariants** —

- **Two address forms coexist and the conversions are explicit.** A pointer the program uses and an address a device uses
  are different things, and every buffer handed to a device must be converted. The modern counterpart is a device or
  graphics API's own address space, and the requirement is identical: *a buffer shared with a device is addressed
  differently by each party.*
- **Memory a device writes to must be locked.** The modern counterpart is pinning for asynchronous transfer. Same reason:
  the device cannot wait for a page fault.
- Firmware calls take a register block by reference and return through it, which is why [`vregset.c`](vregset.c.md) exists.

**Notes** — the two invariants above are the only transferable content. Everything else is a specific machine.
