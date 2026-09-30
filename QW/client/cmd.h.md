# QW/client/cmd.h

> The command interpreter's interface, with the issuer's identity added.

**Needs** — nothing
**Used by** — every subsystem in both programs
**Tier floor** — none

## Purpose

Read [`cmd.h`](../../WinQuake/cmd.h.md) for the interface.

## State

As [`cmd.h`](../../WinQuake/cmd.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**A registered command declares where it may be issued from**, and the interpreter records where the current command came from
([`cmd.c`](cmd.c.md)). A command handler can therefore refuse a remote issuer, and the dispatcher can refuse before calling. That is
the whole delta and it is the security-relevant part.

**Front insertion into the deferred buffer** is declared.
