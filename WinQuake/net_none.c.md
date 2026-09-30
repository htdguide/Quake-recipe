# WinQuake/net_none.c

> The empty driver table: builds an engine with no networking at all, leaving only the loopback path.

**Needs** — [`quakedef.h`](quakedef.h.md)
**Used by** — [`net_main.c`](net_main.c.md) reads both tables
**Tier floor** — none

## Purpose

Proof that the client-server split is load-bearing and networking is not. With both tables empty the engine still
plays single player, because the loopback path is a driver like any other and single player is a session against a
local server.

## State

```text
VARIABLE net_drivers    = [ ]        # empty
VARIABLE net_landrivers = [ ]        # empty
```

**Notes** — a rebuild that only wants single player can start here and stay here; that is the useful fact this file
records. Note that the loopback driver must still be present for that to work — an "empty" table in the original
still leaves single player broken unless the build also links the loopback rows, which is why the shipping tables all
list it first.
