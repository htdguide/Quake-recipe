# QW/client/menu.c

> The menus, reduced to what a client that only joins servers needs: options, controls, video and help, with the single-player and network-configuration pages replaced by a line of text.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`draw.h`](draw.h.md) · [`keys.h`](keys.h.md) · [`client.h`](client.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — [`keys.c`](keys.c.md) routes input to it; [`screen.c`](screen.c.md) draws it
**Tier floor** — none

## Purpose

Read [`menu.c`](../../WinQuake/menu.c.md) for the substance: a menu is a draw function and a key function, the stack of pages, the
slider and binding editors, and the rule that every page is drawn in the pixel-exact two-dimensional space.

This file is a third smaller and the deletions are the interesting content.

## State

As [`menu.c`](../../WinQuake/menu.c.md); the records are unchanged except where **What differs** says otherwise.

## What was removed, and why

**The whole single-player branch**: new game, difficulty, save and load. There are no saved games in a client that has no server.

**The whole network-configuration branch**: choosing a transport, the serial and modem configuration pages, the address entry, the
game-options page and the server browser. Every one of them configured something that no longer exists — there is one transport
([`net_udp.c`](net_udp.c.md)), no serial link, and starting a server is a separate program.

**Invariants** — what remains is **options, key bindings, video and help**, plus two pages that exist only to say the feature is
gone. The measurement is worth keeping: **roughly half of the original's menu system existed to configure networking that the
QuakeWorld design deleted.** A rebuild starting from QuakeWorld's architecture never writes that half.

## What differs in what remains

**The options page gained the settings the networking introduced**: the bandwidth rate, the display lag, and whether to extrapolate
other players. Those are the three settings a player actually has to tune for their connection
([`net_chan.c`](net_chan.c.md), [`cl_pred.c`](cl_pred.c.md), [`cl_ents.c`](cl_ents.c.md)), and putting them in the menu rather than
leaving them to the console is a decision about who is expected to tune them.

**The player-setup page edits the information dictionary** ([`common.c`](common.c.md)) rather than local variables, so a change is
sent to the server immediately ([`cl_main.c`](cl_main.c.md)). The skin name is a text field, and the model preview recolours live from
the kept pixel copy ([`gl_draw.c`](gl_draw.c.md)).

**Invariants** — **the menu is drawn every frame with no state beyond which page is open and where the cursor is.** That is the
original's design and it survives: no widgets, no layout, no event objects — a draw function and a key function per page. For an
interface of this size it is the right amount of machinery, and it is worth resisting the urge to build more.

**Notes** — the help pages are images, not text, which is why the interface archive is a dependency.
