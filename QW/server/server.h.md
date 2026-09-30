# QW/server/server.h

> The server's data model rebuilt for many players over slow links: per-client snapshot rings, per-client datagrams, spillover queues, a challenge table, a multi-block level description, and the states a connection passes through including the one that exists only to stop a stale packet reviving a dead player.

**Needs** — [`protocol.h`](../client/protocol.h.md) · [`net.h`](../client/net.h.md) · [`progs.h`](progs.h.md) · [`pmove.h`](../client/pmove.h.md)
**Used by** — every `sv_*.c` in this directory, and [`pr_cmds.c`](pr_cmds.c.md)
**Tier floor** — none

## Purpose

Read [`server.h`](../../WinQuake/server.h.md) first: the world state, the entity pool, the precache lists and the light styles
are unchanged, and the server's job is the same. This twin records the differences, and they are almost entirely about one thing
— **state that used to be shared by all clients is now per client**, because clients now differ in what they have received.

## State

As [`server.h`](../../WinQuake/server.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs in the world state

```text
VARIABLE state : dead | loading | active   # replaces a bare loaded flag
VARIABLE pvs, phs                          # both fully decompressed (sv_init.c)
VARIABLE datagram, reliable_datagram       # broadcast to everyone, then cleared
VARIABLE multicast                          # composed, then routed to a subset
VARIABLE master                             # the kill log being accumulated
VARIABLE signon : the level description, in up to 8 blocks with their sizes
VARIABLE model_player_checksum, eyes_player_checksum
```

**Invariants** —

- **A three-way load state, not a boolean.** Some game-logic operations are legal only while the level is spawning — precaching,
  placing static entities — and the interpreter checks this state to refuse them later
  ([`pr_cmds.c`](pr_cmds.c.md)). The original checks a different flag for the same purpose, less clearly.
- **The level description is several blocks, each a packet's worth**, because a large map's description exceeds one reliable
  message and the client must acknowledge each before the next
  ([`sv_main.c`](sv_main.c.md) drives that sequence, [`sv_init.c`](sv_init.c.md) fills the blocks). The original sends one
  message and simply fails on a large map.
- **There is a multicast buffer distinct from the broadcast datagram.** A message composed there is then routed to whichever
  clients should receive it ([`sv_send.c`](sv_send.c.md)); the broadcast datagram goes to everyone. The original has only the
  latter and therefore cannot filter by audience.
- Two model checksums are held to compare against what clients report
  ([`sv_init.c`](sv_init.c.md)) — a weak anti-modification measure, recorded as such.

## What differs in the per-client state

```text
RECORD ClientFrame                     # one per remembered packet
  senttime : time ;  ping_time : real
  entities : the snapshot sent in that packet
RECORD Client
  state : free | zombie | connected | spawned
  spectator : bool
  userid : int ;  userinfo : dictionary text
  lastcmd, localtime, oldbuttons        # for filling gaps and for prediction
  maxspeed, entgravity                  # per-player, overriding the server's
  datagram + its storage                # THIS client's unreliable events
  backbuf queue                         # the reliable spillover (sv_nchan.c)
  frames[ring]                          # the snapshot ring, for deltas
  stats[]                               # last values sent, for the stat delta
  old_frags
  download, downloadsize, downloadcount # a file being sent to this client
  upload, uploadfn, snap_from, remote_snap
  spec_track                            # which player a spectator follows
  whensaid[10], whensaidhead, lockedtill   # flood protection history
  chokecount, delta_sequence            # -1 means "send me an absolute snapshot"
  netchan
```

**Invariants** —

- **Each client has its own unreliable datagram.** That is what makes per-audience filtering possible at all
  ([`sv_send.c`](sv_send.c.md)): a sound only some players can hear is written into only their buffers. Overflow of this buffer
  is *tolerated* — the events are simply dropped — because an unreliable event is expendable. The buffer is cleared when it is
  sent, not per frame, so events accumulate across frames a client was choked on.
- **Each client remembers the last several snapshots it was sent**, indexed by packet sequence, and the sequence it last
  acknowledged. That ring is the delta reference store ([`sv_ents.c`](sv_ents.c.md)) and its size is the maximum a client may
  fall behind before it must be sent an absolute snapshot. A distinguished value means "no reference available".
- **Each client remembers the statistic values last sent to it**, so the health and ammunition display is itself a delta
  ([`sv_send.c`](sv_send.c.md)).
- **Maximum speed and gravity are per player**, overriding the server's, because the game logic may change them for one player
  — and they must then be sent to that client, or its prediction diverges
  ([`sv_user.c`](sv_user.c.md), [`pmove.h`](../client/pmove.h.md)).
- **The last command received is kept** and re-used when a packet is lost, so a player whose commands stop arriving keeps moving
  in the same direction rather than stopping dead. That is what makes a brief outage look like lag rather than a freeze.
- The flood-protection history is **a small ring of timestamps per client**
  ([`sv_ccmds.c`](sv_ccmds.c.md), [`sv_user.c`](sv_user.c.md)), which is the minimum state a rate limit needs.
- A client record holds **a file being downloaded to it and one being uploaded from it**, which is new: the server can send a
  player a map they lack, and can request a screen capture. Both are streamed a block per packet
  ([`sv_user.c`](sv_user.c.md)).

## The connection states

```text
free      -- the slot may be reused
zombie    -- disconnected, but the slot is held for a few seconds
connected -- assigned, going through the spawn sequence
spawned   -- fully in the game
```

**Invariants** — **the zombie state is the one worth understanding.** When a player leaves, their slot is not immediately
reusable: packets already in flight from the old connection would otherwise be accepted as belonging to whoever takes the slot
next. Holding the slot for a few seconds outlives those packets. The original engine has the same requirement stated as a
comment on its connection pool ([`net.h`](../../WinQuake/net.h.md)) and meets it by never reusing storage; this is the explicit
version, and a rebuild needs one or the other.

The source's list of the four ways a client leaves — quitting, timing out, being kicked, and overflowing its reliable stream — is
worth reproducing as a checklist: each needs its own path and each must reach the zombie state.

## The server-wide static state

```text
VARIABLE spawncount                    # incremented per level, to reject late spawns
VARIABLE clients[], serverflags
VARIABLE last_heartbeat, heartbeat_sequence
VARIABLE stats : active/idle time and packet counts, latched per 100 frames
VARIABLE info : the public configuration dictionary
VARIABLE log[2] : two kill-log buffers, swapped
VARIABLE challenges[1024] : (address, challenge, time)
```

**Invariants** —

- **A count of levels spawned is kept and echoed through the spawn sequence**, so a client that was mid-spawn when the level
  changed is detected and restarted rather than joining a level it was not told about
  ([`sv_main.c`](sv_main.c.md)). Without it a slow client can spawn into the wrong map.
- **The challenge table is deliberately large, and the comment says why**: a small table could be cycled out by an attacker
  sending many connection requests, denying service to real players. That is a denial-of-service consideration designed into a
  1996 data structure, and it is the right instinct — *size a table that an unauthenticated party can fill so that filling it is
  expensive.*
- **The kill log is double-buffered and swapped on a timer**, so an external score-tracking process can read one buffer while
  the other fills ([`sv_ccmds.c`](sv_ccmds.c.md)).
- The statistics are **latched** — accumulated over a window then published — so the operator's report shows a stable number
  rather than an instantaneous one.

**Notes** — the single most transferable observation is the first: **going from one client to many over unreliable links turns
almost every piece of shared server state into per-client state.** The snapshot ring, the datagram, the statistic history, the
spillover queue and the speed overrides are all the same change applied five times. A rebuild that starts from the original's
data model will make this change file by file; starting from this one is faster.
