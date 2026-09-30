# WinQuake/host.c

> The frame loop and the process lifecycle: what happens in what order every frame, how time is filtered before it reaches the simulation, and the two-level error unwind that lets a bad level drop you to the console instead of killing the program.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`r_local.h`](r_local.h.md) · [`server.h`](server.h.md) · [`client.h`](client.h.md) · [`zone.h`](zone.h.md) · [`cmd.h`](cmd.h.md) · [`cvar.h`](cvar.h.md) · [`net.h`](net.h.md) · [`sound.h`](sound.h.md) · [`cdaudio.h`](cdaudio.h.md) · [`input.h`](input.h.md) · [`screen.h`](screen.h.md) · [`keys.h`](keys.h.md) · [`console.h`](console.h.md) · [`menu.h`](menu.h.md) · [`view.h`](view.h.md) · [`wad.h`](wad.h.md) · [`model.h`](model.h.md) · [`sbar.h`](sbar.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface)
**Used by** — every platform backend's entry point ([`sys_win.c`](sys_win.c.md), [`sys_linux.c`](sys_linux.c.md), [`sys_dos.c`](sys_dos.c.md), …) calls exactly three functions here; [`host_cmd.c`](host_cmd.c.md) is its other half; the error functions are called from everywhere
**Tier floor** — none for the loop; the error unwind needs a non-local jump or an equivalent

## Purpose

This file is the program. A platform backend does three things — set up a memory block and paths,
call the initializer, then call the frame function forever — and everything else is here or reached
from here.

Three things are worth a rebuilder's attention.

**The frame order** is a fixed sequence of thirteen steps, and several adjacent pairs cannot be
swapped. Most notably, a locally-hosted game sends the player's intentions *before* the server
runs and a remotely-hosted one sends them *after*, which is a one-frame latency decision made in
two lines.

**Time filtering** sits between the platform's clock and the simulation. It enforces a maximum
frame rate, clamps the step to between one millisecond and a tenth of a second, and can replace it
with a fixed value for slow motion. The clamp is what keeps a stalled machine from tunnelling
players through walls.

**The two error levels** — one that ends the game and one that ends the level — both unwind by
non-local jump from arbitrary depth, including from inside the bytecode interpreter. That unwind
is why an error in game logic drops you to the console with the server still alive.

## State

```text
VARIABLE host_parms       : QuakeParms   # what the platform handed us
VARIABLE host_initialized : bool         # command registration is now closed
VARIABLE host_frametime   : real         # the FILTERED step the simulation uses
VARIABLE host_time        : real         # accumulated filtered time
VARIABLE realtime         : real         # accumulated RAW time, unbounded
VARIABLE oldrealtime      : real         # when the last frame ran
VARIABLE host_framecount  : int
VARIABLE host_hunklevel   : int          # the mark a level change resets to
VARIABLE host_abortserver : jump target
VARIABLE host_client      : Client       # whose turn it is
VARIABLE host_basepal     : bytes        # 256 RGB triples
VARIABLE host_colormap    : bytes        # 256 x 64 shading table
VARIABLE minimum_memory   : int

VARIABLE host_framerate : Cvar = 0      # non-zero forces a fixed step
VARIABLE host_speeds    : Cvar = 0      # print per-frame timings
VARIABLE sys_ticrate    : Cvar = 0.05   # a dedicated server's sleep granularity
VARIABLE serverprofile  : Cvar = 0
VARIABLE fraglimit, timelimit, teamplay : Cvar   # server-announced
VARIABLE samelevel, noexit, skill, developer, deathmatch, coop, pausable : Cvar
VARIABLE temp1 : Cvar                   # a scratch value for game logic
```

**Invariants** — `realtime` is raw and monotonically accumulating; `host_frametime` is filtered
and clamped. Physics uses the latter, timeouts the former, and confusing them is the classic bug
in this shape of loop.

