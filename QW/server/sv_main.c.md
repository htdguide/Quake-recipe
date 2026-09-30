# QW/server/sv_main.c

> The server's spine: a frame that simulates then reads then replies, a connectionless protocol for connecting and querying, a challenge that proves the requester owns its address, an address filter, remote administration, and the directory announcement.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`net_chan.c`](../client/net_chan.c.md) · [`sv_send.c`](sv_send.c.md) · [`sv_user.c`](sv_user.c.md) · [`sv_init.c`](sv_init.c.md) · [`sv_phys.c`](sv_phys.c.md) · [`sv_ccmds.c`](sv_ccmds.c.md) · [`pr_edict.c`](pr_edict.c.md) · [`md4.c`](../client/md4.c.md)
**Used by** — [`sys_unix.c`](sys_unix.c.md) and [`sys_win.c`](sys_win.c.md) call its frame
**Tier floor** — none

## Purpose

Read [`sv_main.c`](../../WinQuake/sv_main.c.md) for what the original's server does, then note that almost none of this file
resembles it. The original's server is half of a program whose other half is a client, connected by a driver table; this one is a
whole program that talks only to strangers. So the content here is **everything a server needs that a listen server does not**:
admission control, identity, filtering, remote control, and a public presence.

## State

```text
VARIABLE sv : the world state ;  svs : the server-wide state (server.h)
VARIABLE host_client, sv_player        # whichever client is being served
VARIABLE the settings: timeout, zombie time, passwords, rate limits,
         the remote-administration password, the allowed name characters,
         spectator limits, the directory addresses
VARIABLE the address filter list
```

## `SV_Frame`

**Contract** — one server frame: advance time unless paused, time out silent clients, roll the kill log if full, run entity
physics, read every waiting packet, run any console commands, apply changed settings, send each client that spoke a reply, and
announce to the directory if due. Accumulates busy and idle time.

```text
FUNCTION frame(time)
  start = clock ;  idle += start - previous_end
  advance the random sequence                  # keep it time-dependent
  IF not paused  realtime += time ;  sv.time += time
  check_timeouts()
  roll the kill log if it is full
  IF not paused  physics()                     # the world moves FIRST
  read_packets()                               # then clients are heard
  run console commands
  apply changed settings
  send_client_messages()                       # then clients are answered
  announce to the directory if due
  end = clock ;  active += end - start
  every 100 frames, latch the accumulated timings and reset
```

**Invariants** —

- **The world is simulated before clients are heard, and clients are answered after.** So a player's command acts on a world that
  has already moved this frame, and the snapshot they receive reflects both. The original interleaves differently
  ([`host.c`](../../WinQuake/host.c.md)) and it matters for what a player can react to. The design note named this ordering
  explicitly ([`newnet.txt`](newnet.txt.md)).
- **The frame rate is the platform's, not a fixed tick.** The server runs as often as it is called and integrates by the elapsed
  time; player movement is integrated by the *command's* duration instead
  ([`sv_user.c`](sv_user.c.md)), which is what decouples the two.
- **Busy and idle time are measured and latched**, which is what tells an operator whether the machine is the bottleneck
  ([`sv_ccmds.c`](sv_ccmds.c.md)). Measuring this is cheap and a rebuild should.
- The random sequence is stepped every frame so that a server's randomness does not depend only on how many calls the game logic
  made — a small thing, and it is why two servers running the same map diverge.

## `SV_ReadPackets`, `SV_ConnectionlessPacket`

**Contract** — read every waiting packet; route one with the out-of-band marker to the connectionless handler; otherwise find the
client it belongs to by address and connection identifier, update its recorded port if it moved, hand the packet to the channel and
then to the client parser. Unmatched packets are ignored.

**Invariants** —

- **A client is matched on the address's host part plus its connection identifier, and the port is then updated** to whatever the
  packet came from ([`net_chan.c`](../client/net_chan.c.md)). That is the address-translation workaround, and this is where it
  takes effect.
- **An unmatched packet is silently dropped.** A server on a public network receives a great deal of traffic that is not for it,
  and logging it would be the denial of service.
- **The connectionless protocol is text commands in an out-of-band datagram**, parsed by the same command tokenizer the console
  uses ([`cmd.c`](../client/cmd.c.md)). The commands are: query the server's status, ping it, request a challenge, connect, log,
  and run a remote command. Nothing needs server state to arrive.

## `SVC_GetChallenge`

