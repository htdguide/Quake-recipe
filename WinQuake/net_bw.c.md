# WinQuake/net_bw.c

> A vendor TCP stack as a datagram transport: the same interface reached by writing request structures to a character device.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net.h`](net.h.md) · [`net_bw.h`](net_bw.h.md) · [`dosisms.h`](dosisms.h.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`net_dos.c`](net_dos.c.md) lists it as a transport, when built for it
**Tier floor** — none

## Purpose

The third filling of the datagram interface with an IP address family, and the most alien: there is no socket call at
all. Every operation is a structure written to an open device, whose reply is read back from the same structure.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs from the reference driver

**One operation, one message.** Open, close, bind, read, write and the address queries are each a request record with
a command code, handed to the device; the device fills in a result code and any output in place.

```text
FUNCTION request(op, payload) -> result
  fill a request record: command = op, fields = payload
  write the record to the device
  RETURN the record's result field, translated to this engine's error codes
```

**Errors must be translated**, because the device's codes share nothing with the sockets codes the rest of the driver
family reports. That translation table is the file's only real content, and it exists because the layer above
distinguishes exactly three outcomes: bytes, nothing-yet, and dead.

**The local address comes from querying the interface list**, since the device has no notion of the host's name.

**Notes** — worth one page for the same reason as [`net_ipx.c`](net_ipx.c.md): it shows the transport interface
survives a transport with no shared vocabulary whatsoever. Nothing in it needs rebuilding.
