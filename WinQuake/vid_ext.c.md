# WinQuake/vid_ext.c

> The extended-mode sub-driver: higher resolutions obtained through the display firmware's own interface, with the framebuffer reached either as one flat region or through a moving window over it.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`vid_dos.h`](vid_dos.h.md) · [`dosisms.h`](dosisms.h.md) · [`vregset.h`](vregset.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface)
**Used by** — [`vid_dos.c`](vid_dos.c.md) fills its extended mode entries from here
**Tier floor** — T1: the framebuffer may be larger than the window through which it is addressed, so a write's destination depends on a bank register

## Purpose

How the engine ran above the standard resolution on hardware that had no standard way to offer it. The firmware exposes a
mode list and a way to enter a mode; what it does *not* reliably expose is a flat address for the whole framebuffer. So
this file handles the two cases, and the second one is the interesting one.

## State

```text
VARIABLE the firmware's mode list, and per mode: its number, size, stride,
         whether a flat region is available, and the window's size and granularity
VARIABLE the currently selected bank
VARIABLE a buffer in ordinary memory, drawn into and then copied out
```

## `VID_InitExtra`, `VID_ExtraGetModeInfo`, `VID_ExtraInitMode`, `VID_ExtraFarToLinear`

**Contract** — ask the firmware for its capabilities and mode list, keep the indexed modes at usable sizes, describe each,
and enter one; and convert a firmware-reported address into one the program can use.

**Invariants** — the firmware's version determines whether a flat region can be requested at all, which is why the
capability survey precedes the mode list. The engine **requires** one byte per pixel and skips every other mode, because
the renderer writes single bytes ([`d_local.h`](d_local.h.md)).

## `VID_ExtraSwapBuffers`, `VID_ExtraSwapBuffers` (banked form), `VID_SetVESAPalette`

**Contract** — copy the drawing buffer to the display: in the flat case, one copy honouring the update rectangle; in the
banked case, a loop that selects each bank in turn and copies only the rows that fall inside it. Program the palette
through the firmware or through the controller's ports, whichever the firmware prefers.

```text
FUNCTION present_banked(buffer, rect)
  FOR EACH row of the rectangle
    offset = row * stride + rect.left
    bank   = offset / window_granularity
    IF bank is not the selected one  select it through the firmware
    copy the row's bytes into the window at (offset MOD window size)
    # a row that straddles the window's end must be SPLIT across two banks
```

**Invariants** —

- **The framebuffer is addressed through a window smaller than itself**, and a write's destination depends on which bank
  is selected. So a sequential copy must re-select whenever it crosses a boundary, and a single row can straddle one.
  Getting the straddle case wrong tears one row per bank boundary — a distinctive symptom.
- The bank selection is a **firmware call, and it is slow**, so the loop selects as rarely as possible rather than per
  row.
- Because presentation is a copy either way, the renderer always draws into ordinary memory here — so the dirty-rectangle
  tracking works and there is no flip-versus-copy conflict ([`vid_vga.c`](vid_vga.c.md)).

## `VID_ExtraWaitDisplayEnable`, `VID_ExtraVidLookForState`, `VID_ExtraStateFound`, `VGA_BankedBeginDirectRect`, `VGA_BankedEndDirectRect`

**Contract** — wait for the display's active period, save and restore the controller's state around a mode change, and the
direct-rectangle pair through the bank window.

**Invariants** — the controller's state is **saved before the first mode change and restored at shutdown**, because the
firmware's mode changes do not restore everything and the text mode afterwards can come back wrong.

**Notes** — bank switching is dead and a rebuild will never meet it. It is recorded for one transferable idea: when a
resource is addressed through a smaller window, **the copy loop and not the caller must own the windowing**, and the
straddle case is the one that will be wrong.
