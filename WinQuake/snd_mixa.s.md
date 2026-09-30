# WinQuake/snd_mixa.s

> Two audio mixing loops: accumulate one eight-bit sound into the paint buffer, and convert the accumulator to sixteen-bit stereo output.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_i386.h`](asm_i386.h.md) (the channel and sound layouts) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`snd_mix.c`](snd_mix.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Reads the channel, decoded-sound and sample-pair layouts of [`asm_i386.h`](asm_i386.h.md).

## `SND_PaintChannelFrom8`

**Contract** — as [`snd_mix.c`](snd_mix.c.md#snd_paintchannelfrom8-snd_paintchannelfrom16): for a run of samples, add one
channel's contribution to the stereo accumulator, scaling each sample by the channel's two ear volumes
through the precomputed scale table.

**Invariants** — the scale table ([`sound.h`](sound.h.md#snd_initscaletable)) turns a volume and a
sample into a contribution with one indexed load, which is why there is no multiply in this loop.

## `Snd_WriteLinearBlastStereo16`

**Contract** — as [`snd_mix.c`](snd_mix.c.md): convert a run of wide accumulator pairs into
sixteen-bit signed output, **clamping** rather than wrapping on overflow.

**Invariants** — the clamp is load-bearing: the accumulator is wider than the output precisely so that
several loud sounds can sum without wrapping, and wrapping instead of clamping produces a loud click.

**Notes** — the only non-graphics member of the assembly seam.
