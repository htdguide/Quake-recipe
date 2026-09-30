# WinQuake/gl_draw.c

> The texture manager and the 2D layer: every image uploaded once under a name, small images packed into shared sheets, palette-indexed data expanded through the palette, and the screen furniture drawn as textured quads in a pixel-exact orthographic mode.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`wad.h`](wad.h.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)
**Used by** — [`gl_screen.c`](gl_screen.c.md) · [`sbar.c`](sbar.c.md) · [`menu.c`](menu.c.md) · [`console.c`](console.c.md) · [`gl_model.c`](gl_model.c.md) · [`gl_rsurf.c`](gl_rsurf.c.md)
**Tier floor** — none

## Purpose

The hardware counterpart of [`draw.c`](draw.c.md), and additionally the engine's whole texture manager: every texture in
the game, from map surfaces to the font, arrives through the upload routines here. So the file answers two questions a
rebuild must answer — how an indexed image becomes a texture, and how a fixed-layout 2D interface is drawn when the only
primitive available is a textured triangle.

## State

```text
RECORD Texture                        # the registry
  identifier : text                   # usually a file path
  texnum     : handle
  width, height : int
  mipmap     : bool
VARIABLE gltextures : Texture[limit] ;  numgltextures

CONSTANT scrap_sheets = 2 ;  sheet size 256 by 256
VARIABLE scrap_allocated : int[sheets][256]      # skyline, as in gl_rsurf.c
VARIABLE scrap_texels    : byte[sheets][256*256*4]
VARIABLE scrap_dirty : bool ;  scrap_texnum : handle base

VARIABLE menu_cachepics : (name, image, appended handle record)[128]
VARIABLE menuplyr_pixels : byte[4096]            # kept to recolour in the menu
VARIABLE char_texture, conback, draw_disc, draw_backtile
VARIABLE gl_filter_min, gl_filter_max            # the current filter mode
VARIABLE gl_picmip, gl_max_size                  # player-set detail limits
VARIABLE d_8to24table                            # the palette, as true colour
```

**Invariants** — the registry is keyed by **name**, and a second request for the same name returns the existing handle.
That is what makes "load the texture for this surface" idempotent across the many maps that share a texture name, and it
is also the deduplication that keeps a map's texture count down. A size mismatch under a repeated name is fatal — the
names are a namespace, and a collision is a content bug worth failing on.

An image loaded with an **empty** name is never findable again and always uploads fresh. That is the escape hatch for
per-instance textures such as translated player skins.

## `Scrap_AllocBlock`, `Scrap_Upload`, `Draw_PicFromWad`, `Draw_CachePic`

**Contract** — pack an image smaller than a threshold into one of two shared sheets, returning the sheet and the
position; upload every sheet when anything in them changed; and load an interface image either from the interface
archive or from a file, returning it with its handle and its coordinates within a sheet appended.

**Invariants** —

