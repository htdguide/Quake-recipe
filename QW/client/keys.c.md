# QW/client/keys.c

> Key handling: a name per key, a binding per key, the routing of a key to the game, the console, the chat line or the menu, and command completion.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`keys.h`](keys.h.md) · [`cmd.h`](cmd.h.md) · [`console.h`](console.h.md) · [`menu.h`](menu.h.md)
**Used by** — every platform input backend delivers keys here
**Tier floor** — none

## Purpose

Read [`keys.c`](../../WinQuake/keys.c.md) for the substance: the key name table, the binding table, the destination routing, the
line editor with its history, and the rule that a binding prefixed to indicate a press-and-release control generates two commands.

## State

As [`keys.c`](../../WinQuake/keys.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**Command completion.** Pressing the completion key matches the partial word against the command table and the settings table and
fills in the unique match, or lists the candidates.

**Invariants** — completion reads the same two tables the interpreter dispatches from
([`cmd.c`](cmd.c.md), [`cvar.c`](cvar.c.md)), so it can never offer something that does not exist and never omits something that
does. **Deriving completion from the dispatch tables rather than a separate list is the only maintainable way**, and it is the whole
content of the addition.

**A check for whether a typed word is a command at all**, used to decide whether an unrecognized console line should be sent to the
server as a text command instead of reported as an error. That is how a modification's own commands work from the console with no
local registration — an elegant fallthrough, and the reason the server's command table has a closed list with a game-logic
fallthrough ([`sv_user.c`](../server/sv_user.c.md)).

**Notes** — the routing of a key to one of four destinations is unchanged and remains the cleanest part of the design: one variable
says where keys go, and every consumer is passive.
