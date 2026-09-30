# WinQuake/sbar.c

> The status bar and the score overlays: draws health, armour, ammunition, the item icons and the face, unpacking the composite item field and switching layout for each of the three game variants.

**Needs** — [`sbar.h`](sbar.h.md) · [`client.h`](client.h.md) · [`draw.h`](draw.h.md) · [`screen.h`](screen.h.md) · [`vid.h`](vid.h.md) · [`quakedef.h`](quakedef.h.md) (the statistic slots and item bits) · [`cmd.h`](cmd.h.md) · [`wad.h`](wad.h.md)
**Used by** — [`screen.c`](screen.c.md) draws it; [`cl_parse.c`](cl_parse.c.md) and [`view.c`](view.c.md) notify it of changes
**Tier floor** — none

## Purpose

Almost all of it is layout, and a rebuild reproduces the layout from screenshots. Three things are genuinely
load-bearing: the **item field's composite packing**, the **three game variants' different item numberings**, and
the **redraw discipline**.

## State

```text
CONSTANT stat_minus = 10               # the digit frame used for a minus sign
VARIABLE sb_updates : int              # the redraw counter
VARIABLE sb_showscores : bool
VARIABLE the loaded graphics: digit sets, the six faces, the item icons, the
         armour and ammunition icons, the two number colours
VARIABLE fragsort : int[16] ;  scoreboardtext, scoreboardtop, scoreboardbottom
```

**Invariants** — the redraw counter is compared against the display's page count
([`sbar.h`](sbar.h.md)), so a change draws the bar for as many frames as there are pages.

## `Sbar_Init`

**Contract** — loads every graphic the bar needs from the interface archive, selecting different sets for the
two mission packs, and registers the score commands.

**Invariants** — the graphic *names* differ per game variant, which is why the load is conditional rather than the
draw.

## `Sbar_Changed`

**Contract** — sets the redraw counter to the page count.

## `Sbar_DrawPic`, `Sbar_DrawTransPic`, `Sbar_DrawCharacter`, `Sbar_DrawString`

**Contract** — the drawing helpers, each offsetting its position so the bar's own coordinate space has its origin
at the bar's top-left rather than the display's.

**Invariants** — the offset centres the bar horizontally and places it at the bottom, which is what makes the
layout constants resolution-independent.

## `Sbar_itoa`, `Sbar_DrawNum`

**Contract** — render an integer right-aligned in a field of a given digit count, using the large digit graphics
in one of two colours, with a dedicated frame for the minus sign. Values are clamped to the field.

**Invariants** — the minus sign is **digit frame 10** of the digit graphic set, which is why the constant exists.
The two colours distinguish ordinary from low values.

## `Sbar_DrawInventory`

**Contract** — draws the weapon icons, the ammunition counts, the item icons and the sigils, reading the
composite item field. Selects a different icon set and layout for each of the three game variants.

```text
FUNCTION sbar_draw_inventory()
  # WEAPONS: one icon per weapon bit, with a flash frame for a recent pickup.
  FOR EACH weapon bit
    IF cl.items HAS it
      draw its icon, using the FLASHING frame when the pickup was recent
  # AMMUNITION: four counts from the numbered statistic slots.
  FOR EACH of the four ammunition kinds  draw its count
  # ITEMS: the keys, the powerups and the armour.
  FOR EACH item bit IN the item range  draw its icon if set
  # SIGILS: the top four bits of the composite field.
  FOR EACH sigil bit  draw its icon if set
```

**Invariants** — three things.

**The item field is a composite** the server packed
([`sv_main.c`](sv_main.c.md#sv_writeclientdatatomessage)): the low bits are the item set, and the high bits are
either a second item set or the episode-completion flags, depending on which game is loaded. The bar unpacks it
with the same shift the server used — 23 for the second set, 28 for the sigils. **Both ends must agree or the
sigils appear as items.**

**Each of the three variants has a different item numbering**
([`quakedef.h`](quakedef.h.md)), so the bit-to-icon mapping is selected by the startup flags
([`common.c`](common.c.md#com_initargv)). A rebuild should model the three as three named sets rather than one.

**A recently picked-up weapon flashes**, driven by the per-item acquisition times the client records
([`client.h`](client.h.md)).

## `Sbar_DrawFace`

**Contract** — draws the player's face, choosing among the pain frames by health, an invulnerability frame, a
quad-damage frame, and a temporary pain expression whose deadline the damage message set.

**Invariants** — the face is the health display's second channel: five pain levels plus three special states. The
temporary pain expression is triggered by taking damage ([`view.c`](view.c.md#v_parsedamage)) and lasts a fixed
time.

## `Sbar_Draw`

**Contract** — draws nothing when the redraw counter has expired. Otherwise draws the bar's background, the
inventory, the health, armour and ammunition figures, the face, and — in a multiplayer game — the frag counts or
the miniature scoreboard. Decrements the counter.

**Invariants** — **the decision not to draw belongs to the bar, not the caller**
([`sbar.h`](sbar.h.md)), which is why it is called unconditionally every frame.

## `Sbar_SortFrags`, `Sbar_ColorForMap`, `Sbar_UpdateScoreboard`, `Sbar_DrawScoreboard`, `Sbar_DrawFrags`

**Contract** — sort the players by score; map a colour index to a palette entry; rebuild the scoreboard's text and
colour bars; draw the scoreboard; draw the frag counts across the bar with each player's colours.

**Invariants** — each player's two colour bars are drawn from their packed colour byte
([`host_cmd.c`](host_cmd.c.md#host_color_f-color)), which is how a player is identified in a list without reading their
name.

## `Sbar_DeathmatchOverlay`, `Sbar_MiniDeathmatchOverlay`, `Sbar_SoloScoreboard`

**Contract** — the full scoreboard, the compact one drawn beside the bar in a multiplayer game, and the
single-player line showing the level name, the elapsed time and the secret and monster counts.

**Invariants** — the single-player line reads the four counter statistics
([`quakedef.h`](quakedef.h.md)), two of which the client also increments locally
([`cl_parse.c`](cl_parse.c.md#cl_parseservermessage)) so they animate before the next push.

## `Sbar_IntermissionNumber`, `Sbar_IntermissionOverlay`, `Sbar_FinaleOverlay`

**Contract** — render a number in the intermission's own digit style; draw the end-of-level statistics — time,
secrets, kills; and draw the end-of-episode text.

## `Sbar_ShowScores`, `Sbar_DontShowScores`

**Contract** — the plus-and-minus pair for holding the score key.

**Invariants** — bound as a held action ([`keys.h`](keys.h.md)), so releasing the key hides the scores — which is
why it needs both halves.
