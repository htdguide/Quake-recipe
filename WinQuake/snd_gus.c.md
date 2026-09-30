# WinQuake/snd_gus.c

> A second sound-card driver, for a card with on-board sample memory — including its own miniature parser for the card's configuration file.

**Needs** — [`sound.h`](sound.h.md) · [`dosisms.h`](dosisms.h.md) · direct hardware access · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`snd_dma.c`](snd_dma.c.md) through [Seam: Audio output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)
**Tier floor** — T0

## Purpose

The largest of the audio backends and the only one that ships a **configuration file parser**: this card's settings
lived in a text file with sections and fields, and the driver reads them itself. Two thirds of the file is that
parser.

Like its sibling, nothing here survives into a rebuild. What is worth recording is why it exists: the card's ring
lives in **on-board memory**, not host memory, so the engine cannot write into it directly — and the driver bridges
that by keeping a host buffer and copying.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## The configuration parser

**Contract** — `add_section`, `add_field`, `is_section`, `is_field`, `get_section_name`, `get_field_name`,
`get_field_string`, `stripped_fgets`, `reset_buffer` and a local case-folding helper together read a
sections-and-fields text file and build an index of it, so that a named field in a named section can be looked up.

**Invariants** — a self-contained parser for a format nothing else in the engine reads. A rebuild deletes it
entirely.

## The device interface

**Contract** — the four seam calls: discover and program the card from the parsed configuration; report the
consumed position from the card's own play counter; copy the host buffer's newly-written region into the card's
memory on submission; and stop and release it.

**Invariants** — **submission is a real copy here**, unlike every other backend where it is empty. The card's ring
is not host-addressable, so the engine's writes must be transferred — which is exactly the case the seam's fourth
call exists for ([Seam: Audio
output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)). This file is the reason that call is in the interface at all.
