# WinQuake/keys.h

> The key numbering every input backend must produce, and the binding table that turns a key into console text.

**Needs** — nothing beyond the base types
**Used by** — every input backend calls the event function with these numbers; [`console.c`](console.c.md), [`menu.c`](menu.c.md), [`cl_input.c`](cl_input.c.md), [`host.c`](host.c.md), [`sv_user.c`](sv_user.c.md) and [`host_cmd.c`](host_cmd.c.md) read the destination; implemented by [`keys.c`](keys.c.md)
**Tier floor** — none

## Purpose

Two things, and the numbering is the contract.

**The key numbers are an engine-wide namespace of 256 values** that covers the keyboard, the mouse, the
joystick and the mouse wheel. Every platform backend translates its own scancodes into these, and
bindings are stored against them — so a configuration file is portable between platforms, which is the
whole point.

**The input destination** decides who receives a key: the game, the console, a chat prompt, or a menu.
One variable, four states, and it is checked in several places outside the input system —
[`sv_user.c`](sv_user.c.md) consults it to decide whether to pause a single-player game.

## State

```text
ENUM KeyDest = { key_game, key_console, key_message, key_menu }
VARIABLE key_dest      : KeyDest
VARIABLE keybindings   : text[256]     # the console text each key produces
VARIABLE key_repeats   : int[256]      # how long each has been held
VARIABLE key_count     : int           # incremented on every event
VARIABLE key_lastpress : int
```

**Invariants** — the binding table is indexed by key number, so **a binding is a property of a key, not
of an action**. The reverse mapping — which key does this action — is found by searching, which is why
the menu's key-binding page scans the table.

The repeat counter is how held-key auto-repeat is implemented for the console without the platform
providing it.

`key_count` exists so that code can detect "any key was pressed since I last looked", which is how the
loading screen and the attract mode are dismissed.

## The numbering

```text
# Printable keys pass through as LOWERCASED ASCII.
CONSTANT k_tab = 9 ;  k_enter = 13 ;  k_escape = 27 ;  k_space = 32
CONSTANT k_backspace = 127

# 128..152: the non-printable keyboard
CONSTANT k_uparrow = 128, k_downarrow, k_leftarrow, k_rightarrow
CONSTANT k_alt = 132, k_ctrl, k_shift
CONSTANT k_f1 = 135 .. k_f12 = 146
CONSTANT k_ins = 147, k_del, k_pgdn, k_pgup, k_home, k_end
CONSTANT k_pause = 255

# 200..202: mouse buttons, as virtual keys
CONSTANT k_mouse1 = 200, k_mouse2, k_mouse3

# 203..206: joystick buttons
CONSTANT k_joy1 = 203 .. k_joy4 = 206

# 207..238: thirty-two auxiliary keys, so a many-buttoned joystick can use
#           the ordinary binding mechanism
CONSTANT k_aux1 = 207 .. k_aux32 = 238

# 239..240: the mouse wheel, as press/release pairs rather than an axis
CONSTANT k_mwheelup = 239, k_mwheeldown = 240
```

**Invariants** — five properties a rebuild must preserve.

**Printable keys are lowercased ASCII**, so a binding is stored against the unshifted key and shift is
applied only for text entry. That is why binding the shifted form of a key is impossible in this engine.

**Mouse buttons, joystick buttons and the mouse wheel are all *keys*.** There is no separate button
concept anywhere. That is what makes "bind mouse1 +attack" work with the same machinery as any keyboard
binding, and it is a genuinely good decision.

**The wheel is a press and release, not an axis.** A backend must synthesize both, because a binding
fires on the press and a plus-command's release half must also arrive.

**The thirty-two auxiliary numbers exist purely so an unusual device can be bound**, with no meaning of
their own. The comment says so.

The gap between 152 and 200 and the isolated pause at 255 are historical and unused; a rebuild should
keep them empty rather than compacting, because published configuration files contain the numbers.

## Entry points

**Contract** — `Key_Event` is called by a backend with a key number and whether it went down; it routes
to the current destination, handles the console's editing keys, or executes the key's binding.
`Key_Init` builds the default bindings. `Key_SetBinding` assigns one. `Key_WriteBindings` writes the whole
table to the configuration file. `Key_ClearStates` releases every held key.

**Invariants** — a binding beginning with a plus sign is a **held action**: pressing the key runs the
binding and releasing it runs the same name with a minus sign
([`cl_input.c`](cl_input.c.md)). That convention is implemented in the event handler and is how movement
keys work at all.

`Key_ClearStates` is called when focus is lost, and it must run every held binding's release half — not
merely clear the state — or the player keeps moving.
