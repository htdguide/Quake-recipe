# QW/docs/readme.qwcl

> Data: the client's release note — the command-line options, the console commands a player needs, and the settings that govern the connection.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The user-facing description of the client, and the fullest statement of **what a player must configure to play over a link**. That list
is the recipe-relevant content, because each item is a decision the networking pushed onto the player.

## State

Data; no run-time state.

## What it records

**The three connection settings and what they trade:**

- The **bandwidth rate**, which the channel's budget uses
  ([`net_chan.c`](../client/net_chan.c.md)) and the server clamps
  ([`sv_user.c`](../server/sv_user.c.md)). Too high and packets are lost in bursts; too low and entities go stale.
- The **display lag**, which decides how far behind the present the world is drawn
  ([`cl_pred.c`](../client/cl_pred.c.md)). More lag means smoother motion and later reactions.
- Whether to **extrapolate other players** ([`cl_ents.c`](../client/cl_ents.c.md)), which trades a plausible opponent position for an
  occasionally wrong one.

**The identity settings** — name, colours, skin, team — which live in the information dictionary and travel with the player
([`common.c`](../client/common.c.md)).

**The commands for joining, spectating, recording and the scoreboards.**

**That downloads happen automatically** and where the files land
([`cl_parse.c`](../client/cl_parse.c.md)).

**Notes** — the three settings above are the honest cost of the design: **a player on a variable link has to tune their own client**,
and no amount of engine work in 1996 removed that. A rebuild on a modern network should be able to derive all three automatically —
the rate from measurement, the lag from the observed jitter — and the fact that the original could not is a reason to try, not a reason
to copy the settings.
