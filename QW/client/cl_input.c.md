# QW/client/cl_input.c

> Building and sending one command per frame: accumulate the keyboard and mouse into a movement intent, quantize it exactly as the wire will, keep it in a ring for prediction, and send it with the two before it and a checksum.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`protocol.h`](protocol.h.md) · [`net_chan.c`](net_chan.c.md) · [`cl_cam.c`](cl_cam.c.md) · [`crc.h`](crc.h.md) · [`input.h`](input.h.md)
**Used by** — [`cl_main.c`](cl_main.c.md) calls it once per frame
**Tier floor** — none

## Purpose

Read [`cl_input.c`](../../WinQuake/cl_input.c.md) for the shared half: the key-state machine that survives a key being pressed and
released within one frame, the speed and strafe modifiers, and the assembly of the movement intent. Unchanged and load-bearing.

What changed is everything about the command *after* it is built, and each change serves prediction.

## State

As [`cl_input.c`](../../WinQuake/cl_input.c.md); the records are unchanged except where **What differs** says otherwise.

## `CL_FinishMove`

**Contract** — completes a command: discard the first two commands of a session, fold the edge-triggered buttons in, set the
command's duration from the frame time with an upper sanity bound, copy the view angles, take the pending impulse, and then
**quantize every field to exactly the precision the wire carries**.

```text
FUNCTION finish_move(cmd)
  IF this is one of the first two commands of the session  RETURN
  IF attack was pressed or held  set its button ;  clear its edge
  IF jump   was pressed or held  set its button ;  clear its edge
  ms = frame time in milliseconds
  IF ms > 250  ms = 100                        # an implausible frame: substitute
  cmd.msec = ms
  cmd.angles = the current view angles
  cmd.impulse = the pending impulse ;  clear it
  cmd.forwardmove, sidemove, upmove = each ROUNDED to a signed byte
  FOR EACH angle  angle = round to 1/65536 of a turn and back
```

**Invariants** —

- **The command is quantized to the wire's precision before it is stored.** This is the most important line in the file: the
  client predicts from the *same* command the server will receive, not from the full-precision one it built. Without it, prediction
  and authority diverge by the rounding error on every command, continuously
  ([`cl_pred.c`](cl_pred.c.md), [`pmove.c`](pmove.c.md)'s grid snap is the companion rule for position).
  **Quantize, then predict — never predict, then quantize.**
- **The first two commands of a session are discarded**, because they can contain input left over from the previous level's key
  states. A cheap fix for a real class of "I spawned already moving" bug.
- **An implausibly long frame has its duration replaced rather than clamped**, on the grounds that a 250-millisecond frame is a
  stall and not real elapsed play. Substituting a plausible value keeps the player from lurching after a disk access.
- **Buttons are read from the edge-triggered key state and their edges cleared here**, so a press and release inside one frame still
  produces one button-down command. That mechanism is the original's and it matters more here, because a command is now the unit of
  simulation.
- The angles are quantized to a **sixteen-bit** fraction of a turn, which is finer than the byte the protocol uses for most angles
  ([`common.c`](common.c.md)) — the command's angles get the precise form because aim precision is the whole game.

## `CL_SendCmd`

**Contract** — once per frame: build the command, store it in the ring slot of the packet about to be sent, then compose the
movement block — a checksum placeholder, the reported loss, and **the last three commands each delta-encoded against the previous** —
fill in the checksum over the block seeded with the packet's sequence, request a delta reference if one is usable, and transmit.

```text
FUNCTION send_cmd()
  IF replaying a recording  RETURN            # commands come from the recording
  slot = the ring slot for the outgoing sequence
  slot.senttime = now ;  slot.receivedtime = none
  build the command: keyboard, then external controllers, then
    the spectator camera if spectating, then finish_move, then the camera again
  write the movement command kind
  remember where the checksum byte goes ;  write a placeholder
  write the measured loss percentage
  write the command from two packets ago, delta from empty
  write the command from one packet ago,  delta from that
  write this packet's command,            delta from that
  fill in the checksum over everything after it, seeded with the sequence
  IF the newest usable snapshot is older than the ring holds
    forget it                               # must ask for an absolute snapshot
  IF a usable snapshot exists AND deltas are enabled AND we are playing
    record it as this packet's reference ;  write the delta-reference command
  ELSE
    record this packet as having no reference
  IF recording  write the command to the recording
  transmit
```

**Invariants** —

- **Three commands per packet is the loss compensation**, matching the server's expectation
  ([`sv_user.c`](../server/sv_user.c.md)): losing one or two packets costs nothing because the missing commands are re-sent. They
  are delta-encoded against each other so the cost is a few bytes.
- **The command is stored in the ring before it is sent**, because prediction replays from the ring
  ([`cl_pred.c`](cl_pred.c.md)) and the ring index is the packet sequence — which is how a server reply names the command it
  answers.
- **The client asks for the snapshot it wants to delta from**, by sequence, rather than the server guessing. So the client is the
  authority on what it still holds, and a client that lost its reference simply stops asking and receives an absolute snapshot.
  *Putting the choice on the receiving side is the correct design and it is why the scheme needs no negotiation.*
- **The reference is abandoned when it ages out of the ring**, checked here rather than on receipt, so the request is never for
  something already discarded.
- **Deltas are disabled while recording**, because a recording must be replayable without the reference chain
  ([`cl_demo.c`](cl_demo.c.md)).
- **Exactly one packet per frame is sent**, which is what sets the server's reply rate
  ([`sv_send.c`](../server/sv_send.c.md)) — the client's frame rate is the packet rate in both directions.
- The checksum is seeded with the packet's sequence so a replayed movement block is detectable
  ([`sv_user.c`](../server/sv_user.c.md)); it is not a security measure and the recipe records it as such.
- The spectator camera contributes **both before and after** the command is finished, because it needs to influence the movement
  intent and then to override the view angles ([`cl_cam.c`](cl_cam.c.md)).

## `MakeChar`

**Contract** — rounds a value to the signed byte the wire carries.

**Invariants** — it rounds to a multiple that the byte encoding can represent exactly. This is the quantization above, factored, and
the reason it exists as a named function is that **it must be the same rounding the encoder performs** or the client's stored
command differs from the sent one.

## `CL_ClearStates`

**Contract** — releases every held key and clears the pending impulse.

**Invariants** — called on losing focus and on disconnection, the same release-everything rule as every platform backend
([`vid_win.c`](../../WinQuake/vid_win.c.md)).

## `CL_BaseMove`, `CL_AdjustAngles`, `CL_KeyState`, the key handlers and `CL_InitInput`

**Contract** — as the original ([`cl_input.c`](../../WinQuake/cl_input.c.md)): turn the key states into a movement intent with
run and strafe modifiers, adjust the view angles by the turn keys with acceleration, and register every control.

**Notes** — the one-line summary a rebuilder needs: **the command is the unit of simulation, so it must be exactly what the server
will see before anything predicts from it, it must be kept until acknowledged, and it must be sent more than once.** All three are
in this file.
