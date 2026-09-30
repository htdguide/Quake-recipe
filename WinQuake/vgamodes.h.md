# WinQuake/vgamodes.h

> Data: the display controller register values that produce each non-standard resolution the DOS build offers.

**Needs** — [`vregset.h`](vregset.h.md)
**Used by** — [`vid_vga.c`](vid_vga.c.md) · [`vid_dos.c`](vid_dos.c.md)
**Tier floor** — T0

## Purpose

A data file, not code. Each mode is a table in the form [`vregset.h`](vregset.h.md) describes, listing every register the
controller needs set to produce a given resolution and refresh timing. These are the "tweaked" modes the firmware does not
offer.

## State

Data; no run-time state.

## The format

```text
One table per mode, each naming:
  the horizontal and vertical timing registers -- total, display end,
    blank start and end, sync start and end
  the addressing mode, the row stride and the scan-line multiplier
  the memory addressing configuration
```

**Invariants** —

- **The values are a working set discovered by experiment and must be written in the table's order**
  ([`vregset.c`](vregset.c.md)).
- Each table is paired with a width, height and stride in the mode list ([`vid_dos.c`](vid_dos.c.md)); the registers and
  those numbers must agree or the engine draws into the wrong shape.
- A mode's timing can be **out of range for a given monitor**, which is why every mode is confirmed by the player before it
  is kept ([`vid_win.c`](vid_win.c.md) records the same safeguard).

**Notes** — the numbers themselves have no meaning outside this controller and a rebuild carries none of them. What a
rebuilder should take is only that the DOS build's higher resolutions were obtained by programming the display directly
rather than asking for them — which is why they exist at all and why they are riskier than the firmware modes.
