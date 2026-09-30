# WinQuake/snd_dos.c

> A sound-card driver: programs a SoundBlaster's digital signal processor and the machine's direct-memory-access controller by hand, and reads the transfer's remaining count to derive the consumed position.

**Needs** — [`sound.h`](sound.h.md) · [`dosisms.h`](dosisms.h.md) · direct hardware port access
**Used by** — [`snd_dma.c`](snd_dma.c.md) through [Seam: Audio output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)
**Tier floor** — T0; it programs hardware registers and a transfer controller directly

## Purpose

Not an audio backend in any modern sense: it *is* the driver. It discovers the card's address, interrupt and
transfer channel from an environment variable, resets and programs the signal processor, sets up an auto-initializing
transfer from a buffer below the sixteen-megabyte line, and reads the controller's remaining count to work out how
far playback has reached.

Nothing here survives into a rebuild. It is preserved because it documents what the seam's "report the consumed
position" requirement looked like on hardware that offered nothing else.

## State

```text
VARIABLE the card's base port, interrupt number and transfer channel
VARIABLE the transfer buffer, allocated below the addressing limit
```

## `GetBLASTER`

**Contract** — parses the card's configuration from an environment variable of the era — base address, interrupt,
eight-bit and sixteen-bit transfer channels — and reports what it found.

## `ResetDSP`, `ReadDSP`, `WriteDSP`, `ReadMixer`, `WriteMixer`

**Contract** — the card's register protocol: reset it and wait for the acknowledgement byte; read and write the
signal processor's data port with the required busy-wait on its status port; and read and write the mixer's
indexed registers.

**Invariants** — each is a port write followed by a status poll. The reset sequence's acknowledgement byte is how
presence is detected.

## `StartSB`, `StartDMA`

**Contract** — program the signal processor for the chosen rate and width and start an auto-initializing transfer;
and program the machine's transfer controller with the buffer's physical address, length and mode.

**Invariants** — **auto-initializing** means the controller restarts the transfer at the buffer's beginning when it
reaches the end, which is what makes the buffer a ring with no software involvement. That is the hardware
equivalent of the seam's continuously-consuming device.

The buffer must lie below the sixteen-megabyte line and must not cross a 64-kilobyte boundary, because the
controller's address register is segmented — a constraint on where the buffer is allocated.

## `SNDDMA_Init`

**Contract** — discovers the card, allocates a conforming buffer, programs everything, and fills in the device
record. Returns whether it succeeded.

## `SNDDMA_GetDMAPos`

**Contract** — reads the transfer controller's remaining-count register and converts it into a position within the
buffer.

**Invariants** — the count **counts down**, so the position is the buffer's length minus it. The register must be
read twice and compared, because it is latched in two halves and can be read mid-update — which is the classic
hazard of this hardware and is why the read is not a single operation.

## `SNDDMA_Submit`, `SNDDMA_Shutdown`, `SB_Info_f`, `PrintBits`

**Contract** — nothing to submit; stop the transfer and reset the card; and two diagnostics reporting the card's
configuration.
