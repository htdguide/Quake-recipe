# QW/client/protocol.h

> The wire protocol: version 28, the connectionless message letters, the ports, the message kinds in both directions, and the field flags for the entity, player and command encodings.

**Needs** — nothing
**Used by** — every file that reads or writes the wire, in both programs
**Tier floor** — none

## Purpose

Read [`protocol.h`](../../WinQuake/protocol.h.md) for the original's protocol and for the encoding rules — coordinates at an eighth
of a unit, angles at a 256th of a turn — which are unchanged. This is version 28 against the original's 15, and **the two are not
compatible in either direction**. The differences are the protocol-level record of everything QuakeWorld changed.

## State

```text
CONSTANT protocol_version = 28
CONSTANT the client port, the directory port, and the default server port
CONSTANT the connectionless message letters (below)
CONSTANT the server-to-client message kinds, and the client-to-server ones
CONSTANT the entity field flags (U_*), including the removal flag
CONSTANT the player field flags (PF_*)
CONSTANT the command field flags (CM_*)
CONSTANT the temporary-entity kinds
CONSTANT a fixed value used to seed the movement checksum
```

## The connectionless letters

**Contract** — a single letter identifies each message sent outside any connection, with the parties named by convention: server to
client, client to server, directory to server, or any to any.

```text
challenge, connection accepted, print this text     -- server to client
ping, acknowledge, refuse, echo                     -- either direction
heartbeat, shutdown                                 -- server to directory
run this command line                               -- to a client
```

**Invariants** — **the whole connection and query protocol is single letters and text** in out-of-band datagrams
([`net_chan.c`](net_chan.c.md)). That is why it needs no state on either side and why any tool can speak it. The letters are a
public contract with server browsers, so a rebuild changing them breaks external software.

**Three fixed port numbers** are declared: one the client binds, one the directory uses, and one servers default to. Having a default
server port is what lets a player type a bare address.

## The message kinds

**What was removed** — and each removal is informative:

- **Absolute time.** The original sends the server's time in every frame; here time is implicit in the packet sequence and in each
  player's own state stamp ([`cl_ents.c`](cl_ents.c.md)). Removing it saves four bytes a frame and, more importantly, removes the
  client's dependence on a clock it cannot trust.
- **The signon sequence number.** The original's spawn is a numbered handshake driven by the server; here it is client-driven named
  stages ([`sv_user.c`](../server/sv_user.c.md)), so the number has no meaning.
- **The full server-information message**, replaced by a server-data message plus the separately requested lists — because the
  original's single message does not fit a large map.
- **Name, colour and client-data messages**, replaced by the information dictionary
  ([`cl_parse.c`](cl_parse.c.md)), which carries all three and anything a modification adds.
- **The particle message**, replaced by the multicast facility
  ([`pr_cmds.c`](../server/pr_cmds.c.md)) — a particle effect is now a message the game logic composes and routes.
- **The cut-scene message.** No single-player campaign.

**What was added:**

```text
snapshot, absolute and delta forms      -- the entity stream (sv_ents.c)
player state                            -- one per visible player
nails                                   -- the packed projectile encoding
download block                          -- a file being sent (cl_parse.c)
dictionary update, whole and single-key  -- a player's or the server's info
ping, connect time                      -- the scoreboard's extra columns
small kick, big kick                    -- a view punch as a message KIND
intermission with a position            -- rather than with music
```

**Invariants** —

- **A retired message number is commented out and never reused.** The numbering is the compatibility surface, and reusing a number
  would make two versions silently misparse each other rather than failing cleanly.
- **A view punch became two message kinds rather than a value**, because the two magnitudes are the only ones used and a kind costs
  no payload at all. A small economy, and an example of the protocol being shaped by what the game actually sends.
- **The statistic update shrank from a long to a byte**, with a separate long form where needed — the common case is small.
- **A print message now carries an importance level** ([`sv_send.c`](../server/sv_send.c.md)), so a client can filter.
- **Messages are not length-prefixed**, so an unknown kind cannot be skipped and ends the session
  ([`cl_parse.c`](cl_parse.c.md)). That is why the version check is the first thing the handshake does.

## The client-to-server kinds

```text
no operation, delta reference request, movement block, text command,
spectator teleport, upload block
```

**Invariants** — **six kinds, and one of them carries everything that matters.** The movement block
([`cl_input.c`](cl_input.c.md)) is the whole of a playing client's output. Text commands cover everything else, which means the
client-to-server protocol is almost entirely extensible without a version change — a modification adds commands, not message kinds.
That asymmetry is deliberate and worth copying.

## The field flags

**Contract** — three sets of flags, one per encoding: entity fields, player fields, and command fields. Each names which optional
fields follow, and each encoding's flag word shares bits with something else.

**Invariants** —

- **The entity flag word shares its low nine bits with the entity identifier**
  ([`sv_ents.c`](../server/sv_ents.c.md)), so the identifier is limited to 512 and one bit signals that a second flag byte follows.
- **A removal is the identifier with a reserved flag set** — no payload at all.
- **The player flag word is sixteen bits with no shared identifier**, because a player's slot is a separate byte.
- **The command flags mark which of the angles, movements, button and impulse fields differ from the previous command**
  ([`common.c`](common.c.md) encodes and decodes), which is what makes three commands per packet cheap.

**Notes** — the protocol is where every one of QuakeWorld's decisions becomes visible at once, so this file is the best single page
to read after [`SYSTEM-REQUIREMENTS.md`](../../SYSTEM-REQUIREMENTS.md). Its two rules for a rebuilder: **a number's meaning may never
change, only be added to**, and **extend through text commands rather than new message kinds wherever the cost allows.**
