# WinQuake/snd_mix.c

> The mixer: accumulates every active channel into a wide stereo buffer in 512-sample blocks, then converts and clamps that buffer into the device's format.

**Needs** — [`sound.h`](sound.h.md) · [`quakedef.h`](quakedef.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) ([`snd_mixa.s`](snd_mixa.s.md))
**Used by** — [`snd_dma.c`](snd_dma.c.md) drives it
**Tier floor** — none

## Purpose

Two stages: **paint** every channel into a wide accumulator, then **transfer** the accumulator into the ring in
the device's format. The separation exists so that summation can happen at a wider precision than the output and
be clamped once.

## State

```text
CONSTANT paintbuffer_size = 512                # sample pairs per block
VARIABLE paintbuffer : SamplePair[512+...]     # the accumulator
VARIABLE snd_scaletable : int[32][256]         # volume by sample value
VARIABLE snd_vol : int                         # the master volume, as an
                                               # integer scale
```

**Invariants** — the accumulator is **512 sample pairs**, and mixing proceeds in blocks of that size regardless of
how far ahead the target is. So a long mix is several blocks, each painted and transferred.

## `SND_InitScaletable`

**Contract** — builds a table giving, for each of 32 volume levels and each of 256 sample values, the scaled
contribution.

```text
FUNCTION snd_init_scaletable()
  FOR EACH volume level i IN 0..31
    FOR EACH sample value j IN 0..255
      # The sample is 8-bit UNSIGNED, so subtract 128 to centre it; the volume
      # index is the top five bits of a 0..255 volume.
      snd_scaletable[i][j] = (j - 128) * i * 8
```

**Invariants** — **32 volume levels, not 256**, so a channel's volume is quantized to the top five bits. That is
the quantization that makes the table 32 kilobytes rather than 256, and it is why a very slow fade steps audibly.

The factor of 8 compensates for the five-bit index. The subtraction of 128 converts the stored unsigned sample to a
signed one, which is why the table is indexed by the raw byte.

## `SND_PaintChannelFrom8`, `SND_PaintChannelFrom16`

**Contract** — add a run of one channel's samples into the accumulator, scaled by its two ear volumes. The
eight-bit form uses the scale table; the sixteen-bit form multiplies directly. Advances the channel's position.

```text
FUNCTION snd_paint_channel_from_8(ch, sc, count)
  lscale = snd_scaletable[ch.leftvol  SHIFTED RIGHT 3]     # top five bits
  rscale = snd_scaletable[ch.rightvol SHIFTED RIGHT 3]
  samples = sc.data + ch.pos
  FOR EACH i IN 0..count-1
    paintbuffer[i].left  = paintbuffer[i].left  + lscale[samples[i]]
    paintbuffer[i].right = paintbuffer[i].right + rscale[samples[i]]
  ch.pos = ch.pos + count

FUNCTION snd_paint_channel_from_16(ch, sc, count)
  # No table: a 16-bit sample would need a 64K-entry table per volume.
  FOR EACH i
    paintbuffer[i].left  += (samples[i] * ch.leftvol)  SHIFTED RIGHT 8
    paintbuffer[i].right += (samples[i] * ch.rightvol) SHIFTED RIGHT 8
  ch.pos = ch.pos + count
```

**Invariants** — the eight-bit path is **two indexed loads and two additions per sample** with no multiply, which
is why the table exists. The sixteen-bit path cannot use a table and multiplies — so sixteen-bit sounds are more
expensive to mix, which is why the engine can be told to load everything as eight-bit
([`sound.h`](sound.h.md)).

**Accumulation, not replacement.** Every channel adds, and the accumulator is wide enough that several loud sounds
can sum without wrapping — which is what the transfer's clamp then resolves.

## `S_PaintChannels`

**Contract** — mixes from the current position to a target, in blocks: clears the accumulator, paints every active
channel into it, handling loop points and channel completion, then transfers the block and advances.

```text
FUNCTION s_paint_channels(endtime)
  WHILE paintedtime < endtime
    end = min(endtime, paintedtime + 512)
    zero the accumulator for (end - paintedtime) pairs

    FOR EACH channel
      IF it has no sound OR both ear volumes are zero  CONTINUE
      sc = its decoded samples, RELOADING them if the cache dropped them
      IF none  CONTINUE
      ltime = paintedtime
      WHILE ltime < end
        count = min(ch.end, end) - ltime
        IF count > 0
          paint `count` samples, choosing the 8- or 16-bit painter by width
          ltime = ltime + count
        IF ltime reached ch.end                   # the sound ran out
          IF sc.loopstart >= 0
            ch.pos = sc.loopstart                 # LOOP
            ch.end = ltime + sc.length - ch.pos
          ELSE
            ch.sfx = nothing ;  BREAK             # the channel stops
    s_transfer_paint_buffer(end)
    paintedtime = end
```

**Invariants** — four things.

**The decoded samples are reloaded if the cache dropped them**, inside the mix loop
([`zone.h`](zone.h.md)). So a sound evicted between frames is silently reloaded, and the mixer never sees a missing
sound.

**Looping is handled here, not by the device.** A sound with a loop point rewinds to it and extends its end time,
so a looping sound never stops. That is what makes ambient and static sounds work
([`snd_dma.c`](snd_dma.c.md#s_staticsound)).

**A channel is freed by clearing its sound**, which is the only place a dynamic channel ends naturally.

**The inner loop handles a channel ending mid-block** by painting up to its end and then either looping or
stopping — so a block can contain the tail of one sound and the head of its loop.

## `S_TransferPaintBuffer`, `S_TransferStereo16`, `Snd_WriteLinearBlastStereo16`

**Contract** — convert the accumulator into the device's ring: scale by the master volume, clamp to the output
range, convert to the device's width and channel count, and write at the correct position with wrapping. The
sixteen-bit stereo case has a dedicated path.

```text
FUNCTION s_transfer_paint_buffer(endtime)
  IF the device is 16-bit stereo  s_transfer_stereo_16(endtime) ;  RETURN
  # The general case: 8-bit, or mono, or both.
  FOR EACH pair FROM paintedtime TO endtime-1
    val = (the accumulated left) * snd_vol, SHIFTED RIGHT 8
    CLAMP val INTO the output range                     # clamp, do NOT wrap
    write it at (position MODULO the ring's length)
    ...the same for the right channel when the device is stereo

FUNCTION snd_write_linear_blast_stereo_16()
  FOR EACH pair
    val = (left * snd_vol) SHIFTED RIGHT 8
    CLAMP INTO -32768 .. 32767
    store it ;  ...the same for right
```

**Invariants** — three things.

**The clamp is essential and must not be a wrap.** The accumulator is wider than the output precisely so that
several sounds can sum past the output's range; wrapping instead of clamping turns a loud moment into a loud
click. That is the one behaviour a rebuild must not get wrong in this file.

**The master volume is applied at transfer, not at paint**, so changing it does not invalidate anything already
mixed — it takes effect from the next block.

**The write wraps at the ring's length**, which is why the transfer is split into at most two runs. The dedicated
sixteen-bit stereo path exists because that is the common case and it can write two samples per store.

An eight-bit device's output is **unsigned**, offset by 128, which the general path handles and the dedicated one
does not.