The hunk mark is taken once at the end of initialization, and every level change resets to it
([`Host_ClearMemory`](#host_clearmemory)). That single integer is the level-lifetime boundary.

## Error handling

### `Host_EndGame`

**Contract** — ends the current game, not the process: shuts down a running server, then either
advances the demo sequence or disconnects, then unwinds to the top of the frame loop. On a
dedicated server it is fatal instead, because there is nothing to drop back to.

```text
FUNCTION host_end_game(message...)
  print the message at developer verbosity
  IF a server is running  host_shutdown_server(crash = false)
  IF this is a dedicated server  FAIL FATALLY WITH the message
  IF a demo sequence is in progress  advance to the next demo
  ELSE                               disconnect
  JUMP to the top of the frame loop
```

**Invariants** — this is the *graceful* end: the game finished, the demo ran out, the server said
goodbye. It is not an error.

### `Host_Error`

**Contract** — reports a failure, shuts down the server and the client, and unwinds to the top of
the frame loop. Re-entry is fatal. On a dedicated server it is fatal. Cancels the loading screen
first, so the message is visible.

```text
FUNCTION host_error(message...)
  IF already inside host_error  FAIL FATALLY WITH "recursively entered"
  mark that we are inside
  end the loading plaque                       # re-enable screen updates
  print the message
  IF a server is running  host_shutdown_server(crash = false)
  IF this is a dedicated server  FAIL FATALLY WITH the message
  disconnect ;  cancel any demo sequence
  clear the inside mark
  JUMP to the top of the frame loop
```

**Invariants** — the **recursion guard** is essential: the shutdown path itself can fail, and
without the guard a failure inside error handling loops until the stack is gone. It is cleared
before the jump, so a later unrelated error still works.

The loading-screen cancellation comes *first*, because the screen is suppressed during a load and
the message would otherwise be invisible.

There are three failure levels in this engine and they form a hierarchy: this one loses the level,
[`Host_EndGame`](#host_endgame) loses the game, and the platform's fatal error loses the process.
The bytecode interpreter's own error ([`pr_exec.c`](pr_exec.c.md#pr_runerror)) routes here, which
is why buggy game logic does not kill the program.

## Setup

### `Host_FindMaxClients`

**Contract** — decides the player limit and the process's role from the command line. A dedicated
switch makes this a dedicated server; a listen switch makes it a listen server; neither makes it a
pure client. Either switch takes an optional count, defaulting to eight. The limit is clamped to
between one and sixteen. Allocates the player slots, never fewer than four. Sets the deathmatch
variable from whether the limit exceeds one. Both switches together is a fatal error.

```text
FUNCTION host_find_max_clients()
  svs.maxclients = 1
  IF "-dedicated" IS present
    this process is dedicated
    svs.maxclients = the following value, ELSE 8
  ELSE
    this process starts disconnected
  IF "-listen" IS present
    IF already dedicated  FAIL WITH "Only one of -dedicated or -listen"
    svs.maxclients = the following value, ELSE 8
  clamp svs.maxclients INTO 1..16
  svs.maxclientslimit = max(svs.maxclients, 4)
  allocate the player slots on the hunk
  set the deathmatch variable TO 1 IF svs.maxclients > 1 ELSE 0
```

**Invariants** — the allocated count has a **floor of four** while the active count can be one, so
the slots can be raised to four later without reallocating. The array is hunk memory taken before
the level mark, so it cannot be grown at all
([`server.h`](server.h.md)).

Deathmatch is turned on merely by allowing more than one player, which is why a listen server
defaults to deathmatch rules and why the level loader then forces cooperative off if both are set
([`sv_main.c`](sv_main.c.md#sv_spawnserver)).

### `Host_InitLocal`

**Contract** — registers the console commands from [`host_cmd.c`](host_cmd.c.md) and the fifteen
variables above, decides the player limit, and sets the accumulated time to 1.0.

**Invariants** — the accumulated time starts at 1.0 for the same reason the server clock does: so
that a schedule of zero means "never" rather than "now"
([`sv_main.c`](sv_main.c.md#sv_spawnserver)).

### `Host_WriteConfiguration`

**Contract** — writes the key bindings and every archive-flagged variable into the game directory's
configuration file. Does nothing before initialization completes or on a dedicated server.

**Invariants** — written on shutdown and read by the startup script, so the round trip is the
configuration's whole persistence mechanism. The file is console script and replays through the
normal dispatcher ([`cvar.c`](cvar.c.md#cvar_writevariables)).

**Notes** — the guard uses a bitwise rather than a logical conjunction on two boolean values; the
values are 0 or 1 so it happens to work.

### `Host_InitVCR`

**Contract** — sets up the session recorder: with a record switch, writes a file holding the
command line (with the record switch rewritten to a playback switch); with a playback switch,
reads the command line back out of the file and replaces the process's own. A playback switch with
any other argument is a fatal error, as is a missing or badly signed file.

**Invariants** — the recorder captures the command line *and* — through
[`net_vcr.c`](net_vcr.c.md) — every network message, so a networked session can be replayed
exactly. The command line has to be captured because the session's behaviour depends on it, and
the record switch is rewritten so that replaying is a single step.

**Notes** — it allocates with the language's own allocator rather than from the engine's block,
because it runs before the memory system is fully set up. The only such allocation in the engine.

A rebuild interested in deterministic replay for testing should keep this; it is the closest
thing the engine has to a test harness.

### `Host_Init`

**Contract** — brings the process up. Checks the memory budget, initializes each subsystem in a
fixed order, loads the palette and shading table, brings up the output devices unless dedicated,
queues the startup script, and takes the hunk mark that level changes reset to. Too little memory
is a fatal error.

```text
FUNCTION host_init(parms)
  minimum_memory = 5.4 MB, OR 6.4 MB when a mission pack is selected
  IF "-minmemory"  parms.memsize = minimum_memory
  host_parms = parms
  IF parms.memsize < minimum_memory  FAIL WITH "Only <n> megs available"

  memory_init(parms.membase, parms.memsize)
  cbuf_init() ;  cmd_init()
  v_init() ;  chase_init()
  host_init_vcr(parms)
  com_init(parms.basedir)               # filesystem and byte order
  host_init_local()                     # variables, commands, player limit
  w_load_wad_file("gfx.wad")            # interface graphics
  key_init() ;  con_init() ;  m_init()
  pr_init() ;  mod_init() ;  net_init() ;  sv_init()
  print the build time and the heap size
  r_init_textures()                     # needed EVEN on a dedicated server

  IF NOT dedicated
    host_basepal   = load "gfx/palette.lmp"   ;  fatal if missing
    host_colormap  = load "gfx/colormap.lmp"  ;  fatal if missing
    # Mouse before video on some platforms, after on others — see the note.
    in_init() ;  vid_init(host_basepal)
    draw_init() ;  scr_init() ;  r_init()
    s_init()
    cdaudio_init() ;  sbar_init() ;  cl_init()

  queue "exec quake.rc"
  hunk_alloc_named(0, "-HOST_HUNKLEVEL-")     # a zero-size marker
  host_hunklevel = the hunk's low mark
  host_initialized = true
```

**Invariants** — the order has five hard constraints.

**Memory first**, because everything else allocates from it.

**The command buffer and dispatcher before anything that registers a command**, which is nearly
everything.

**The filesystem before anything that loads a file.** The palette, the interface graphics and the
game logic all come through it.

**The texture registry even on a dedicated server**, because the map loader builds texture records
whether or not anything will draw them.

**Command registration closes when initialization completes.** Registrations are hunk-allocated
before the level mark, so a later one would be freed by a level change —
[`cmd.c`](cmd.c.md#cmd_addcommand) makes that a fatal error.

The **zero-size hunk allocation** is a marker whose only purpose is to make the memory dump
readable: it names the boundary between permanent and per-level data.

The startup script is **queued, not executed**, so it runs on the first frame — after
initialization has completed and `host_initialized` is set. That is what lets the script contain
commands that are illegal during initialization.

**Notes** — the mouse-versus-video order differs per platform and the source gives the reason as
security: on some systems the mouse must be grabbed before the display is taken over, or a
privileged process could be left with an ungrabbed input device. Sound has a symmetric ordering
constraint on one platform so that a "device in use" dialogue can be shown before the display is
seized. Both are real platform constraints, not cleanup opportunities.

### `Host_Shutdown`

**Contract** — writes the configuration and releases the output devices, in reverse order of
initialization. Guards against re-entry with a printed message. Suppresses screen updates first.

**Notes** — the source's own comment complains that this is a callback from both the quit path and
the fatal-error path, and would be better placed. A rebuild should call it explicitly from one
place.

## Per-player messages

### `SV_ClientPrintf`, `SV_BroadcastPrintf`, `Host_ClientCommands`

**Contract** — append a formatted print to the current client's reliable stream, to every
spawned client's, or a formatted *console command* to the current client's. Each formats into a
1024-byte buffer with no bound check.

**Invariants** — these live here rather than in [`sv_main.c`](sv_main.c.md) because they are
called from the host's command handlers. The third one is the mechanism behind the host function
that stuffs a command to a client ([`pr_cmds.c`](pr_cmds.c.md#pf_stuffcmd-stuffcmdentity-string-21)), and it is the
trust boundary discussed in [`protocol.h`](protocol.h.md).

The broadcast reaches only **spawned** clients, so a player still loading misses it.

### `SV_DropClient`

**Contract** — removes a player. Unless this is a crash, sends a final disconnect, runs the game's
disconnect callback, and logs it. Then closes the connection, frees the slot while leaving the
entity in place, and tells every remaining client to clear that player's name, score and colours.

```text
FUNCTION sv_drop_client(crash)
  IF NOT crash
    IF the reliable channel will accept a message
      append a disconnect ;  send the reliable buffer
    IF the client has an entity AND is spawned
      save the global self ;  set it to their entity
      run the game's ClientDisconnect              # sets the body to a corpse
      restore the global self
    print "Client <name> removed"
  close the connection ;  clear the handle
  the slot is no longer active ;  clear the name
  old_frags = a large negative value               # force a score update
  FOR EACH remaining active client
    append: an empty name, a score of 0, and colours of 0 FOR this slot
```

**Invariants** — the **entity stays**, which is how a disconnecting player leaves a corpse: the
game's callback turns the body into one, and nothing frees it.

Setting the remembered score to a large negative value forces the next frame's comparison
([`sv_main.c`](sv_main.c.md#sv_updatetoreliablemessages)) to notice a change when the slot is
reused.

On a crash, no callback runs and no goodbye is sent — so a crashed player's body is left in
whatever state it was in, and their score is not updated on other clients until the slot is
reused. Deliberate: the callback could itself be what crashed.

### `Host_ShutdownServer`

**Contract** — ends a game. Disconnects the local client, then spends up to three seconds trying
to flush every client's pending reliable data — reading incoming messages meanwhile to let the
reliable layer advance — then broadcasts a disconnect with a five-second budget, drops every
client, and clears both server records.

```text
FUNCTION host_shutdown_server(crash)
  IF no server is running  RETURN
  sv.active = false
  IF the local client is connected  disconnect it      # stops its sounds

  # Flush pending reliable data — "like the score!!!"
  start = now
  REPEAT
    pending = 0
    FOR EACH active client with reliable data buffered
      IF the channel will accept a message
        send it ;  clear the buffer
      ELSE
        read an incoming message      # lets the reliable layer acknowledge
        pending = pending + 1
    IF more than 3 seconds have passed  BREAK
  WHILE pending != 0

  broadcast a disconnect, blocking up to 5 seconds
  IF any client did not receive it  print a warning
  FOR EACH active client  sv_drop_client(crash)
  zero the per-level record and every player slot
```

**Invariants** — the flush loop **reads while waiting**, which is what makes it terminate: the
reliable layer only frees its outstanding message on receiving an acknowledgement
([`net_dgrm.c`](net_dgrm.c.md)), and that acknowledgement arrives through the read. A loop that
only wrote would spin for the full three seconds every time.

The three-second budget and the five-second broadcast budget are the only places the engine waits
on the network, and they exist so that the final scores reach everyone — which the source's three
exclamation marks emphasize.

### `Host_ClearMemory`

**Contract** — releases everything belonging to a level: flushes the renderer's caches, discards
every loaded model, resets the hunk to the mark taken at the end of initialization, and zeroes
both the server and client records.

**Invariants** — this is called at the *start* of a level load, not at the end of the previous
one, and the source's comment at the top of the file states the principle: memory is cleared when
a server or client begins. That is what lets an error mid-load leave the previous level's memory
intact for inspection.

The renderer cache flush must come first, because the surface cache holds pointers into model
data.

## The frame

### `Host_FilterTime`

**Contract** — accumulates raw elapsed time and decides whether a frame should run. Returns false
when less than 1/72 of a second has passed, unless a timed benchmark is in progress. Otherwise
computes the simulation step as the time since the last frame, replaced by a fixed value when
requested, and otherwise clamped between one millisecond and a tenth of a second.

```text
FUNCTION host_filter_time(elapsed) -> bool
  realtime = realtime + elapsed
  IF NOT running a timed benchmark AND realtime - oldrealtime < 1/72
    RETURN false                               # too soon; do nothing at all
  host_frametime = realtime - oldrealtime
  oldrealtime = realtime
  IF host_framerate > 0
    host_frametime = host_framerate            # fixed step: slow motion
  ELSE
    clamp host_frametime INTO 0.001 .. 0.1
  RETURN true
```

**Invariants** — **72 frames per second is the ceiling**, and the number is not arbitrary: it is
the server tick rate the protocol assumes, and running faster would send packets faster than
clients expect. A rebuild raising it must decouple the send rate from the frame rate, which is what
[`QW`](../QW/client/cl_main.c.md) does.

**The tenth-of-a-second clamp is a safety property, not a nicety.** A frame representing a full
second would advance every entity a second's worth in one collision sweep, and a player falling at
800 units per second would move 800 units — through several walls
([`world.c`](world.c.md) sweeps a segment, so a long segment can pass entirely through a thin
obstacle in one step only if the sweep misses it, but the *stepping* logic in
[`sv_phys.c`](sv_phys.c.md) genuinely breaks). Clamping means the simulation runs slow rather than
wrong.

**The one-millisecond floor** prevents a division by zero in the friction and acceleration
computations.

**A fixed step makes the game deterministic in time**, which is what the slow-motion variable is
for and what demo recording depends on for reproducibility.

The benchmark bypass is what lets a timed demo run at whatever rate the machine manages.

### `Host_GetConsoleCommands`

**Contract** — drains the platform's console input, queueing each line as if typed.

### `Host_ServerFrame`

**Contract** — runs one server tick: publish the step to game logic, clear the broadcast, accept
connections, read player messages, run physics unless paused, and send.

```text
FUNCTION host_server_frame()
  the global frametime = host_frametime
  sv_clear_datagram()
  sv_check_for_new_clients()
  sv_run_clients()
  IF NOT sv.paused AND (more than one player OR the game has keyboard focus)
    sv_physics()
  sv_send_client_messages()
```

**Invariants** — the order is: accept, receive, simulate, send. Receiving before simulating is
what makes a player's command take effect in the same tick it arrived.

The pause condition includes the **single-player focus test**, so a solo game freezes when the
console opens. The same test appears in [`sv_user.c`](sv_user.c.md#sv_runclients); both are
needed, because one suppresses movement and the other suppresses the whole simulation.

**Notes** — a build switch replaces this with a version that subdivides the frame into steps of at
most 0.05 seconds and runs the simulation repeatedly, to give a fixed 20-tick simulation rate
regardless of frame rate. It is not enabled. A rebuild wanting a fixed tick should start from it —
but note that it changes the animation interval too
([`pr_exec.c`](pr_exec.c.md#state)), so the two must move together.

### `_Host_Frame`

**Contract** — one whole frame. Establishes the error unwind target, advances the random sequence,
filters time and returns if too soon, then runs the thirteen steps below. Optionally prints a
three-way timing breakdown.

```text
FUNCTION host_frame_inner(elapsed)
  IF arriving here BY the error unwind  RETURN
  advance the random generator                 # keeps randomness time-dependent
  IF NOT host_filter_time(elapsed)  RETURN     # too soon

  send_key_events()                            # drain the platform's input
  in_commands()                                # let controllers queue commands
  cbuf_execute()                               # run queued console commands
  net_poll()                                   # the discovery timer queue

  IF a server is running  cl_send_cmd()        # BEFORE the server runs
  host_get_console_commands()
  IF a server is running  host_server_frame()
  IF NO server is running  cl_send_cmd()       # AFTER the incoming reads

  host_time = host_time + host_frametime
  IF connected  cl_read_from_server()
  scr_update_screen()
  IF the connection is fully established
    s_update(the camera position and basis) ;  cl_decay_lights()
  ELSE
    s_update(no position, no basis)            # silence the world
  cdaudio_update()
  host_framecount = host_framecount + 1
```

**Invariants** — five orderings matter.

**The unwind target is established at the top of the frame**, so an error anywhere inside — including
inside the interpreter, inside physics, inside a command handler — resumes here and the frame is
abandoned. Everything the error path cleaned up is already consistent by then.

**The random generator is advanced once per frame unconditionally**, so that the sequence depends
on how many frames have run and is therefore not reproducible between machines. A rebuild wanting
deterministic replay must remove it, and must then accept that monster behaviour becomes
frame-rate-independent in a way the original's is not.

**The player's intentions are sent before the server frame when the server is local, and after
when it is remote.** Locally, sending first means the command is in hand when physics runs, saving
a frame of latency. Remotely, sending after means the command is composed from state just updated
by the incoming messages. Two lines, one frame of latency each way, and a rebuild that picks one
order for both cases makes one of the two cases worse.

**Console commands run before the server frame**, so a command that changes the level takes effect
before the old level is simulated again.

**Sound is updated from the camera only when the connection is fully established**; otherwise it
is updated with a null listener, which silences the world without stopping the mixer. A rebuild
that skips the update entirely lets the ring buffer run dry
([`sound.h`](sound.h.md)).

### `Host_Frame`

**Contract** — the platform's entry point. Runs a frame, and when profiling is enabled accumulates
timings over a thousand frames and prints the average with the player count.

## `Host_InitCommands`

**Contract** — declared here, implemented in [`host_cmd.c`](host_cmd.c.md): registers the
console commands that manage games, levels, saves and players.
