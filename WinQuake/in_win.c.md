# WinQuake/in_win.c

> The Windows input backend: relative mouse motion taken either from the window's messages or from a direct device, the pointer confined and hidden while the game has focus, and a joystick surveyed and mapped onto named axes.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`winquake.h`](winquake.h.md) · [`input.h`](input.h.md) · [`client.h`](client.h.md) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`cl_input.c`](cl_input.c.md) calls the move accumulation; [`host.c`](host.c.md) initializes it
**Tier floor** — none

## Purpose

The reference filling of the input seam ([`input.h`](input.h.md)). The engine wants three things per frame: accumulated
relative mouse motion, the state of every key, and axis positions. Keys arrive through the window handler
([`vid_win.c`](vid_win.c.md)); this file owns the other two, and the mouse half contains the decisions.

## State

```text
VARIABLE mouse_x, mouse_y                 # accumulated this frame
VARIABLE old_mouse_x, old_mouse_y         # for the filter
VARIABLE mouse_buttonstate, mouse_oldbuttonstate
VARIABLE mouseactive, mouseinitialized, mouseparmsvalid, restore_spi
VARIABLE the saved system mouse parameters
VARIABLE the direct device and its buffered-event state, if available
VARIABLE the joystick's capabilities, the axis map, and the named settings
```

## `IN_ActivateMouse`, `IN_DeactivateMouse`, `IN_ShowMouse`, `IN_HideMouse`, `IN_SetQuakeMouseState`, `IN_RestoreOriginalMouseState`, `IN_UpdateClipCursor`

**Contract** — take the mouse: hide the pointer, replace the system's acceleration settings with linear ones, confine the
pointer to the window, and centre it; and give it all back.

**Invariants** —

- **The system's pointer acceleration must be disabled and restored.** The engine wants raw counts; an accelerated pointer
  makes aiming speed depend on how fast you were already moving. Failing to restore it leaves the user's desktop pointer
  behaving strangely after the game exits — which is why the restoration also runs from the error path.
- **The pointer is confined to the window**, and the confinement must be re-applied whenever the window moves or resizes
  ([`vid_win.c`](vid_win.c.md) republishes the rectangle). Without confinement the pointer leaves the window and motion
  stops arriving.
- Taking and releasing the mouse is **paired with focus and with the console and menu being open**, because a player who
  opens the menu expects their pointer back.

## `IN_InitInput`, `IN_InitDInput`, `IN_StartupMouse`, `IN_Init`, `IN_Shutdown`

**Contract** — try the direct device interface first and fall back to the window's mouse messages; record how many buttons
and whether a wheel is present; register the settings and commands.

**Invariants** — **the direct device gives buffered relative counts with no pointer involved at all**, which is strictly
better, so it is tried first and the message path is the fallback. The same detect-and-degrade rule as everywhere else. A
rebuild should look for its platform's raw-input facility before doing anything with a pointer.

## `IN_MouseEvent`, `IN_Accumulate`, `IN_MouseMove`, `IN_Move`

**Contract** — record a button state change; accumulate the motion since the last call; convert the accumulated motion into
view angle changes or movement, applying sensitivity, the optional filter and the optional mouse-look mode; and combine the
mouse and joystick contributions into the command being built.

```text
FUNCTION mouse_move(command)
  read the accumulated relative counts (from the device buffer, or from
    the pointer's offset from the window centre, then re-centre it)
  IF filtering is on
    use the average of this frame's and last frame's counts
    remember this frame's
  scale by sensitivity
  IF mouse-look is active (held, or toggled on)
    yaw   -= horizontal * yaw sensitivity
    pitch += vertical  * pitch sensitivity, then CLAMP to the pitch limits
  ELSE
    horizontal becomes a sidestep, vertical becomes forward motion
```

**Invariants** —

- **Motion is accumulated between frames and consumed once per frame**, never applied as it arrives. That keeps input
  sampling aligned with the command being built ([`cl_input.c`](cl_input.c.md)) and makes the amount of motion per command
  proportional to the frame time, which is what makes turning speed frame-rate independent.
- **Pitch is clamped and yaw is not**, because looking past vertical is meaningless while turning is unbounded. The clamp
  belongs here, not in the game logic.
- **The filter averages two frames.** It halves jitter and adds half a frame of latency. It is a setting because players
  disagree, and the recipe records it as a deliberate trade rather than a smoothing detail.
- When mouse-look is off, vertical motion moves the player rather than the view — the mode the game shipped with. Both
  modes must exist because the key that toggles them is a game control.
- Re-centring the pointer after reading it is the same technique, and the same hazard, as
  [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md).

## `IN_StartupJoystick`, `IN_ReadJoystick`, `IN_JoyMove`, `RawValuePointer`, `Joy_AdvancedUpdate_f`, `IN_Commands`, `IN_ClearStates`, `Force_CenterView_f`

**Contract** — survey the joystick and its axis count; read its raw values; map each physical axis to one of the named
logical axes — turn, look, forward, side — applying a dead zone, a sensitivity and an optional absolute-versus-relative
interpretation; report button changes as key presses; clear all state; and recentre the view.

**Invariants** —

- **Axes are mapped by name, not by index**, configured from the console, because no two devices agreed on ordering. That
  indirection is the whole reason this code is long.
- An axis can be **absolute** (a position, giving a target angle) or **relative** (a rate, giving an angular velocity), and
  which it is depends on the control: a throttle is absolute, a twist is a rate. A rebuild supporting analogue input needs
  the same distinction.
- A **dead zone** is mandatory: an unloaded analogue axis never reads exactly centre, and without a dead zone the view
  drifts forever.
- Joystick buttons are **injected as key presses** into the same key state the keyboard uses, so every binding mechanism
  works on them with no extra code. That is the right structure and worth copying.

**Notes** — the file also clears every held key and button whenever the mouse is deactivated, which is the same
release-everything-on-focus-loss rule stated in [`vid_win.c`](vid_win.c.md). It must be here as well, because the mouse
buttons are tracked here.
