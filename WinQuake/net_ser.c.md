# WinQuake/net_ser.c

> A reliable message channel over a raw byte stream: framed messages with an escape-marked end, a checksum, one-byte sequence numbers and an acknowledge, decoded by a byte-at-a-time state machine.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net.h`](net.h.md) · [`net_ser.h`](net_ser.h.md) · [`crc.h`](crc.h.md) · [`net_comx.c`](net_comx.c.md) · [`dosisms.h`](dosisms.h.md)
**Used by** — [`net_win.c`](net_win.c.md) and [`net_dos.c`](net_dos.c.md) list it as a message driver
**Tier floor** — none

## Purpose

The other half of the answer that [`net_dgrm.c`](net_dgrm.c.md) gives. Both deliver the same service — ordered
reliable messages plus unreliable ones, one reliable message outstanding at a time
([`net.h`](net.h.md)) — but a datagram channel already has message boundaries and this one does not. So this file is
worth reading as the *framing* problem stated cleanly: how to turn a byte pipe into a message channel, with error
recovery, in about four hundred lines.

It includes [`net_comx.c`](net_comx.c.md) textually, so the driver and the hardware layer are one translation unit.

## State

```text
CONSTANT serial_protocol_version = 3
CONSTANT mtype_reliable = 1 ; mtype_unreliable = 2 ; mtype_control = 3 ; mtype_ack = 4
CONSTANT mtype_client = 0x80        # OR'd into the type by the client end
CONSTANT escape_command = 0xE0 ;  escape_eom = 0x19

RECORD SerialLine                   # one per port
  sock          : Connection
  tty           : port handle
  lengthStated, lengthFound : int
  crcStated, crcValue       : 16-bit
  currState, prevState      : state
  mtype, sequence           : byte
  connected, connecting, client : bool
  connect_time  : real
VARIABLE serialLine[ports] ;  listening : bool
```

**Invariants** — the decoder's state is **per line and persists between calls**, because a message arrives across many
frames. Everything about the file follows from that: it can never assume a whole message is present.

## The wire format

```text
reliable / unreliable :  type  sequence  length(16)  data...  crc(16)  EOM
acknowledge           :  type  sequence  crc(16)  EOM
control               :  type  length(16)  data...  crc(16)  EOM
```

**Invariants** —

- Multi-byte fields are **big-endian**, matching [`net_dgrm.c`](net_dgrm.c.md) and unlike the game protocol
  ([`common.c`](common.c.md)). Control information is network order; game payload is little-endian.
- The **checksum covers the whole message except itself**, and it is the cyclic checksum of
  [`crc.h`](crc.h.md) — the same one the file system uses, reused rather than a second polynomial.
- **The end of a message is marked, not merely implied by the length.** An escape byte followed by an
  end-marker byte terminates. A length field alone would be unrecoverable: one lost byte and the decoder would
  consume the next message as payload forever. With a marker the decoder resynchronizes at the next boundary.
- Any payload byte equal to the escape value is **doubled**, which is the standard cost of an in-band marker:
  the marker can then never appear inside a message.
- The client end **sets a bit in the message type**, so an end that receives its own echo — a real hazard on a
  loopback-capable serial line — recognizes and drops it.
- Control messages are **neither sequenced nor acknowledged**, because they only establish the session.

## `Serial_SendMessage`, `ReSendMessage`

**Contract** — frame a reliable message onto the line: type with the client bit, sequence, length, escaped payload,
checksum, end marker; flush; then mark the channel busy, copy the message into the connection's retransmit slot and
stamp the send time. Always reports success. Re-sending replays the saved copy through the same path.

**Invariants** — **the channel is marked busy until the acknowledge arrives**, which is the one-outstanding-message
rule the layer above depends on ([`net_main.c`](net_main.c.md#net_sendtoall)).

The checksum is accumulated over the *unescaped* bytes while the escaped bytes go to the line, so the receiver must
un-escape before checking. Getting that order wrong is the obvious rebuild bug.

## `Serial_SendUnreliableMessage`

**Contract** — the same framing with the unreliable type, no copy kept and no busy flag. Reports success.

**Invariants** — it is dropped if the output queue is not empty (see the can-send query), rather than queued, because
a stale movement update is worth less than a fresh one.

## `Serial_SendACK`, `Serial_SendControlMessage`

**Contract** — frame the two short message kinds: an acknowledge carrying just the sequence being acknowledged, and a
control message carrying a payload with no sequence.

## `ProcessInQueue`

**Contract** — drain every byte available on the port through the decoder, dispatching each complete message: an
acknowledge frees the channel, a reliable message in sequence is delivered and acknowledged, a duplicate is
acknowledged and dropped, an unreliable message is delivered if not older than the last seen, a control message goes
to the connection logic. Reports whether a message was delivered.

```text
FUNCTION decode(line, byte)
  IF the previous state was escape
    IF byte is the end marker
      IF everything expected was received AND the checksum matches  DELIVER
      ELSE  reset to ready                      # resynchronized, message lost
    ELSE IF byte equals the escape value        # a doubled escape
      fall through and treat it as data
    ELSE  reset to ready                        # unknown escape: abort
  ELSE IF byte is the escape value
    remember the state and enter escape
  ELSE advance the field machine:
    ready    -> read the type, start the checksum, pick the next field by type
    sequence -> read it
    length   -> accumulate two bytes, big-endian
    data     -> append, until the stated length is reached
    crc      -> accumulate two bytes; then expect the escape/end pair
