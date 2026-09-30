# WinQuake/menu.h

> The menu system's four-call interface, and the one place a network driver reports its availability upward.

**Needs** — nothing beyond the base types
**Used by** — [`host.c`](host.c.md), [`keys.c`](keys.c.md), [`screen.c`](screen.c.md); the network drivers set its availability flags; implemented by [`menu.c`](menu.c.md)
**Tier floor** — none

## Purpose

Four functions: initialize, handle a key, draw, and toggle. The menus are a state machine entirely
internal to their implementation, and this header deliberately exposes none of it.

The one substantive declaration is the network availability mask, and the header's comment explains it:
rather than having the menu inspect the network layer's internal tables, **each driver sets a bit** to
say it is present. The same upward-reporting inversion appears in
[`vid.h`](vid.h.md#the-backends-menu-page) and [`net.h`](net.h.md#serial-and-modem-configuration).

## State

```text
CONSTANT mnet_ipx = 1
CONSTANT mnet_tcp = 2
VARIABLE m_activenet : int        # a bit per available transport
```

**Invariants** — two bits for two transports, set by [`net_wipx.c`](net_wipx.c.md) /
[`net_ipx.c`](net_ipx.c.md) and [`net_wins.c`](net_wins.c.md) /
[`net_udp.c`](net_udp.c.md) at initialization. The serial transport is *not* represented here — it is
detected through the availability flags in [`net.h`](net.h.md) instead — so there are two mechanisms for
the same question, which a rebuild should unify.

## Entry points

**Contract** — `M_Init` registers the menu commands and loads its images. `M_Keydown` handles one key
while a menu is open. `M_Draw` draws the current menu. `M_ToggleMenu_f` opens the main menu or closes
whatever is open.

**Invariants** — the menus own the keyboard while open, through the input destination in
[`keys.h`](keys.h.md), and the key handler is called instead of the game's. That is the whole mechanism
by which a menu captures input.
