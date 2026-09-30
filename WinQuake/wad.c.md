# WinQuake/wad.c

> Loads the one texture archive the engine keeps open, and resolves a 16-character lump name to bytes.

**Needs** — [`wad.h`](wad.h.md) · [`quakedef.h`](quakedef.h.md) · [`common.h`](common.h.md) (the file loader and byte-order accessors) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`draw.c`](draw.c.md) and [`gl_draw.c`](gl_draw.c.md) · [`model.c`](model.c.md) · [`sbar.c`](sbar.c.md) · [`host.c`](host.c.md) loads the archive
**Tier floor** — T1 as written, because the directory is byte-swapped in place inside the loaded file and every lookup returns a pointer into it. A T2 rebuild returns byte ranges and meets every contract.

## Purpose

One hundred and forty lines, and the only thing in it worth a second look is a
deliberate choice about name comparison: names are normalized to a fixed 16-byte field
at load and at lookup, specifically so the comparison can proceed four bytes at a time.
The engine loads exactly one of these archives — the interface graphics — and keeps it
for the process's life.

## State

```text
VARIABLE wad_base     : bytes             # the whole file, on the hunk
VARIABLE wad_lumps    : list<LumpInfo>    # a view INTO wad_base, swapped in place
VARIABLE wad_numlumps : int
```

**Invariants** — the directory is not copied. It is read where it lies and byte-swapped
where it lies, and every returned lump is an offset into the same buffer. So the
archive must be loaded onto the hunk (never the evictable cache) and must outlive every
pointer handed out, which it does by never being released.

## `W_CleanupName`

**Contract** — takes a name and a 16-byte destination, which may be the same storage;
copies up to 16 characters, folding the twenty-six ASCII uppercase letters to lowercase,
and zero-fills the remainder of the field.

```text
FUNCTION w_cleanup_name(in, out)
  i = 0
  WHILE i < 16 AND in[i] IS NOT the terminator
    out[i] = lowercase_ascii(in[i])
    i = i + 1
  WHILE i < 16
    out[i] = 0
    i = i + 1
```

**Invariants** — the result is exactly 16 bytes with every byte past the name set to
zero. That total zero-fill is what makes the fixed-width comparison in
[`W_GetLumpinfo`](#w_getlumpinfo) correct: two names that agree up to their common
length and are both zero-filled compare equal only if they are the same name.

A 16-character name has no terminator inside the field, so the field is not a
zero-terminated string in general — despite the format's own comment claiming it must
be. The engine's own lumps are all shorter.

**Notes** — the case folding means lookups are case-insensitive, unlike the *asset*
archive in [`common.c`](common.c.md#com_findfile), which is case-sensitive. Two archive
formats in one engine with opposite case rules is a trap for a rebuild; both rules are
observable, so keep both.

## `W_LoadWadFile`

**Contract** — takes a path; loads the whole file onto the hunk through the virtual
filesystem, validates the four-byte magic, reads the directory's location and size,
byte-swaps each entry's offset and length, normalizes each name in place, and
byte-swaps the two dimension fields of every interface graphic. Fatal error when the
file cannot be loaded or the magic is not the expected four bytes.

```text
FUNCTION w_load_wad_file(filename)
  wad_base = load filename onto the hunk
  IF wad_base IS nothing  FAIL WITH "couldn't load <filename>"
  IF the first 4 bytes ARE NOT "WAD2"
    FAIL WITH "Wad file <filename> doesn't have WAD2 id"
  wad_numlumps = little_endian(header.numlumps)
  wad_lumps    = wad_base + little_endian(header.infotableofs)
  FOR EACH lump IN wad_lumps
    lump.filepos = little_endian(lump.filepos)
    lump.size    = little_endian(lump.size)
    w_cleanup_name(lump.name, lump.name)             # in place
    IF lump.type IS type_qpic
      byte-swap the two dimension fields at wad_base + lump.filepos
```

**Invariants** — the `disksize` field is **not** byte-swapped, because nothing reads
it. Neither is the compression field checked. A rebuild should swap everything it
intends to read and should reject a non-zero compression field.

The interface graphics are swapped eagerly at load rather than at use, so that
[`draw.c`](draw.c.md) can read the dimensions as plain fields. That eager pass is the
reason the type field is consulted here at all.

**Notes** — the loop's counter is an unsigned value compared against a signed count, so
a negative count from a corrupt file becomes a very large loop bound. One of several
places where this format's loader trusts its input; a rebuild should validate the
directory's offset and length against the file's actual size before walking it.

## `W_GetLumpinfo`

**Contract** — takes a name; normalizes it, then scans the directory for an exact match
over the full 16-byte field. Returns the entry. A name that is not present is a **fatal
error**.

```text
FUNCTION w_get_lumpinfo(name) -> LumpInfo
  clean = w_cleanup_name(name)
  FOR EACH lump IN wad_lumps
    IF lump.name == clean                 # full 16 bytes
      RETURN lump
  FAIL WITH "W_GetLumpinfo: <name> not found"
```

**Invariants** — failing rather than returning nothing is the right contract for this
engine: every caller names a lump the shipped content contains, so absence means the
content is wrong and continuing would draw garbage. A rebuild loading third-party
content should still fail, loudly, rather than substitute a placeholder.

**Notes** — the scan is linear over roughly 160 entries and runs once per interface
graphic per level load, so its cost never mattered. The four-bytes-at-a-time comparison
the normalization enables is not actually used — the code compares as
zero-terminated strings. The normalization is still required, for the case folding.

## `W_GetLumpName`, `W_GetLumpNum`

**Contract** — return a lump's data by name or by index. The index form validates the
index and fails on one out of range.

**Notes** — the index form's bound test admits an index exactly equal to the count,
reading one entry past the directory. No caller uses the index form, so it has never
fired; a rebuild should write the correct test.

## `SwapPic`

**Contract** — byte-swaps an interface graphic's width and height in place.
