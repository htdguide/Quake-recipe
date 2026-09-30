# WinQuake/net.h

> Two nested seams: a fifteen-call transport driver, and above it a five-call reliable message layer built out of unreliable datagrams.

**Needs** — [`common.h`](common.h.md) (the byte buffer) · [`cvar.h`](cvar.h.md)
**Used by** — [`net_main.c`](net_main.c.md) (the dispatcher) · [`net_dgrm.c`](net_dgrm.c.md), [`net_loop.c`](net_loop.c.md), [`net_vcr.c`](net_vcr.c.md) (the message-layer drivers) · [`net_udp.c`](net_udp.c.md), [`net_wins.c`](net_wins.c.md), [`net_wipx.c`](net_wipx.c.md), [`net_ipx.c`](net_ipx.c.md), [`net_ser.c`](net_ser.c.md), [`net_bw.c`](net_bw.c.md), [`net_mp.c`](net_mp.c.md), [`net_bsd.c`](net_bsd.c.md), [`net_wso.c`](net_wso.c.md), [`net_dos.c`](net_dos.c.md), [`net_comx.c`](net_comx.c.md) (the transport drivers) · [`sv_main.c`](sv_main.c.md) and [`cl_main.c`](cl_main.c.md) (consumers)
**Tier floor** — none for the interface; the layer above needs only datagrams

## Purpose

This file declares two layers and the distinction between them is the whole point.

The **transport driver** is [Seam: Unreliable datagram
transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport): fifteen
functions that move bytes between addresses with no guarantees. Eleven
implementations exist in this tree — two kinds of socket API, two kinds of IPX, a
direct serial cable, a modem, and a commercial network library.

The **message driver** is the engine's own reliability layer, and it is *not* a seam.
It is load-bearing logic: sequencing, acknowledgement, retransmission, fragmentation
and reassembly, all over datagrams, implemented in
[`net_dgrm.c`](net_dgrm.c.md). A rebuild must implement it, not shop for it, because
its exact behaviour is what a same-generation client and server agree on.

Two drivers of the second kind are not networks at all: a loopback driver that hands
the client's messages straight to the server in the same process, and a recording
driver that replays a captured session for debugging. That a single-player game goes
through the same message layer as a networked one — over loopback rather than around
it — is a design decision worth naming: it means there is only one code path to get
right.

## State

```text
RECORD SockAddr                 # an opaque transport address
  sa_family : int (16-bit)
  sa_data   : byte[14]

CONSTANT net_namelen     = 64
CONSTANT net_maxmessage  = 8192
CONSTANT net_headersize  = 8    # two 32-bit words
CONSTANT net_datagramsize = 1024 + 8
CONSTANT max_net_drivers = 8
```

**Invariants** — the address is a fixed 16-byte blob deliberately shaped like a 1996
socket address, and the engine never looks inside it: every operation on one goes
through the driver. That opacity is what lets IPX and a serial cable share the
interface. A rebuild should keep the opacity and can widen the blob freely.

### The reliable-message header

```text
# Every datagram the message layer sends begins with two 32-bit big-endian words:
#   word 0: flags in the high bits, length in the low 16 bits
#   word 1: a sequence number
CONSTANT netflag_length_mask = 0x0000ffff
CONSTANT netflag_data        = 0x00010000   # a reliable fragment
CONSTANT netflag_ack         = 0x00020000   # an acknowledgement
CONSTANT netflag_nak         = 0x00040000   # a negative acknowledgement
CONSTANT netflag_eom         = 0x00080000   # the last fragment of a message
CONSTANT netflag_unreliable  = 0x00100000   # an unreliable payload
CONSTANT netflag_ctl         = 0x80000000   # a connection-control packet
```

**Invariants** — the length field includes the header, and it is checked on receipt: a
datagram whose stated length disagrees with what arrived is discarded. The flags are
mutually exclusive in practice except that the end-of-message flag accompanies a data
flag.

The header is **big-endian**, unlike every other multi-byte value in the engine. That
is a deliberate choice for a network header and a rebuild must not normalize it.

### The connection protocol

