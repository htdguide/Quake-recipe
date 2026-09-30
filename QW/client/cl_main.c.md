# QW/client/cl_main.c

> The client program: the frame loop, the connection attempt with its retries, the connectionless protocol, the configuration a player carries with them, and the error handling that returns to the console instead of exiting.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`net_chan.c`](net_chan.c.md) · [`cl_parse.c`](cl_parse.c.md) · [`cl_input.c`](cl_input.c.md) · [`cl_pred.c`](cl_pred.c.md) · [`cl_ents.c`](cl_ents.c.md) · [`cl_cam.c`](cl_cam.c.md) · [`cl_demo.c`](cl_demo.c.md) · [`screen.h`](screen.h.md) · [`sound.h`](sound.h.md) · [`md4.c`](md4.c.md)
**Used by** — the platform entry point ([`sys_win.c`](sys_win.c.md), [`sys_linux.c`](sys_linux.c.md))
**Tier floor** — none

## Purpose

This file absorbed the original's [`host.c`](../../WinQuake/host.c.md) as well as its own
[`cl_main.c`](../../WinQuake/cl_main.c.md). With no server in the process there is no host to arbitrate between two halves, so the
frame loop, the initialization order, the error handling and the configuration writing all moved here.

Read both originals for the shared substance: the subsystem initialization order, the configuration file, the frame-rate limiting,
the disconnection handling. This twin records the changes, and they are substantial.

## State

As [`cl_main.c`](../../WinQuake/cl_main.c.md); the records are unchanged except where **What differs** says otherwise.

## `Host_Frame`, `Host_SimulationTime`

**Contract** — one client frame: bound the frame rate, pump input, run console commands, read packets, send a command or retry a
connection, extrapolate the other players, predict the local player, extrapolate again, build the visible entity list, draw, update
sound and music. Reports timings when asked. A fatal error unwinds to the top of this function rather than exiting.

```text
FUNCTION host_frame(time)
  establish the error-recovery point; a fatal error returns here
  realtime += time
  fps = clamp(the player's frame limit, or one derived from their bandwidth,
              between 30 and 72)
  IF not timing a recording AND too little time has passed  RETURN
  host_frametime = elapsed, clamped to 0.2
  pump platform input ;  poll controllers ;  run console commands
  read_packets()
  IF disconnected  check_for_resend()  ELSE  send_cmd()
  set_up_player_prediction(without predicting)      # positions only
  predict_move()                                    # the local player
  set_up_player_prediction(with predicting)          # now extrapolate others
  emit_entities()
  update_screen()
  IF playing  update sound from the view, and decay the lights
  ELSE        update sound from the origin
  update music
```

**Invariants** —

