# WinQuake/crc.h

> Declares the sixteen-bit checksum the engine uses to detect that content has been altered.

**Needs** — nothing; this is a leaf
**Used by** — [`crc.c`](crc.c.md) · [`common.c`](common.c.md) (archive directory checksum) · [`pr_edict.c`](pr_edict.c.md) (game logic field-layout checksum) · [`sv_main.c`](sv_main.c.md) (serverinfo checksum sent to clients)
**Tier floor** — none

## Purpose

Three places in the engine need to know "is this exactly the data I expect, or has
somebody changed it", and none of them needs cryptographic strength. This declares
the one checksum they share.

## State

Stateless. The running value is owned by the caller, which is why the interface is
three operations over a caller-held accumulator rather than a one-shot over a
buffer: [`common.c`](common.c.md) checksums a buffer it already holds, while
[`pr_edict.c`](pr_edict.c.md) checksums a string it builds as it goes.

## `CRC_Init`

**Contract** — takes a caller-owned 16-bit accumulator and sets it to the initial
value. Must be called before the first byte.

## `CRC_ProcessByte`

**Contract** — takes the accumulator and one byte; folds the byte into the running
value. Order matters; the same bytes in a different order give a different result.

## `CRC_Value`

**Contract** — takes the accumulator by value and returns the finished checksum.

**Notes** — the finishing step is a fixed exclusive-or that happens to be with
zero in this engine, so the finished value equals the accumulator. Two of the
three callers skip this call and read the accumulator directly, which is why the
identity matters: a rebuild that changes the final constant breaks those callers
silently. Keep the value at zero or fix all three sites.
