# WinQuake/wad.h

> The texture archive format: a flat directory of typed, named lumps, of which the engine reads only the palette, the interface graphics and the wall textures.

**Needs** — nothing; this is a leaf
**Used by** — [`wad.c`](wad.c.md) · [`draw.c`](draw.c.md) and [`gl_draw.c`](gl_draw.c.md) (interface graphics) · [`model.c`](model.c.md) (textures a map did not embed) · [`sbar.c`](sbar.c.md) · [`quakedef.h`](quakedef.h.md) includes it for everyone
**Tier floor** — none; a byte layout

## Purpose

Two different archive formats exist in this engine and they are easy to confuse. The
one in [`common.c`](common.c.md) is the *asset* archive: a flat bag of files, addressed
by path, backing the virtual filesystem. This one is the *texture* archive: a flat bag
of named lumps, addressed by a 16-character name, holding the palette, the interface
graphics and the lighting table.

Exactly one of these is loaded at a time, into a global, and it is the interface
graphics archive. Wall textures normally live embedded inside the map
([`bspfile.h`](bspfile.h.md)) rather than here.

## State

```text
VARIABLE wad_base     : bytes                # the whole archive, in memory
VARIABLE wad_lumps    : list<LumpInfo>       # pointing into wad_base
VARIABLE wad_numlumps : int
```

**Invariants** — one archive, globally. Loading a second replaces the first. The
directory is not copied out of the file — it is read in place and byte-swapped in
place, which is why the archive is loaded onto the hunk and kept for the process's
life.

## The file

```text
RECORD WadInfo
  identification : text[4]      # "WAD2"
  numlumps       : int
  infotableofs   : int          # offset of the directory from the file start

RECORD LumpInfo                 # one directory entry, 32 bytes
  filepos     : int
  disksize    : int             # as stored
  size        : int             # uncompressed
  type        : byte
  compression : byte            # 0 none, 1 an LZSS scheme
  pad1, pad2  : byte
  name        : text[16]        # lowercase, zero-padded, zero-terminated
```

## Lump types

```text
CONSTANT type_none    = 0
CONSTANT type_label   = 1     # a marker, no data
CONSTANT type_palette = 64    # 256 RGB triples
CONSTANT type_qtex    = 65
CONSTANT type_qpic    = 66    # an interface graphic: two dimensions then pixels
CONSTANT type_sound   = 67
CONSTANT type_miptex  = 68    # a wall texture with four mip levels
```

**Notes** — the numbering starts at 64 because the values above it are "64 plus a tool
command number", a convention from the asset pipeline. The engine reads only the
palette, interface graphics and wall-texture types; the sound and texture-source types
are produced by the tools and never consumed here.

## The interface graphic

```text
RECORD Pic
  width, height : int          # little-endian
  data          : byte[width * height]   # palette indices, row major
```

**Invariants** — palette index 255 is transparent, by convention shared with every
other image in the engine. There is no stride, no padding, and no alignment: rows are
exactly `width` bytes.

## Compression

**Invariants** — the compression field exists and names an LZSS scheme. **Nothing in
the engine decompresses.** Every lump in the shipped archive is stored, and the loader
does not check the field. A rebuild should assume stored and reject anything else
loudly, rather than reproducing a silent misread.

## `W_CleanupName`

**Contract** — normalizes a lump name into the 16-byte field's canonical form:
lowercased, and zero-filled to the full width. Safe to perform in place.

**Notes** — the file's comment says the padding is spaces, and the code writes zeros.
The zeros are what matters, because lookup compares the fixed-width fields.

## `W_LoadWadFile`

**Contract** — takes a path; loads the whole archive onto the hunk, validates the
magic, byte-swaps the directory in place, normalizes every name, and byte-swaps the
header of every interface graphic. Fatal error if the file is missing or the magic is
wrong.

## `W_GetLumpinfo`, `W_GetLumpName`, `W_GetLumpNum`

**Contract** — find a lump by name or by index and return its directory entry or its
data. A name that is not present is a **fatal error**, not a failure return — every
caller asks for a lump it believes is there, and a missing one means the content is
wrong.

## `SwapPic`

**Contract** — byte-swaps an interface graphic's two dimension fields in place.
Idempotent only on a little-endian host, where it does nothing.
