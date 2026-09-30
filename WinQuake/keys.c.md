# WinQuake/keys.c

> Key dispatch: routes each event to the game, the console, a chat prompt or a menu; edits the console's input line with history and completion; and holds the binding table.

**Needs** — [`keys.h`](keys.h.md) · [`console.h`](console.h.md) · [`menu.h`](menu.h.md) · [`client.h`](client.h.md) · [`cmd.h`](cmd.h.md) · [`cvar.h`](cvar.h.md) · [`zone.h`](zone.h.md) · [`screen.h`](screen.h.md) · [`common.h`](common.h.md)
**Used by** — every input backend calls the event function; [`host.c`](host.c.md) initializes it; [`host.c`](host.c.md) and [`sv_user.c`](sv_user.c.md) read the destination
**Tier floor** — none

## Purpose

The routing decision and the console's line editor. The routing is four-way and the substance is in which keys
are intercepted before the destination sees them — because a few must always work regardless of where input is
going.

## State

```text
CONSTANT maxcmdline = 256
VARIABLE key_lines : text[32][256]       # the console's history ring
VARIABLE edit_line, history_line : int
VARIABLE key_linepos : int
VARIABLE key_dest : KeyDest
VARIABLE keybindings : text[256]
VARIABLE consolekeys, menubound : bool[256]     # which keys each destination
                                                # claims
VARIABLE keyshift : int[256]             # the shifted form of each key
VARIABLE key_repeats : int[256]
VARIABLE chat_buffer : text ;  team_message : bool
```

**Invariants** — the console's history is a **32-entry ring of complete lines**, and the current line is an
entry in it. So editing a recalled line edits a copy, which is why recalling and then editing does not destroy
the original.

Two per-key tables decide routing: one says a key is meaningful to the console, the other that a key must reach
its *binding* even while a menu is open.

## `Key_Console`

**Contract** — edits the console's input line: enter executes and appends to the history; tab completes a command
or variable name; backspace and the arrows move and delete; the page keys scroll the backscroll; the up and down
arrows walk the history; a printable key inserts. Anything else is ignored.

```text
FUNCTION key_console(key)
  SELECT key
    enter       append the line (minus its leading prompt) to the command queue
                print it ;  advance to the next history entry ;  reset the
                cursor ;  update the screen if disconnected
    tab         complete the first token against commands, then variables
    backspace   remove the character before the cursor
    left arrow  move the cursor left ;  right arrow, right
    up arrow    walk BACK through the history, skipping empty entries
    down arrow  walk forward, ending at a fresh empty line
    page up     scroll the backscroll up ;  page down, down
    home / end  jump to the first or last line
    a printable character  insert it, if the line has room
```

**Invariants** — three things.

**Completion tries commands first, then variables**, matching the dispatcher's own priority
([`cmd.c`](cmd.c.md#cmd_executestring)) — so the name it offers is the one that would actually run.

**The history walk skips empty entries**, so the ring's unused slots do not appear as blank recalls.

**The screen is updated explicitly after a command when disconnected**, because the frame loop is not producing
frames to show the result.

## `Key_Message`

**Contract** — edits the chat prompt: enter sends it as a say command, escape cancels, backspace deletes, a
printable key appends.

**Invariants** — the line is sent as `say` or `say_team` depending on which key opened the prompt, which is why
the mode is remembered in a flag.

## `Key_StringToKeynum`, `Key_KeynumToString`

**Contract** — convert between a key's name and its number, using a table of names for the non-printable keys and
the character itself for the printable ones.

**Invariants** — the names are what a configuration file contains, so they are a persisted format. A rebuild must
keep them exactly ([`keys.h`](keys.h.md)).

**Notes** — the source asks how a quote character should be handled, since bindings are written quoted. It is not
handled; a binding containing a quote writes a file that reads back wrongly.

## `Key_SetBinding`, `Key_Unbind_f`, `Key_Unbindall_f`, `Key_Bind_f`, `Key_WriteBindings`

**Contract** — assign a key's binding, releasing any previous one; the three commands that unbind one key, unbind
everything, and bind or report; and write every non-empty binding to the configuration file as a `bind` command.

**Invariants** — the bind command with one argument *reports* the binding rather than clearing it, which is why
unbinding needs its own command.

## `Key_Init`

**Contract** — initializes the history, marks which keys the console and the menus claim, builds the shift table,
and registers the three binding commands.

**Invariants** — the shift table is built by hand rather than from a locale, so shifted punctuation is fixed to a
US layout. That is a real limitation and a rebuild should take the shifted character from the platform.

## `Key_Event`

**Contract** — the entry point every backend calls. Records the key's state and repeat count, suppresses excessive
auto-repeat, handles the keys that must always work, then routes to the current destination — or, on a release,
always runs the binding's release half.

```text
FUNCTION key_event(key, down)
  keydown[key] = down
  IF NOT down  key_repeats[key] = 0
  ELSE
    key_repeats[key] = key_repeats[key] + 1
    # Suppress runaway auto-repeat except for the keys where repeat is wanted.
    IF key_repeats[key] > 1 AND key is not a console-editing key
      RETURN
    IF the key is unbound AND the destination is the game
      print "<name> is unbound, hit F4 to set"

  # --- keys that always work ---
  IF key IS escape
    IF NOT down  RETURN
    SELECT the destination
      message   cancel the chat prompt
      menu      m_keydown(escape)
      game      toggle the menu
    RETURN
  IF key IS the console toggle  ...likewise

  # --- a RELEASE always runs the binding's minus half ---
  IF NOT down
    kb = the binding
    IF kb begins with '+'  queue "-" + the rest + " " + the key number
    IF the shifted form has its own binding  queue its minus half too
    RETURN

  # --- a PRESS: route it ---
  IF the destination is a menu AND the key is not menu-bound
    m_keydown(key) ;  RETURN
  IF the destination is the game AND fully connected, OR the key is menu-bound
    kb = the binding
    IF kb begins with '+'  queue kb + " " + the key number
    ELSE                   queue kb
    RETURN
  apply the shift state
  SELECT the destination
    message  key_message(key)
    game, console  key_console(key)
```

**Invariants** — six things, and every one matters.

**A release *always* runs the binding's release half**, before any routing. Otherwise opening the console while
holding a movement key would leave the player moving forever. This single rule is why the plus-and-minus
convention is safe.

**Escape and the console toggle are intercepted before routing**, so they always work — which is what makes the
engine recoverable from any state.

**The key number is appended to a plus-binding's arguments**, which is how the two-slot button model knows which
key pressed it ([`cl_input.c`](cl_input.c.md#keydown-keyup)).

**Auto-repeat is suppressed for everything except the console's editing keys**, so holding a movement key does not
re-run its binding sixty times a second.

**A menu-bound key reaches its binding even with a menu open**, which is how the screenshot and console keys work
from a menu.

**A press is only bound when fully connected**, so keys pressed during a load do nothing — which pairs with the
client discarding its first few movement commands ([`client.h`](client.h.md)).

## `Key_ClearStates`

**Contract** — releases every held key by calling the event handler with a release for each, so every binding's
release half runs.

**Invariants** — it must **route through the event handler** rather than merely clearing the table, precisely so
the release halves run. Called when focus is lost.
