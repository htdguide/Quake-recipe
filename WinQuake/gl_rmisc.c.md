# WinQuake/gl_rmisc.c

> Renderer startup and per-map reset, the generated particle texture, per-player skin recolouring, and the two diagnostic commands.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`gl_model.h`](gl_model.h.md) · [`gl_draw.c`](gl_draw.c.md)
**Used by** — [`host.c`](host.c.md) at startup; [`cl_parse.c`](cl_parse.c.md) on a new map and on a player's appearance change
**Tier floor** — none

## Purpose

The hardware counterpart of [`r_misc.c`](r_misc.c.md): the same registration of the renderer's console variables and
commands, the same per-map reset. Two things here have no software counterpart and are worth their own sections.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## `R_InitParticleTexture`

**Contract** — builds a small round texture in memory — a filled disc with a soft edge, opaque in the middle and
transparent outside — and uploads it. All particles are drawn as this one texture.

**Invariants** — the software renderer draws a particle as a **square of solid pixels** ([`d_part.c`](d_part.c.md)); the
hardware renderer draws a **textured quad**, and the texture is generated rather than shipped so the content does not
have to carry it. That difference is why particles look round in one renderer and square in the other, and a rebuild
choosing the hardware path should generate the texture the same way rather than adding an asset.

## `R_TranslatePlayerSkin`

**Contract** — takes a player slot; builds that player's skin by remapping the two colourable ranges of the palette
according to their chosen shirt and trouser colours, reduces it by the player-skin detail setting, and uploads it into
that player's reserved texture handle.

```text
FUNCTION translate_player_skin(slot)
  build a 256-entry identity translation
  top    = the player's shirt colour ;  bottom = their trouser colour
  FOR each of the 16 entries of the shirt range
    translation[shirt_base + i] = (top    < 8) ? top_base    + i : top_base    + 15 - i
  FOR each of the 16 entries of the trouser range
    translation[trouser_base + i] = (bottom < 8) ? bottom_base + i : bottom_base + 15 - i
  source = the player model's kept pixel copy, reduced by the detail setting
  FOR each pixel  out[p] = translation[source[p]]
  upload into the handle reserved for this slot
```

**Invariants** —

- **Two sixteen-entry ranges of the palette are reserved for player colours**, and a colour choice selects which range
  those sixteen entries map to. This is a property of the game's palette, shared with the software renderer and with the
  server's colour field in the protocol ([`protocol.h`](protocol.h.md)).
- **The upper eight colour choices reverse the ramp.** Those palette ranges run dark-to-light where the others run
  light-to-dark, so mapping them straight would make half the colours look inverted. The reversal is the fix and it is
  not optional.
- The result goes into the **handle reserved for this slot** ([`glquake.h`](glquake.h.md)), which is why handles are
  allocated in ranges: the draw path needs "player 3's skin" without a lookup.
- This runs whenever a player's colours change mid-game, so it must be cheap enough to do live — hence the detail
  reduction, which is a separate setting from the general one because player models are the most numerous skins in a
  deathmatch.
- It relies on the player model's pixels having been kept at load ([`gl_model.c`](gl_model.c.md)), since a texture cannot
  be read back.

## `R_Init`, `R_NewMap`, `D_FlushCaches`

**Contract** — register the renderer's commands and variables and build the particle texture and the particle pool; reset
per-map state, clear the light styles to a neutral value, rebuild every lightmap page, and find the map's mirror surface
by texture name; and a no-op cache flush kept so shared code links.

**Invariants** — the cache flush **does nothing** here, because there is no surface cache. It exists only because
[`host.c`](host.c.md) and [`vid_*`](vid_win.c.md) call it. A rebuild with one renderer deletes the call.

## `R_TimeRefresh_f`, `R_Envmap_f`

**Contract** — render 128 frames while rotating a full turn and report the frame rate; and render six views along the
axes, reading each back to build a cube map of the current location.

**Invariants** — the timing command is the engine's only benchmark and is worth reproducing as one: a fixed number of
frames over a fixed rotation is a comparable number across machines and across rebuilds. It is also the measurement the
recipe's conformance section refers to.

The environment capture sets the flag that suppresses the view model and the screen tint
([`gl_rmain.c`](gl_rmain.c.md)), which is the only reason that flag exists.
