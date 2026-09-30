# QW/server — the dedicated server

> A whole program that talks only to strangers: it simulates the world, builds a private view of it for each of sixteen players, and paces itself to each of their links.

Read [`../README.md`](../README.md) for the three ideas this chapter implements, and
[`../../WinQuake/README.md`](../../WinQuake/README.md) sections 3–5 for the simulation, which is largely unchanged. Twelve files here
are new; twenty are pointers to their originals.

**What makes this a server rather than half of a game**: admission control, identity, address filtering, remote administration, a public
presence, flood protection, and reconfiguration without restart. None of those exist in a listen server and all of them are forced by
running unattended for weeks.

## Read in this order

**1. The contract** — [`qwsvdef.h`](qwsvdef.h.md), the umbrella header, which *is* the definition of the dedicated server: this list of
includes and nothing else.

**2. The data model** — [`server.h`](server.h.md). The most informative single page in the chapter. Its thesis: **going from one client
to many over unreliable links turns almost every piece of shared server state into per-client state** — a snapshot ring, a datagram, a
statistic history, a spillover queue, and speed overrides, all per player. Plus the zombie connection state, and a challenge table sized
so that filling it is expensive.

**3. The output stage** — the three files that build and pace a packet:
[`sv_ents.c`](sv_ents.c.md) (the snapshot as a difference from an acknowledged reference, players encoded separately, nails in six bytes
each, visibility from a slightly enlarged leaf set) →
[`sv_send.c`](sv_send.c.md) (reply only when replied to, one spillover buffer per packet, skip a choked client rather than truncating it,
and multicast by audience) →
[`sv_nchan.c`](sv_nchan.c.md) (the spillover queue in front of the reliable stream, and why every reliable write in the server goes
through ten functions).

**4. The input stage** — [`sv_user.c`](sv_user.c.md). Three commands per packet so a lost packet loses nothing; at most one movement
block per packet because a client will repeat anything it can; the client-driven staged join; the file transfer; and chat with flood
protection.

**5. The spine** — [`sv_main.c`](sv_main.c.md). Simulate, then read, then reply. The challenge that proves address ownership. The
connection request and the password stripped out of the dictionary before it is stored. The address filter. Remote administration —
**which is plaintext and must be redesigned.**

**6. Level setup** — [`sv_init.c`](sv_init.c.md). Two precomputations: the hearable set (one step of transitive closure over
visibility, because sound reaches around a corner) and the entity baselines. Plus the level change that does not disconnect anyone.

**7. Operation** — [`sv_ccmds.c`](sv_ccmds.c.md). The configuration dictionary with a private half, the per-player diagnostic report,
flood protection, and the content-directory switch mid-game.

**8. The platform** — [`sys.h`](sys.h.md) (seven operations) · [`sys_unix.c`](sys_unix.c.md) (**waits on the socket and the console
together** — the right way, and the correction to the original's dedicated server) · [`sys_win.c`](sys_win.c.md).

**9. The build** — [`makefile`](makefile.md). The authoritative statement of what the server is, and of which files are shared with the
client: the channel, the transport, the mover, the collision queries and the foundation.

## The author's own notes — read these

[`newnet.txt`](newnet.txt.md) — **the design document for the whole networking**, written before the code. It contains the prediction
loop in four lines, the packet contents as first sketched, and a task list whose items map one to one onto the shipped code. Read it
before rebuilding anything networked.
[`move.txt`](move.txt.md) — the movement model's state machine as a table: four probes and what each combination means.
[`notes.txt`](notes.txt.md) — a challenge-response scheme for verifying servers to a directory. **Designed and never built.**
[`profile.txt`](profile.txt.md) — a real execution profile. **Collision dominates**, at about six million hull sweeps in the sample;
the game interpreter is about six percent. Use it as a target shape: if a rebuild's profile differs sharply, something is wrong
structurally rather than locally.

## The simulation — pointers to the original

Unchanged in substance; read the originals and these twins for the deltas.

[`world.h`](world.h.md) · [`world.c`](world.c.md) — collision. The leaf list an entity records now does double duty as the visibility
index, which is two responsibilities in one structure.
[`sv_phys.c`](sv_phys.c.md) — physics **with the player's movement removed**, plus a transactional push and an immediate pass so a
projectile fired this frame moves this frame.
[`sv_move.c`](sv_move.c.md) — monster locomotion, untouched, and unused in a game with no monsters.
[`model.h`](../client/model.h.md) · [`model.c`](model.c.md) — **roughly half a map file and no model files**: the inventory of what a server needs
to parse.
[`progs.h`](progs.h.md) · [`pr_comp.h`](pr_comp.h.md) · [`progdefs.h`](progdefs.h.md) · [`pr_exec.c`](pr_exec.c.md) ·
[`pr_edict.c`](pr_edict.c.md) · [`pr_cmds.c`](pr_cmds.c.md) — the game-logic interface, extended with message routing by audience,
dictionary access, a kill log and a trajectory query.
[`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`math.s`](math.s.md) · [`worlda.s`](worlda.s.md) — **of the entire
server, exactly one function was worth hand writing**, and the profile says which.

## Cycles

- **[`sv_main.c`](sv_main.c.md) ↔ [`sv_user.c`](sv_user.c.md) ↔ [`sv_send.c`](sv_send.c.md).** The spine dispatches to the input stage,
  which writes into the output stage's buffers, which the spine then sends. Broken by reading
  [`server.h`](server.h.md) first: the data model is the shared vertex.
- **[`sv_phys.c`](sv_phys.c.md) ↔ [`pr_cmds.c`](pr_cmds.c.md) ↔ the game logic.** As in the original. Broken at
  [`pr_exec.c`](pr_exec.c.md).

## Not twinned

The editor and project files (`qwsv.dsp`, `qwsv.dsw`, `qwsv.mak`, `qwsv.mdp`, `qwsv.plg`).
