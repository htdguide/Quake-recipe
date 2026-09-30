# WinQuake/vid_win.c

> The Windows software-video backend: a byte-per-pixel buffer the renderer draws into, presented as a window, a scaled window, or an exclusive display mode, with the palette programmed and every held key released on focus loss.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`winquake.h`](winquake.h.md) · [`vid.h`](vid.h.md) · [`d_local.h`](d_local.h.md) · [`resource.h`](resource.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`host.c`](host.c.md) · [`screen.c`](screen.c.md) · [`draw.c`](draw.c.md) · [`menu.c`](menu.c.md)
**Tier floor** — T1: the renderer writes single bytes into a buffer whose stride and base address the platform chooses, and the buffer may be locked and moved between frames

## Purpose

The largest platform file in the engine, and the one that defines the software renderer's side of the framebuffer seam.
The whole software renderer exists to fill one array of palette indices; this file obtains that array from the operating
system, hands it over, and presents it. Read [`vid.h`](vid.h.md) for the interface; this twin records how it is filled
and, more usefully, **what a rebuild must demand of its own framebuffer**.

## State

```text
RECORD Mode
  type : windowed | fullscreen-exclusive | fullscreen-windowed
  width, height, bpp, stretch factor : int
  description : text
VARIABLE modelist, nummodes, vid_modenum, vid_default, windowed_default
VARIABLE vid : the published buffer descriptor (base, stride, size, palette state)
VARIABLE vid_buffer, zbuffer, surfcache   # the three big allocations
VARIABLE the display contexts, the window, its remembered position
VARIABLE lockcount, vid_palettized, ActiveApp, Minimized, in_mode_set
```

**Invariants** — the backend owns **three** allocations, not one: the pixel buffer, the depth buffer, and the surface
cache ([`d_surf.c`](d_surf.c.md)). All three scale with resolution, so they are reallocated on every mode change, and all
three come from the **high end of the load heap** ([`zone.h`](zone.h.md)) — high, so that growing them on a mode change
does not leave a hole under the game's own allocations. That placement is a real constraint and it is documented nowhere
but the allocator's own header.

The surface cache's size is derived from the resolution by a formula the renderer supplies, with a floor, and the
remaining memory must still leave room for temporary allocations. A mode is **refused before it is entered** if the memory
is not there — which is why the mode list carries a memory check and the menu shows modes it will not allow.

## `VID_SetMode`, `VID_SetWindowedMode`, `VID_SetFullscreenMode`, `VID_SetFullDIBMode`, `VID_RestoreOldMode`, `VID_SetDefaultMode`

**Contract** — switch to a numbered mode: refuse it if memory is short, tear down the current mode, create the new
window or take the display, allocate the three buffers, publish the buffer descriptor, reprogram the palette, tell input
the new window rectangle, force a full redraw, and release any held keys. On failure restore the previous mode, and
failing that the default.

**Invariants** —

- **A failed mode change must land in a working mode.** The fallback chain is: requested, previous, default windowed,
  fatal error. Anything less strands the player at a black screen with no way back.
- A flag marks that a mode change is in progress, and the window message handler must ignore activation traffic while it
  is set, because the platform generates focus events during the change and acting on them re-enters the change. That
  re-entrancy guard is a real bug this code fixes and a rebuild will meet the same one.
- The window position is **remembered and restored**, and clamped back on screen if the display got smaller.
- Three kinds of fullscreen exist — exclusive through the graphics library, and a borderless window at desktop depth —
  because exclusive modes were unreliable. A rebuild needs at most two.

## `VID_Update`, `FlipScreen`

**Contract** — present the finished buffer: for a windowed mode, copy the buffer to the window, honouring the current
update rectangle; for an exclusive mode, swap or copy to the visible page. Handles a pending palette change and a pending
mode change first.

**Invariants** —

- **Only the changed rectangle is copied.** The engine tracks what it dirtied ([`screen.c`](screen.c.md)) and this is
  where that tracking pays: a frame that only changed the view rectangle copies only the view rectangle. A rebuild whose
  presentation is all-or-nothing throws the tracking away — as the hardware path does
  ([`gl_screen.c`](gl_screen.c.md)) — and should then also delete it from the shared code rather than maintaining it for
  nothing.
- A **palette change is deferred to the next present**, because programming the palette between two presents makes the
  currently visible frame flash in the new palette. The flash is visible and the deferral removes it.
