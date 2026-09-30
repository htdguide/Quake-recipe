# WinQuake/snd_win.c

> The Win32 audio backend: tries a direct memory-mapped device first and falls back to a queued-buffer interface, exposing the same four-call seam either way.

**Needs** — [`sound.h`](sound.h.md) · [`winquake.h`](winquake.h.md) · the platform's audio interfaces
**Used by** — [`snd_dma.c`](snd_dma.c.md) through [Seam: Audio output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)
**Tier floor** — T2; it needs a device whose consumed position is readable

## Purpose

One of seven implementations of the four-call audio seam, and the most instructive because it implements the same
contract two ways: a **direct** mode where the engine writes into the device's own ring and reads its play cursor,
and a **queued** mode where the engine writes into buffers it hands over and infers the position from how many have
been returned.

The queued mode is the interesting one, because it shows how to satisfy "report the consumed position" on a device
that does not offer it.

## State

```text
VARIABLE the direct device and its buffer, or the queued device and a ring of
         buffer headers
VARIABLE snd_sent, snd_completed : int      # queued mode: buffers handed over
                                            # and returned
VARIABLE wav_buffers, wav_samples : int
```

## `SNDDMA_Init`

**Contract** — tries the direct device; on failure or when the command line forbids it, tries the queued one;
reports which it got. Fills in the device record with the negotiated rate, width, channel count and ring, and a
submission granularity.

**Invariants** — the direct mode is preferred because it needs no copy. The command line can force the fallback,
which is what a player with a broken driver does.

The negotiated format is written into the shared record
([`sound.h`](sound.h.md)) and the engine reads it back rather than assuming.

## `SNDDMA_InitDirect`

**Contract** — opens the device in the cooperative mode that allows direct buffer access, creates a ring of the
requested size, locks it to obtain an address, starts it looping, and records the address as the device's ring.

**Invariants** — the ring is **started looping immediately and never stopped**, which is what makes the play cursor
a continuously advancing position. That is the seam's central requirement
([Seam: Audio output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)).

## `SNDDMA_InitWav`

**Contract** — opens the queued device, allocates a fixed number of equally-sized buffers, and presents a
*synthetic* ring: the engine writes into a contiguous block that the backend then slices into buffers and submits.

**Invariants** — the synthetic ring is the trick. The engine believes it is writing into one continuous ring; the
backend hands out slices of it as they are filled. That is how a queued device satisfies a memory-mapped
interface.

## `SNDDMA_GetDMAPos`

**Contract** — returns the device's consumed position in mono samples. In direct mode, reads the play cursor and
converts. In queued mode, derives it from the number of buffers the device has returned.

```text
FUNCTION snddma_get_dma_pos() -> int
  IF direct mode
    read the play cursor in bytes
    RETURN (cursor / the sample width) MODULO the ring's length
  ELSE
    # Count the buffers the device has finished with; each is a known size.
    RETURN (snd_completed * wav_samples) MODULO the ring's length
```

**Invariants** — the queued mode's position advances **one buffer at a time**, so its granularity is a whole
buffer rather than a sample. The engine tolerates that because it only ever compares the position against its own
write cursor; but the effective latency is at least one buffer.

## `SNDDMA_Submit`

**Contract** — in direct mode, does nothing. In queued mode, hands over every buffer the engine has filled since
the last call and reclaims the ones the device has returned.

**Invariants** — that the direct mode's submission is empty is what makes the seam's fourth call optional in
practice, and why other backends leave it empty too.

## `S_BlockSound`, `S_UnblockSound`

**Contract** — silence and restore the device when the application loses and regains focus, using a nesting count.

**Invariants** — the count is why it nests ([`sound.h`](sound.h.md)); several independent reasons to silence can
overlap.

## `FreeSound`, `SNDDMA_Shutdown`

**Contract** — release the buffers and the device.
