# WinQuake/console.h

> The console: a scrolling text buffer that is also the engine's only output channel, and the notification lines that show recent messages during play.

**Needs** — [`vid.h`](vid.h.md)
**Used by** — every module, through the print functions; [`screen.c`](screen.c.md) draws it; implemented by [`console.c`](console.c.md)
**Tier floor** — none

## Purpose

Every diagnostic, every server message and every error in this engine goes through the print functions
declared here. The console is therefore not a debugging feature — it is the engine's standard output,
and it is the reason nearly every file in the tree depends on this header.

The second thing it provides is **notification lines**: the most recent few console lines are drawn over
the game for a few seconds, which is how chat and pickup messages reach a player who does not have the
console open.

## State

```text
VARIABLE con_totallines   : int      # lines the buffer holds
VARIABLE con_backscroll   : int      # how far back the player has scrolled
VARIABLE con_forcedup     : bool     # the console is forced open because there
                                     # is no world to draw
VARIABLE con_initialized  : bool
VARIABLE con_chars        : bytes    # the font
VARIABLE con_notifylines  : int      # scanlines the notification area occupies
```

**Invariants** — `con_forcedup` is set when there is no connection, which is what makes the console fill
the screen at startup and after a disconnect rather than floating over a blank world.

The `con_initialized` flag matters because printing happens **before** the console exists — during the
memory and filesystem setup — and the print functions must tolerate it.

## Printing

**Contract** — `Con_Printf` formats and appends to the console, echoing to the platform's own output.
`Con_DPrintf` does the same only when the developer variable is set. `Con_SafePrintf` does the same while
suppressing any screen update, for use from inside the renderer. `Con_Print` appends already-formatted
text.

**Invariants** — three variants for one operation, and the distinction is real. The developer variant is
how the engine's chatter is kept out of a player's way. The **safe** variant exists because printing
normally can trigger a screen update, and a print from inside the drawing code would then recurse; it
sets and restores the update-suppression flag ([`screen.h`](screen.h.md)).

Appending also writes to the platform's console
([`sys.h`](sys.h.md)), which is what makes a dedicated server's output visible at all.

## Display

**Contract** — `Con_DrawConsole` draws a given number of lines, optionally including the input line.
`Con_DrawNotify` draws the recent lines over the game. `Con_ClearNotify` discards them.
`Con_DrawCharacter` draws one glyph at a console cell position. `Con_CheckResize` rebuilds the buffer
when the display's width changes, **preserving the text** by rewrapping it.

**Invariants** — the resize rewraps rather than discarding, which is what lets a player change resolution
without losing the log. It is the only place in the engine that reflows text.

## Modal notice

**Contract** — `Con_NotifyBox` shows a message and waits for a key, for startup warnings about missing
sound or CD hardware.

**Invariants** — used before the menus exist, which is why it is here rather than in
[`menu.h`](menu.h.md).

## Commands

**Contract** — `Con_Init` allocates the buffer and registers the commands. `Con_ToggleConsole_f` opens and
closes it. `Con_Clear_f` empties it.
