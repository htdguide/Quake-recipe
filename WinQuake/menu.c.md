# WinQuake/menu.c

> Twenty-odd menu pages, each a draw routine and a key routine, dispatched through one pair of function pointers — including the network configuration pages that are the only interface to the serial and modem transports.

**Needs** — [`menu.h`](menu.h.md) · [`client.h`](client.h.md) · [`draw.h`](draw.h.md) · [`screen.h`](screen.h.md) · [`keys.h`](keys.h.md) · [`net.h`](net.h.md) · [`sound.h`](sound.h.md) · [`cdaudio.h`](cdaudio.h.md) · [`vid.h`](vid.h.md) · [`cmd.h`](cmd.h.md) · [`cvar.h`](cvar.h.md) · [`common.h`](common.h.md) · [`quakedef.h`](quakedef.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`keys.c`](keys.c.md) routes keys here; [`screen.c`](screen.c.md) draws; [`host.c`](host.c.md) initializes
**Tier floor** — none

## Purpose

The largest file in the client and almost all of it is layout. Three things are worth a rebuilder's attention:
the **dispatch shape**, the **save-game enumeration**, and the **network pages**, which are the only way to reach
the serial and modem transports' settings.

## State

```text
VARIABLE m_state : which page is open
VARIABLE m_entersound, m_recursiveDraw : bool
VARIABLE m_return_onerror : bool ;  m_return_reason : text
VARIABLE one cursor position per page
VARIABLE m_activenet : int                      # menu.h: the transports present
VARIABLE translationTable : byte[256]           # for the player preview
CONSTANT max_savegames = 12
VARIABLE m_filenames : text[12][16] ;  loadable : bool[12]
```

**Invariants** — the dispatch is **one state value plus two switch statements** — one in the draw function and one
in the key function — rather than a table of page records. So adding a page means editing three places, which is
why the file is as long as it is. A rebuild should use a table of (draw, key) pairs.

The error-return flag and reason let a page that starts a connection be re-entered with a message when the
connection fails, which is how a failed join shows an explanation.

## The helpers

**Contract** — `M_DrawCharacter`, `M_Print`, `M_PrintWhite`, `M_DrawPic`, `M_DrawTransPic` draw at menu
coordinates centred on the display; `M_DrawTextBox` draws a resizable framed box from nine corner and edge
graphics; `M_BuildTranslationTable` and `M_DrawTransPicTranslate` build and apply a player-colour remap for the
setup page's preview.

**Invariants** — the text box is assembled from edge and corner pieces, which is what lets it resize. The
translation table is a 256-entry map rather than the renderer's full 64-level one
([`cl_parse.c`](cl_parse.c.md#cl_newtranslation)), because the preview is drawn as a flat image.

## `M_ToggleMenu_f`, `M_Draw`, `M_Keydown`

**Contract** — open the main menu or close whatever is open; draw the current page, fading the world behind it
first; and hand a key to the current page.

```text
FUNCTION m_draw()
  IF no menu is open  RETURN
  IF NOT m_recursiveDraw
    force a full screen update
    IF the console is partly down  draw the console ;  clear the notifications
    ELSE                          draw_fade_screen()
  SELECT m_state
    ...one draw call per page
  IF m_entersound  play the menu-enter sound ;  clear the flag
```

**Invariants** — **the world behind a menu is faded**, which in the software renderer is a checkerboard
([`draw.c`](draw.c.md#draw_fadescreen)). The recursion guard exists because one page (the quit confirmation)
draws another page beneath itself.

## The pages

| Page | Content |
|---|---|
| Main | five entries: single player, multiplayer, options, help, quit |
| Single player | new game, load, save |
| Load / Save | twelve slots, enumerated from disk |
| Multiplayer | join, start, setup |
| Setup | name, colours, with a live player-model preview |
| Net | four transports: serial, direct, IPX, network |
| Serial config | port, baud rate, modem settings — **the only interface to them** |
| LAN config | the address and port to join or host on |
| Game options | skill, limits, level selection, team play |
| Search / Server list | the discovery results |
| Options | the tunables, plus the video backend's own page |
| Keys | the binding editor |
| Video | delegated to the video backend's own draw and key handlers |
| Help | the shipped help screens |
| Quit | a confirmation with a randomly chosen message |

**Invariants** — three things.

**The serial and modem settings are reachable only from here**, through the four handler variables the transport
driver fills in ([`net.h`](net.h.md#serial-and-modem-configuration)). That is the inversion
[`menu.h`](menu.h.md) describes, and a rebuild without a serial transport deletes the page and the handlers
together.

**The video page is delegated entirely to the backend**
([`vid.h`](vid.h.md#the-backends-menu-page)), because resolution selection is irreducibly platform-specific.

**Which network pages are offered depends on the availability bits** the transports set, so a build with no IPX
shows three entries rather than four.

## `M_ScanSaves`

**Contract** — probes twelve fixed save filenames, reading each one's version line and comment, and records which
exist.

**Invariants** — **twelve fixed names rather than a directory listing**, because the platform seam has no
enumeration ([`sys.h`](sys.h.md)). The comment is read from the file's second line and displayed with its
underscores turned back into spaces
([`host_cmd.c`](host_cmd.c.md#host_savegamecomment)).

## The key handlers

**Contract** — each page's key handler moves its cursor, wrapping, plays the cursor sound, and acts on enter or
escape. Most enter actions queue a console command rather than calling directly.

**Invariants** — **menu actions are console commands**, queued through the command buffer. So every menu action is
scriptable and the menu is a front end to the same interface the console offers. That is a genuinely good
decision and it is why the load page can simply queue `load s0`.

**Notes** — the file's length is almost entirely the twenty pages' layout constants. A rebuild reproduces them
from screenshots; nothing in them is derivable.
