# WinQuake/common.c

> The engine's own standard library, the wire codec, and the layered virtual filesystem that lets archives and loose directories shadow one another.

**Needs** — [`common.h`](common.h.md) · [`quakedef.h`](quakedef.h.md) · [`zone.h`](zone.h.md) · [`crc.h`](crc.h.md) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md) · [`console.h`](console.h.md) · [`draw.h`](draw.h.md) (the disc-access indicator) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — every module. The filesystem is reached by [`model.c`](model.c.md), [`snd_mem.c`](snd_mem.c.md), [`wad.c`](wad.c.md), [`host_cmd.c`](host_cmd.c.md), [`cl_demo.c`](cl_demo.c.md), [`pr_edict.c`](pr_edict.c.md) and [`sv_main.c`](sv_main.c.md); the wire codec by [`sv_send.c`](sv_main.c.md), [`cl_parse.c`](cl_parse.c.md) and [`net_dgrm.c`](net_dgrm.c.md)
**Tier floor** — T1 as written; every contract here is satisfiable at T3

## Purpose

Four unrelated subsystems share this file: reimplementations of standard string and
memory operations, the typed codec that turns values into protocol bytes and back,
the growable byte buffer both the codec and the network layer are built on, and the
virtual filesystem. They are together because all four are prerequisites of
everything else, and the file is best read as four chapters.

The filesystem is the part worth the most attention. It implements a rule that
third-party content depends on — later-added sources shadow earlier ones, and a
loose file shadows an archived one — and it is where the engine's one security
decision lives.

## State

```text
RECORD SearchPath              # one element of the lookup chain
  filename : text              # a directory, when this element is a directory
  pack     : optional<Pack>    # an archive, when this element is an archive
  next     : optional<SearchPath>
  # exactly one of filename and pack is meaningful

RECORD Pack                    # an opened archive
  filename : text
  handle   : int               # one OS handle, shared by every read from it
  files    : list<PackEntry>

RECORD PackEntry
  name    : text[64]           # the in-game path, forward slashes, lowercase
  filepos : int
  filelen : int

VARIABLE search_paths  : optional<SearchPath>   # head of the chain; newest first
VARIABLE com_gamedir   : text                   # where writes go
VARIABLE com_cachedir  : text                   # development mirror; usually empty
VARIABLE com_filesize  : int                    # length of the last file found
VARIABLE com_modified  : bool                   # content is not the original
VARIABLE com_argc, com_argv                     # the parsed command line
VARIABLE com_token     : text[1024]             # the last token parsed
```

**Invariants** — the chain is searched front to back, and every source is pushed
onto the *front* as it is added, so the last-added source wins. Within one
directory's addition, the directory itself is pushed first and then its archives,
so **archives shadow the loose files beside them**. Across directories, a later
directory and all its archives shadow everything before. This ordering is the
whole modding model and a rebuild must reproduce it exactly.

---

## Chapter 1 — standard-library replacements

These exist because 1996 platform libraries disagreed. Their contracts are given
only where they *differ* from the obvious; where a rebuild's own library matches,
it should use it.

### `Q_memset`, `Q_memcpy`

**Contract** — fill and copy. Both check whether the addresses and the count are
all multiples of four and, if so, move four bytes at a time.

**Notes** — pure optimization for a machine without a fast byte-move instruction,
and the alignment test reads a pointer as an integer. Incidental; use the language's
own.

### `Q_memcmp`

**Contract** — takes two buffers and a count; returns 0 when every byte matches and
−1 otherwise. Compares **backwards**, from the last byte to the first.

**Notes** — it is an equality test, not an ordering, despite the name. No caller
uses the sign. The backwards scan is not a bug but it is also not a decision worth
keeping.

### `Q_strcpy`, `Q_strncpy`, `Q_strlen`, `Q_strcat`, `Q_strrchr`

**Contract** — the obvious operations on zero-terminated byte strings, with one
difference: `Q_strncpy` writes the terminator only when it stopped *before*
exhausting the count, so a source exactly as long as the count leaves the
destination unterminated.