- A mode change requested from the menu or console is also deferred to here, so it never happens mid-frame.
- In a page-flipped mode there are **two buffers and the renderer draws into the hidden one**, but the engine's dirty
  tracking assumes the buffer it drew last frame is the one in front of it. So a flipped mode must either copy rather
  than flip, or force full updates. Both paths exist. This is the one place where the dirty-rectangle optimization and
  double buffering genuinely conflict, and the recipe records it because a rebuild that keeps the tracking must resolve
  it the same way.

## `VID_LockBuffer`, `VID_UnlockBuffer`, `VID_ForceLockState`, `VID_ForceUnlockedAndReturnState`

**Contract** — obtain the buffer's base address and stride, counting nested locks so only the outermost pair touches the
platform; force the count to a value; and unlock whatever is held while reporting what it was, so it can be restored.

**Invariants** — **the buffer's address and stride may change on every lock.** That is the constraint the whole interface
exists for, and it is why the renderer re-reads them rather than caching them across frames
([`d_local.h`](d_local.h.md)). A rebuild whose framebuffer is a stable array can delete the locking entirely; one on a
surface-based API cannot.

The nested count exists because the renderer locks in several places and the platform's lock is not re-entrant. The
force-and-restore pair exists for the error path: printing a fatal error needs the buffer unlocked, and the caller has no
idea how deep the nesting was.

## `D_BeginDirectRect`, `D_EndDirectRect`

**Contract** — draw a small block of pixels directly to the visible screen, outside the frame, and restore what was
there.

**Invariants** — this exists for exactly one feature: the **disc-activity indicator during a load**, when the frame loop
is not running ([`draw.c`](draw.c.md)). It is the only thing in the engine that writes to the front buffer. The hardware
path cannot do it at all ([`gl_vidnt.c`](gl_vidnt.c.md)), which is why the feature silently disappears there.

## `MapKey`, `ClearAllStates`, `AppActivate`, `MainWndProc`, `VID_HandlePause`, `VID_UpdateWindowStatus`

**Contract** — translate platform key codes to the engine's numbering, release every held key and button, and handle the
window's messages: activation, minimization, palette loss, focus, mouse buttons and wheel, and the close request.

**Invariants** —

- **Every held key and button is released on focus loss**, without exception. Same rule as every other backend.
- On regaining focus the **palette must be reprogrammed**, because another application owned it meanwhile. Forgetting
  this gives a correct image in wrong colours, which is the classic indexed-colour symptom.
- Activation pauses single player and silences the mixer.
- The window rectangle is republished to the input layer whenever it moves, because the mouse is confined to it.

## `VID_CheckAdequateMem`, `VID_AllocBuffers`, `initFatalError`, `VID_Suspend`

**Contract** — decide whether a mode's three buffers will fit in the remaining heap; allocate them; report a fatal
initialization failure after restoring the display; and suspend for a mode the graphics library needs the display for.

## `VID_NumModes`, `VID_GetModePtr`, `VID_GetModeDescription` and its variants, `VID_CheckModedescFixup`, `VID_InitMGLFull`, `VID_InitMGLDIB`, `VID_InitFullDIB`, `registerAllDispDrivers`, `registerAllMemDrivers`, `createDisplayDC`, `DestroyDIBWindow`, `DestroyFullscreenWindow`, `DestroyFullDIBWindow`

**Contract** — build the mode list from the graphics library's fullscreen modes plus the desktop's own settings plus a
fixed set of windowed sizes; describe each; and create and destroy the contexts and windows for each kind.

**Invariants** — the mode list is **deduplicated and sorted**, and modes wider than the engine's limits are dropped. A
duplicate mode in a menu is a support problem, which is why the fix-up exists.

## `VID_DescribeCurrentMode_f`, `VID_NumModes_f`, `VID_DescribeMode_f`, `VID_DescribeModes_f`, `VID_TestMode_f`, `VID_Windowed_f`, `VID_Fullscreen_f`, `VID_Minimize_f`, `VID_ForceMode_f`, `VID_MenuDraw`, `VID_MenuKey`

**Contract** — the console commands and the menu page for choosing a mode, including a test command that enters a mode for
a few seconds and returns, and a force command that skips the memory check.

**Invariants** — **testing a mode before committing to it** is the same safeguard as the confirmation countdown in
[`gl_vidnt.c`](gl_vidnt.c.md), and for the same reason. The force command exists because the memory check was
conservative and players wanted past it — a reminder that a refusal a user can see is a refusal a user will want to
override.

**Notes** — the file's size is almost entirely the three fullscreen paths and the mode enumeration. The *design* content
is small and is in the lock semantics, the deferred palette and mode changes, the dirty-rectangle interaction with page
flipping, and the release-everything-on-focus-loss rule. A rebuild should read those five and take the rest from its own
platform.