```

**Invariants** — **the escape state is entered from any field**, so a message can be terminated mid-payload, which is
exactly the recovery path. The saved previous state is what lets a doubled escape resume the field it interrupted.

A failed checksum **is not retransmitted from here**: the message is simply dropped, and the sender's timer
([`net_main.c`](net_main.c.md)) resends it. That keeps the receiver stateless about failure.

An out-of-sequence reliable message is **acknowledged anyway** before being dropped, because the likely cause is a
lost acknowledge rather than a lost message — the same reasoning as
[`net_dgrm.c`](net_dgrm.c.md#datagram_getmessage).

## `Serial_GetMessage`, `_Serial_GetMessage`

**Contract** — drain the port, then report the kind of message delivered, or nothing. Also drives the retransmit
timer: if the channel is busy and the send is older than the retry interval, resend.

**Invariants** — retransmission is driven by **reads, not by a clock**, so a connection nobody polls never retries.
That is acceptable because the frame loop always polls.

## `Serial_Connect`, `_Serial_Connect`

**Contract** — takes a host name, which for this driver is a phone number or a bare port request; enables the port,
dials if a modem, then exchanges control messages carrying the protocol version until the far end agrees; returns a
connection or nothing. Reports progress to the player and can be cancelled by a key press.

**Invariants** — **the version is negotiated in the control exchange and a mismatch refuses the connection**, which is
the only version check in the whole network layer — the datagram driver checks the game protocol version instead.

Dialling can take thirty seconds, so this is the only connect path that reports progress and accepts a cancel. A
rebuild that keeps a slow transport needs the same.

## `Serial_CheckNewConnections`, `_Serial_CheckNewConnections`

**Contract** — for each enabled port while listening: answer an incoming ring or accept a raw connection, then wait
for the far end's control message, reply with agreement, and hand back a connection.

## `Serial_SearchForHosts`

**Contract** — reports nothing found.

**Invariants** — discovery is meaningless on a point-to-point line; the player must supply the number. The operation
exists only to fill the interface.

## `Serial_CanSendMessage`, `Serial_CanSendUnreliableMessage`, `Serial_Close`, `Serial_Init`, `Serial_Shutdown`, `Serial_Listen`, `ResetSerialLineProtocol`

**Contract** — report the busy flag; report whether the hardware output queue has drained; hang up and reset the line;
open every port at startup and close every connection at shutdown; set the listening flag; and clear a line's decoder
state.

**Invariants** — the *unreliable* can-send query asks the **hardware queue**, not the protocol, because an unreliable
message is worth sending only if it will go out now. The reliable one asks the protocol. Two different questions
sharing a name is the one confusing part of this interface.
