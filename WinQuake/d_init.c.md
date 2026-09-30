# WinQuake/d_init.c

> The rasterizer's initialization and per-frame setup: declares its capabilities to the renderer, chooses the destination buffer, and computes the mip thresholds.

**Needs** — [`d_local.h`](d_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`vid.h`](vid.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — [`r_main.c`](r_main.c.md) calls the initializer and the frame setup
**Tier floor** — none

## Purpose

Where the software rasterizer answers the capability questions
[`d_iface.h`](d_iface.h.md) poses, and where the per-frame decision about *where* to draw is made — the
display, or the warp buffer when the player is underwater.

## State

```text
VARIABLE d_subdiv16 : Cvar = 1     # use the sixteen-bit span filler when
                                   # available
VARIABLE d_mipcap   : Cvar = 0     # never use a mip level finer than this
VARIABLE d_mipscale : Cvar = 1     # bias the whole mip selection
CONSTANT basemip : real[3] = { 1.0, 0.4, 0.2 }     # 1.0, 0.5*0.8, 0.25*0.8
```

**Invariants** — the base thresholds are 1, 0.4 and 0.2, written as products so their derivation is
visible: nominally one half and one quarter, each multiplied by 0.8 to bias mip transitions slightly
later than geometry would suggest. That bias is a quality choice — it keeps a finer level in use a little
longer than strictly needed.

## `D_Init`

**Contract** — registers the three variables and declares the rasterizer's capabilities.

```text
FUNCTION d_init()
  r_skydirect = 1
  register d_subdiv16, d_mipcap, d_mipscale
  r_drawpolys = false                    # this rasterizer wants a SPAN LIST
  r_worldpolysbacktofront = false        # front to back
  r_recursiveaffinetriangles = true      # it can use recursive subdivision
  r_pixbytes = 1                         # one byte per destination pixel
  r_aliasuvscale = 1.0
```

**Invariants** — these five assignments *are* the negotiation
([`d_iface.h`](d_iface.h.md#the-capability-flags)). A hardware rasterizer would set the first three
oppositely and the fourth to false, and the renderer would then take entirely different code paths.

## `D_SetupFrame`

**Contract** — per frame: selects the destination buffer and its stride, resets the surface cache's rover
bookkeeping, computes the three mip thresholds from the tunables, and selects the span filler.

```text
FUNCTION d_setup_frame()
  IF the underwater warp is active
    d_viewbuffer = the warp buffer ;  screenwidth = 320
  ELSE
    d_viewbuffer = the display surface ;  screenwidth = the surface's stride

  d_roverwrapped = false
  d_initial_rover = the surface cache's current rover     # to detect a full lap

  d_minmip = d_mipcap, CLAMPED INTO 0..3
  FOR EACH i IN 0..2  d_scalemip[i] = basemip[i] * d_mipscale

  d_drawspans = the sixteen-bit filler IF accelerated AND d_subdiv16
                ELSE the eight-bit filler
  d_aflatcolor = 0
```

**Invariants** — **rendering into the warp buffer changes the stride to 320 regardless of the display's
resolution**, which is why the underwater effect is blockier at high resolutions
([`d_scan.c`](d_scan.c.md#d_warpscreen)).

The rover snapshot is how thrashing is detected: if the cache's allocation cursor comes all the way back
around within one frame, it has evicted something it will need again
([`d_surf.c`](d_surf.c.md)).

**Notes** — the filler selection reads as choosing between eight- and sixteen-*bit* fillers, but the names
mean eight- and sixteen-*pixel subdivision intervals*, and the accelerated build offers both. So the
tunable is a quality-versus-speed control, not a pixel-depth one — the naming is genuinely misleading and
a rebuild should name it for what it does.

## The do-nothing operations

**Contract** — `D_CopyRects` and `D_UpdateRects` do nothing, because this rasterizer draws straight into
the surface the video backend exposes. `D_TurnZOn` does nothing, because the depth buffer is always on.
`D_EnableBackBufferAccess` and `D_DisableBackBufferAccess` lock and unlock the surface
([`vid.h`](vid.h.md#vid_lockbuffer-vid_unlockbuffer)).

**Invariants** — the comment on the first explains the case it exists for: a backend with no direct access
to the display, where the engine draws somewhere else and the backend copies. No such backend ships.