- **The frame rate is bounded above as well as below, and the upper bound is derived from the player's bandwidth setting.** Every
  frame sends a packet ([`cl_input.c`](cl_input.c.md)), so an uncapped frame rate is an uncapped packet rate, which the channel's
  budget would then choke arbitrarily. Tying the two together is the correct coupling and it is easy to miss: **in a
  packet-per-frame design, the frame limiter is a network control.**
  The floor exists because below it the command durations become long enough to change the physics
  ([`pmove.c`](pmove.c.md)'s split threshold).
- **Player extrapolation is called twice, once without predicting and once with.** The first pass establishes the other players'
  positions so the local prediction can collide with them; the second then extrapolates them for drawing
  ([`cl_ents.c`](cl_ents.c.md)). Doing it in one pass gives either collision against un-extrapolated positions or drawing from a
  list the prediction has disturbed.
- **A fatal error unwinds to the top of the frame loop.** A client that loses its server, or meets a malformed packet, returns to the
  console and stays running. The original's equivalent tears the whole host down
  ([`host.c`](../../WinQuake/host.c.md)); here a disconnection is routine, because it is.
- The frame time is clamped, and the clamp is the same safeguard the original has.

## `CL_CheckForResend`, `CL_BeginServerConnect`, `CL_Connect_f`, `CL_SendConnectPacket`

**Contract** — the connection attempt: resolve the server's name, apply the default port if none was given, send a challenge
request, and **retry every few seconds until answered or abandoned**. On receiving the challenge, send the connection request
carrying the protocol version, this client's connection identifier, the challenge, and the player's information dictionary.

```text
FUNCTION check_for_resend()
  IF no attempt is in progress  RETURN
  IF fewer than 5 seconds since the last attempt  RETURN
  resolve the server name ;  abandon on failure
  reject an address the client is not permitted to contact
  IF no port was given  use the default
  record the attempt time                       # measured around the resolve
  send an out-of-band challenge request
```

**Invariants** —

- **Connection is a retry loop with no state machine**, because both messages are out of band
  ([`net_chan.c`](net_chan.c.md)) and idempotent: a repeated challenge request returns the same challenge
  ([`sv_main.c`](../server/sv_main.c.md)). *That idempotence is what lets the retry be three lines.*
- **The time spent resolving the name is measured and excluded from the retry interval**, because a slow resolve would otherwise
  consume the whole interval and the retry would fire immediately.
- The address is checked against a permitted-address rule before contacting it, which exists so a web page cannot use the client to
  contact arbitrary hosts.
- **The player's whole information dictionary travels in the connection request**, which is why a password is carried there and must
  be stripped server-side ([`sv_main.c`](../server/sv_main.c.md)).

## `CL_ConnectionlessPacket`

**Contract** — handles an out-of-band reply: a challenge, an acceptance, a text message to print, a server's query response, a
pointer to another server, or a command to execute. Anything unrecognized is reported.

**Invariants** —

- **A command to execute arriving from a server is honoured**, which is how a server configures its clients
  ([`sv_send.c`](../server/sv_send.c.md)). It is also a trust boundary the recipe should name plainly: ***the server can run
  console commands on the client.*** In 1996 that was a feature; a rebuild should scope it to a safe subset, because a command
  interpreter with file access reachable from an unauthenticated packet is a serious hole.
- The reply is only accepted while a connection attempt is outstanding for that address, which limits but does not close the
  exposure.

## `CL_ReadPackets`

**Contract** — read every waiting packet: route an out-of-band one to the handler above, otherwise hand it to the channel and then
to the message parser. Detect a connection that has gone silent and disconnect.

**Invariants** — **a packet from an unexpected address is dropped silently**, and a timeout disconnects rather than hanging. Both are
the client's side of the same rules the server applies.

## `CL_SetInfo_f`, `CL_FullInfo_f`, `CL_FullServerinfo_f`, `CL_Color_f`, `CL_Rcon_f`, `CL_Packet_f`, `CL_Download_f`, `CL_User_f`, `CL_Users_f`, `CL_Version_f`

**Contract** — read and write entries of the player's own information dictionary, which is sent to the server on connection and on
each change; set the two colours; send a remote administration command; send a raw packet; request a file; and print player and
version information.

**Invariants** —

- **The player's identity is a dictionary that travels with them**, not server-side state: name, colours, chosen skin, bandwidth
  rate and anything a modification wants. Changing an entry mid-game sends only that entry
  ([`sv_user.c`](../server/sv_user.c.md)). That is the right model — *the client owns its own description* — and it is why the server
  must sanitize it.
- **The remote administration password is stored in a setting and sent in the clear**
  ([`sv_main.c`](../server/sv_main.c.md)). The recipe repeats the warning here because this is the end that holds the password: a
  player's configuration file contains a server's administration password in plaintext. A rebuild must not reproduce this.
- The raw-packet command exists for diagnosis and is exactly the facility that makes the address restriction above necessary.

## `Host_Init`, `Host_Shutdown`, `Host_Error`, `Host_EndGame`, `Host_WriteConfiguration`

**Contract** — initialize every subsystem in order; shut them down in reverse; report a fatal error, disconnect, and unwind to the
frame loop; end the session cleanly with a message; and write the player's settings and key bindings to a file.

**Invariants** —

- **The initialization order is the dependency order** and is the same list as the original's
  ([`host.c`](../../WinQuake/host.c.md)) minus the server. Memory, then the command interpreter and settings, then the file system,
  then video, then sound, then the network.
- **Errors are re-entrancy guarded**: an error while handling an error exits immediately.
- **The configuration is written on a clean exit only**, so a crash does not persist a broken setting. Worth keeping.

## `Host_FixupModelNames`, `simple_crypt`

**Contract** — adjust the names of two player models; and obscure a short buffer by inverting every byte.

**Invariants** — the obscuring function is **not encryption and must not be mistaken for it**: it inverts each byte, which is
reversible by anyone. It exists so that a string is not plainly visible in the executable, which is an anti-casual-inspection
measure and nothing more. The recipe records it as such; a rebuild has no reason to reproduce it and should not treat anything
protected this way as protected.

## `CL_Disconnect`, `CL_Disconnect_f`, `CL_Reconnect_f`, `CL_Changing_f`, `CL_NextDemo`, `CL_Quit_f`, `CL_Init`, `CL_ClearState`, `CL_EstablishConnection`

**Contract** — leave a server, telling it; reconnect to the same one; note that the server is changing level and wait; advance a
recording sequence; quit; register everything; and clear the per-session state.

**Invariants** — **a level change is not a disconnection.** The server instructs the client to expect a new level
([`sv_init.c`](../server/sv_init.c.md)) and the client keeps its channel, clears its world, and goes through the spawn handshake
again. That distinction is what makes a persistent server possible and it is the piece the original engine lacks entirely.

**Notes** — the three things a rebuilder should take from this file: **the frame limiter is a network control**, **connection is an
idempotent retry rather than a state machine**, and **a fatal error should return to a console rather than exit**. The first is
non-obvious, the second is a simplification worth copying, and the third is what makes a client usable on a flaky network.

The two trust issues — a server executing commands on the client, and a plaintext administration password — are the recipe's
strongest "do not copy" markers in the client chapter.
