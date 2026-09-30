# QW/qwfwd/misc.c

> Copies of the shared helpers the forwarder needs: the command-line search, the tokenizer, and the information dictionary operations.

**Needs** — nothing
**Used by** — [`qwfwd.c`](qwfwd.c.md)
**Tier floor** — none

## Purpose

A duplicate of parts of [`common.c`](../client/common.c.md) — read that twin for the tokenizer and for the dictionary operations,
including the reserved-namespace rule and the separator restriction.

## State

As described under the sections below; the forwarder holds only its two sockets and the client's address.

## What differs

Nothing of substance. The functions are copied rather than shared, because the forwarder does not link the engine.

**Notes** — recorded because the duplication is real and a reader will wonder. **A rebuild should factor the dictionary and the
tokenizer into a library both programs link**, which is the obvious fix and the reason this file is worth one page rather than none: it
marks the place where the source's file organization forced a copy.
