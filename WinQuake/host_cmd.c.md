# WinQuake/host_cmd.c

> Thirty-four console commands: the level transitions, the save format, the three-stage connection handshake, chat, and the cheats — each of which decides what it will do based on whether a local user or a remote player asked.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`server.h`](server.h.md) · [`client.h`](client.h.md) · [`progs.h`](progs.h.md) · [`cmd.h`](cmd.h.md) · [`cvar.h`](cvar.h.md) · [`net.h`](net.h.md) · [`protocol.h`](protocol.h.md) · [`model.h`](model.h.md) · [`keys.h`](keys.h.md) · [`screen.h`](screen.h.md) · [`menu.h`](menu.h.md) · [`zone.h`](zone.h.md) · [`pr_exec.c`](pr_exec.c.md) · [`pr_edict.c`](pr_edict.c.md) · [`sv_main.c`](sv_main.c.md) · [`host.c`](host.c.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`host.c`](host.c.md) registers everything here; [`sv_user.c`](sv_user.c.md)'s authorization list names nineteen of these commands
**Tier floor** — none

## Purpose

The engine's whole user-facing command surface. Its structure is a pattern applied thirty-four
times: **look at where the command came from, and act differently**. A command typed locally on a
client is forwarded to the server; the same command arriving from a player is executed; a command
typed on a dedicated server's console acts as the server. That one branch, at the top of nearly
every function here, is the engine's authorization model in practice
([`cmd.h`](cmd.h.md)).

