# QW/client/md4.c

> A message digest, used to summarize a block of bytes into sixteen.

**Needs** — nothing
**Used by** — [`sv_main.c`](../server/sv_main.c.md) · [`common.c`](common.c.md)
**Tier floor** — none

## Purpose

A standard cryptographic digest function, carried in the tree because no platform of the era provided one. It is used for two
things: summarizing a file's contents to compare against another party's claim, and as the basis of the sequence-dependent
checksum over a packet's movement block.

## State

```text
RECORD Context
  state  : four 32-bit words, initialized to fixed constants
  count  : the message length in bits, as two words
  buffer : 64 bytes of partial block
```

## The operations

**Contract** — initialize a context; absorb bytes, processing each complete 64-byte block; and finish, appending the padding and
the length and producing sixteen bytes. A convenience operation digests one buffer in a single call.

```text
FUNCTION transform(state, block)
  load the block as 16 little-endian 32-bit words
  three rounds of 16 operations each, each operation:
    a = rotate_left(a + f(b,c,d) + word[k] + round_constant, s)
  where f is a different bitwise function per round, and the word order,
    rotation amounts and constants are fixed by the specification
  add the round results back into state
```

**Invariants** —

- **The words are little-endian regardless of the machine**, as the specification requires, so two machines of different byte
  order produce the same digest. That is the only portability requirement in the file and it is the one a rebuild will get wrong
  first.
- **The length is appended in bits, not bytes**, and as two words — also specified.
- The four initial constants, the per-round functions, the word orders, the rotation amounts and the additive constants are all
  **fixed by the specification and none may be altered**. This is a case where the recipe's usual advice inverts: there is nothing
  to decide, only to reproduce.

**Notes** — ***do not use this function in a rebuild.*** It has been broken for decades: collisions are cheap to construct, which
means the file-content comparison it supports ([`sv_init.c`](../server/sv_init.c.md)) can be defeated deliberately, and any
authentication built on it would be unsound. The correct action for a rebuild is to use the platform's modern digest.

What is worth carrying is *why* a digest is here at all: to compare content without transferring it, and to make a packet's
checksum depend on its sequence number. Both needs are real; the function is not the answer any more.