```text
FUNCTION q_strncpy(dest, src, count)
  WHILE src has a byte remaining AND count > 0
    copy one byte ;  count = count - 1
  IF count != 0
    append a zero byte
  # when count reached 0 exactly, no terminator is written
```

**Notes** — this is a real trap. The engine relies on it in one place —
[`zone.c`](zone.c.md) copies an 8-byte hunk name into an 8-byte field deliberately
without a terminator — and would be wrong with the standard behaviour there. A
rebuild should make the fixed-width-field case explicit and use ordinary
terminated copies everywhere else.

### `Q_strcmp`, `Q_strncmp`

**Contract** — return 0 when equal, −1 when not. **Not an ordering**: the negative
result carries no information about which string sorts first.

**Notes** — every caller uses these as equality tests, so the missing ordering is
harmless in this engine. A rebuild that swaps in a real comparison must check no
caller started depending on the sign.

### `Q_strcasecmp`, `Q_strncasecmp`

**Contract** — as above, ignoring case for the twenty-six ASCII letters only. The
counted form checks the count *after* fetching both bytes, so it compares one byte
past the limit in the sense that it reads it; but it returns equal before using it.
The uncounted form is the counted form with a limit of 99,999.

**Notes** — ASCII-only folding is a platform assumption worth stating: byte values
above 127 are the engine's alternate glyph set, not letters, and folding them would
be wrong.

### `Q_atoi`

**Contract** — parses an integer from the front of a string, stopping at the first
byte that does not fit. Accepts an optional leading minus, then one of three forms:
a `0x` or `0X` prefix introduces hexadecimal; a leading apostrophe means "the
numeric value of the next byte"; otherwise decimal. No whitespace skipping, no
overflow detection, no error signal — an unparseable string yields zero.

**Notes** — the apostrophe form is used by console scripts to bind a key by
character. The hex form is used for item bit masks. Both are part of the
configuration file format and a rebuild that only handles decimal will fail to load
a real `config.cfg`.

### `Q_atof`

**Contract** — as `Q_atoi` but producing a float, additionally accepting a decimal
point. Accumulates all digits as an integer and then divides by a power of ten, in
double precision. No exponent notation. No overflow or underflow detection.

```text
FUNCTION q_atof(str) -> real
  sign = -1 IF str starts with '-' ELSE 1
  IF str starts with "0x" or "0X"   RETURN sign * hex_digits_as_integer
  IF str starts with an apostrophe  RETURN sign * byte_value_of_next_character
  decimal_position = -1 ;  digits_seen = 0 ;  val = 0  # val is double precision
  FOR EACH character c
    IF c IS '.'
      decimal_position = digits_seen ;  CONTINUE
    IF c IS NOT a digit  BREAK
    val = val*10 + digit_value(c) ;  digits_seen = digits_seen + 1
  IF decimal_position == -1  RETURN sign * val
  RETURN sign * (val / 10^(digits_seen - decimal_position))
```

**Notes** — no exponent notation is a real limit: a console variable set to `1e-3`
parses as 1. The accumulate-then-divide order is more accurate than repeated
multiplication by a tenth, which is the reason for the double-precision
accumulator; a rebuild using its language's parser gets this and more.

---

## Chapter 2 — byte order

### `ShortSwap`, `LongSwap`, `FloatSwap`, and the no-swap identities

**Contract** — reverse the bytes of a 16-, 32- or 32-bit-float value, or return it
unchanged.

### `COM_Init`

**Contract** — probes the host byte order by writing the bytes `1, 0` and reading
them as a 16-bit value; binds the six accessors accordingly; registers the
`registered` and `cmdline` console variables and the `path` command; brings up the
filesystem; then verifies the content licence marker. Called once, early.

```text
FUNCTION com_init(basedir)
  probe = the two bytes (1, 0)
  IF reading probe as a 16-bit value gives 1
    big_endian = false
    big_* = the swapping forms ;  little_* = the identities
  ELSE
    big_endian = true
    big_* = the identities ;  little_* = the swapping forms
  register the "registered" and "cmdline" variables and the "path" command
  com_init_filesystem()
  com_check_registered()
```

