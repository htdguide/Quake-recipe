# WinQuake/net_loop.c

> The loopback driver: hands the client's messages to the server in the same process through two byte rings, so single player uses the same code path as a network game.

**Needs** — [`net.h`](net.h.md) · [`net_loop.h`](net_loop.h.md) · [`common.h`](common.h.md)
**Used by** — [`net_main.c`](net_main.c.md) registers it as the first message driver
**Tier floor** — none

## Purpose

The reason single player in this engine is a client-server session. Two connections — the client's and the server's
— share two byte rings, and each writes into the ring the other reads. No socket, no serialization boundary, no
reliability layer: delivery is guaranteed and instantaneous.

That this exists is the single most important structural decision in the engine's networking: **there is only one
game loop to get right.**

## State

```text
VARIABLE loop_client, loop_server : QSocket        # the two ends
VARIABLE localconnectpending : bool
# Each end's inline message arrays are used as byte RINGS holding several
# length-prefixed messages, not as one message.
```

## `Loop_Connect`

**Contract** — takes a host name; returns the client's end of the loopback pair, creating both ends if a connection
is pending. Accepts only the name `local`. Clears both ends and cross-links them.

**Invariants** — the pair is created by the *server* side first, and this call collects the client's end, which is
why a pending flag mediates.

## `Loop_CheckNewConnections`

**Contract** — returns the server's end of the pair when a local connection is pending, clearing the flag.

## `Loop_SendMessage`, `Loop_SendUnreliableMessage`

**Contract** — append the message to the *other* end's receive ring, prefixed by its length and a kind byte. Fatal
error if the ring cannot hold it.

```text
FUNCTION loop_send_message(sock, data) -> int
  buffer = the OTHER end's receive ring
  IF it cannot hold 4 + the message  FAIL WITH "overflow"
  write the length as a 32-bit value ;  write the kind byte (1 or 2)
  append the message, ALIGNED up to a four-byte boundary
  sock.canSend = true                     # always: there is no flow control
  RETURN 1
```

**Invariants** — **the ring holds several messages at once**, each length-prefixed, unlike a network connection's
single outstanding message. So the loopback has no flow control and the sender is never blocked — which is why the
can-send query is unconditionally yes and why a local server never buffers reliable data across frames.

## `Loop_GetMessage`

**Contract** — takes the first message from this end's receive ring into the global incoming buffer and returns its
kind. Returns nothing when the ring is empty.

**Invariants** — messages come out in order, one per call, which makes the ring a reliable ordered stream for free.

## `Loop_CanSendMessage`, `Loop_CanSendUnreliableMessage`

**Contract** — both always report yes.

## `Loop_Init`, `Loop_Shutdown`, `Loop_Listen`, `Loop_SearchForHosts`, `Loop_Close`

**Contract** — initialize and shut down trivially; listening is a no-op; discovery reports the local server itself
into the host cache when one is running; closing clears both ends.

**Invariants** — **discovery reporting the local server** is what makes a listen server appear in its own browser,
which is how the multiplayer menu shows a game you are hosting.

## `IntAlign`

**Contract** — rounds a length up to a four-byte multiple.
