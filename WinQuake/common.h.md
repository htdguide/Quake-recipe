# WinQuake/common.h

> Declares the four vocabularies every other file assumes: the growable message buffer, the intrusive doubly linked list, the byte-order accessors, and the wire read/write primitives.

**Needs** — nothing; this is a leaf
**Used by** — [`quakedef.h`](quakedef.h.md) includes it before everything else, so: every file in the engine
**Tier floor** — T1 as written (the list uses address arithmetic to recover the containing record); the contracts themselves are tier-free

## Purpose

This is the engine's standard library declaration. Four unrelated things share it
because all four are needed before anything else can be declared: a byte buffer
with a length and an overflow policy, a list you can splice an object into without
allocating, the accessors that make file and wire data byte-order-independent, and
the typed readers and writers that sit on top of the buffer.

## State

Stateless, except for two globals owned by the reader:

```text
VARIABLE msg_readcount : int    # bytes consumed from the message being read
VARIABLE msg_badread   : bool   # a read ran past the end
```

The reader is a **singleton over one global buffer**, not a cursor over an argument.
There is exactly one message being decoded at any moment, and every read function
consumes from it. This is a real design decision, not an accident: it is why the
client can hand a half-decoded message to a subroutine without threading a cursor
through, and it is why a rebuild cannot decode two messages concurrently without
restructuring. A rebuild should make the cursor explicit; nothing in the logic
depends on it being global.

## Types

```text
TYPE byte  = int (8-bit, unsigned)
TYPE bool  = enum { false = 0, true = 1 }

RECORD SizeBuf                  # a byte buffer with a policy
  allow_overflow : bool         # false: overflowing is fatal
  overflowed     : bool         # set when an overflow was tolerated
  data           : bytes
  maxsize        : int
  cursize        : int
  # invariant: 0 <= cursize <= maxsize

RECORD Link                     # an intrusive list node
  prev, next : pointer to Link
```

### Numeric limits

The file declares named limits for the signed 8-, 16- and 32-bit ranges. Two of
them — the ones naming the float extremes — hold the *integer* maximum rather than
a float value, which is a bug in the original. Nothing reads them; a rebuild should
omit all of them and use its language's limits.

## The intrusive list

```text
FUNCTION clear_link(l)                 # make l an empty circular list header
  l.prev = l ;  l.next = l

FUNCTION remove_link(l)                # unlink l from whatever list holds it
  l.next.prev = l.prev
  l.prev.next = l.next

FUNCTION insert_link_before(l, before)
FUNCTION insert_link_after(l, after)   # the mirror
```

**Notes** — the list is intrusive and circular, with a header node that is itself
a `Link`. The containing record is recovered from a node by subtracting the field's
offset within the record — the source spells this with a macro and its own comment
calls it a mess. What survives is the *requirement*: the collision system
([`world.c`](world.c.md)) needs to splice an entity into and out of several
per-area lists per frame with no allocation and no separate node object, and it
needs to get from a node back to its entity. A rebuild solves that with whatever
its language offers — a list of indices into the entity array is the obvious choice
here, since entities already live in one contiguous array and are already
identified by index.

## The byte-order accessors

```text
VARIABLE big_endian : bool
# Six accessors, each resolved once at startup:
#   big_short, little_short, big_long, little_long, big_float, little_float
```

**Contract** — each takes a value in the named order and returns it in host order,
and is its own inverse. Which of each pair is the identity and which swaps is
decided once at startup by inspecting the host's byte order directly, not by a
compile-time switch.

**Notes** — the source implements this as six function pointers assigned at
startup, which costs an indirect call on every field of every loaded file. That
indirection is incidental; what is load-bearing is that the *decision* is made at
run time from an actual probe rather than from a build flag, because the same
source builds for several platforms. A rebuild in a language with byte-order-aware
readers should use them and delete this entirely.

## The message writers