Three parts deserve close reading. **The connection handshake** is three commands the client sends
in sequence, and the server's three responses are what put a player into a world. **The save
format** is name-keyed text and the twin states it field by field, because it is one of the
conformance inputs ([`SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md#6-conformance)). And
**the level transitions** differ in ways that are not obvious from their names.

## State

```text
VARIABLE current_skill     : int    # the skill the loaded level actually uses
VARIABLE noclip_anglehack  : bool   # see the note under the noclip command
CONSTANT savegame_version  = 5
```

## The source dispatch

Almost every command below begins with one of these three shapes:

```text
# Shape A — "this belongs to the server"
IF the command came from the console
  forward it to the server ;  RETURN
# ... now running on the server, on behalf of a player

# Shape B — "this is console-only"
IF the command did NOT come from the console  RETURN

# Shape C — "the server may do this itself"
IF the command came from the console
  IF no server is running  forward it to the server ;  RETURN
  act as the server
```

**Invariants** — Shape A is how a cheat typed on a client reaches the server that must apply it.
The client forwards the line verbatim ([`cmd.c`](cmd.c.md#cmd_forwardtoserver)), the server's
authorization list admits it ([`sv_user.c`](sv_user.c.md#the-authorization-list)), and the same
function runs again with a different source. A rebuild must keep the double dispatch or must
invent an equivalent.

## Process and status

### `Host_Quit_f` — `quit`

**Contract** — on a client with the console closed, opens the confirmation menu instead. Otherwise
disconnects, shuts down the server, and exits the process.

**Invariants** — the menu diversion is why quitting from the game asks for confirmation and
quitting from the console does not.

### `Host_Version_f` — `version`

**Contract** — prints the version and build timestamp.

### `Host_Status_f` — `status`

**Contract** — prints the host name, version, available network addresses, current level, player
count, and one line per player giving their slot, name, score and connected duration, plus their
address. Output goes to the local console when typed locally and to the asking player otherwise.
Forwards to the server when typed on a client with no local server.

**Invariants** — the output function is selected into a variable and then called, which is how one
body serves two destinations. The duration is computed from the connection's own timestamp and the
network layer's clock, not from the server clock, so it survives level changes.

### `Host_Ping_f` — `ping`

**Contract** — prints every player's average round-trip time, computed as the mean of their sixteen
most recent samples, to the asking player.

**Invariants** — the average includes uninitialized samples for a freshly connected player, so
early readings are wrong. The samples come from the client echoing a server timestamp
([`sv_user.c`](sv_user.c.md#sv_readclientmove)).

## Cheats

### `Host_God_f` — `god`, `Host_Notarget_f` — `notarget`, `Host_Fly_f` — `fly`, `Host_Noclip_f` — `noclip`

**Contract** — each toggles one property of the asking player and tells them the new state: damage
immunity, invisibility to monsters, gravity-free movement, and passing through geometry. Each
forwards from the console. Each **refuses silently in deathmatch** unless the player is privileged.

```text
FUNCTION host_god_f()
  IF from the console  forward to the server ;  RETURN
  IF deathmatch AND NOT privileged  RETURN              # silently
  toggle the player's godmode flag
  tell the player whether it is now on or off
```

**Invariants** — the deathmatch refusal is **silent**, giving no indication that the command was
rejected. Deliberate: telling a player that cheats are disabled invites them to keep trying.

The flying and noclipping commands *replace* the movement type rather than toggling a flag, and
restore it to walking when turned off — so a cheat used on a player whose movement type was
something else leaves them walking.

**Notes** — `noclip_anglehack` is set when noclipping begins and cleared when it ends. Its only
consumer is [`cl_input.c`](cl_input.c.md), which uses it to make the up/down movement keys control
vertical motion rather than swimming. A flag crossing from the server's command handler to the
client's input code, in one process — a layering violation that exists because both halves live
together.

### `Host_Give_f` — `give`

**Contract** — grants the asking player a weapon, ammunition or health. Forwards from the console.
Refuses silently in deathmatch unless privileged. The first character of the argument selects what
to grant: a digit selects a weapon by position, and letters select ammunition types and health.
The mapping differs per mission pack.

```text
FUNCTION host_give_f()
  IF from the console  forward ;  RETURN
  IF deathmatch AND NOT privileged  RETURN
  t = argument 1 ;  v = argument 2 as an integer
  SELECT the first character OF t
    a digit:
      IF the second mission pack is loaded
        '6' with a following 'a' grants its proximity gun, else the grenade
                                 launcher
        '9' grants its laser cannon ;  '0' grants its hammer
        otherwise a digit from '2' up grants the weapon at that offset
      ELSE
        a digit from '2' up grants the weapon at that offset
    's' shells ;  'n' nails ;  'r' rockets ;  'c' cells ;  'h' health
      ...and, when the FIRST mission pack is loaded, each also writes a
      SECOND ammunition field looked up BY NAME at run time, and mirrors
      the value into the standard field only when the player's current
      weapon is on the matching side of a threshold
    'l' lava nails ;  'm' multi-rockets ;  'p' plasma
      ...which exist ONLY in the first mission pack, through the same
      run-time field lookup
```

**Invariants** — this is where the three game variants' divergence is most visible in the engine.
The first mission pack gives every ammunition type a second variant and the player carries both;
which one the standard field mirrors depends on which weapon they hold. The engine reaches those
extra fields **by name at run time**
([`pr_edict.c`](pr_edict.c.md#getedictfieldvalue)), because they are not in the compiled-in layout.

Digits below '2' grant nothing in the base game, because weapon bit positions start there.

A rebuild targeting only the original game implements the digit case and the five letters, and
should still route the extra fields through a named lookup so that mission-pack game logic works.

## Level transitions

### `Host_Map_f` — `map`

**Contract** — starts a fresh game on a named level. Console only. Kicks every player off, closes
the console and menu, shows the loading screen, resets the episode flags, spawns the server, and —
on a non-dedicated process — connects the local client to it.

```text
FUNCTION host_map_f()
  IF not from the console  RETURN
  cancel any demo sequence
  disconnect ;  host_shutdown_server(crash = false)
  give the keyboard to the game ;  begin the loading plaque
  remember the whole command line, so a restart can repeat it
  svs.serverflags = 0                    # a NEW game: no episodes completed
  sv_spawn_server(argument 1)
  IF the spawn failed  RETURN
  IF not dedicated
    remember arguments 2 onward as the spawn parameters
    execute "connect local"
```

**Invariants** — **the episode flags are cleared**, which is the difference between this and a
level change: starting a level by name is starting a new game, so collected sigils are forgotten.

The whole command line is remembered because a restart must repeat it including any arguments.

The local client connects through the *loopback* transport by the literal name `local`
([`net.h`](net.h.md)), which is how single player is a client-server session.

### `Host_Changelevel_f` — `changelevel`

**Contract** — moves to a new level keeping every player and their inventory. Refuses if no server
is running or a demo is playing. Saves each player's carry values, then spawns the new level.

**Invariants** — the difference from starting a level is entirely these two things: the episode
flags survive, and the carry values are captured first
([`sv_main.c`](sv_main.c.md#sv_savespawnparms)). That is the whole of level-to-level continuity.

### `Host_Restart_f` — `restart`

**Contract** — respawns the current level. Console only; refuses during demo playback or with no
server. Copies the level name out first, because spawning clears it.

**Invariants** — the carry values are **not** saved, so a restart loses the player's inventory —
which is correct, because a restart is what happens when the player dies.

### `Host_Reconnect_f` — `reconnect`

**Contract** — makes the client await the handshake again. Shows the loading screen and resets the
handshake stage to zero.

**Invariants** — this is what the server stuffs to every client on a level change
([`sv_main.c`](sv_main.c.md#sv_sendreconnect)). It does not touch the connection; only the
handshake state. That is why a level change does not drop anyone.

### `Host_Connect_f` — `connect`

**Contract** — connects to a named host. Cancels any demo sequence, stops any playback, establishes
the connection, and resets the handshake.

## The connection handshake

Three commands, sent by the client in sequence, each answered by the server appending to that
client's reliable stream and advancing a stage number.

### `Host_PreSpawn_f` — `prespawn`

**Contract** — sends the client the whole signon stream — every baseline, every static entity,
every ambient sound — and advances to stage two. Refuses from the console and refuses if the client
has already spawned.

### `Host_Spawn_f` — `spawn`

**Contract** — puts the player into the world. On a restored game, merely unpauses. Otherwise clears
their entity, sets their colours, team and name, loads their carry values into the globals, and runs
the game's connect and spawn callbacks. Then, in all cases, discards whatever was buffered and
sends: the server time; every player's name, score and colours; every light style; four statistic
values; an absolute view angle with the roll forced to zero; the player's own state; and stage
three. Refuses from the console and refuses if already spawned.

```text
FUNCTION host_spawn_f()
  IF from the console  print "not valid from the console" ;  RETURN
  IF already spawned   print "already spawned" ;  RETURN

  IF restoring a saved game
    sv.paused = false               # the last client to connect unpauses
  ELSE
    ent = the client's entity
    zero its interpreter fields
    ent.colormap = its own entity number
    ent.team     = (the client's colours BITAND 15) + 1
    ent.netname  = the client's name
    copy the client's sixteen carry values INTO the globals
    the global time = sv.time ;  the global self = ent
    run the game's ClientConnect
    IF the connection began no later than the current server time
      print "<name> entered the game"          # suppresses it during a load
    run the game's PutClientInServer

  DISCARD the client's buffered reliable data
  write the time
  FOR EACH player slot: its name, its REMEMBERED score, its colours
  FOR EACH light style: its animation string
  write four statistics: total secrets, total monsters, found secrets,
        killed monsters
  # An absolute view angle, with roll forced to zero — see the note.
  write the set-angle tag, the player's pitch and yaw, and a roll of 0
  sv_write_clientdata_to_message(the player, their stream)
  write the signon stage 3
```

**Invariants** — several decisions here are load-bearing.

**The buffered reliable data is discarded**, not appended to. Anything queued for this client
between the previous stage and now is lost, which is safe because the block that follows is a
complete snapshot.

**The roll is explicitly sent as zero**, and the comment explains why at length: a saved game can
capture the server expecting the client to correct a roll angle, and after a load that correction
never comes — leaving the player with a permanent head tilt. Forcing zero removes the possibility.

**The scores sent are the remembered ones**, not the live ones, so that the change detection in
[`sv_main.c`](sv_main.c.md#sv_updatetoreliablemessages) still fires for anything that has changed
since.

**A restored game skips the spawn callbacks entirely**, because the entity was already restored from
the save file with its full state. Running them would reset the player's inventory.

**The "entered the game" message is suppressed during a load** by comparing the connection's
wall-clock age against the server's simulation clock — a connection older than the simulation is
one that predates this level.

The team is derived from the lower four bits of the player's colour, which is how team play is
configured by trouser colour.

### `Host_Begin_f` — `begin`

**Contract** — marks the client as spawned, so that it begins receiving frame datagrams. Refuses
from the console.

**Invariants** — the handshake is therefore: the client asks for the signon block, then asks to be
spawned, then declares itself ready. The server sends nothing unreliable until the third step,
which is what keeps a loading client from receiving frames it cannot interpret.

## Save and restore

### `Host_SavegameComment`

**Contract** — builds a fixed 39-character description: the level's title, then at offset 22 a
kill count, with every space replaced by an underscore and a terminator appended.

**Invariants** — the space substitution is so that the whole comment reads back as a single
whitespace-delimited token. The menu displays it with the substitution reversed.

### `Host_Savegame_f` — `save`

**Contract** — writes the current single-player game to a named file. Console only. Refuses when no
local server is running, during the end-of-level screen, in a multiplayer game, with the wrong
argument count, with a path containing a parent-directory reference, or with any dead player.
Writes the version, the comment, the sixteen carry values, the skill, the level name, the server
clock, the 64 light-style strings, the flagged globals, and every entity.

```text
FUNCTION host_savegame_f()
  IF not from the console                RETURN
  IF no server is running                print "Not playing a local game" ;  RETURN
  IF at the intermission                 print "Can't save in intermission" ;  RETURN
  IF more than one player slot exists    print "Can't save multiplayer games" ;  RETURN
  IF the argument count is not 2          print usage ;  RETURN
  IF the name contains ".."               print "Relative pathnames are not allowed"
                                          RETURN
  IF any active player has health <= 0    print "Can't savegame with a dead player"
                                          RETURN

  path = the game directory + "/" + argument 1, defaulting the extension to ".sav"
  open it for writing ;  print an error and RETURN on failure
  write the version (5)
  write the comment
  write the sixteen carry values, one per line, as floats
  write the skill as an integer
  write the level name
  write the server clock as a float
  FOR EACH of the 64 light styles  write its string, OR the single letter "m"
  write the flagged globals as a brace-delimited block
  FOR EACH entity  write it as a brace-delimited block, FLUSHING after each
```

**Invariants** — the parent-directory check is the engine's only path-traversal defence for a
user-supplied filename, and it is a substring test, so it also rejects an innocent name containing
two dots.

The **letter `m` stands for a fully-lit light style**; every style slot must be written so that the
restore's fixed-count read stays aligned.

**Every entity is written, free ones included** ([`pr_edict.c`](pr_edict.c.md#ed_write) writes an
empty block for a free slot), which is what keeps entity numbering stable across the round trip.

Flushing after each entity is a 1996 crash-tolerance measure.

Refusing to save with a dead player is because the restore would put the player into a corpse with
no way to respawn.

### `Host_Loadgame_f` — `load`

**Contract** — restores a saved game. Console only. Reads and checks the version, reads the comment,
the carry values, the skill, the level name and the clock; disconnects; spawns that level; then
pauses the server, marks it as a restored game, reads the light styles, and reads the globals and
every entity back. Finally reconnects the local client.

```text
FUNCTION host_loadgame_f()
  IF not from the console  RETURN
  IF the argument count is not 2  print usage ;  RETURN
  cancel any demo sequence
  open the file ;  print an error and RETURN on failure
  read the version ;  IF it is not 5  print the mismatch and RETURN
  read and discard the comment
  read the sixteen carry values
  read the skill AS A FLOAT and round it            # 1.06 files stored floats
  write it back to the skill variable
  read the level name and the clock
  disconnect
  sv_spawn_server(that level)
  IF the spawn failed  print "Couldn't load map" ;  RETURN
  sv.paused = true                                  # until every client connects
  sv.loadgame = true                                # changes how spawning works
  FOR EACH of the 64 light styles  read a token and hunk-copy it

  # Then a sequence of brace-delimited blocks: the FIRST is the globals,
  # each subsequent one is entity 0, 1, 2, ...
  entnum = -1
  WHILE not at end of file
    accumulate characters INTO a 32768-byte buffer until a '}' or the end
    IF the buffer filled  FAIL WITH "Loadgame buffer overflow"
    parse the first token ;  IF empty  BREAK
    IF it is not "{"  FAIL WITH "First token isn't a brace"
    IF entnum == -1  parse the globals
    ELSE
      ent = entity entnum
      zero its interpreter fields ;  mark it in use
      parse the block into it
      IF it is still in use  sv_link_edict(ent, touch_triggers = false)
    entnum = entnum + 1
  sv.num_edicts = entnum
  sv.time = the clock read earlier
  copy the carry values into the single player's slot
  IF not dedicated  establish a local connection and reset the handshake
```

**Invariants** — the level is **spawned first and then overwritten**. So the restore runs every
entity's spawn function, builds the collision index and the precache lists, and then replaces the
entity array's contents with the saved state. That is why the save format does not need to record
anything about models, sounds or geometry.

**The restored-game flag changes two later behaviours**: a connecting client keeps its carry values
rather than asking the game for new ones ([`sv_main.c`](sv_main.c.md#sv_connectclient)), and
spawning skips the game's connect callbacks
([`Host_Spawn_f`](#host_spawn_f-spawn)). Without it a restore resets the player.

**The server is paused until the client connects**, and unpaused by the spawn command. That is
what stops the world advancing while the restore completes.

Entities are relinked **without firing triggers**, so a restored player standing in a trigger does
not activate it on load.

The skill is read as a float and rounded because an earlier version's files stored it that way —
a compatibility shim the source names explicitly.

The loading screen is **not** shown, and the comment explains: too much stack has been used by the
time this runs, so the menu shows it before stuffing the command instead. A rebuild without a 1996
stack budget can simply show it.

**Notes** — the block accumulator is a 32-kilobyte stack buffer and overflow is fatal. An entity
with very long string fields overflows it.

A build switch adds a second pair of save and restore functions that persist *one level's* state
into a side file, so that returning to a level within an episode restores it. That is the sequel's
hub-level model and is not compiled here.

## Chat and identity

### `Host_Name_f` — `name`

**Contract** — sets the asking player's name, truncated to fifteen characters. Printed when given no
argument. From the console it sets the local variable and forwards; from a player it renames them,
announces the change, and tells every client.

**Invariants** — the truncation writes a terminator **into the argument buffer**, mutating the
tokenizer's storage. Harmless because the buffer is rebuilt each line, but a rebuild should copy.

Using the raw remainder rather than a single token when there is more than one argument is what
allows a name with spaces.

### `Host_Say`, `Host_Say_f` — `say`, `Host_Say_Team_f` — `say_team`

**Contract** — broadcasts a message to every spawned player, or to the sender's team only. From a
client's console, forwards; from a dedicated server's console, sends as the server. Strips
surrounding quotes, prefixes a highlight character and the sender's name, truncates to fit
64 bytes, and appends a newline. Also prints locally.

```text
FUNCTION host_say(team_only)
  IF from the console
    IF dedicated  send as the server, ignoring team_only
    ELSE          forward to the server ;  RETURN
  IF fewer than 2 arguments  RETURN
  p = the raw remainder
  IF it begins with a quote  strip the leading and trailing quote
  prefix = a highlight byte THEN "<sender>: ", or "<hostname> " from the server
  truncate p so that prefix + p + newline fits in 64 bytes
  FOR EACH spawned client
    IF team play AND team_only AND their team differs FROM the sender's  CONTINUE
    print the text TO that client
  print the text locally, WITHOUT the highlight byte
```

**Invariants** — the leading byte is a control character that selects the alternate glyph set, so a
chat line renders highlighted ([`SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)).
It is skipped for the local print because a terminal cannot show it.

The **quote stripping** is because the client's key binding for chat quotes the text. It strips the
last character unconditionally when the first is a quote, so an unterminated quote eats a character.

The raw remainder is used rather than the tokens, preserving the sender's spacing.

Team filtering applies only when team play is enabled, so `say_team` in a free-for-all reaches
everybody.

### `Host_Tell_f` — `tell`

**Contract** — sends a message to one named player. Forwards from the console. Requires at least
three arguments. Matching is case-insensitive on the whole name.

**Invariants** — the message text is taken from the raw remainder, which **still includes the
recipient's name**, so the recipient sees their own name at the front of the message. A real defect,
present in the shipped game.

### `Host_Color_f` — `color`

**Contract** — sets the asking player's shirt and trouser colours, each 0 to 13. One argument sets
both. From the console, sets the local variable and forwards; from a player, records the colours,
derives their team from the trousers, and tells every client.

**Invariants** — each colour is masked to four bits and then clamped to 13, because the top two
palette rows are not player colours. The two are packed into one byte as shirt in the high nibble
and trousers in the low.

The **team is the trouser colour plus one**, so team assignment is by clothing. That is the whole
team mechanism.

## Game actions

### `Host_Kill_f` — `kill`

**Contract** — asks the game to kill the asking player. Forwards from the console. Refuses if
already dead.

### `Host_Pause_f` — `pause`

**Contract** — toggles the pause. Forwards from the console. Refuses when pausing is disabled.
Announces who paused and tells every client.

### `Host_Kick_f` — `kick`

**Contract** — removes a player by name, or by slot number when the first argument is a hash.
Refuses in deathmatch from an unprivileged player. Refuses to kick the caller. An optional trailing
message is passed to the kicked player.

```text
FUNCTION host_kick_f()
  IF from the console AND no server is running  forward ;  RETURN
  IF from a player AND deathmatch AND NOT privileged  RETURN
  saved = the current client
  IF argument 1 is "#"  select the slot numbered by argument 2
  ELSE                  find the slot whose name matches argument 1
  IF found
    who = "Console" on a dedicated server, the local player's name on a client,
          or the asking player's name
    IF the target IS the caller  RETURN                # cannot kick yourself
    IF there are further arguments
      extract the message from the raw remainder, skipping the hash and the
      number when the by-number form was used, then any leading spaces
    tell the target they were kicked, with the reason if any
    sv_drop_client(crash = false)
  restore the current client
```

**Invariants** — the current-client global is saved and restored around the operation, because the
message functions read it ([`host.c`](host.c.md#sv_clientprintf-sv_broadcastprintf-host_clientcommands)).

The message extraction skips the hash and the number by *string arithmetic over the raw
remainder*, which is fragile: extra spaces between the hash and the number shift the result.

## Debugging

### `FindViewthing`, `Host_Viewmodel_f`, `Host_Viewframe_f`, `Host_Viewnext_f`, `Host_Viewprev_f`

**Contract** — find the map entity of a specific class used as a model viewer, and load a model
into it or step its animation frame, printing each frame's name. Report when the map has no such
entity.

**Notes** — an asset inspection tool that works by placing a specially-named entity in a map.
Worth keeping in a rebuild: it is the only way to look at a model in isolation.

The model is written into the **client's** precache array rather than the server's, so the change
is purely visual and does not affect collision.

### `Host_Please_f` — `please`

**Contract** — toggles a player's privilege, by name or slot; revoking it also cancels their
cheats. Present only behind a build switch, not compiled here.

**Notes** — the counterpart to the privilege flag whose consequences
[`sv_user.c`](sv_user.c.md#the-authorization-list) documents. A rebuild should not port either.

## Demo sequence

### `Host_Startdemos_f` — `startdemos`, `Host_Demos_f` — `demos`, `Host_Stopdemo_f` — `stopdemo`

**Contract** — record a list of demo names to cycle through, resume cycling, and stop playback. On a
dedicated server, the first instead starts a level if none is running.

**Invariants** — the demo list is how the game's attract mode works: the startup script calls it,
and the cycle runs until the player does something. A dedicated server has no attract mode, so the
same command starts a real level — which is why a dedicated server with no map argument still comes
up playable.

## `Host_InitCommands`

**Contract** — registers all thirty-four commands, plus the model-cache dump from
[`model.c`](model.c.md).

**Invariants** — the names registered here are the ones
[`sv_user.c`](sv_user.c.md#the-authorization-list) admits from remote players. The two lists must
be read together: a command registered here and not in that list is console-only in practice, and
one in that list without a handler here is refused by the dispatcher.
