# QW/client/cvar.c

> Named settings: a linked list of name-value pairs readable as text or as a number, settable from anywhere, with a flag marking those that must be published.

**Needs** — [`common.h`](common.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — every subsystem in both programs
**Tier floor** — none

## Purpose

Read [`cvar.c`](../../WinQuake/cvar.c.md) for the substance: a setting is a name, a string, a cached numeric form and flags; setting
it re-parses the number; a setting can be marked to be written to the configuration file.

## State

As [`cvar.c`](../../WinQuake/cvar.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**A setting can be marked as belonging to a dictionary**, and changing it writes through into the server's public dictionary or the
client's own ([`common.c`](common.c.md)). So a value has one definition and two representations — a setting for local use and a
dictionary entry for the wire — kept in step automatically.

**Invariants** — that write-through is what makes the movement tunables work
([`sv_phys.c`](../server/sv_phys.c.md)): the operator changes a setting, the dictionary changes, every client is told, and every
client's prediction updates. Without it the tunables would have to be published by hand at every change and one missed change is a
prediction that diverges.

**A setting can be created by name at run time**, rather than only registered at startup, so a modification's settings exist without
engine support.

**Notes** — the pattern is worth naming: **a value that must be both local and published should have one owner and an automatic
projection, not two copies.**