**Notes** — deciding at run time from a probe rather than at build time is the
load-bearing choice. A rebuild in a language that exposes explicit little-endian
readers should use those and delete this section.

---

## Chapter 3 — the byte buffer and the wire codec

### `SZ_Alloc`, `SZ_Clear`, `SZ_Free`

**Contract** — `SZ_Alloc` attaches hunk storage of at least 256 bytes and sets the
length to zero. `SZ_Clear` sets the length to zero. `SZ_Free` also sets the length
to zero and nothing else — the storage is never released, because it is hunk
memory and the hunk does not release.

**Notes** — the two names for one operation are a leftover. A rebuild has one.

### `SZ_GetSpace`

**Contract** — takes a buffer and a length; reserves that many bytes at the end and
returns the reservation. When the reservation would not fit, the buffer's policy
decides: a buffer that does not permit overflow is a fatal error; a buffer that
does is **cleared**, flagged, and the reservation is satisfied from the front.
A single reservation larger than the whole buffer is always a fatal error.

```text
FUNCTION sz_get_space(buf, length) -> byte range
  IF buf.cursize + length > buf.maxsize
    IF NOT buf.allow_overflow   FAIL WITH "overflow without allowoverflow set"
    IF length > buf.maxsize     FAIL WITH "<length> is > full buffer size"
    buf.overflowed = true
    print "SZ_GetSpace: overflow"
    buf.cursize = 0                     # DISCARD everything already written
  range = buf.data + buf.cursize ... + length
  buf.cursize = buf.cursize + length
  RETURN range
```

**Invariants** — on a tolerated overflow the buffer's **existing contents are
thrown away**, not truncated and not preserved. This is load-bearing and
surprising: a server datagram that overflows loses the whole frame's worth of
updates, not just the last one, and the `overflowed` flag is how the caller learns
to send nothing at all this frame rather than a half message. A rebuild that
instead truncates sends a corrupt message that a client will try to decode.

### The writers

