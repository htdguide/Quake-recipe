# WinQuake/cl_input.c

> Turns key presses into a movement command: a two-key button model, a fractional key-held measure that survives a key pressed and released within one frame, and the composition of turning, strafing and looking.

**Needs** — [`client.h`](client.h.md) · [`keys.h`](keys.h.md) · [`cmd.h`](cmd.h.md) · [`cvar.h`](cvar.h.md) · [`protocol.h`](protocol.h.md) · [`common.h`](common.h.md) · [`view.h`](view.h.md) · [`mathlib.h`](mathlib.h.md)
**Used by** — [`cl_main.c`](cl_main.c.md) builds and sends the command; [`keys.c`](keys.c.md) dispatches the plus and minus commands here
**Tier floor** — none

## Purpose

The header comment states the design: a logical button can be held by **two** physical keys at once, so pressing
an action on two keys and releasing one does not release the action. And a key that was pressed *and* released
within a single frame still contributes — as a fraction — so that a very brief tap is not lost at a high frame
rate.

Both properties come from one two-bit state word, and they are the file's whole content.

## State

```text
VARIABLE in_mlook, in_klook, in_strafe, in_speed : KButton    # modifiers
VARIABLE in_up, in_down, in_left, in_right, in_forward, in_back : KButton
VARIABLE in_moveleft, in_moveright, in_lookup, in_lookdown : KButton
VARIABLE in_attack, in_jump, in_use : KButton
VARIABLE in_impulse : int
```

**Invariants** — a button's state word uses **bit 0 for "currently held" and bit 1 for "pressed at some point
since the last command was built"**. The second bit is what makes a tap within one frame register.

## `KeyDown`, `KeyUp`

**Contract** — `KeyDown` records the pressing key in the first free of the button's two slots, ignores a repeat
of a key already held, and sets both state bits. `KeyUp` clears the releasing key's slot and clears the held bit
only when no key remains; a release with no argument clears both slots outright.

```text
FUNCTION key_down(b)
  k = the key number from the command's argument, or -1 when typed at the console
  IF k IS b.down[0] OR b.down[1]  RETURN          # already held by this key
  IF b.down[0] is free       b.down[0] = k
  ELSE IF b.down[1] is free  b.down[1] = k
  ELSE  print "Three keys down for a button!" ;  RETURN
  IF b.state HAS the held bit  RETURN             # already down
  b.state = b.state BITOR 1 BITOR 2               # held, AND pressed-this-frame

FUNCTION key_up(b)
  k = the key number, or -1
  IF k == -1                                      # typed at the console
    b.down[0] = b.down[1] = 0 ;  b.state = 4 ;  RETURN
  clear whichever slot holds k ;  IF neither does, RETURN
  IF either slot is still occupied  RETURN         # still held by the other key
  IF NOT (b.state HAS the held bit)  RETURN
  b.state = b.state WITHOUT the held bit, BITOR 4  # released-this-frame
```

**Invariants** — **three keys on one button prints a complaint and drops the third.** That is the practical
limit of the two-slot model, and it is the source of the classic stuck-movement bug: hold three keys bound to
one action, release the first two, and the action stays held because the third was never recorded.

**A press typed at the console uses key number −1** and a release clears both slots, which is how a bound action
can be forced off from a script.

Bit 2 records a release within the frame, the mirror of bit 1.

## `CL_KeyState`

**Contract** — takes a button; returns how much of the frame it was held, as 0, 0.5 or 1, and clears the
transition bits.

```text
FUNCTION cl_key_state(key) -> real
  down = 0
  impulsedown = key.state HAS bit 1        # pressed during this frame
  impulseup   = key.state HAS bit 2        # released during this frame
  down        = key.state HAS bit 0        # still held now

  IF impulsedown AND NOT impulseup
    down = 0.5 IF still held ELSE 0        # pressed mid-frame
  IF impulseup AND NOT impulsedown
    down = 0   IF still held ELSE 0.5      # released mid-frame
  IF NOT impulsedown AND NOT impulseup
    down = 1   IF still held ELSE 0        # steady state
  IF impulsedown AND impulseup
    down = 0.5 IF still held ELSE 0.25     # pressed AND released mid-frame

  key.state = key.state WITHOUT bits 1 and 2
  RETURN down
```

**Invariants** — the four cases are the whole point. A steady press gives 1; a press or release part-way through
gives 0.5, the expected value of a uniformly distributed transition; a press *and* release within one frame
gives 0.25 if it ended released. So **a tap shorter than a frame still moves the player**, by a quarter of a
frame's worth. Without it, at 70 frames a second a fast tap would sometimes do nothing at all.

The transition bits are cleared here, which means **this must be called exactly once per button per frame** — a
second call in the same frame sees a steady state.