**Contract** — return a random number bound to the requester's address, reusing an existing entry for that address or overwriting
the oldest.

```text
FUNCTION get_challenge()
  scan the table for an entry matching the requester's host address
  IF none, pick the OLDEST entry and overwrite it with
    a fresh random number, this address, and the current time
  send the entry's number back
```

**Invariants** —

- **The challenge proves that whoever asks to connect can receive at the address they claim.** A forged source address gets the
  challenge sent to the real owner, and the forger cannot answer. That is the whole purpose and it costs one extra round trip at
  connection time only. It is the same mechanism every protocol since has used against reflection and spoofed connections, and a
  rebuild should not omit it.
- **Repeating the request for the same address returns the same challenge**, so a lost connection attempt can be retried.
- **The table is replaced oldest-first and is deliberately large** ([`server.h`](server.h.md)) so that an attacker cannot cycle
  legitimate challenges out of it cheaply.
- The number is built from two draws of the ordinary random generator, which is **not** unpredictable to a determined attacker.
  The recipe records this honestly: the challenge stops spoofing, not a knowledgeable adversary. A rebuild should use a real
  random source — the cost is nothing.

## `SVC_DirectConnect`

**Contract** — admit a client: check the protocol version, verify the challenge against the requester's address, check the
password or the spectator password, sanitize the client's information dictionary, reclaim any slot already held by that address,
find a free slot honouring the player and spectator limits, set up the channel, and put the client in the state that begins the
spawn handshake.

```text
FUNCTION direct_connect()
  IF the stated protocol version is not ours  reject with our version ;  RETURN
  read the connection identifier, the challenge and the information dictionary
  IF no challenge exists for this host address  reject
  IF the challenge does not match             reject
  IF the dictionary claims spectator
    IF a spectator password is set and does not match  reject
    strip the password from the dictionary ;  mark the entry as a server-set one
  ELSE
    IF a password is set and does not match  reject
    strip the password from the dictionary
  IF high characters are disallowed  filter the dictionary to printable characters
  IF a slot already exists for this address and identifier
    IF it is still connecting  reuse it     # a retried connection attempt
    IF its zombie period has not elapsed  reject as too soon
    otherwise drop the old one
  find a free slot within the player or spectator limit ;  reject if full
  assign a fresh unique identifier ;  set up the channel with the connection id
  tell the client it is connected ;  state = connected
```

**Invariants** —

- **Passwords arrive inside the client's information dictionary and are removed from it before it is stored.** The dictionary is
  broadcast to every other player ([`sv_send.c`](sv_send.c.md)); leaving the password in it would publish it. That removal is a
  one-line security-critical step and it is exactly the kind of thing a rebuild forgets.
- **A key the server sets itself is marked distinctly** from one the client set, so a client cannot claim to be a spectator by
  setting the key directly. The naming convention that marks it is enforced by the dictionary code
  ([`common.c`](../client/common.c.md)), and it is the mechanism by which server-asserted facts and client-asserted ones share
  one dictionary safely. *Worth copying: separate namespaces for who asserted what.*
- **Non-printable characters are filtered out of the dictionary by default.** A player name containing control characters
  corrupts other players' displays and can impersonate server messages. The filter is a setting because some communities wanted
  the characters.
- **A reconnecting address reclaims its own slot** if it is still connecting — which is what makes a retried connection work —
  but is refused while its previous slot is in the zombie period, which is what stops a stale connection's packets being
  confused with a new one ([`server.h`](server.h.md)).
- **Players and spectators have separate limits**, counted separately, so spectators cannot fill the game.
- The client is given a **unique identifier that is never reused**, distinct from its slot number, because external tools track
  players across reconnections ([`sv_ccmds.c`](sv_ccmds.c.md)).

## `SVC_Status`, `SVC_Ping`, `SVC_Log`

**Contract** — answer a query with the server's public dictionary and a line per player; answer a ping; and send the accumulated
kill log to whoever asked.

**Invariants** — **the status reply is plain text and needs no connection**, which is what lets any tool list servers. Its format
is therefore a public contract with software the engine does not know about, and a rebuild changing it breaks server browsers.

## `Rcon_Validate`, `SVC_RemoteCommand`

**Contract** — check a submitted password against the configured administration password, and if it matches, run the rest of the
line as a console command with output redirected back to the sender.

**Invariants** —