**Contract** — each appends one value in the protocol's encoding. See
[`common.h`](common.h.md#the-message-writers) for the table. Three deserve their
own note.

#### `MSG_WriteCoord`

```text
FUNCTION msg_write_coord(buf, f)
  msg_write_short(buf, truncate_to_int(f * 8))
```

**Invariants** — 1/8 world unit per step, truncating toward zero, sixteen bits
signed. The world is therefore bounded to ±4096 units and a coordinate's wire form
is not its exact value. Both the server and the client quantize the same way, which
is why prediction in [`QW`](../QW/client/pmove.c.md) can agree with the server at
all.

#### `MSG_WriteAngle`

```text
FUNCTION msg_write_angle(buf, f)
  # The multiplication binds to the TRUNCATED value, not to f:
  msg_write_byte(buf, (truncate_to_int(f) * 256 / 360) BITAND 255)
```

**Invariants** — this is not the obvious "scale then truncate". The source's
expression truncates `f` to a whole number of degrees *first*, then scales by
256/360 in **integer** arithmetic, then masks. The result is that sub-degree
precision is lost before the scaling, and the integer division introduces a second
rounding: 179 degrees and 180 degrees both encode to 127.

A rebuild must reproduce this exactly. It is almost certainly not what was
intended, but it is what every client decodes and what every recorded demo
contains, so "fixing" it desynchronizes playback and makes entity angles differ
from the original by up to a degree and a half.

#### `MSG_WriteString`

**Contract** — writes the bytes followed by a zero terminator; a null string writes
only the terminator, so a null and an empty string are indistinguishable on the
wire.

### The readers

**Contract** — see [`common.h`](common.h.md#the-message-readers). Each checks
whether the requested width fits in what remains, and on failure sets the bad-read
flag and returns −1 without advancing the cursor.

```text
FUNCTION msg_read_short() -> int
  IF msg_readcount + 2 > net_message.cursize
    msg_badread = true ;  RETURN -1
  value = data[msg_readcount] + (data[msg_readcount+1] SHIFTED LEFT 8)
  value = sign_extend_from_16_bits(value)
  msg_readcount = msg_readcount + 2
  RETURN value
```

**Invariants** — the short reader sign-extends; the byte reader does not; the char
reader does. `MSG_ReadFloat` **omits the bounds check entirely** and will read past
the end of the message. That is a genuine out-of-bounds read reachable from a
malicious server, and a rebuild should add the check — it changes nothing for
well-formed input.

`MSG_ReadString` accumulates into a 2048-byte static buffer, stopping at a zero
byte, at end of message, or at 2047 characters, and always terminates. Consecutive
calls reuse the buffer.

#### `MSG_ReadCoord`, `MSG_ReadAngle`

```text
FUNCTION msg_read_coord() -> real   RETURN msg_read_short() * (1/8)
FUNCTION msg_read_angle() -> real   RETURN msg_read_char()  * (360/256)
```

**Invariants** — the angle reader reads a **signed** byte, so the decoded range is
−180 to +178.6 degrees rather than 0 to 358.6. The writer masked to an unsigned
byte. The two are consistent modulo a full turn, which is all that matters, but a
rebuild that reads unsigned gets angles off by 360 in half the cases and will see
it immediately in any interpolation that averages two angles.

### `SZ_Write`, `SZ_Print`

**Contract** — `SZ_Write` copies a byte range in. `SZ_Print` appends a string and
its terminator, but if the buffer's last byte is already a zero it *overwrites*
that zero, so a sequence of prints produces one terminated string rather than
several.

```text
FUNCTION sz_print(buf, text)
  len = length(text) + 1                     # including the terminator
  IF the byte at buf.cursize-1 IS NOT zero
    append text and its terminator            # a fresh string
  ELSE
    back up over the existing terminator and append text and its terminator
```

**Notes** — reading `data[cursize-1]` on an empty buffer reads one byte before the
buffer. The original gets away with it because every caller has already written
something. A rebuild should model this as "a buffer holding one accumulating
zero-terminated string" and the anomaly disappears.

---

## Chapter 4 — token parsing and paths

### `COM_Parse`

**Contract** — takes a position in a text buffer; extracts one token into the shared
token buffer and returns the position after it, or nothing at end of input. Skips
whitespace (anything at or below the space character) and line comments introduced
by two slashes. A double-quoted run is one token with the quotes stripped and no
escape processing. Six punctuation characters — braces, parentheses, apostrophe and
colon — are each a token on their own. Otherwise a token runs to the next character
at or below the space character, or to one of those six punctuation characters.

```text
FUNCTION com_parse(data) -> optional<position>     # token left in com_token
  IF data IS nothing  RETURN nothing
  LOOP
    WHILE the current character <= ' '
      IF it is the terminator  RETURN nothing
      advance
    IF the next two characters are "//"
      skip to the end of the line ;  CONTINUE          # and re-skip whitespace
    BREAK

  IF the current character IS '"'
    advance past it
    accumulate until the next '"' or the terminator     # quotes stripped
    RETURN the position after the closing quote

  IF the current character IS one OF { '{', '}', '(', ')', '\'', ':' }
    the token is that one character
    RETURN the position after it

  accumulate characters until one IS <= ' ' OR IS one of those six
  RETURN the current position
```

**Invariants** — no bounds check on the 1024-byte token buffer: a longer token
overruns it. No escape sequences inside quotes, so a quoted string cannot contain a
quote. The six single-character tokens are what let the same parser read map entity
text, the compiled-game definition file and console scripts.

**Notes** — this one parser reads every text format the engine understands. A
rebuild should add the length check and keep everything else, including the
whitespace rule (`<= ' '`, which treats every control character as whitespace).

### `COM_SkipPath`, `COM_StripExtension`, `COM_FileExtension`, `COM_FileBase`, `COM_DefaultExtension`

**Contract** — `COM_SkipPath` returns the position after the last forward slash.
`COM_StripExtension` copies up to the **first** dot. `COM_FileExtension` returns up
to seven characters after the first dot, in a shared static buffer, or an empty
string. `COM_FileBase` extracts the name between the last slash and the last dot,
substituting the literal `?model?` when the result would be shorter than two
characters. `COM_DefaultExtension` appends an extension unless the final path
component already contains a dot.

**Notes** — the first-dot versus last-dot inconsistency between stripping and
basename extraction is real; a name like `b_shell0.bsp` is affected. The `?model?`
fallback exists because the result is used as an eight-character hunk tag and an
empty tag made the memory dump unreadable. Only forward slashes are recognized as
separators anywhere.

---

## Chapter 5 — command line

### `COM_InitArgv`

**Contract** — takes the process arguments; copies up to 50 of them into the
engine's own array, rebuilds them into a single 256-byte string for the `cmdline`
console variable, and terminates the array with a pointer to a single space rather
than with nothing. Detects `-safe` and, if present, appends seven fixed arguments
that disable every optional device. Detects `-rogue` and `-hipnotic` and sets the
game-variant flags.

```text
FUNCTION com_init_argv(argv)
  reconstruct up to 256 bytes of "arg arg arg " into com_cmdline
  copy up to 50 arguments into the engine's array, noting whether "-safe" appears
  IF "-safe" appeared
    append: -stdvid -nolan -nosound -nocdaudio -nojoy -nomouse -dibonly
  append a pointer to a single space as the terminator
  IF "-rogue"    IS present  rogue    = true ;  standard_quake = false
  IF "-hipnotic" IS present  hipnotic = true ;  standard_quake = false
```

**Notes** — the array is over-allocated by exactly seven slots so the safe-mode
append never needs a bounds check, and the terminator is a space rather than
nothing so that code scanning for the next argument finds something harmless. Both
are incidental; a rebuild uses a list.

### `COM_CheckParm`

**Contract** — takes an argument name; returns its 1-based position in the argument
list, or 0 if absent. Skips null entries, because one 1990s platform cleared
argument slots after reading them.

**Notes** — the convention throughout the engine is "check for the flag, then read
the next argument if the returned position is not the last". A rebuild should keep
the convention but return an optional index rather than overloading zero.

### `COM_CheckRegistered`

**Contract** — opens a specific 256-byte asset from the archive and compares it
against a copy compiled into the engine, treating a mismatch as corrupt data and a
fatal error. Its presence sets the `registered` variable and unlocks the parts of
the search path outside the base directory; its absence means the shareware
content, and combining shareware content with modified content is a fatal error.

**Notes** — this is a licence check, and it has one consequence a rebuild must
understand even if it drops the check: when the flag is clear,
[`COM_FindFile`](#com_findfile) refuses any path containing a separator in a
directory search path. That is not only an anti-piracy measure — it is also the
engine's sole defence against a server telling a client to write outside its game
directory.

---

## Chapter 6 — the virtual filesystem

### `COM_LoadPackFile`

**Contract** — takes an explicit archive path; opens it, validates the four-byte
magic, reads the directory, and returns an archive record holding one shared OS
handle and a table of entries. Fatal error on a wrong magic or on more than 2048
entries. A directory whose entry count or checksum differs from the shipped
original's marks the content as modified.

```text
FUNCTION com_load_pack_file(path) -> optional<Pack>
  IF the file cannot be opened  RETURN nothing
  read the header: 4 magic bytes, directory offset, directory length
  IF magic != "PACK"  FAIL WITH "<path> is not a packfile"
  directory offset and length are little-endian
  count = directory length / 64          # each entry is 56 name + 4 + 4 bytes
  IF count > 2048  FAIL WITH "<path> has <count> files"
  IF count != 339  com_modified = true           # not the shipped archive
  read the whole directory
  crc = CCITT-16 OVER the directory bytes
  IF crc != 32981  com_modified = true
  FOR EACH entry: copy the name, byte-swap the offset and length
  RETURN the archive with ONE shared OS handle
```

**Invariants** — every read from an archive uses the archive's single handle and
seeks to the entry's offset first. That is why [`COM_CloseFile`](#com_openfile-com_fopenfile-com_closefile) must
not close a handle that belongs to an archive, and why nothing may read from two
archived files concurrently.

The two constants — 339 entries and checksum 32981 — describe the shipped first
archive. They are only used to set the modified flag.

### `COM_AddGameDirectory`

**Contract** — takes a directory; records it as the write destination, pushes it
onto the front of the search chain, then pushes each of `pak0.pak`, `pak1.pak` and
so on until one is missing.

```text
FUNCTION com_add_game_directory(dir)
  com_gamedir = dir
  push a directory element for dir onto the front of the chain
  FOR i FROM 0 UPWARD
    pak = com_load_pack_file(dir + "/pak" + i + ".pak")
    IF pak IS nothing  BREAK
    push an archive element for pak onto the front of the chain
```

**Invariants** — because each is pushed onto the front, the resulting order is:
highest-numbered archive first, then downward, then the loose directory last. So
**within a game directory, archives shadow loose files**, and a higher-numbered
archive shadows a lower one. Numbering stops at the first gap, so `pak0` and `pak2`
without `pak1` loads only `pak0`.

**Notes** — the interaction with [`COM_FindFile`](#com_findfile) is subtle and
worth stating plainly, because it is the opposite of what most engines do and what
most people remember: a loose file **does not** override an archived one inside the
same game directory. It overrides only files in *earlier* game directories.

### `COM_InitFilesystem`

**Contract** — assembles the search chain. Reads the base directory from the
command line or from the platform backend, trimming one trailing separator. Reads
the development cache directory similarly, where a value beginning with a hyphen
disables caching. Adds the default game directory, then the mission-pack directories
if requested, then a user-specified game directory. A `-path` argument discards the
whole generated chain and builds one from the listed directories and archives, each
of which must load or it is a fatal error.

```text
FUNCTION com_init_filesystem()
  basedir = "-basedir" value, ELSE the platform's base directory
  strip one trailing '/' or '\' from basedir
  cachedir = "-cachedir" value (a leading '-' means none), ELSE the platform's
  com_add_game_directory(basedir + "/id1")
  IF "-rogue"     com_add_game_directory(basedir + "/rogue")
  IF "-hipnotic"  com_add_game_directory(basedir + "/hipnotic")
  IF "-game" <name>
    com_modified = true
    com_add_game_directory(basedir + "/" + name)
  IF "-path" <entries...>
    com_modified = true
    discard the entire chain
    FOR EACH following argument until one starts with '+' or '-'
      IF its extension IS "pak"
        load it as an archive; FAIL WITH "Couldn't load packfile" on failure
      ELSE
        add it as a directory
      push it onto the front
```

**Invariants** — the game directory is fixed for the life of the process. The
source states why: a server must not be able to make a client write files
elsewhere. Any rebuild that adds runtime game-directory switching has to re-derive
that protection.

### `COM_FindFile`

**Contract** — takes a game-relative path and exactly one of two output forms — an
OS handle or a buffered stream — and walks the chain front to back. An archive
element is searched by exact name comparison; a match seeks the shared handle to the
entry's offset, or opens a fresh stream positioned there. A directory element is
tried by concatenation and a modification-time stat. Sets the found length in
`com_filesize` and returns it, or sets the output to "not found", sets the length to
−1, and returns −1. Asking for both output forms, or neither, is a fatal error.

```text
FUNCTION com_find_file(filename, want_handle) -> int    # length, or -1
  FOR EACH element IN the search chain, front to back
    IF element IS an archive
      FOR EACH entry IN element.files
        IF entry.name == filename                 # exact, case-sensitive
          IF want_handle
            hand back the archive's SHARED handle, seeked to entry.filepos
          ELSE
            open a NEW stream on the archive file, seeked to entry.filepos
          com_filesize = entry.filelen
          RETURN com_filesize
    ELSE                                          # a directory
      IF NOT registered AND filename contains '/' or '\'
        CONTINUE                                  # refuse to leave the base
      path = element.filename + "/" + filename
      IF the file does not exist  CONTINUE
      IF a cache directory is configured
        mirror path into the cache when the cached copy is older, then read
        from the cache instead
      open it; com_filesize = its length
      RETURN com_filesize
  com_filesize = -1 ;  RETURN -1
```

**Invariants** — archive lookup is a **linear scan with exact byte comparison**, so
lookup is proportional to archive size and names must match case exactly. The
shipped archives are all lowercase; the engine lowercases nothing. A rebuild should
build a map at load time and must decide explicitly about case — folding it accepts
content the original rejects, which is usually what people want but is a change.

The path-separator refusal when unregistered is the engine's only sandbox.

**Notes** — the development cache-directory mirror copies a file locally when the
source is newer, and reads the copy. It exists because the authors worked over ISDN
from home. A rebuild should drop it; it is dead weight and its Windows-drive-letter
special case is a bug magnet.

### `COM_OpenFile`, `COM_FOpenFile`, `COM_CloseFile`

**Contract** — the two open forms differ only in which output the lookup fills.
`COM_CloseFile` closes the handle **unless** it belongs to an archive, in which case
it does nothing.

```text
FUNCTION com_close_file(handle)
  FOR EACH element IN the search chain
    IF element IS an archive AND element.pack.handle == handle
      RETURN                              # shared; must stay open
  close the handle
```

**Notes** — a rebuild that gives each archived read its own stream, or that models
an opened asset as a byte range over a memory-mapped archive, deletes this special
case and the concurrency restriction with it.

### `COM_LoadFile` and its four wrappers

**Contract** — takes a path and a destination discipline; finds the file, allocates
length-plus-one bytes under that discipline, reads the file in, appends a zero byte,
and returns the buffer. Returns nothing if the file was not found. Fatal error on an
unknown discipline or on an allocation that returns nothing. Shows the disc-access
indicator around the read.

The five disciplines and their wrappers:

| Discipline | Wrapper | Lifetime |
|---|---|---|
| hunk, low, tagged with the file's base name | `COM_LoadHunkFile` | until the level changes |
| hunk, temporary | `COM_LoadTempFile` | until the next temporary load |
| zone | — | until explicitly freed |
| cache, into a caller's handle | `COM_LoadCacheFile` | until evicted |
| a caller's stack buffer, falling back to a temporary hunk allocation when the file does not fit | `COM_LoadStackFile` | the caller's frame, or the next temporary load |

```text
FUNCTION com_load_file(path, discipline) -> optional<bytes>
  len = com_open_file(path)
  IF not found  RETURN nothing
  tag = com_file_base(path)                   # 8 characters, for the memory dump
  buf = allocate len+1 bytes under `discipline`
  IF buf IS nothing  FAIL WITH "not enough space for <path>"
  buf[len] = 0                                # always zero-terminated
  show the disc indicator
  read len bytes ;  close
  hide the disc indicator
  RETURN buf
```

**Invariants** — **every loaded file is zero-terminated**, one byte past its
length. Every text parser in the engine relies on it and none of them carries a
length. A rebuild that hands out exact-length buffers must revisit
[`COM_Parse`](#com_parse) and the map entity reader.

**Notes** — the caller's buffer and its size travel in globals rather than as
arguments, which makes the stack-buffer variant non-reentrant. Incidental.

### `COM_WriteFile`

**Contract** — takes a game-relative path; writes into the *game directory*
specifically, never through the search chain, never into an archive. Prints and
returns quietly on failure rather than failing.

### `COM_CopyFile`, `COM_CreatePath`

**Contract** — `COM_CreatePath` creates every directory named by an intermediate
separator in a path, ignoring errors. `COM_CopyFile` copies a file in 4096-byte
chunks, creating the destination's directories first. Both serve only the
development cache mirror.

### `COM_Path_f`

**Contract** — the `path` console command; prints each search chain element, giving
the entry count for archives.

---

## `va`

**Contract** — formats into a shared 1024-byte static buffer and returns it, so
that a formatted string can be passed to any function taking a string. No length
check; the source's own comment flags it.

**Notes** — one buffer means one live result. Two `va` calls in one argument list is
a bug, and the engine has a handful of near misses. A rebuild returns a value.

## `memsearch`

**Contract** — finds the first occurrence of a byte in a buffer, or −1. Unused;
a debugging leftover.
