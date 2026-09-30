# WinQuake/snd_mem.c

> Loads a sound: parses the container format, extracts the loop point, and resamples to the device's rate into the evictable cache.

**Needs** — [`sound.h`](sound.h.md) · [`common.h`](common.h.md) · [`zone.h`](zone.h.md) · [`console.h`](console.h.md)
**Used by** — [`snd_dma.c`](snd_dma.c.md) and [`snd_mix.c`](snd_mix.c.md) call the loader
**Tier floor** — none

## Purpose

The asset path for sound. Two things matter: the **loop point** comes from an optional chunk of the container and
is what makes ambient sounds work, and the **resampling** is nearest-neighbour with a fixed-point step — which is
why the game's sounds are gritty at any rate other than the one they were authored at.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## `GetWavinfo`

**Contract** — takes a name and a raw file; walks the container's chunks and returns the rate, sample width,
channel count, sample count, the loop point and the data offset. Reports and returns nothing for a file that is
not the expected container, has no format chunk, is compressed, or has no data chunk.

```text
FUNCTION get_wavinfo(name, wav, length) -> WavInfo
  IF the first four bytes are not the container magic  print and RETURN nothing
  find the format chunk ;  IF absent  print "missing fmt chunk" ;  RETURN
  IF the format is not uncompressed  print "Microsoft PCM format only" ;  RETURN
  channels = a short ;  rate = a long ;  width = a short / 8

  # The LOOP POINT lives in an optional cue chunk.
  find the cue chunk
  IF present
    loopstart = the cue's sample offset
    # An optional label chunk after it gives the loop's LENGTH.
    find the next chunk ;  IF it is a label chunk
      samples = loopstart + its recorded length
  ELSE
    loopstart = -1

  find the data chunk ;  IF absent  print "missing chunk" ;  RETURN
  samples = min(the data chunk's sample count, samples if a loop set one)
  dataofs = where the data begins
```

**Invariants** — three things.

**The loop point is a cue chunk and the loop length an optional label chunk after it.** That is an unusual use of
those chunks — they are meant for markers and annotations — and it is the convention this engine's asset pipeline
adopted. A rebuild must read the same chunks or every ambient sound plays once and stops.

**Only uncompressed samples are accepted**, and anything else is refused with a message rather than
misinterpreted.

**The sample count is the smaller of the data's own and the loop's**, so a file whose loop extends past its data
is truncated rather than reading beyond it.

## `FindNextChunk`, `FindChunk`, `GetLittleShort`, `GetLittleLong`, `DumpChunks`

**Contract** — walk the container's chunk list looking for a four-character identifier; read little-endian values
from the walk position; and print the chunk list for diagnosis.

**Invariants** — the walk uses a file-level cursor, so the searches are sequential and order-dependent — which is
why the cue chunk must be sought before the data chunk.

## `ResampleSfx`

**Contract** — takes a sound, its source rate and width, and its raw samples; converts to the device's rate and to
the configured width, into a cache allocation. Copies directly when the rates match and the width is already
correct.

```text
FUNCTION resample_sfx(sfx, inrate, inwidth, data)
  info = the sound's container information
  stepscale = inrate / the device's rate            # 1 means no resampling
  outcount = info.samples / stepscale
  # Scale the loop point by the same ratio.
  sc.loopstart = info.loopstart / stepscale, or -1
  sc.speed = the device's rate
  sc.width = 1 IF the eight-bit tunable is set ELSE inwidth
  allocate outcount * width bytes IN THE CACHE
  IF stepscale == 1 AND inwidth == 1 AND sc.width == 1
    # Fast path: a plain copy, converting unsigned to SIGNED.
    FOR EACH sample  out[i] = data[i] - 128
  ELSE
    # NEAREST-NEIGHBOUR resampling with a 16.16 fixed-point step.
    samplefrac = 0 ;  fracstep = stepscale * 0x10000
    FOR EACH output sample i
      srcsample = samplefrac SHIFTED RIGHT 16
      samplefrac = samplefrac + fracstep
      sample = the source sample, converted to signed and to the output width
      store it
```

**Invariants** — four things.

**Resampling is nearest-neighbour**, with no filtering, which is why a sound authored at 11 kilohertz played
through a 44-kilohertz device is audibly stepped. A rebuild with a real resampler sounds noticeably better and
noticeably unlike the original.

**The loop point is scaled by the same ratio**, which it must be or a resampled looping sound clicks.

**Eight-bit samples are converted from unsigned to signed** by subtracting 128, because the container stores them
unsigned and the mixer expects signed.

**The engine can be told to load everything as eight-bit** ([`sound.h`](sound.h.md)), halving memory and letting
the mixer use its table path ([`snd_mix.c`](snd_mix.c.md#snd_paintchannelfrom8-snd_paintchannelfrom16)). That is a real
quality-versus-cost control and it is worth keeping.

## `S_LoadSound`

**Contract** — takes a sound record; returns its decoded samples, loading and resampling if the cache does not
hold them. Returns nothing when the file is missing or unparseable, after printing.

```text
FUNCTION s_load_sound(s) -> optional<SfxCache>
  IF the cache holds it  RETURN it
  name = "sound/" + s.name
  data = load that file, into a stack buffer if it fits
  IF not found  print "Couldn't load <name>" ;  RETURN nothing
  info = get_wavinfo(s.name, data, its length)
  IF it is not mono  print "Sound <name> is not mono" ;  RETURN nothing
  resample_sfx(s, info.rate, info.width, data + info.dataofs)
  RETURN the cache entry
```

**Invariants** — **only mono sounds are accepted**, because the mixer spatializes into two ears itself and a stereo
source would have no meaningful position. That is a hard constraint on the asset set.

Sounds live under a fixed `sound/` prefix, so a sound's name on the wire is relative to it.

The cache check first is what makes this safe to call from the mixer's inner loop
([`snd_mix.c`](snd_mix.c.md#s_paintchannels)).
