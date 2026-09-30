# QW/server/sv_user.c

> Everything a client can ask of the server: the spawn handshake in named stages, three commands per packet so a lost packet loses nothing, a checksum over the movement, the file transfer, chat with flood protection, and the spectator's commands.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`pmove.h`](../client/pmove.h.md) · [`pmove.c`](../client/pmove.c.md) · [`world.h`](world.h.md) · [`sv_nchan.c`](sv_nchan.c.md) · [`sv_send.c`](sv_send.c.md) · [`pr_exec.c`](pr_exec.c.md) · [`crc.h`](../client/crc.h.md)
**Used by** — [`sv_main.c`](sv_main.c.md) dispatches every client packet here
**Tier floor** — none

## Purpose

The server's input stage, and three times the size of the original's
([`sv_user.c`](../../WinQuake/sv_user.c.md)) because it now holds the spawn handshake, the file transfer and the chat facilities
as well as the movement. Read the original for the shared shape — running the game logic's pre- and post-think, the view roll, the
touch handlers — and this twin for what changed, which is nearly everything about *when* commands are run.

The one structural change that matters most: **movement is no longer computed here.** It is delegated to
[`pmove.c`](../client/pmove.c.md), and this file's job around it is to translate between the server's entity representation and
the movement record.

## State

As [`sv_user.c`](../../WinQuake/sv_user.c.md); the records are unchanged except where **What differs** says otherwise.

## `SV_ExecuteClientMessage`

**Contract** — parses one packet from one client: computes the round trip from the acknowledged packet, aligns the reply
sequence, then reads a sequence of commands — no operation, a delta reference request, a movement block, a text command, a
spectator teleport, an upload block — until the packet is exhausted. A malformed packet drops the client.

```text
FUNCTION execute_client_message(client)
  frame = the snapshot ring slot the client just acknowledged
  frame.ping_time = now - frame.senttime                # the round trip
  IF the client's incoming sequence has caught up to ours
    align our outgoing sequence to it
  ELSE  do not reply this frame                         # the sequences slipped
  stamp the new ring slot's send time
  localtime = server time            # so the client knows how far to extrapolate
  delta_sequence = none              # unless the packet asks for one
  LOOP
    IF the read has run off the end  drop the client ;  RETURN
    c = next command byte ;  IF exhausted  BREAK
    DISPATCH on c:
      no-operation      -- nothing
      delta reference   -- read which snapshot to delta from
      movement          -- see below; at most ONE per packet
      text command      -- hand the string to the user-command table
      spectator move    -- read a position; honoured ONLY for a spectator
      upload block      -- append to the file being received
      anything else     -- drop the client
```

**Invariants** —

- **At most one movement block per packet, enforced.** The comment says why: *someone is trying to cheat*. Two movement blocks in
  one packet would run the player's physics twice for one frame, doubling their speed. That check is the whole of the defence and
  it is the general lesson — **anything a client can repeat, it will**, so count it.
- **The spectator teleport is honoured only for spectators**, checked here and not in the game logic. Trusting the game logic to
  check would make every modification a potential hole.
- **The round trip is measured per snapshot, not per packet**, using the send time stored in the ring — which is the reason the
  ring stores a time at all.
- **The reply sequence is aligned to the client's**, and if the client is somehow ahead, the server declines to reply this frame
  rather than sending out of order.
- A parse failure **drops the client** rather than skipping the rest, because the byte stream has no resynchronization point.
- The server time is recorded per client as it processes their packet, and that value is what
  [`sv_ents.c`](sv_ents.c.md) sends as the extrapolation interval.

## The movement block

**Contract** — the movement block carries a checksum byte, a reported loss percentage, and **three consecutive commands**, each
delta-encoded against the previous. The checksum is verified over the block; a mismatch discards the rest of the packet. Then, if
packets were lost, the missing commands are made up before the newest is run.

```text
movement block:  checksum(1)  loss percent(1)
                 command delta from empty     -- the oldest
                 command delta from that      -- the middle
                 command delta from that      -- the newest

FUNCTION handle_movement(client)
  verify the checksum over the block, seeded with the packet's sequence number
  IF it fails  discard the rest of the packet
  IF not paused
    pre_run_cmd()
    IF fewer than 20 packets were lost
      WHILE more than 2 were lost   run the client's LAST known command  # fill
      IF 2 were lost  run the oldest of the three
      IF 1 was lost   run the middle of the three
    run the newest of the three
    post_run_cmd()
  remember the newest as the client's last command, with its buttons CLEARED
```

**Invariants** —

- **Three commands per packet is the loss compensation.** A packet carries the newest command and the two before it, so losing
  one or two packets costs nothing at all: the missing commands arrive in the next packet and are run then. Only from the third
  consecutive loss onward does the server have to invent motion, and it does so by repeating the last known command. That single
  decision — *send a short history, not just the present* — is why QuakeWorld tolerates packet loss that the original cannot,
  and it costs a few bytes because the commands are delta-encoded against each other.