```text
CONSTANT net_protocol_version = 3

# Requests, sent to a server's control port, often by broadcast:
CONSTANT ccreq_connect     = 0x01   # game name, protocol version
CONSTANT ccreq_server_info = 0x02   # game name, protocol version
CONSTANT ccreq_player_info = 0x03   # a player number
CONSTANT ccreq_rule_info   = 0x04   # a rule name, or empty to start enumerating

# Responses; note the high bit distinguishes them:
CONSTANT ccrep_accept      = 0x81   # the port to talk to from now on
CONSTANT ccrep_reject      = 0x82   # a reason string
CONSTANT ccrep_server_info = 0x83   # address, host name, level name,
                                    # current and maximum players, version
CONSTANT ccrep_player_info = 0x84   # number, name, colours, frags,
                                    # connect time, address
CONSTANT ccrep_rule_info   = 0x85   # a rule name and its value
```

**Invariants** — the acceptance response carries only a **port**, not an address, and
the header's own comment explains why: the address is "wherever this reply came from",
which offloads the multi-homed-host problem onto the operating system. The longer form
carrying a full address string exists only for reporting a *third* server's address
during discovery.

Rule enumeration works by asking for the rule *after* a given name, with an empty name
starting the walk — a cursor built out of a stateless request.

The game name field is always the literal `QUAKE`, and the header's comment says it is
there for a future game. A rebuild should still send and check it.

### The connection

```text
RECORD QSocket                  # one connection, either end
  connecttime, lastMessageTime, lastSendTime : real
  disconnected, canSend, sendNext : bool
  driver, landriver, socket : int         # which message driver, which
                                         # transport driver, which handle
  driverdata : opaque

  ackSequence, sendSequence, unreliableSendSequence : int
  sendMessageLength : int
  sendMessage       : byte[8192]          # the reliable message being sent,
                                          # held for retransmission
  receiveSequence, unreliableReceiveSequence : int
  receiveMessageLength : int
  receiveMessage    : byte[8192]          # the reliable message being reassembled

  addr    : SockAddr
  address : text[64]                      # a printable form
```

**Invariants** — **one outstanding reliable message per direction**, held in full for
retransmission, capped at 8192 bytes. That is the reliability model in one sentence: not
a window, not a queue — a single message, fragmented into datagram-sized pieces, each
acknowledged before the next is sent. The `canSend` flag is the interlock.

Reliable and unreliable traffic carry **separate sequence numbers**, and unreliable
datagrams arriving out of order are simply dropped by comparing against the last one
seen. A rebuild must keep both counters.

Both buffers are inline in the record, so a connection is 16 kilobytes. With eight
drivers and sixteen players that is a fixed cost the engine pays up front; connections
come from a free list rather than being allocated.

### The two driver records

```text
RECORD LanDriver                # the transport seam: 15 operations
  name : text ;  initialized : bool ;  controlSock : int
  Init, Shutdown
  Listen(enable)
  OpenSocket(port) -> handle ;  CloseSocket(handle)
  Connect(handle, addr)
  CheckNewConnections() -> handle
  Read(handle, buf, len) -> (count, from addr)
  Write(handle, buf, len, to addr) -> count
  Broadcast(handle, buf, len) -> count
  AddrToString(addr) -> text ;  StringToAddr(text) -> addr
  GetSocketAddr(handle) -> addr
  GetNameFromAddr(addr) -> text ;  GetAddrFromName(text) -> addr
  AddrCompare(a, b) -> int
  GetSocketPort(addr) -> int ;  SetSocketPort(addr, port)

RECORD NetDriver                # the message layer: 11 operations
  name : text ;  initialized : bool ;  controlSock : int
  Init ;  Shutdown
  Listen(enable)
  SearchForHosts(transmit)
  Connect(host) -> QSocket
  CheckNewConnections() -> QSocket
  QGetMessage(sock) -> int
  QSendMessage(sock, data) -> int
  SendUnreliableMessage(sock, data) -> int
  CanSendMessage(sock) -> bool ;  CanSendUnreliableMessage(sock) -> bool
  Close(sock)

VARIABLE net_landrivers : LanDriver[8] ;  net_numlandrivers : int
VARIABLE net_drivers    : NetDriver[8] ;  net_numdrivers    : int
VARIABLE net_driverlevel : int          # which message driver is being asked
```