- **The password travels in the clear in every remote command**, and the comparison is a plain string comparison. There is no
  challenge, no nonce, no encryption; anyone observing the traffic obtains the password and full control of the server. The
  recipe records this plainly: ***remote administration as implemented here is not secure and must not be reproduced as is.*** A
  rebuild needs at minimum a challenge-response — the scheme sketched in
  [`notes.txt`](notes.txt.md) is the right shape — and better, a separate authenticated channel.
- An empty configured password **refuses everything**, which is the correct default.
- The output is redirected to the sender ([`sv_send.c`](sv_send.c.md)), which is what makes remote administration usable.

## `StringToFilter`, `SV_AddIP_f`, `SV_RemoveIP_f`, `SV_ListIP_f`, `SV_WriteIP_f`, `SV_FilterPacket`, `SV_SendBan`

**Contract** — parse an address pattern with a variable number of significant octets into a mask and value; add, remove, list and
save such patterns; and test a packet's source against the list, in either allow or deny sense.

**Invariants** — **the filter is a mask-and-value list applied to the host address**, checked before a connection is admitted, and
its sense — whether the list is what to block or all that is permitted — is a setting. Saving the list to a file that is executed
at startup is how it persists. That is the minimum a public server needs and a rebuild should include it.

A rejected connection is **told it was banned** rather than ignored, which is a usability choice with a cost: it confirms the
server exists.

## `SV_CheckTimeouts`, `SV_DropClient`, `SV_FinalMessage`, `SV_CalcPing`, `SV_FullClientUpdate`, `SV_FullClientUpdateToClient`

**Contract** — drop a client that has not been heard from within the timeout, with a shorter timeout while connecting; drop a
client, telling the game logic, announcing it, and moving it to the zombie state; send a final message to everyone at shutdown;
average a client's recorded round trips; and broadcast one client's full public state.

**Invariants** —

- **Dropping goes to the zombie state, not the free state** ([`server.h`](server.h.md)).
- **The game logic is told**, so a modification can run its own disconnect handling, and the player's entity is removed.
- **The round trip is the average over the whole snapshot ring**, not the latest, because a single sample is noisy.
- A **final message is sent to every client at shutdown**, so a restarting server does not leave twenty clients waiting for a
  timeout.

## `SV_ExtractFromUserinfo`

**Contract** — read the fields the engine itself cares about out of a client's dictionary: the display name, the bandwidth rate,
the message filter, the colours, and the spectator marker. Rejects a name that duplicates another player's or that consists only
of digits, and rate-limits name changes.

**Invariants** —

- **A name that is only digits is refused**, because commands address players by number and a numeric name is ambiguous. A
  duplicate name is made unique. Both are protections against impersonation through the command interface, and both are the kind
  of thing that only shows up once real players arrive.
- **Name changes are rate-limited**, because each one is broadcast to every player.
- The engine reads a **fixed small set of keys** and ignores the rest, which is what lets modifications carry their own.

## `Master_Heartbeat`, `Master_Shutdown`

**Contract** — periodically send an out-of-band announcement to each configured directory address, and a withdrawal at shutdown.

**Invariants** — **the server announces itself rather than being polled**, so a directory needs no protocol beyond receiving a
datagram. The announcement carries a sequence number so a directory can detect a restart.

## `SV_InitLocal`, `SV_InitNet`, `SV_Init`, `SV_Shutdown`, `SV_Error`, `SV_CheckVars`, `SV_CheckLog`, `SV_GetConsoleCommands`, `ServerPaused`

**Contract** — register every setting and command, open the socket on the configured port, initialize every subsystem in order,
shut down cleanly, report a fatal error after sending the final message, detect settings that changed and publish them, roll the
kill log, read the console, and report the pause state to the channel.

**Invariants** — a fatal error **still sends the final message to every client** before exiting, so players see a reason rather
than a timeout. Settings that clients must know about are published when they change, which is why they are polled once per frame
rather than hooked.

**Notes** — the security content of this file deserves a summary, because a rebuilder will inherit whatever it copies:

- The **challenge** is sound in shape and weak in its random source.
- The **movement checksum** ([`sv_user.c`](sv_user.c.md)) catches corruption and casual tampering only.
- The **model checksums** ([`sv_init.c`](sv_init.c.md)) are self-reported and prove nothing.
- The **address filter** works as intended.
- **Remote administration is plaintext and must be redesigned.**
- The **channel itself is unauthenticated** ([`net_chan.c`](../client/net_chan.c.md)), so an observer can inject.

A rebuild targeting the public internet should treat the first, fourth and fifth as the list of things to do properly, and should
not mistake the second and third for protections.