- **Beyond twenty lost packets, nothing is made up.** Filling in hundreds of commands would let a player who suffered an outage
  reappear far away, and would cost the server unbounded time. The player simply resumes from where they were.
- **The buttons are cleared from the remembered command**, so a repeated fill command does not fire the player's weapon over and
  over during a loss. Movement persists, actions do not. That distinction is exactly right and it is easy to get wrong.
- **The checksum is seeded with the packet's sequence number** and covers the movement block, which makes a replayed or
  hand-edited movement block detectable. It is not a security measure against a determined attacker — the algorithm is in the
  client — but it catches corrupted packets and naive tampering. The recipe records it honestly as the latter.
- The client's own **reported loss percentage** is taken on trust and only displayed
  ([`sv_ccmds.c`](sv_ccmds.c.md)), never acted on.
- The impulse is **cleared on the second half of a split command** (below) so that one key press produces one weapon change.

## `SV_RunCmd`, `SV_PreRunCmd`, `SV_PostRunCmd`

**Contract** — run one command: split it if it is long, apply the command's angles and buttons to the player entity, run the game
logic's pre-think, fill the movement record from the entity, collect the nearby world, run the movement model, write the results
back to the entity, link it into the world and run the touch handlers of everything it touched; then the post-think.

```text
FUNCTION run_cmd(cmd)
  IF cmd.msec > 50
    halve it ;  run_cmd(first half) ;  run_cmd(second half with impulse cleared)
    RETURN
  unless the game logic has fixed the view, take the angles from the command
  set the player's two button fields and its impulse from the command
  IF alive and the view is not fixed
    entity pitch = -(view pitch) / 3        # the body leans a third of the look
    entity yaw   = view yaw
    entity roll  = calculated from velocity and view, scaled
  frametime = cmd.msec / 1000, clamped to 0.1
  IF not a spectator
    run the game logic's pre-think ;  run the entity's own think
  # into the movement record
  origin   = entity origin, ADJUSTED for a difference in box size
  velocity, angles, spectator flag, water-jump timer, death, buttons-held
  the world list starts with just the world model
  the movement tunables are taken from THIS PLAYER's overrides
  collect everything within 256 units into the world list
  player_move()
  # back out of the movement record
  buttons-held, water-jump timer, water level and type
  IF on ground  set the flag and record which entity, via its tag
  origin = movement origin, un-adjusting the box size difference
  velocity, view angles
  IF not a spectator
    link the entity into the world and touch triggers
    FOR EACH entity the movement touched
      IF it has a touch handler AND has not already been touched this frame
        run its touch handler with the player as the other party
        mark it touched
```

**Invariants** —

- **The command's duration drives everything, and the split threshold must match the client's**
  ([`cl_pred.c`](../client/cl_pred.c.md)). If the two differ, prediction diverges precisely on the frames where the player's
  frame rate dropped — an intermittent bug with an invisible cause.
- **The origin is adjusted between the entity and the movement record by the difference between the entity's own box and the
  movement model's fixed box.** The movement model assumes one body size ([`pmove.c`](../client/pmove.c.md)); an entity whose
  box the game logic changed — a crouched or gibbed player — must be shifted so its feet stay in the same place. Getting this
  adjustment wrong, or forgetting to undo it, sinks players into the floor.
- **Only entities within a fixed radius are collected** into the movement world list, walking the world's spatial tree. That
  bound is what keeps the list under its cap ([`pmove.h`](../client/pmove.h.md)) and is measurably one of the server's larger
  costs ([`profile.txt`](profile.txt.md)).
- **The ground entity is recovered through the movement record's opaque tag**, which the collection step set to the entity
  number. That is the whole purpose of the tag field.
- **Each touched entity's handler runs at most once per frame per player**, tracked in a bitset. Without it a player resting on a
  trigger fires it every command, several times per frame.
- **A spectator runs no game logic, does not link into the world and touches nothing.** The checks are scattered through the
  function rather than factored, which is incidental; the rule is not.
- The body's pitch is **a third of the view pitch** and the roll is computed from sideways velocity — both so that other players
  can see roughly where someone is looking and which way they are strafing. A rebuild wanting the same readability needs both.
- An **alternative collection routine that adds every entity** is present and disabled, kept for comparison. A **stuck-detection
  check** around the movement call is likewise present and disabled. Both are worth keeping in a rebuild as diagnostics.

## The spawn handshake

**Contract** — a sequence of named text commands the client issues in order, each answered with reliable data and an instruction
to issue the next: ask for a new connection, fetch the sound list, fetch the model list, fetch the level description blocks, spawn
the player entity, and begin playing.

```text
new         -> the server sends the protocol version, the client's slot,
               the level name, the game directory and the movement tunables
soundlist   -> a run of sound names, then "ask for the next run"
modellist   -> a run of model names, then "ask for the next run"
prespawn    -> one block of the level description, then "ask for the next block"
spawn       -> the light styles, every player's name, score and colours,
               the player's statistics, and the signal to begin
begin       -> the player entity is spawned by the game logic; the client is live
```