**Invariants** — both are tables of function pointers, filled in at startup by the
platform build, and the engine loops over them. The loop is the load-bearing part: a
server listens on **every** available driver at once, and a client trying to connect
tries each in turn. So a single server serves loopback, two socket families and a
serial cable simultaneously, and that is why a listen server works for the local player
and remote ones through one code path.

A rebuild with one transport can collapse both tables, but should keep the *loopback*
message driver separate from the network one — that separation is what makes
single-player free of any socket.

## The public interface

### `NET_Init`, `NET_Shutdown`

**Contract** — initialize and release every available driver of both kinds. Determines
which transports exist and sets the three availability flags.

### `NET_CheckNewConnections`

**Contract** — polls every message driver for a pending inbound connection; returns one
connection or nothing.

### `NET_Connect`

**Contract** — takes a host name or address; tries each message driver in turn and
returns a connection or nothing. The literal name `local` selects loopback.

### `NET_GetMessage`

**Contract** — takes a connection; when data is available, leaves it in the global
incoming buffer and returns 1 for a reliable message, 2 for an unreliable one, 0 for
nothing available, and −1 for a dead connection.

**Invariants** — the decoded message lands in a **global** buffer, which is why only
one connection's message can be in flight through the decoder at a time. The client has
one connection so this is free; the server loops over players, fully handling each
message before fetching the next.

### `NET_SendMessage`, `NET_SendUnreliableMessage`

**Contract** — take a connection and a buffer. Return 1 on success, 0 when a reliable
message cannot be accepted *right now* because one is already outstanding but the
connection is still healthy, and −1 when the connection has died. The caller must retry
on 0.

### `NET_CanSendMessage`

**Contract** — reports whether a reliable send would be accepted. The server checks it
before composing a message, so that it does not build one it cannot send.

### `NET_SendToAll`

**Contract** — takes a buffer and a timeout in seconds; sends it reliably to every
connected player, **blocking** until each has accepted or the timeout expires. Used only
for the messages that must not be lost at a level change.

**Notes** — the only blocking call in the engine. A rebuild should keep it, because the
alternative — a client missing the level-change message — leaves that client stuck.

### `NET_Close`

**Contract** — takes a connection whose operations reported death, or one being
deliberately dropped; releases it. A connection's slot is not reused until this is
called.

### `NET_Poll`, `SchedulePollProcedure`

**Contract** — a tiny timer queue: a procedure with an argument and a due time, run by
`NET_Poll` when due. Used for the multi-stage server-discovery broadcast, which must
send, wait, and collect.

**Notes** — a two-entry cooperative scheduler, and the engine's only asynchronous
machinery. A rebuild with real timers or coroutines replaces it; a rebuild without them
needs exactly this.

### Host cache

```text
CONSTANT hostcachesize = 8
RECORD HostCacheEntry
  name, map, cname : text
  users, maxusers  : int
  driver, ldriver  : int
  addr : SockAddr
VARIABLE hostcache : HostCacheEntry[8] ;  hostCacheCount : int
```

**Contract** — what server discovery found: up to eight servers with their names, current
level, player counts and the driver pair to reach each by. The multiplayer menu reads
it.

### Discovery state and availability

```text
VARIABLE slistInProgress, slistSilent, slistLocal : bool
VARIABLE serialAvailable, ipxAvailable, tcpipAvailable : bool
VARIABLE my_ipx_address, my_tcpip_address : text
VARIABLE net_hostport, DEFAULTnet_hostport : int
VARIABLE hostname : Cvar
VARIABLE playername : text ;  playercolor : int
VARIABLE messagesSent, messagesReceived : int
VARIABLE unreliableMessagesSent, unreliableMessagesReceived : int
VARIABLE net_time : real ;  net_message : SizeBuf ;  net_activeconnections : int
```

### Serial and modem configuration

**Contract** — four handler variables through which the menu reads and writes a serial
port's hardware settings and a modem's dial strings, filled in by the serial transport
driver if present.

**Notes** — this is the same inversion as the video backend's menu page: irreducibly
platform-specific configuration contributed upward by the driver. A rebuild with no
serial transport deletes all four.
