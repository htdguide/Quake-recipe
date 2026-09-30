# QW/client/common.c

> The shared foundation: the byte encoders including the command and angle forms the protocol needs, the information dictionaries, the file system with its archive search path, and the sequence-seeded checksum.

**Needs** — [`quakedef.h`](quakedef.h.md) or [`qwsvdef.h`](../server/qwsvdef.h.md) · [`crc.h`](crc.h.md) · [`md4.c`](md4.c.md) · [Seam: Operating system services](../../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — every file in both programs
**Tier floor** — none

## Purpose

Read [`common.c`](../../WinQuake/common.c.md) in full: the byte-order handling, the coordinate and angle encodings with the angle
encoder's precedence quirk, the growable buffers, the tokenizer, the command-line parameter set, the archive format and the search
path with its override order. All present and unchanged in substance.

The additions are three, and each supports something specific in the networking.

## State

As [`common.c`](../../WinQuake/common.c.md); the records are unchanged except where **What differs** says otherwise.

## The information dictionaries

**Contract** — a dictionary is a single string of alternating separated keys and values. Operations read a key's value, set a key,
remove a key, remove every key marked as server-set, print a dictionary, and validate one.

```text
representation:  \key1\value1\key2\value2
FUNCTION value_for_key(info, key) -> text        # returns "" when absent
FUNCTION set_value_for_key(info, key, value, maxsize)
FUNCTION set_value_for_star_key(info, key, value, maxsize)
FUNCTION remove_key(info, key)
FUNCTION remove_prefixed_keys(info, prefix)
```

**Invariants** —

- **A key beginning with a reserved character may only be set through the distinguished operation**, and the ordinary setter refuses
  it. That is the mechanism by which a server marks a fact as its own assertion rather than the client's
  ([`sv_main.c`](../server/sv_main.c.md) marks the spectator flag this way), and it is what stops a client claiming a privilege by
  setting a key. **Two namespaces in one string, separated by a character convention** — crude, and effective, and a rebuild needs
  the separation however it spells it.
- **The separator character is forbidden in keys and values**, and a value containing it is rejected — otherwise a client could
  inject arbitrary keys through a value. That check is security-relevant.
- The dictionary has a **maximum length** and a set that would exceed it fails rather than truncating, because a truncated dictionary
  is a corrupted one.
- Keys are **unordered and absent means default**, so a dictionary is extensible by anyone and old software ignores what it does not
  know. That is the whole reason player and server configuration is a dictionary rather than a record
  ([`sv_ccmds.c`](../server/sv_ccmds.c.md)).

## The command encoding

**Contract** — write a command as a difference from another: a flag byte saying which fields differ, then only those fields. Read the
same.

```text
FUNCTION write_delta_usercmd(buf, from, to)
  bits = which of (3 angles, forward, side, up, buttons, impulse) differ
  write bits
  write each differing field: angles as 16-bit, movements as 16-bit,
    buttons and impulse as bytes
  write the duration as a byte, ALWAYS
```

**Invariants** —

- **The duration is always sent and everything else is conditional**, because the duration always differs and everything else usually
  does not. Three commands per packet ([`cl_input.c`](cl_input.c.md)) therefore cost little more than one.
- **Angles are 16-bit here**, finer than the byte form the protocol uses elsewhere, because aim precision is the game. The client
  quantizes to exactly this precision before predicting
  ([`cl_input.c`](cl_input.c.md)) — the two must be the same rounding.
- The encoding is symmetric and both programs use it, which is why it lives here.

## `COM_BlockSequenceCRCByte`

**Contract** — computes a one-byte checksum over a block, seeded by a sequence number, using a fixed table of seeds.

**Invariants** — **the checksum depends on the packet's sequence number**, so the same movement block in a different packet checksums
differently and a replayed block is detectable ([`sv_user.c`](../server/sv_user.c.md)). The recipe repeats the honest assessment: the
algorithm and its table are in the client, so this deters corruption and casual tampering and stops nothing determined.

## What else differs

**The file system gained the ability to rebuild its search path at run time**, because the server can tell a client to change content
directory mid-game ([`sv_ccmds.c`](../server/sv_ccmds.c.md)). The original builds the path once at startup. That means **every
assumption that the path is fixed had to go**, which is a small change with a wide blast radius, and a rebuild should decide early
whether its content path is mutable.

**A file's existence can be tested without opening it**, which the download check needs
([`cl_parse.c`](cl_parse.c.md)).

**Path creation is available**, because downloads land in directories that may not exist.

**Notes** — the dictionary is the most reusable thing here. Its four properties — unordered, absent-means-default, a reserved
namespace for the authority's own assertions, and a hard length limit — are what let a 1996 protocol carry configuration nobody had
designed yet, and they are why modifications could add fields without an engine release.
