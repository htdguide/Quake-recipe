# QW/client/common.h

> The shared foundation's interface: the buffers, the byte encoders including the command form, the tokenizer, the file system, and the dictionary operations.

**Needs** — nothing
**Used by** — every file in both programs
**Tier floor** — none

## Purpose

Read [`common.h`](../../WinQuake/common.h.md) for the interface.

## State

As [`common.h`](../../WinQuake/common.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The dictionary operations, the command encoder and decoder, the sequence-seeded checksum, the existence test and the path creation
are declared here — see [`common.c`](common.c.md) for all five.

**Invariants** — the header is included by both programs, so **everything declared here is a shared contract**. That is the same
discipline as [`bothdefs.h`](bothdefs.h.md), applied to functions rather than constants.