- **Small images are packed into two shared sheets** by the same skyline allocator as the lightmaps
  ([`gl_rsurf.c`](gl_rsurf.c.md#allocblock)) — including the same off-by-one that never tries the rightmost column.
  The reason is stated plainly in the source: some hardware handled many small textures badly. A packed image is
  addressed by a handle *plus* a coordinate rectangle, which is why every interface image carries four texture
  coordinates rather than assuming zero-to-one.
- The sheets are **uploaded lazily**, once, before the first draw that needs them, because packing happens across many
  separate load calls.
- The player-model image is **also kept as raw pixels**, because the multiplayer menu recolours it live and cannot read
  a texture back.
- An interface image record has **padding after it** so the handle and coordinates can be written into the bytes
  following the image header. That is a memory-layout trick to avoid a second allocation, and it is exactly the kind of
  incidental thing a rebuild replaces with a record containing both.

## `GL_LoadTexture`, `GL_LoadPicTexture`, `GL_FindTexture`

**Contract** — look the name up and return the existing handle if present, otherwise take the next handle, record the
entry, and upload the indexed image. Find returns a handle by name or nothing.

**Invariants** — **the handle is the engine's own counter** ([`glquake.h`](glquake.h.md)), incremented per texture.
There is no free operation: textures live until the process ends, and a map change re-registers the names it needs. A
rebuild on an API that owns its handles keeps the registry and stores the API's handle in it.

## `GL_Upload8`, `GL_Upload8_EXT`

**Contract** — expand an indexed image through the palette into true colour and hand it to the true-colour path;
detecting first whether the image actually uses the transparent index, and if not, treating it as opaque. When the
hardware accepts indexed textures directly and the image is opaque, upload the indices instead.

```text
FUNCTION upload_indexed(pixels, width, height, mipmap, alpha)
  IF alpha was requested
    uses_transparency = false
    FOR EACH pixel
      IF its index is the transparent index  uses_transparency = true
      expanded[i] = palette[index]
    IF NOT uses_transparency  treat the image as opaque after all
  ELSE
    expand every pixel, four at a time
  IF the hardware takes indexed textures AND the image is opaque
    upload the indices directly ;  RETURN
  upload the expanded image
```

**Invariants** —

- **The last palette entry is the transparent index**, by convention throughout the game's content. An image is
  declared as possibly-transparent by its caller, and this routine *demotes* it to opaque when no such pixel is
  present — which matters because a transparent texture cannot be drawn in the fast opaque path.
- The indexed upload path exists because expanding to true colour quadrupled texture memory, which was scarce. It is
  gated on an extension and excluded for the packed sheets. A rebuild ignores it.
- The opaque path processes **four pixels per iteration and fails on a size not divisible by four**. That is a hand
  unroll, incidental, and the failure is a latent bug for any content with an odd-sized opaque image. A rebuild drops
  both.

## `GL_Upload32`, `GL_ResampleTexture`, `GL_Resample8BitTexture`, `GL_MipMap`, `GL_MipMap8Bit`

**Contract** — round the dimensions up to powers of two, then reduce them by the player's detail setting and clamp to
the hardware's maximum; resample the image to that size if it changed; upload it; then, if mipmapping was asked for,
repeatedly halve the image and upload each level. Set the filter mode: filtered between levels when mipmapped, and the
plain magnification filter otherwise.

```text
FUNCTION upload_true_colour(data, width, height, mipmap, alpha)
  scaled_w = the next power of two at or above width ;  likewise height
  scaled_w >>= detail_reduction ;  scaled_h >>= detail_reduction
  clamp both to the hardware maximum
  FAIL if the result exceeds the scratch buffer
  IF the size is unchanged AND no mipmaps  upload directly ;  RETURN
  resample into the scratch buffer at the new size
  upload level 0
  IF mipmap
    level = 0
    WHILE either dimension exceeds 1
      halve the scratch in place ;  clamp each dimension to at least 1
      upload the next level
```

**Invariants** —

- **Dimensions are rounded up to powers of two, not down**, because the hardware of the day required it; content is
  already power-of-two so the path is rarely taken. A rebuild on hardware without the restriction deletes the rounding
  and keeps the clamping.
- The **detail reduction is applied by discarding the top mip levels**, so a player on weak hardware trades sharpness
  for memory with one setting. This applies to interface images too, via a separate setting, which is why the menu can
  go blurry.
- Halving is a **box filter over each two-by-two group**, applied *in place* on the scratch buffer. It must be a box
  filter and not point sampling, or distant surfaces shimmer. The indexed variant cannot average indices, so it picks
  one of the four — which is why the indexed path looks worse at distance and is another reason to ignore it.
- **A non-mipmapped texture uses the magnification filter for both directions.** Using a mip filter on a texture with
  no mip levels is undefined on some hardware, and the interface images are exactly that case.
- Mip generation is done by the engine rather than the library's utility, and the source keeps the library version
  beside it, disabled. The reason is control over the filter and the resampling; a rebuild should generate its own for
  the same reason.

## `Draw_Init`, `Draw_TextureMode_f`

**Contract** — load the font, the console background and the tiling and disc images; build the console background by
drawing the version text into it; register the texture-mode command and the detail settings. The command sets the
filter mode by name and re-applies it to every mipmapped texture already loaded.

**Invariants** — changing the filter mode must **walk the registry and re-set the parameter on each texture**, since the
setting is per texture and not global. That is the only reason the registry stores whether a texture was mipmapped.

## `Draw_Character`, `Draw_String`, `Draw_DebugChar`

**Contract** — draw one character of the fixed 128-by-128 font sheet at a pixel position, skipping the space character;
draw a string as successive characters advancing by the cell width.

**Invariants** — the font is a **16 by 16 grid of 8 by 8 cells**, so a character's coordinates are its code's low and
high nibbles scaled by a sixteenth. Space is skipped rather than drawn, which is a real saving when the console is full.

A small inset is applied to each cell's coordinates to keep the filter from bleeding a neighbouring glyph in. Without it
every character shows fringes of its neighbours — a rebuild symptom worth naming.

## `Draw_Pic`, `Draw_AlphaPic`, `Draw_TransPic`, `Draw_TransPicTranslate`

**Contract** — draw an image at a pixel position, opaque, blended at a given opacity, or with its palette remapped
through a translation table. Uploads the packed sheets first if they are dirty.

**Invariants** — the translated variant **re-uploads the image every call** through a temporary handle, because it is
used only for the recolouring preview in the menu and the table changes as the player drags a slider. Recorded because
it is the one place the engine uploads a texture per frame on purpose.

## `Draw_ConsoleBackground`, `Draw_TileClear`, `Draw_Fill`, `Draw_FadeScreen`, `Draw_BeginDisc`, `Draw_EndDisc`

**Contract** — draw the console image at a given height, blended when partially down; fill a rectangle with a tiling
image, the texture coordinates taken from the screen position so adjacent calls line up; fill a rectangle with a palette
colour; darken the whole screen with a blended black quad; and the disc-activity indicator.

**Invariants** — the tiling fill derives its texture coordinates from **screen coordinates divided by the tile size**,
which is what makes several separate fills form one continuous pattern. Deriving them from the rectangle instead makes
every panel restart the pattern — visible and wrong.

The disc indicator is a **no-op in the hardware renderer**, because drawing it would require presenting a frame
mid-load. In software it wrote directly to the front buffer ([`draw.c`](draw.c.md)). This is a capability the hardware
path loses, and the recipe records it because a rebuild will wonder why the function is empty.

## `GL_Set2D`

**Contract** — installs an orthographic projection matching the engine's logical resolution with the origin at the top
left, disables depth testing and culling, and enables alpha testing.

**Invariants** — **the origin is at the top left and one unit is one pixel**, so every 2D coordinate in the game's
interface code ([`sbar.c`](sbar.c.md), [`menu.c`](menu.c.md)) is a pixel position in the logical resolution and needs no
conversion. That is what lets the entire interface be shared with the software renderer unchanged, and it is the single
most valuable thing in this file for a rebuilder: **define the 2D layer as a pixel-exact top-left orthographic space and
the whole interface becomes renderer-independent.**

Depth testing off plus alpha testing on means the interface draws in submission order with hard-edged transparency —
order is the only depth, which is why the interface code's drawing order is load-bearing.

## `GL_Bind`, `GL_SelectTexture`

**Contract** — bind a texture handle, doing nothing if it is already bound; and select which texture unit subsequent
calls affect, remembering the last selection.

**Invariants** — the redundancy check is the point ([`glquake.h`](glquake.h.md)). Note that the cached value must be
**per unit**, and the unit selector keeps its own record for exactly that reason; a single cached binding shared across
units is a subtle rebuild bug that shows up only with multitexture enabled.