## `CL_AdjustAngles`

**Contract** — applies keyboard turning and looking to the client's own view angles, scaled by the frame time
and the speed modifier, and clamps pitch and roll. Cancels the automatic pitch centring whenever the player
looks manually.

```text
FUNCTION cl_adjust_angles()
  speed = host_frametime, multiplied by the angle-speed modifier when held
  IF NOT strafing
    yaw = yaw - speed * cl_yawspeed * key_state(right)
              + speed * cl_yawspeed * key_state(left)
    yaw = anglemod(yaw)                        # quantized to the wire's grid
  IF in keyboard-look mode
    stop the pitch drift
    pitch = pitch -+ speed * cl_pitchspeed * key_state(forward / back)
  pitch = pitch -+ speed * cl_pitchspeed * key_state(lookup / lookdown)
  IF either look key was held  stop the pitch drift
  clamp pitch INTO -70 .. 80
  clamp roll  INTO -50 .. 50
```

**Invariants** — four things.

**The pitch limits are asymmetric: 80 down, 70 up.** (Positive pitch looks down,
[`mathlib.c`](mathlib.c.md#anglevectors).) So a player can look further down than up. That asymmetry is
deliberate and is in every engine of this lineage.

**The yaw is passed through the angle reduction** ([`mathlib.c`](mathlib.c.md#anglemod)), quantizing it to the
wire's 16-bit grid — so the client's own angle and the one the server receives agree exactly.

**The strafe modifier repurposes the turn keys as strafe keys**, which is why the yaw adjustment is skipped
while it is held.

**Any manual look cancels the automatic pitch centring** ([`view.c`](view.c.md)), which is
what stops the view fighting the player.

## `CL_BaseMove`

**Contract** — builds a movement command from the keyboard: side, up and forward speeds from the movement keys
and the modifiers, each scaled by its own configured speed and then by the run modifier. Does nothing until
fully connected.

```text
FUNCTION cl_base_move(cmd)
  IF not fully signed on  RETURN
  cl_adjust_angles()
  zero cmd
  IF strafing
    cmd.sidemove = cmd.sidemove +- cl_sidespeed * key_state(right / left)
  cmd.sidemove = cmd.sidemove +- cl_sidespeed * key_state(moveright / moveleft)
  cmd.upmove   = cmd.upmove   +- cl_upspeed   * key_state(up / down)
  IF NOT in keyboard-look mode
    cmd.forwardmove = cmd.forwardmove + cl_forwardspeed * key_state(forward)
                                      - cl_backspeed    * key_state(back)
  IF the run modifier is held
    scale all three BY cl_movespeedkey
```

**Invariants** — **forward and backward have separate speeds**, and the default backward speed is lower. So
retreating is slower than advancing, which is a deliberate combat-feel decision.

**The keyboard-look mode repurposes forward and back as pitch keys**, which is why the forward movement is
skipped while it is active.

The command carries **intended velocities in units per second**, not key states
([`client.h`](client.h.md)), so everything above is resolved before the command leaves the client.

## `CL_SendMove`

**Contract** — writes the movement message: the server timestamp being responded to, the three view angles, the
three movement speeds, a button bit set, and the pending impulse. Clears the impulse and the transient button
bits.

```text
FUNCTION cl_send_move(cmd)
  cl.cmd = cmd
  write the move tag
  write cl.mtime[0] as a float            # echoed so the server can time the
                                          # round trip
  FOR EACH axis  write the view angle     # one byte each
  write the three movement speeds as shorts
  bits = 0
  IF in_attack.state HAS either the held OR the pressed-this-frame bit
    bits = bits BITOR 1
  clear in_attack's pressed bit
  IF in_jump.state likewise  bits = bits BITOR 2
  clear in_jump's pressed bit
  write bits as a byte
  write in_impulse as a byte ;  in_impulse = 0
```

**Invariants** — three things.

**A button pressed and released within one frame still reaches the server**, because the test is against the
held bit *or* the pressed-this-frame bit. That is the same guarantee `CL_KeyState` gives the movement axes,
applied to the discrete buttons — and it is why a very fast click always fires.

**The timestamp is echoed back** so the server can compute the round trip without keeping a send history
([`sv_user.c`](sv_user.c.md#sv_readclientmove)).

**The impulse is one-shot and cleared on send**, which is how weapon selection reaches the server exactly once.

**Notes** — the message is written into a local 128-byte buffer and sent unreliably, so an unsent command is
simply lost — which is correct: the next frame's command supersedes it.

## `CL_InitInput`

**Contract** — registers the plus and minus command pairs for every logical button, and the impulse command.

**Invariants** — the plus-and-minus convention ([`keys.h`](keys.h.md)) is what makes every one of these
bindable, and the pairs are registered here rather than being special-cased in the key handler.
