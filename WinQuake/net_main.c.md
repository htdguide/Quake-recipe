# WinQuake/net_main.c

> The network dispatcher: a free list of connections, a loop over every available driver for each public operation, and the multi-stage server-discovery broadcast driven by a two-entry timer queue.

**Needs** — [`net.h`](net.h.md) · [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`server.h`](server.h.md) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md) · [`console.h`](console.h.md) · [`zone.h`](zone.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`sv_main.c`](sv_main.c.md), [`cl_main.c`](cl_main.c.md), [`host.c`](host.c.md) and [`menu.c`](menu.c.md) call its public operations
**Tier floor** — none

## Purpose

The layer above the message drivers. Its structure is one idea applied to every operation: **loop over every driver
that initialized successfully.** A server listens on all of them at once; a client connecting tries each in turn.
That is what makes a listen server serve the local player over loopback and remote players over a socket through one
code path.

## State

```text
VARIABLE net_activeSockets, net_freeSockets : QSocket    # two linked lists
VARIABLE net_numsockets : int
VARIABLE net_drivers, net_numdrivers                     # the message drivers
VARIABLE net_landrivers, net_numlandrivers               # the transport drivers
VARIABLE net_driverlevel : int                 # which driver is being asked
VARIABLE net_time : real ;  net_message : SizeBuf
VARIABLE hostcache, hostCacheCount             # discovery results
VARIABLE slistInProgress, slistSilent, slistLocal : bool
VARIABLE the poll-procedure queue
```

**Invariants** — connections come from a **free list built at startup**, one per player slot plus a few, because
each is sixteen kilobytes ([`net.h`](net.h.md)) and allocating them dynamically would fragment the hunk.

`net_driverlevel` is a global naming the driver currently being invoked, which is how a driver's own code learns
which slot it occupies. A rebuild should pass it.

## `NET_Init`, `NET_Shutdown`

**Contract** — allocate the connection pool, initialize every message driver and every transport driver recording
which succeeded, read the port from the command line, and register the network commands and variables; and shut every
initialized driver down, closing every active connection first.

## `NET_NewQSocket`, `NET_FreeQSocket`

**Contract** — take a connection from the free list, clearing it and recording the connect time; and return one to
the free list, removing it from the active list.

**Invariants** — a connection's slot is **not reusable until it is explicitly freed**
([`net.h`](net.h.md#net_close)), which is what lets a dead connection be detected and reported before its storage is
recycled.

## `SetNetTime`

**Contract** — samples the platform clock into the network layer's own time.

**Invariants** — the network layer keeps its **own** clock rather than using the frame time, because timeouts and
retransmission must advance even when the frame loop stalls.

## `NET_Connect`

**Contract** — takes a host name; tries each message driver in turn until one returns a connection. The literal name
`local` restricts the attempt to the loopback driver. Reports failure.

```text
FUNCTION net_connect(host) -> optional<QSocket>
  set_net_time()
  IF host names the loopback  try ONLY the loopback driver
  ELSE
    IF a discovery is in progress, wait for it
    FOR EACH initialized message driver
      net_driverlevel = its index
      sock = its connect operation
      IF it succeeded  RETURN sock
  RETURN nothing
```

**Invariants** — **the loopback driver is tried by name**, which is how single-player avoids touching a socket
([`host_cmd.c`](host_cmd.c.md#host_map_f-map) connects to `local`).

## `NET_CheckNewConnections`

**Contract** — polls every message driver for a pending inbound connection; returns the first. Sets the driver level
so the connection records which driver owns it.

## `NET_GetMessage`, `NET_SendMessage`, `NET_SendUnreliableMessage`, `NET_CanSendMessage`, `NET_Close`

**Contract** — dispatch to the connection's own driver, updating the network clock and the message counters.
`NET_Close` also frees the connection.

**Invariants** — each is a one-line dispatch plus statistics, which is the whole of the abstraction's cost.

## `NET_SendToAll`

**Contract** — takes a buffer and a timeout; sends it reliably to every connected player, **blocking** until each has
accepted or the timeout expires. Reads incoming messages while waiting. Returns how many did not receive it.

```text
FUNCTION net_send_to_all(data, blocktime) -> int
  FOR EACH active client  mark it as needing the message
  start = now
  WHILE any client still needs it AND now - start < blocktime
    FOR EACH client still needing it
      IF its channel will accept a message
        send it ;  mark it done
      ELSE
        read a message from it            # lets the reliable layer acknowledge
  RETURN how many are still marked
```

**Invariants** — **reading while waiting is what makes this terminate.** The reliable layer only frees its
outstanding message on receiving an acknowledgement ([`net_dgrm.c`](net_dgrm.c.md#datagram_getmessage)), and the
acknowledgement arrives through a read. A loop that only wrote would spin for the whole timeout. The same pattern
appears in the server shutdown ([`host.c`](host.c.md#host_shutdownserver)).

This is the only blocking network call in the engine, used for the messages that must not be lost at a level change.

## `NET_Poll`, `SchedulePollProcedure`

**Contract** — a small timer queue: a procedure with an argument and a due time, run when due, in due order.

**Invariants** — the engine's only asynchronous machinery, and it exists for exactly one purpose: the discovery
broadcast must send, wait, and collect. A rebuild with timers or coroutines replaces it.

## `NET_Slist_f`, `Slist_Send`, `Slist_Poll`, and the three printing helpers

**Contract** — the server-browser command: clears the host cache, schedules a broadcast and then repeated polls, and
prints the results as they arrive.

```text
FUNCTION net_slist_f()
  IF a discovery is already in progress  RETURN
  clear the host cache ;  slistInProgress = true
  schedule slist_send immediately
  schedule slist_poll shortly after
  print the header

FUNCTION slist_send()
  FOR EACH message driver  its search operation, TRANSMITTING
  IF the time budget is not exhausted  schedule slist_send again

FUNCTION slist_poll()
  FOR EACH message driver  its search operation, NOT transmitting  # collect only
  print any new entries
  IF the budget is not exhausted  schedule slist_poll again
  ELSE  print the trailer ;  slistInProgress = false
```

**Invariants** — the search operation takes a flag saying whether to *transmit* or merely *collect*, which is how
one operation serves both phases. The send phase repeats because a broadcast may be lost; the poll phase repeats
because replies arrive over time.

## `NET_Listen_f`, `MaxPlayers_f`, `NET_Port_f`

**Contract** — the console commands to start or stop listening, to change the player limit, and to change the port.
Changing the limit while a server is running restarts the level; changing the port stops and restarts listening.

**Invariants** — the player limit is clamped to the allocated slot count
([`host.c`](host.c.md#host_findmaxclients)), which cannot grow — so the command can lower it and can raise it only
to the startup allocation.