**Invariants** —

- **Each stage is client-driven and carries the level counter**, so a client whose stage arrives after a level change is
  restarted rather than spawning into the wrong map ([`server.h`](server.h.md)). Every stage checks it.
- **The lists are sent in runs, not all at once**, because they exceed a reliable message; the client asks for the next run by
  reissuing the command with an index. The same pattern as the description blocks. This is a **resumable transfer built out of
  request-response**, which needs no framing and no state on the server beyond the index the client sends — and that statelessness
  is why it survives packet loss and reordering.
- **The movement tunables are sent in the first stage.** If they did not travel, prediction could not work at all
  ([`pmove.h`](../client/pmove.h.md)).
- A spectator takes a **shortened path** with its own spawn, because it has no player entity in the game logic's sense.

## `SV_BeginDownload_f`, `SV_NextDownload_f`, `SV_NextUpload`

**Contract** — begin sending the client a file it lacks, send the next block, and receive a block of a file the client is sending.

**Invariants** —

- **A block per packet, in the reliable stream**, so a download shares the link with play rather than blocking it. A player
  missing a map is sent it while the game continues around them.
- The path is **checked for escapes** before opening, and only within the content directories. That check is the whole of the
  protection against a client asking for an arbitrary file, and a rebuild must not omit it —
  ***a client-supplied path is an attack surface***.
- Uploads exist only for the screen-capture request ([`sv_ccmds.c`](sv_ccmds.c.md)) and write to a server-chosen name, not a
  client-chosen one.

## `SV_Say`, `SV_Say_f`, `SV_Say_Team_f`

**Contract** — a player speaks: check their flood-protection history and refuse if they have exceeded the allowance, then route
the text to everyone, or to their team, or to spectators only according to the server's settings.

```text
FUNCTION say(client, team_only)
  IF the client is silenced until a time in the future  tell them ;  RETURN
  count how many of the client's last N message times fall within the window
  IF that reaches the allowance
    silence the client for the penalty period ;  tell them ;  RETURN
  record this message's time in the ring
  route the text: to the team, to spectators, or to everyone
```

**Invariants** — **the history is a small ring of timestamps and the test is a count within a window**, which is the minimum
correct rate limiter. Spectator chat is separately controllable because a spectator talking to players is a gameplay decision.

## `SV_ExecuteUserCommand` and the command table

**Contract** — matches a client's text command against a fixed table and runs it; an unmatched command is handed to the game logic
as a client command.

**Invariants** — **the table is a closed list and the fallthrough goes to the game logic**, so a game modification can add
commands without the engine knowing them, but cannot reach any engine command. That boundary is the right one and it is enforced
here rather than by convention.

## `SV_Pings_f`, `SV_Rate_f`, `SV_SetInfo_f`, `SV_Msg_f`, `SV_PTrack_f`, `SV_NoSnap_f`, `SV_Drop_f`, `SV_Kill_f`, `SV_Pause_f`, `SV_TogglePause`, `SV_ShowServerinfo_f`

**Contract** — report every player's round trip; set this client's bandwidth rate; change an entry of this client's own
information dictionary; set the message importance filter; choose which player to spectate; refuse a screen capture; disconnect;
suicide; and pause.

**Invariants** —

- **The client chooses its own bandwidth rate and the server clamps it** to a configured range
  ([`net_chan.c`](../client/net_chan.c.md)). The client knows its link and the server knows its policy; both are needed.
- **A client can refuse a screen capture**, and the refusal is a command. Recorded because the capability is privacy-relevant
  and the refusal path is the mitigation.
- Changing the information dictionary is **rate-limited for the name field specifically**, because a name changing every frame is
  broadcast to every player ([`sv_main.c`](sv_main.c.md) holds the timing fields).
- Pausing is a **game-logic decision the engine asks about**, not an engine right, so a modification can forbid it.

## `AddLinksToPmove`, `AddAllEntsToPmove`

**Contract** — walk the world's spatial tree collecting every solid entity whose bounds overlap a box around the player into the
movement world list, as a brush model or as a box; or, in the disabled alternative, add every entity unconditionally.

**Invariants** — a solid entity that is a brush model contributes **its precompiled hull and its origin**; anything else
contributes **its bounds**, which the movement code then expands by the player's size
([`pmovetst.c`](../client/pmovetst.c.md)). Players are added too, so players collide with each other in both the authoritative
and the predicted move.

The list is **capped and the excess silently dropped**, which in a crowded space means a player can briefly pass through
something. Recorded as a real limitation of the fixed-size design.

## `V_CalcRoll`, `SV_UserInit`, `OutofBandPrintf`

**Contract** — compute the view roll from sideways velocity; register the settings; and print to an address with no channel.

**Notes** — the single most valuable idea in this file for a rebuilder is the **three-commands-per-packet** loss compensation, and
the second is the **client-driven staged handshake**. Both are cheap, both remove whole classes of failure, and neither requires
anything from the transport.
