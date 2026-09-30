# QW/server/sv_ccmds.c

> The operator's commands: level changes, kicking, cheats, logging, the status report, the two information dictionaries, flood protection, and the mid-game content directory switch.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`sv_init.c`](sv_init.c.md) · [`sv_send.c`](sv_send.c.md) · [`sv_nchan.c`](sv_nchan.c.md) · [`cmd.h`](../client/cmd.h.md)
**Used by** — the server console and remote administration ([`sv_main.c`](sv_main.c.md) routes authenticated commands here)
**Tier floor** — none

## Purpose

What the original spreads through [`host_cmd.c`](../../WinQuake/host_cmd.c.md), gathered and extended for a server that runs
unattended for weeks. The interesting content is not the commands themselves but four facilities they establish, each of which
is a decision a long-running server forces.

## State

```text
VARIABLE the log file and the frag log file
VARIABLE host_client, sv_player            # whichever player a command is acting on
VARIABLE the flood-protection settings: how many messages, in what period,
         and how long the silence lasts
VARIABLE the master server addresses
```

## The information dictionaries

**Contract** — two named-value dictionaries: one public, sent to every client and to anyone querying the server, and one local,
readable by the game logic only. Commands read and write both; changing a public entry notifies every connected client.

**Invariants** —

- **The server's configuration is a dictionary, not a fixed set of variables**, so a game modification can add its own entries
  and have them appear to clients with no engine change. The game logic reads both dictionaries through the interpreter
  ([`pr_cmds.c`](pr_cmds.c.md)).
- **The public one is visible to anyone who can send a packet**; the local one never leaves the server. That split is the
  mechanism for keeping an administrative password out of the public record, and it only works because the two are separate
  stores rather than one store with a flag.
- A change to a public entry is **pushed to clients immediately**, so a client's display of the server's rules stays current.

## `SV_Status_f`

**Contract** — prints the server's state and a line per player: their identifier, score, connected time, latency, packet loss,
name, address and rate.

**Invariants** — **latency and loss are computed from the channel's counters** ([`net_chan.c`](../client/net_chan.c.md)), which
is why those counters exist. This report is the only diagnostic an operator has, and a rebuild should produce the same four
numbers per player: round trip, loss fraction, chosen rate, and time connected.

## `SV_Map_f`, `SV_Gamedir`, `SV_Gamedir_f`

**Contract** — change the level; and change the content directory the server searches, mid-game, telling every client to do the
same.

**Invariants** — **the content directory can change while players are connected**, which the original cannot do at all. Every
client is instructed to switch, and the file-system search path is rebuilt ([`common.c`](../client/common.c.md)). It is how a
server rotates between game modifications without restarting, and it means the search path must be rebuildable at any moment,
not only at startup.

## `SV_Kick_f`, `SV_SetPlayer`, `SV_God_f`, `SV_Noclip_f`, `SV_Give_f`

**Contract** — select a player by identifier and act on them: disconnect them, or toggle invulnerability, wall-passage, or grant
items.

**Invariants** — the cheats are **the operator's, applied to a named player**, not the player's own — there is no client-side
cheat command. That is the correct placement and it follows from the server being authoritative: a client has nothing to cheat
*with*.

Selecting a player sets two globals that the command then operates on, which is a convention the game interpreter also uses
([`pr_cmds.c`](pr_cmds.c.md)).

## `SV_Floodprot_f`, `SV_Floodprotmsg_f`

**Contract** — configure how many messages a player may send in a given period before being silenced, for how long, and what
they are told.

**Invariants** — **a public server needs rate limiting on player-generated messages**, and the limit is expressed as a count, a
window and a penalty. It is enforced where messages arrive ([`sv_user.c`](sv_user.c.md)). This is the earliest form of a
facility every multiplayer game since has needed, and a rebuild should not consider it optional.

## `SV_Logfile_f`, `SV_Fraglogfile_f`

**Contract** — begin or end logging the console to a numbered file, and logging kills to a separate one.

**Invariants** — **files are numbered, not overwritten**, so a restart never destroys the previous log. The kill log is separate
because it is machine-read by score-tracking tools — an external contract with software the engine knows nothing about, which
is worth recording precisely because nothing in the engine documents it.

## `SV_Heartbeat_f`, `SV_SetMaster_f`

**Contract** — announce this server to the configured directory servers, and set which directory servers to announce to.

**Invariants** — **the server announces itself periodically rather than being polled**, which is what makes a public server
list possible without every browser scanning the internet. The announcement is an out-of-band datagram
([`net_chan.c`](../client/net_chan.c.md)), so the directory needs no protocol of its own.

## `SV_Snap`, `SV_Snap_f`, `SV_SnapAll_f`, `SV_User_f`, `SV_ConSay_f`, `SV_Quit_f`, `SV_SendServerInfoChange`, `SV_Serverinfo_f`, `SV_Localinfo_f`, `SV_InitOperatorCommands`

**Contract** — ask a client for a screen capture and save it; print a client's information dictionary; speak as the console; quit;
notify clients of a configuration change; read and write the two dictionaries; and register every command above.

**Invariants** — the screen-capture request is the one place the **server asks the client for data**, and the client may refuse.
Recorded because it is a privacy-relevant capability that a rebuild should think about before inheriting: the server can obtain
a picture of the player's screen.

**Notes** — the four facilities above — a configuration dictionary with a private half, a per-player diagnostic report, flood
protection, and reconfiguration without restart — are what separate a server meant to be hosted from a server meant to be run
by the player for an evening. A rebuild aiming at the former needs all four and the recipe records them as requirements rather
than features.
