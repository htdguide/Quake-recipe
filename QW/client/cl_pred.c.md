# QW/client/cl_pred.c

> Prediction: re-run every command the server has not yet answered, starting from the last state it confirmed, then interpolate to the moment being displayed.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`pmove.h`](pmove.h.md) · [`pmove.c`](pmove.c.md) · [`client.h`](client.h.md) · [`cl_ents.c`](cl_ents.c.md)
**Used by** — [`cl_main.c`](cl_main.c.md) calls it once per frame before rendering
**Tier floor** — none

## Purpose

The payoff for [`pmove.c`](pmove.c.md)'s extraction. The client's own position as the server last reported it is a round trip
old. Rather than display that, the client takes that confirmed state and **replays its own unacknowledged commands through the
identical movement code**, arriving at where it believes it is now. When the server's answer for one of those commands
arrives, the replay simply starts from the newer confirmed state — so an error corrects itself within one round trip and
without any explicit reconciliation step.

This is the canonical form of client-side prediction and every subsequent game uses a variant of it. The whole file is 200
lines, which is the argument: **prediction is cheap once movement is a pure function.**

## State

```text
VARIABLE cl_pushlatency    # how far back in time to display, in milliseconds
VARIABLE cl_nopred         # disable prediction, for diagnosis
# in the client state:
VARIABLE frames[]          # a ring indexed by packet sequence, holding for each:
                           #   the command sent, the time it was sent,
                           #   and the player state the server reported (if yet)
VARIABLE simorg, simvel, simangles   # the predicted present, used for rendering
VARIABLE validsequence     # the newest sequence with a usable server answer
```

**Invariants** — the ring is indexed by the **outgoing packet sequence**, the same index the server uses for its snapshots
([`sv_ents.c`](../server/sv_ents.c.md)). So a server answer names the command it answers by sequence, and the client finds it
with one masked index. The two rings are the same size and that is not a coincidence — it is the maximum prediction horizon.

## `CL_PredictMove`

**Contract** — computes the position to render from: sets the displayed time slightly behind the present, finds the last
confirmed frame, replays each later command in turn until the replay passes the displayed time, then interpolates between the
last two replayed states. Falls back to the confirmed state when prediction is disabled, and gives up when the connection has
stalled.

```text
FUNCTION predict_move()
  IF paused or in intermission  RETURN
  displayed_time = now - measured latency - the display lag setting
  clamp displayed_time to not exceed now
  IF no valid server answer yet  RETURN
  IF the number of unacknowledged packets fills the ring  RETURN   # connection stalled
  simangles = the player's current view angles                      # never predicted
  from = the frame the server most recently answered
  IF this is the first answered frame  the connection is now fully active
  IF prediction is disabled
    simorg, simvel = the confirmed state ;  RETURN
  make the other players solid for the duration of the replay
  FOR i = 1 upward while sequence+i is still unacknowledged, bounded by the ring
    to = frame[sequence + i]
    predict_usercmd(from.playerstate, to.playerstate, to.cmd)
    IF to.senttime >= displayed_time  BREAK
    from = to
  restore the world list
  IF the ring ran out  RETURN                                       # too far behind
  f = (displayed_time - from.senttime) / (to.senttime - from.senttime), clamped to 0..1
  IF any axis of the two origins differs by more than 128 units
    use `to` outright                                               # a teleport
  simorg = from.origin + f * (to.origin - from.origin)
  simvel likewise
```

**Invariants** —

- **The view angles are never predicted.** They come straight from the player's input and are applied to rendering
  immediately, independent of prediction — so looking around has no latency even when position does. This is why the game
  feels responsive even when prediction is off, and it is worth stating explicitly because a rebuild that predicts angles as
  part of the state will find them fighting the mouse.
