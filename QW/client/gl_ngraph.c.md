# QW/client/gl_ngraph.c

> A rolling bar chart of the last packets' fate, drawn from the channel's own history, so a player can see loss and choking as it happens.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`client.h`](client.h.md) · [`net_chan.c`](net_chan.c.md)
**Used by** — [`gl_screen.c`](gl_screen.c.md) draws it when enabled
**Tier floor** — none

## Purpose

The networking's diagnostic display, and the reason the client keeps a per-packet history at all. One column per recent packet,
its height the packet's size and its colour its fate — received, lost, or choked by the local budget.

## State

```text
VARIABLE a generated texture, one row per graph line, rebuilt as the graph scrolls
CONSTANT the graph's height and the colour per outcome
```

## `R_NetGraph`, `R_LineGraph`, `Draw_CharToNetGraph`

**Contract** — for each of the last several packets, draw a vertical bar whose height is that packet's size and whose colour
encodes whether it arrived, was dropped, or was suppressed by the rate budget; label the axis.

```text
FUNCTION net_graph()
  FOR EACH recent packet in the client's frame ring
    IF it was never acknowledged      colour = lost
    ELSE IF it was suppressed locally colour = choked
    ELSE                              colour = normal
    height = its size, scaled and clamped
    draw the bar
  draw the scale labels
```

**Invariants** —

- **Three outcomes, distinguished.** The distinction between *the network lost it* and *we chose not to send it* is the whole
  value of the display: the first means the link is bad, the second means the rate setting is too low. Collapsing them leaves a
  player unable to tell which. A rebuild with any network diagnostic should keep exactly this distinction.
- The data comes from the client's frame ring and the channel's counters
  ([`client.h`](client.h.md), [`net_chan.c`](net_chan.c.md)) — no extra bookkeeping. The display is free because the information
  is already kept.
- It is drawn by **generating a texture a row at a time** rather than by drawing many small quads, because a bar per packet would
  be dozens of draws per frame.

**Notes** — the software renderer has its own version ([`r_part.c`](r_part.c.md) carries the equivalent for that build). Worth
keeping in a rebuild: a live picture of the link is the difference between "the game feels bad" and "your rate is set to 2500".
