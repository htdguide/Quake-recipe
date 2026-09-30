# QW/cmds.txt

> Data: the complete list of console commands and settings, for the client and the server separately.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The reference the two programs' interfaces are actually built from, enumerated in one place. For a rebuilder it is the **authoritative
surface of the command interpreter** ([`client/cmd.c`](client/cmd.c.md)) — everything a player or an operator can type, and therefore
everything a rebuild must accept if it wants existing configuration files and scripts to work.

## State

Data; no run-time state.

## What it records

Two lists, one per program, each split into commands and settings.

**The client's commands** — the movement controls (each a press-and-release pair, so a binding generates two commands
([`client/keys.c`](client/keys.c.md))), the weapon impulses, the connection commands, the recording commands, the spectator commands, the
scoreboard toggles, the information-dictionary editors, and the diagnostics.

**The client's settings** — the three that govern the connection (bandwidth rate, display lag, extrapolation
([`QW/docs/readme.qwcl`](docs/readme.qwcl.md))), the identity keys, the sensitivity and view preferences, and every renderer switch
([`client/glquake.h`](client/glquake.h.md)).

**The server's commands** — the level and content-directory changes, the player commands, the status report, the address filter, the
logging, the dictionaries, and the directory announcement
([`server/sv_ccmds.c`](server/sv_ccmds.c.md)).

**The server's settings** — the limits, the passwords, the timeouts, the rate clamps, the flood-protection parameters, and the movement
tunables that must reach every client ([`client/pmove.h`](client/pmove.h.md)).

**Invariants** — the list distinguishes **what may be issued from where**, which is the boundary the interpreter enforces with its
per-command source flag ([`client/cmd.h`](client/cmd.h.md)). A command reachable by a remote party is marked as such, and that marking is
security-relevant.

**Notes** — worth using as a checklist rather than reading straight through. Its most useful property is completeness: a rebuild can tick
items off against it, which no header allows.