- **The displayed time is set slightly in the past**, by the measured latency plus an adjustable amount. That is deliberate:
  rendering at exactly the predicted present would show the very newest replayed state, which jumps whenever a correction
  arrives. Rendering a fraction behind means there are always two replayed states to interpolate between and the motion is
  smooth. The setting exists because the right amount is a matter of taste and connection.
- **A correction is never applied as a correction.** There is no "server says you are here, so move there" step. The replay
  simply begins from the newer confirmed state, and if the server disagreed, the newly replayed positions differ from the
  previously displayed ones — which the interpolation smooths over. *The absence of an explicit reconciliation step is the
  elegance of the design.*
- **A large discrepancy is treated as a teleport and not interpolated.** Without the check, a teleport draws the player
  sliding across the map. The threshold is a distance no single command can cover.
- **The other players are made solid for the replay and then removed again**, because the predicted movement must collide with
  them or the player walks through opponents and is then pushed out when the server disagrees. Restoring the list afterwards
  matters because the same list is used for other purposes
  ([`cl_ents.c`](cl_ents.c.md)).
- **Prediction stops when the connection stalls.** If the unacknowledged commands fill the ring, the client has no confirmed
  state within the horizon and continuing would extrapolate indefinitely. It freezes instead, which is the honest behaviour.
- The display-lag setting is **forced non-positive at entry**: a positive value would mean displaying the future, which the
  code refuses rather than trusting.

## `CL_PredictUsercmd`

**Contract** — runs one command through the movement model, from one player state to the next, **splitting a long command in
half recursively** first. Copies the state into the shared movement record, runs the move, and copies the result back.

```text
FUNCTION predict_usercmd(from, to, cmd, spectator)
  IF cmd.msec > 50
    halve the command's duration
    predict_usercmd(from, temp, half) ;  predict_usercmd(temp, to, half)
    RETURN
  load the movement record from `from`, taking ANGLES FROM THE COMMAND
  set dead from the player's own health, and the spectator flag
  run the movement model
  store origin, angles, velocity, ground state, water-jump timer and
    the buttons-held state back into `to`
  carry the weapon animation frame across unchanged
```

**Invariants** —

- **A command longer than a fixed limit is split in half, recursively.** A single large integration step gives different
  results from two small ones — a fast player can pass through a thin wall, and friction applied once over a long interval
  differs from twice over half. **The server splits at the same limit** ([`sv_user.c`](../server/sv_user.c.md)), and if the
  two limits differ the prediction diverges on exactly the frames where the player's frame rate dropped. Matching this
  constant is not optional.
- **The angles come from the command, not from the previous state**, because the command is what the player actually aimed
  with and it is what the server will use.
- **The buttons-held state is carried forward** through every replayed command, so an edge-triggered jump
  ([`pmove.c`](pmove.c.md#jumpbutton)) behaves the same in the replay as it did live.
- The death state is read from the player's own statistics rather than from the replayed state, because health is not
  predicted — it is authoritative and arrives in the snapshot.
- The weapon animation frame is **passed through untouched**, since it is driven by game logic the client does not run.

## `CL_NudgePosition`

**Contract** — if the confirmed position is inside solid matter, try the eight neighbouring eighth-unit positions in the
horizontal plane and take the first that is free.

**Invariants** — needed because **the server's position arrives quantized to an eighth of a unit** and the rounding can put a
player a fraction inside a wall ([`common.c`](common.c.md)). Predicting forward from inside a wall gives a player stuck
against nothing.

Note that this searches only the horizontal plane, whereas the mover's own unsticking
([`pmove.c`](pmove.c.md#nudgeposition)) searches all three axes. The two are not identical, which is a small latent source of
disagreement; a rebuild should use one routine for both.

## `CL_InitPrediction`

**Contract** — registers the two settings.

**Notes** — the diagnostic switch that disables prediction is worth keeping in a rebuild. Prediction bugs present as
jitter, rubber-banding or sticking, all of which have several possible causes; being able to turn it off and see the
unpredicted truth separates "my prediction is wrong" from "my movement model is wrong" in one keystroke.