**Contract** — each takes a buffer and a value, and appends the value in
little-endian order, growing `cursize`. All of them go through
[`SZ_GetSpace`](common.c.md#sz_getspace), so all of them inherit its overflow
policy.

| Writer | Wire form |
|---|---|
| char | one byte, signed |
| byte | one byte, unsigned |
| short | two bytes, little-endian |
| long | four bytes, little-endian |
| float | four bytes: the IEEE 754 single bit pattern, little-endian |
| string | the bytes, then one zero byte; a null string writes just the zero |
| coord | the value multiplied by 8 and truncated, as a short |
| angle | the value scaled so 256 spans a full turn, masked to a byte |

**Invariants** — the coordinate and angle forms are the protocol's quantization and
are not adjustable: 1/8 of a world unit per step, giving a world bounded to roughly
±4096 units, and 360/256 degrees per step. See
[`common.c`](common.c.md#msg_writeangle) for a precedence bug in the angle writer
that a rebuild must reproduce bit-for-bit to stay wire-compatible.

## The message readers

**Contract** — each consumes from the single global message buffer at
`msg_readcount` and advances it. A read that would pass the end sets `msg_badread`
and returns −1 *without* advancing; once set, the flag stays set until
`MSG_BeginReading`, so a caller may run a whole decode and check once at the end
rather than after every field.

| Reader | Returns |
|---|---|
| char | one byte sign-extended, or −1 at end |
| byte | one byte unsigned, or −1 at end |
| short | two bytes little-endian, sign-extended from 16 bits, or −1 at end |
| long | four bytes little-endian, or −1 at end |
| float | four bytes as an IEEE 754 single — **does not bounds-check** |
| string | bytes up to a zero, end of message, or 2047 characters, into a shared static buffer |
| coord | a short divided by 8 |
| angle | a signed char scaled by 360/256 |

**Invariants** — −1 is ambiguous: it is both a legitimate value and the
end-of-message signal, which is exactly why the flag exists and why every decoder
in the engine checks it rather than testing return values.

**Notes** — the string reader returns a pointer to a buffer it reuses on the next
call. Two live strings from one message is a bug the original avoids by
convention. A rebuild returns values.

## `MSG_BeginReading`

**Contract** — resets the read cursor to zero and clears the bad-read flag. Called
once per incoming message, before any read.

## The buffer operations

**Contract** — `SZ_Alloc` attaches hunk storage of at least 256 bytes. `SZ_Clear`
and `SZ_Free` both reset the length to zero and are the same operation; the buffer
is never actually released, because it lives on the hunk. `SZ_GetSpace` reserves
and returns a run of bytes. `SZ_Write` copies bytes in. `SZ_Print` appends a string
with a subtlety about the preceding terminator — see
[`common.c`](common.c.md#sz_write-sz_print).

## The string and number helpers

**Contract** — the file declares its own set of memory, string and
number-conversion operations rather than using the language's. Their behaviour
differs from the standard ones in ways that matter; the differences are documented
per-operation in [`common.c`](common.c.md). A rebuild should read that page before
substituting its own language's equivalents, because at least three of them are not
drop-in replacements.

## The token parser and path helpers

**Contract** — `COM_Parse` pulls one token from a string and leaves it in a shared
buffer; the path helpers split a filename into directory, base and extension.
Details in [`common.c`](common.c.md).

## The filesystem

**Contract** — `COM_OpenFile` and friends resolve a game-relative path through the
search path, which may find the file inside an archive; the four `COM_Load*File`
variants read a whole file into hunk, temporary, zone or cache memory. The global
`com_filesize` carries the length of the last file found, and `com_gamedir` names
the directory writes go to. Details in [`common.c`](common.c.md).

## Game-variant flags

```text
VARIABLE standard_quake : bool = true
VARIABLE rogue          : bool
VARIABLE hipnotic       : bool
```

**Notes** — three mutually exclusive flags selecting which mission pack's rules the
status bar and item logic follow. Set from the command line, never changed after.
A rebuild should make this one enumerated value.
