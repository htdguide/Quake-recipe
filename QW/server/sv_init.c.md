# QW/server/sv_init.c

> Starting a level: load the map, precompute which leaves can hear which, record every entity's initial state as the delta reference, run the game logic's spawn functions, and send the new level to everyone already connected.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`model.c`](model.c.md) · [`pr_edict.c`](pr_edict.c.md) · [`sv_send.c`](sv_send.c.md) · [`world.h`](world.h.md)
**Used by** — [`sv_ccmds.c`](sv_ccmds.c.md) on a map change; [`sv_main.c`](sv_main.c.md) at startup
**Tier floor** — none

## Purpose

The level-change sequence. Most of it is the same work as the original's
([`sv_main.c`](../../WinQuake/sv_main.c.md) and [`host_cmd.c`](../../WinQuake/host_cmd.c.md)): load the map, reset the
entity pool, run the map's spawn functions, precache the content. Two things are specific to QuakeWorld and are the reason
this file exists separately, and both are precomputations.

## State

```text
VARIABLE sv.pvs, sv.phs           # two bitset tables, one row per leaf
VARIABLE sv.signon                # the accumulated level-description message
VARIABLE the model and sound precache lists, and their checksums
```

## `SV_CalcPHS`

**Contract** — at map load, build two tables with one row per leaf: the decompressed visibility set, and the **hearable** set,
which is the union of the visibility sets of every leaf visible from that leaf. Reports the average sizes.

```text
FUNCTION calc_phs()
  rowbytes = enough bits for every leaf, rounded to words
  # table one: decompress every leaf's visibility set
  FOR EACH leaf i
    pvs[i] = the decompressed visibility set of leaf i
  # table two: one more step of transitive closure
  FOR EACH leaf i
    phs[i] = pvs[i]
    FOR EACH bit set in pvs[i]
      phs[i] |= pvs[that leaf]           # OR in that leaf's visibility set
  report the average row population of each table
```

**Invariants** —

- **The hearable set is one step of transitive closure over the visible set.** If you can see leaf B and leaf B can see leaf C,
  then C is hearable from you. The justification is physical: sound reaches around one corner reliably, and the second-order
  set is a cheap and close approximation of that. Using the visible set for sound makes a rocket around a corner silent —
  which players notice at once — and using full reachability makes the set nearly everything and saves no bandwidth.
- **Both tables are decompressed and stored whole**, which costs memory quadratic in the leaf count, and that is the reason this
  is a precomputation with a printed progress message rather than a lookup. The original keeps visibility run-length encoded
  and decompresses per query ([`model.c`](model.c.md)); the server here needs both sets per frame per client
  ([`sv_send.c`](sv_send.c.md)) and cannot afford to.
- **The bit numbering is one-based** — bit *n* of a row refers to leaf *n+1* — because leaf zero is the solid outside-the-map
  leaf and is never visible. That off-by-one is in the map format and must be matched; getting it wrong shifts every
  visibility answer by one leaf, which produces intermittent invisible entities.
- The averages are printed because they are the number that predicts a map's bandwidth cost. A map with a large average
  hearable set will be expensive no matter what the server does — useful information for a level designer, and worth keeping.

## `SV_CreateBaseline`

**Contract** — for every entity that exists after the map's spawn functions have run, record its model, frame, colour map, skin
and origin as its baseline, and write that baseline into the level-description message. Gives players a standard baseline.

**Invariants** — **the baseline is the delta reference for an entity that was not in the client's previous snapshot**
([`sv_ents.c`](sv_ents.c.md)). So it must be captured after spawning and before the first frame, and it must be identical on
client and server — hence it travels in the level description rather than being recomputed.

Entities created *during* play have no baseline, so they are first sent as a difference from an all-zero state. That is why a
newly spawned projectile costs more bytes than a map-placed torch.

## `SV_SpawnServer`

**Contract** — start a level: tell every connected client to reconnect, clear the server state, load the map and its collision
hulls, reset the entity pool and load the map's entity definitions, run the spawn functions, build the two visibility tables,
capture the baselines, precache the standard content, and leave every client in a state where it will be sent the level
description.

```text
FUNCTION spawn_server(level)
  tell every client to expect a new level
  save each player's persistent spawn parameters        # carried across levels
  clear the server state and the entity pool
  load the map and its three collision hulls
  reset the game interpreter's globals and string pool
  create the world entity and the player entities
  parse the map's entity list and spawn each one
  run the game logic's start-of-level function
  calc_phs()
  create_baseline()
  find the special-cased model indices
  set every connected client to the state that triggers a level description
```

**Invariants** —

- **Connected players are not disconnected across a level change**; they are sent a reconnect instruction and go through the
  spawn sequence again on the same channel ([`sv_main.c`](sv_main.c.md)). That is what makes a server persistent across maps,
  and it is a requirement the original's level change does not meet.
- **Each player's persistent parameters are saved before the pool is cleared and restored after**, which is how a coordinated
  campaign carries health and weapons forward ([`pr_edict.c`](pr_edict.c.md)).
- The order is forced: the map must be loaded before the hulls can be indexed, the entities spawned before the baselines can be
  captured, and the visibility tables built before any frame is sent.

## `SV_FlushSignon`, `SV_ModelIndex`, `SV_CheckModel`, `SV_SaveSpawnparms`

**Contract** — close off a block of the level-description message when it grows large and start another; look a model's index
up by name in the precache list; compute a checksum of a model file; and save every player's persistent parameters by calling
into the game logic.

**Invariants** —

- **The level description is sent in several blocks**, because it exceeds one reliable message and the client must acknowledge
  each before the next ([`sv_main.c`](sv_main.c.md) drives the sequence). The flush decides the block boundaries, and a block
  must never split a message.
- **Model files are checksummed** and the sums sent to the client, which compares them
  ([`cl_parse.c`](../client/cl_parse.c.md)). The purpose is to detect a player running modified models — a wall made
  transparent, a player model made larger. It is a weak measure, because the client computes and reports the sum itself, and
  the recipe records it as such: **it deters casual modification and stops nothing determined.** A rebuild wanting real
  integrity must verify server-side, which for client-side assets is not possible at all.

**Notes** — the two precomputations here — the hearable set and the baselines — are both instances of the same principle that
runs through QuakeWorld: **spend memory and load time to make the per-frame per-client cost small.** That is the right trade
whenever the bottleneck is the link rather than the machine.
