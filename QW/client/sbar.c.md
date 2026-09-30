# QW/client/sbar.c

> The status bar and the scoreboards: the player's own state along the bottom, and the full and team score overlays with their sorting.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`draw.h`](draw.h.md) · [`sbar.h`](sbar.h.md) · [`bothdefs.h`](bothdefs.h.md)
**Used by** — [`screen.c`](screen.c.md) and [`gl_screen.c`](gl_screen.c.md)
**Tier floor** — none

## Purpose

Read [`sbar.c`](../../WinQuake/sbar.c.md) for the substance: the bar's layout, the digit and face drawing, the item and armour
indicators, and the rule that the bar is drawn in the pixel-exact two-dimensional space
([`gl_draw.c`](gl_draw.c.md)).

The changes are all about there now being up to sixteen players whose state matters.

## State

As [`sbar.c`](../../WinQuake/sbar.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**A full scoreboard overlay with columns for score, latency, connected time and name**, sorted by score, replacing the original's
small deathmatch overlay. The latency and connected-time columns come from the channel's own statistics
([`net_chan.c`](net_chan.c.md), [`cl_parse.c`](cl_parse.c.md)) — the display exists because the data was already being kept.

**A team scoreboard**, with players grouped by their declared team and each team's total, sorted. Teams are read from the players'
information dictionaries ([`cl_parse.c`](cl_parse.c.md)), so **the engine knows nothing about what a team is** beyond the string
players agree on. That is the right layering and it is why a team game needs no engine change.

**Spectators are listed separately** and marked, since they have no score.

**The overlays can be held open by a key** as well as appearing at the end of a level.

**A sub-image draw operation** is added, because the scoreboard draws parts of the interface images rather than whole ones.

**Invariants** —

- **The sorts are stable and recomputed only when something changed**, because sorting sixteen players every frame is wasteful and a
  changing order makes the board unreadable.
- **The colour shown per player is derived from their declared colours** through the same mapping the skins use
  ([`gl_rmisc.c`](gl_rmisc.c.md)), so the board and the models agree.
- The bar reads the player's item set from the statistics
  ([`bothdefs.h`](bothdefs.h.md)) — game content in the engine, and the same layering compromise the original makes.

**Notes** — the useful observation is that **the scoreboard is built entirely from data the networking already needed**: scores from
the protocol, latency from the channel, names and teams from the dictionaries. A rebuild that keeps those three gets the scoreboard
almost free, and one that does not will find itself adding a protocol message per column.
