# QW/docs/readme.qwsv

> Data: the dedicated server's release note — how to run it and where its settings live.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

Thirteen lines telling an operator how to start a server.

## State

Data; no run-time state.

## What it records

- The program is started with a **configuration file executed at startup**, which is where the port, the player limit, the passwords,
  the address filter and the directory addresses are set
  ([`sv_main.c`](../server/sv_main.c.md), [`sv_ccmds.c`](../server/sv_ccmds.c.md)).
- The content directory and the game-logic file it expects.
- That the server needs no display, sound or input — which is the whole point
  ([`makefile`](../server/makefile.md)).

**Notes** — the operational fact worth carrying: **a server's entire configuration is a file of console commands.** There is no
configuration format, no parser, no schema — the command interpreter is the configuration language
([`cmd.c`](../client/cmd.c.md)). That is a real simplification and it is why the settings and the dictionaries are the same mechanism.
