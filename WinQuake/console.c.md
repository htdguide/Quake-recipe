# WinQuake/console.c

> The console: a wrapping character grid that survives a resolution change, the engine's three print variants, and the notification lines that surface recent messages over the game.

**Needs** — [`console.h`](console.h.md) · [`draw.h`](draw.h.md) · [`vid.h`](vid.h.md) · [`keys.h`](keys.h.md) · [`client.h`](client.h.md) · [`screen.h`](screen.h.md) · [`cmd.h`](cmd.h.md) · [`cvar.h`](cvar.h.md) · [`zone.h`](zone.h.md) · [`common.h`](common.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — every module prints through it; [`screen.c`](screen.c.md) draws it; [`keys.c`](keys.c.md) edits its input line
**Tier floor** — none

## Purpose

The engine's standard output. Two things are worth reading: the buffer is a **character grid**, not a list of
lines, so wrapping is decided as text arrives and a resolution change must reflow it; and there are three print
variants whose differences matter.

## State

```text
CONSTANT con_textsize = 16384          # the grid, in characters
CONSTANT num_con_times = 4             # notification lines
VARIABLE con_text : bytes              # the grid: linewidth by totallines
VARIABLE con_linewidth, con_totallines : int
VARIABLE con_current, con_x, con_display, con_backscroll : int
VARIABLE con_times : real[4]           # when each notification line arrived
VARIABLE con_notifytime : Cvar = 3     # seconds
VARIABLE con_debuglog : bool
```

**Invariants** — the grid is **fixed at 16384 characters** and the line count is derived from the width, so a
wider console holds fewer lines. The buffer is a ring indexed by a monotonically increasing line number modulo the
count.

The notification timestamps are a small ring parallel to the last four lines, which is how their expiry is
tracked without touching the grid.

## `Con_CheckResize`

**Contract** — recomputes the line width from the display's console width; if it changed, rebuilds the grid,
**preserving the existing text** by copying it into the new geometry. A display too narrow for a minimum width
falls back to a fixed layout and clears the buffer.

```text
FUNCTION con_check_resize()
  width = (the console's pixel width SHIFTED RIGHT 3) - 2      # 8-pixel glyphs
  IF width == con_linewidth  RETURN
  IF width < 1                                   # video is not up yet
    con_linewidth = 38 ;  con_totallines = 16384/38 ;  clear the grid
    RETURN
  copy the old grid into a temporary, cleared to spaces
  con_linewidth = width ;  con_totallines = 16384 / width
  clear the grid to spaces
  # Copy back line by line, keeping each line's LEFT portion.
  FOR EACH old line, newest first, as far as either geometry allows
    copy min(old width, new width) characters into the corresponding new line
  con_current = con_totallines - 1 ;  con_display = con_current
```

**Invariants** — **the text is preserved across a resize**, which is the only reflow in the engine. It keeps each
line's left portion rather than rewrapping, so a long line resized narrower loses its tail.

The two-character inset is a margin. The fallback width of 38 is what the console uses before the display exists,
which is when the earliest startup messages are printed.

## `Con_Init`

**Contract** — allocates the grid, sets a provisional width, prints the console's banner, and registers the
commands. Enables the debug log when asked on the command line.

**Invariants** — allocation happens **before** the display is up, which is why the provisional width exists and
why every print before that point is guarded by the initialization flag.

## `Con_Print`, `Con_Linefeed`

**Contract** — `Con_Print` appends text to the grid, wrapping at the line width, handling newline and carriage
return, and honouring a leading control byte that switches the whole line to the bright glyph set. Records a
notification timestamp for each new line. `Con_Linefeed` advances to a new line, clearing it.

```text
FUNCTION con_print(txt)
  mask = 0
  IF the text begins with byte 1 or 2
    mask = 128 ;  skip that byte           # the BRIGHT glyph set
  FOR EACH character c
    # Decide whether a word break must wrap.
    count the characters to the next space; if the line cannot hold them, wrap
    SELECT c
      newline         con_linefeed() ;  start a new line
      carriage return con_x = 0 ;  suppress the next line's timestamp
      otherwise       write (c BITOR mask) at the cursor ;  advance
                      IF the cursor reached the line width  con_linefeed()
```

**Invariants** — three things.

**A leading control byte 1 or 2 sets the high bit on the whole line**, which is what makes chat lines and server
banners render highlighted ([`draw.h`](draw.h.md#text)). The server writes that byte
([`host_cmd.c`](host_cmd.c.md#host_say-host_say_f-say-host_say_team_f-say_team)).

**Word wrapping looks ahead to the next space** and breaks early if the word will not fit, so the console wraps at
word boundaries rather than mid-word.

**A carriage return without a newline rewrites the current line**, which is how progress messages overwrite
themselves.

## `Con_Printf`, `Con_DPrintf`, `Con_SafePrintf`

**Contract** — format into a 4096-byte buffer and print, also echoing to the platform's console; the same only when
the developer variable is set; and the same while suppressing screen updates.

```text
FUNCTION con_printf(fmt, ...)
  format into a 4096-byte buffer
  print it TO the platform's console                  # so a dedicated server
                                                      # shows it
  IF the debug log is enabled  append it to the log file
  IF the console is not initialized yet  RETURN
  IF a dedicated server  RETURN                       # nothing to draw into
  con_print(the text)
  # Update the screen so the message is visible even without a frame loop.
  IF not fully connected AND updates are not suppressed
    scr_update_screen()

FUNCTION con_safe_printf(fmt, ...)
  suppress screen updates ;  con_printf(...) ;  restore
```

**Invariants** — three things.

**Printing updates the screen when there is no frame loop.** That is how startup and loading messages appear at
all, and it is why the safe variant must exist: a print from inside the renderer would otherwise re-enter it.

**A dedicated server prints only to the platform's console** and skips the grid entirely.

**The formatted buffer is 4096 bytes with no bound check**, and the source acknowledges it as a hazard. A rebuild
should bound it.

## `Con_DrawInput`

**Contract** — draws the console's current input line with a blinking cursor, scrolling horizontally when the line
is longer than the console is wide.

## `Con_DrawNotify`

**Contract** — draws the last four console lines that are still within the notification period, over the game; and
the chat prompt when one is open.

**Invariants** — the notification period is a tunable in seconds. Lines older than it are skipped rather than
removed, so the grid is untouched.

## `Con_DrawConsole`

**Contract** — draws the backdrop, then as many lines of the grid as the given height allows, starting from the
scroll position, then a marker when the view is scrolled back, then the input line.

## `Con_ToggleConsole_f`, `Con_Clear_f`, `Con_ClearNotify`, `Con_MessageMode_f`, `Con_MessageMode2_f`

**Contract** — toggle the console, changing the input destination and clearing the notification lines; clear the
grid; clear the notification timestamps; and open the chat prompt in ordinary or team mode.

**Invariants** — toggling the console **refuses while a menu is up** and instead does nothing, so the two cannot
be open at once. And toggling it while the console is forced open by having no world is a no-op
([`console.h`](console.h.md)).

## `Con_NotifyBox`

**Contract** — prints a message, draws it, and blocks until a key is pressed.

**Invariants** — used before the menus exist, for startup warnings about missing sound or CD hardware
([`host.c`](host.c.md#host_init)).

## `Con_DebugLog`

**Contract** — appends a formatted line to a named file, opening and closing it per call.

**Invariants** — opening and closing per line is what makes the log survive a crash, which is the whole reason it
exists.
