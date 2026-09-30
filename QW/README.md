# QW — QuakeWorld: the same game over the internet

> Two programs instead of one, three ideas instead of a driver abstraction, and a protocol that assumes packets are lost.

QuakeWorld is the 1996 engine rebuilt for a link with latency and loss. Almost none of the simulation changed; almost all of the
networking did. Read [`../WinQuake/README.md`](../WinQuake/README.md) first — this chapter is a delta against it, and most files here
are pointers to their originals.

## The three ideas

Everything in this chapter follows from three decisions. If you read only three files, read these.

1. **[`client/net_chan.c`](client/net_chan.c.md) — one packet per frame, nothing ever waits.** The original sends a reliable message and
   blocks until it is acknowledged; here a packet goes out every frame carrying the current unreliable state, with the reliable stream
   riding along when there is room. A lost packet is superseded, not retransmitted. Acknowledgement is one bit in the next packet's
   header, and a byte budget paces the sender to the link.

2. **[`server/sv_ents.c`](server/sv_ents.c.md) — each update is a difference from the last snapshot the client confirmed.** The original
   deltas against a fixed baseline, so a moving entity costs the same bytes forever and every update must be reliable. Here the reference
   is something the client has acknowledged, so a stationary entity costs nothing and nothing needs to be reliable at all.

3. **[`client/pmove.c`](client/pmove.c.md) — player movement is a pure function, compiled into both programs.** The original's movement
   lives inside the server and the client cannot run it, so every step appears a round trip late. Extracted, the client replays its own
   unacknowledged commands from the last confirmed state ([`client/cl_pred.c`](client/cl_pred.c.md)) and corrections need no
   reconciliation step.

Each is independent and a rebuild can adopt them one at a time. The second is the one that matters most for bandwidth; the third for
how the game feels.

## The consequences, in order of how surprising they are

- **The packet rate is the frame rate**, so the frame limiter is a network control
  ([`client/cl_main.c`](client/cl_main.c.md)) and the client's whole state is one ring indexed by packet sequence
  ([`client/client.h`](client/client.h.md)).
- **Commands are quantized to the wire before anything predicts from them**
  ([`client/cl_input.c`](client/cl_input.c.md)), and positions are snapped to the wire's grid inside the mover
  ([`client/pmove.c`](client/pmove.c.md)). Without both, prediction diverges continuously.
- **Three commands travel in every packet**, so losing one or two packets costs nothing
  ([`server/sv_user.c`](server/sv_user.c.md)).
- **Other players are extrapolated by running the same mover on their last command**, and deliberately under-extrapolated so corrections
  read as forward motion ([`client/cl_ents.c`](client/cl_ents.c.md)).
- **Every server-to-client event chooses its own audience** — everyone, everyone who can see a point, everyone who can *hear* it — using
  a precomputed hearable set ([`server/sv_send.c`](server/sv_send.c.md), [`server/sv_init.c`](server/sv_init.c.md)).
- **The connection protocol is text in out-of-band datagrams** with a challenge that proves the requester owns its address, so the server
  holds no state for an unfinished connection ([`server/sv_main.c`](server/sv_main.c.md)).
- **A level change does not disconnect anyone**, which is what makes a server persistent and is the single largest change to the
  client's structure ([`server/sv_init.c`](server/sv_init.c.md), [`client/cl_main.c`](client/cl_main.c.md)).
- **A client fetches content it lacks from the server**, a block per packet, in a staged index-driven join
  ([`client/cl_parse.c`](client/cl_parse.c.md), [`client/skin.c`](client/skin.c.md)).
- **The original's eleven-file network abstraction became three files**
  ([`client/net.h`](client/net.h.md)), because the transports it supported no longer exist.
- **Roughly half the menu system was deleted** along with the networking it configured
  ([`client/menu.c`](client/menu.c.md)).

## The chapters

| | |
|---|---|
| [`client/`](client/README.md) | the client program: the frame, prediction, the delta decompression, the renderers, the interface. 183 twins. |
| [`server/`](server/README.md) | the dedicated server: the snapshot builder, the reliable queue, admission control, the game interpreter. 32 twins. |
| [`progs/`](progs/README.md) | the compiled game logic's build directory — a pointer to [`../qw-qc/`](../qw-qc/README.md), where the sources are twinned. |
| [`qwfwd/`](qwfwd/README.md) | a hundred-line datagram forwarder, and the proof the protocol is relayable. |
| [`gas2masm/`](gas2masm/README.md) | the assembly syntax translator, carried forward. |
| [`docs/`](docs/README.md) | the release notes, read as a specification of observable behaviour. |

## Reading order

1. [`client/bothdefs.h`](client/bothdefs.h.md) — the shared constants, and the 1450-byte message that every other decision follows from.
2. [`client/protocol.h`](client/protocol.h.md) — version 28, and what was added and removed against version 15.
3. [`client/net.h`](client/net.h.md) → [`client/net_chan.c`](client/net_chan.c.md) — the channel.
4. [`client/pmove.h`](client/pmove.h.md) → [`client/pmove.c`](client/pmove.c.md) → [`client/pmovetst.c`](client/pmovetst.c.md) — the
   shared mover.
5. [`server/server.h`](server/server.h.md) → [`server/sv_ents.c`](server/sv_ents.c.md) → [`server/sv_send.c`](server/sv_send.c.md) →
   [`server/sv_user.c`](server/sv_user.c.md) → [`server/sv_main.c`](server/sv_main.c.md) — the server.
6. [`client/client.h`](client/client.h.md) → [`client/cl_input.c`](client/cl_input.c.md) →
   [`client/cl_pred.c`](client/cl_pred.c.md) → [`client/cl_ents.c`](client/cl_ents.c.md) →
   [`client/cl_parse.c`](client/cl_parse.c.md) → [`client/cl_main.c`](client/cl_main.c.md) — the client.
7. [`server/newnet.txt`](server/newnet.txt.md) and [`client/docs.txt`](client/docs.txt.md) — the author's design notes and the list of
   known defects. **Read these before rebuilding anything**: the first shows the order the problems arrive in, the second says where the
   shipped code does not match the intent.
8. [`server/profile.txt`](server/profile.txt.md) — a real measurement of where the server spends its time.

## What is not here

**Single player.** There are no monsters ([`../qw-qc/server.qc`](../qw-qc/server.qc.md) stubs every class), no saved games, no campaign
sequencing. The game logic is the original with everything non-competitive removed.

**A second transport.** One datagram family ([`client/net_udp.c`](client/net_udp.c.md), [`client/net_wins.c`](client/net_wins.c.md)), no
serial, no loopback driver — a player hosting a game runs both programs and they talk over a real socket.
