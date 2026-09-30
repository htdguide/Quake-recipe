# WinQuake/crc.c

> A table-driven CCITT sixteen-bit cyclic redundancy check, used to notice altered game content.

**Needs** — [`crc.h`](crc.h.md) · [`quakedef.h`](quakedef.h.md)
**Used by** — [`common.c`](common.c.md) · [`pr_edict.c`](pr_edict.c.md) · [`sv_main.c`](sv_main.c.md)
**Tier floor** — none

## Purpose

The engine needs to answer "has this been modified" about three things: the
contents of the first asset archive, the field layout of the compiled game logic,
and the game logic a server is running as reported to its clients. This is that
check. It is a standard algorithm and the twin exists mainly to pin down *which*
standard, because the exact parameters are part of the wire and file
compatibility.

## State

```text
CONSTANT polynomial  = 0x1021        # CCITT / X.25 / XMODEM
CONSTANT init_value  : int (16-bit) = 0xFFFF
CONSTANT final_xor   : int (16-bit) = 0x0000
CONSTANT table       : list<int (16-bit)>[256]   # derived from the polynomial
```

The parameterization is: 16-bit width, polynomial `0x1021`, **non-reflected**
(bits shift toward the high end), initial value all ones, final exclusive-or zero,
input bytes not reflected. That is the variant XMODEM uses. A rebuild that reaches
for a library's "CRC-16" will most often get the reflected variant and produce
different values, so check the parameters rather than the name.

## `CRC_Init`

**Contract** — sets the caller's accumulator to all ones.

## `CRC_ProcessByte`

**Contract** — takes the accumulator and a byte; replaces the accumulator with the
next state. Pure; no other effect.

```text
FUNCTION crc_process_byte(crc, data)
  index = (crc SHIFTED RIGHT 8) BITXOR data      # the outgoing high byte
  crc = ((crc SHIFTED LEFT 8) BITXOR table[index]) TRUNCATED TO 16 bits
```

**Invariants** — the shift is left, and the index comes from the *high* byte: that
is what "non-reflected" means concretely. The truncation to sixteen bits is
implicit in the source's use of a 16-bit type and must be explicit in a rebuild
whose integers are wider.

## `CRC_Value`

**Contract** — returns the accumulator combined with the final exclusive-or
constant, which is zero, so it returns the accumulator unchanged.

## The table

**Contract** — 256 entries, where entry *i* is the result of shifting the byte *i*
into an empty register eight times under the polynomial.

```text
FUNCTION build_table() -> list<int (16-bit)>[256]
  FOR EACH i IN 0..255
    r = i SHIFTED LEFT 8
    REPEAT 8 TIMES
      IF the high bit OF r IS SET
        r = ((r SHIFTED LEFT 1) BITXOR 0x1021) TRUNCATED TO 16 bits
      ELSE
        r = (r SHIFTED LEFT 1) TRUNCATED TO 16 bits
    table[i] = r
```

**Notes** — the source stores the table as 256 literals rather than computing it,
which in 1996 saved a startup loop. A rebuild should compute it, and can check its
work against the first few published entries: index 0 is `0x0000`, index 1 is
`0x1021`, index 2 is `0x2042`, index 255 is `0x1EF0`.

## What depends on the exact values

- The archive-directory checksum in [`common.c`](common.c.md) is compared against
  a hard-coded constant (32981) to decide whether the shipped content has been
  modified, which in turn gates a shareware-versus-registered check.
- The game-logic field checksum in [`pr_edict.c`](pr_edict.c.md) is compared
  against a constant compiled into the engine, and a mismatch refuses to load the
  game at all.
- The same field checksum is sent to clients in the server information message, so
  a client and server disagreeing about it disagree about the wire format.

A rebuild that computes different checksums than the original cannot load original
content. This is one of the few places in the engine where an arbitrary-looking
constant is genuinely load-bearing.
