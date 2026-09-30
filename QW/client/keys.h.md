# QW/client/keys.h

> The key numbering and the input routing interface.

**Needs** — nothing
**Used by** — every input backend, [`keys.c`](keys.c.md), [`console.c`](console.c.md), [`menu.c`](menu.c.md)
**Tier floor** — none

## Purpose

Read [`keys.h`](../../WinQuake/keys.h.md). **The key numbering is a contract with every platform backend** and with every saved
configuration file, so a number's meaning may not change. The addition is the chat destination and the completion declarations.
## State

As [`keys.h`](../../WinQuake/keys.h.md); the records are unchanged except where **What differs** says otherwise.

