# WinQuake/gl_vidnt.c

> The Windows hardware-video backend: a window or an exclusive display mode, a rendering context, the run-time extension survey, the gamma ramp, the keyboard map, and the window message loop.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`winquake.h`](winquake.h.md) · [`vid.h`](vid.h.md) · [`resource.h`](resource.h.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`host.c`](host.c.md) initializes it; [`gl_screen.c`](gl_screen.c.md) brackets each frame with it
**Tier floor** — none

## Purpose

Where the hardware engine meets the operating system. It is a large file and most of it is platform ceremony, but four
things in it are decisions a rebuild must make, and they are the sections below. Everything else — creating a window,
describing pixel formats, enumerating display modes — is what the platform demands and the recipe treats as a
**given** ([Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface)).

Compare [`vid_win.c`](vid_win.c.md), the software renderer's version of the same job. That one must also allocate and
present a byte-per-pixel buffer; this one has only to hand over a context.

## State

```text
RECORD Mode                              # one entry per selectable mode
  type : windowed | fullscreen
  width, height, bpp : int
  description : text
VARIABLE modelist, nummodes, vid_modenum, vid_default
VARIABLE mainwindow, the rendering context
VARIABLE glx, gly, glwidth, glheight     # the viewport, published to the renderer
VARIABLE vid_gamma, the saved and the applied gamma ramp
VARIABLE the extension entry points and the flags saying which were found
VARIABLE ActiveApp, Minimized, scr_skipupdate
```

## `GL_BeginRendering`, `GL_EndRendering`

**Contract** — report the viewport the renderer should use, and present the finished frame; the presentation also polls
the sound system and honours the frame-skip flag.

**Invariants** — **these two calls are the entire per-frame platform surface of the hardware renderer.** That is the most
useful fact in this file: a rebuild needs nothing else from its window system inside the frame. Compare the software
path, which needs a lockable buffer, a palette and a blit.

The presentation is also where the sound mixer is nudged, because presentation is where the frame naturally waits. A
rebuild with a proper audio thread does not need this and should not copy it.

## `GL_Init`, `CheckTextureExtensions`, `CheckArrayExtensions`, `CheckMultiTextureExtensions`, `VID_Is8bit`, `VID_Init8bitPalette`

**Contract** — read the vendor, renderer, version and extension strings; set the base rendering state — front-face
winding, culling mode, alpha test threshold, texture wrap and environment; then scan the extension string for each
optional capability and resolve its entry points, leaving a flag per capability.

**Invariants** —

- **Every optional capability is detected by scanning a string and resolving pointers, and its absence is a supported
  configuration** with a slower path behind it ([`glquake.h`](glquake.h.md)). Four are surveyed: separate texture
  objects, vertex arrays, multitexture, and indexed textures.
- The alpha test threshold is a **specific value**, not a half: it must reject the transparent index's expanded pixels
  and keep everything else, and the value is chosen against how the palette expansion rounds
  ([`gl_draw.c`](gl_draw.c.md)). A rebuild that changes the expansion must revisit the threshold or model edges fringe.
- The base state is set **once** and every path that changes it is expected to change it back. That is the state-machine
  discipline the whole renderer is written against, and it is the largest single source of bugs in a rebuild that adds a
  pass.

## `Check_Gamma`

**Contract** — reads the player's gamma setting, then **pre-corrects the palette** by raising each component to a power
before it is used, rather than programming the display's ramp.

**Invariants** — gamma is applied to the **palette used to expand textures**, so it must be set before any texture is
uploaded and changing it requires reloading everything. That is why it is read once at startup from the command line and
is not a live setting in this backend. A rebuild with a post-process or a display ramp makes it live, and should.

## `MapKey`, `ClearAllStates`, `AppActivate`, `MainWndProc`, `VID_HandlePause`

**Contract** — translate a platform key code into the engine's own key numbering; release every held key and button; and
handle activation, minimization, mode loss, focus, mouse and the close request from the window's message loop.

**Invariants** —

- **Every held key must be released when focus is lost**, or the player returns to find themselves walking into a wall
  forever. Same rule in every backend, and the single most commonly forgotten one in a rebuild.
- Losing focus in an exclusive display mode must **restore the desktop mode and the saved gamma ramp**, and regaining it
  must reapply them. A process that exits or crashes without restoring leaves the display wrong, which is why the
  restoration also runs from the error path ([`sys_win.c`](sys_win.c.md)).
- Activation also **pauses the game** in single player and silences the mixer, which is a game decision expressed in the
  platform layer.

## `VID_SetMode`, `VID_SetWindowedMode`, `VID_SetFullDIBMode`, `VID_SetDefaultMode`, `CenterWindow`, `bSetupPixelFormat`, `VID_UpdateWindowStatus`

**Contract** — switch to a numbered mode: destroy the old window, create a window of the requested kind and size,
request a pixel format with a depth buffer, create and make current a rendering context, publish the viewport, tell the
input layer the new window rectangle, and report failure by falling back to the default mode.

**Invariants** — **the engine's logical resolution and the window's pixel size are separate**, with the ratio applied when
the viewport is computed ([`gl_rmain.c`](gl_rmain.c.md#mygluperspective-r_setupgl)) and the 2D layer always drawing in logical units
([`gl_draw.c`](gl_draw.c.md#gl_set2d)). That is what lets the interface be pixel-designed at one size and the world
render at another.

A failed mode change must **land somewhere working**, not exit. The fallback chain is: requested mode, then the default
windowed mode, then a fatal error.

## `VID_NumModes`, `VID_GetModePtr`, `VID_GetModeDescription`, `VID_GetExtModeDescription`, `VID_InitDIB`, `VID_InitFullDIB`, and the four describe commands

**Contract** — build the mode list from the desktop's current setting plus every enumerated display mode of an acceptable
depth, and expose it to the console and the menu by number and description.

## `VID_MenuDraw`, `VID_MenuKey`

**Contract** — the video menu page: list the modes, let the player pick one, and require a confirmation before committing.

**Invariants** — a mode change is **confirmed after the fact**, with a countdown that reverts if the player does not
accept. On hardware that could show nothing at all in a given mode, that is the difference between a settings screen and
a reboot. A rebuild targeting anything with an unreliable display path should keep it.

## `D_BeginDirectRect`, `D_EndDirectRect`, `VID_LockBuffer`, `VID_UnlockBuffer`, `VID_ForceLockState`, `VID_ForceUnlockedAndReturnState`

**Contract** — all empty.

**Invariants** — they exist because the software renderer's interface ([`vid.h`](vid.h.md)) demands them and shared code
calls them. In a hardware backend there is no buffer to lock and no way to draw outside a frame, so the disc indicator
([`gl_draw.c`](gl_draw.c.md)) simply does not appear. A rebuild with one renderer deletes the whole group.
