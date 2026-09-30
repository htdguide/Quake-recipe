# WinQuake/sound.h

> The audio vocabulary: a ring buffer the hardware drains, a fixed channel array, and the record that makes a sound follow the entity that made it.

**Needs** — nothing beyond the engine's base types; a near-leaf
**Used by** — [`snd_dma.c`](snd_dma.c.md) · [`snd_mix.c`](snd_mix.c.md) · [`snd_mem.c`](snd_mem.c.md) · [`cl_parse.c`](cl_parse.c.md) · [`cl_main.c`](cl_main.c.md) · [`view.c`](view.c.md) · [`host.c`](host.c.md) · and implemented per platform by [`snd_win.c`](snd_win.c.md), [`snd_linux.c`](snd_linux.c.md), [`snd_dos.c`](snd_dos.c.md), [`snd_gus.c`](snd_gus.c.md), [`snd_sun.c`](snd_sun.c.md), [`snd_next.c`](snd_next.c.md) and [`snd_null.c`](snd_null.c.md)
**Tier floor** — T2: the mixer writes into a buffer the hardware is concurrently reading, and its correctness depends on knowing how far the hardware has got

## Purpose

Declares both halves of the sound system: the platform seam ([Seam: Audio
output](../SYSTEM-REQUIREMENTS.md#seam-audio-output), four calls) and the engine's own
mixer, which is entirely in this directory.

The design's defining property is that **the mixer is driven by the hardware's read
position, not by a timer**. There is no audio thread and no callback. Once per frame,
and again at several points during level loading, the engine asks the device how many
samples it has consumed and mixes forward from there to a target a fixed distance
ahead. Everything else — the channel allocation, the spatialization, the looping —
follows from that.

## State

```text
RECORD SamplePair               # one stereo sample in the accumulator
  left, right : int             # wider than the output, to allow summation
                                # before clamping

RECORD Sfx                      # a named sound, possibly not resident
  name  : text[64]
  cache : CacheHandle           # the decoded samples live in the evictable
                                # cache, so they may vanish

RECORD SfxCache                 # the decoded form of one sound
  length    : int               # in samples
  loopstart : int               # -1 for no loop
  speed     : int               # samples per second, after resampling
  width     : int               # 1 or 2 bytes per sample
  stereo    : int
  data      : bytes             # variable length

RECORD Dma                      # the device's ring buffer, as the device
                                # describes it
  channels          : int
  samples           : int       # total MONO samples in the ring
  submission_chunk  : int       # never submit fewer than this many
  samplepos         : int       # in mono samples; the device's read cursor
  samplebits        : int
  speed             : int
  buffer            : bytes
  splitbuffer       : bool      # the ring is two halves alternately consumed
  gamealive, soundalive : bool

RECORD Channel                  # one playing sound
  sfx         : optional<Sfx>   # nothing means the slot is free
  leftvol     : int             # 0..255, after spatialization
  rightvol    : int
  end         : int             # the global paint time at which it finishes
  pos         : int             # the next sample to read from the sound
  looping     : int             # where to loop back to; -1 for none
  entnum      : int             # the entity that owns it
  entchannel  : int             # which of that entity's channels
  origin      : vec3            # where it is, for spatialization
  dist_mult   : real            # attenuation per unit of distance
  master_vol  : int             # 0..255, before spatialization

RECORD WavInfo                  # what the file parser extracts
  rate, width, channels : int
  loopstart : int
  samples   : int
  dataofs   : int

CONSTANT max_channels          = 128
CONSTANT max_dynamic_channels  = 8
CONSTANT default_volume        = 255
CONSTANT default_attenuation   = 1.0

VARIABLE channels       : Channel[128]
VARIABLE total_channels : int
VARIABLE paintedtime    : int    # global mix cursor, in mono samples;
                                 # monotonically increasing
VARIABLE listener_origin, listener_forward, listener_right, listener_up : vec3
VARIABLE sound_nominal_clip_dist : real
VARIABLE shm : optional<Dma>     # the device, once open
```

**Invariants** — the 128 channel slots are partitioned by *position*, not by a field:

```text
slots 0 .. 7                              dynamic: any entity's sounds
slots 8 .. 11                             the four ambient sounds
                                          (water, sky, slime, lava)
slots 12 .. total_channels-1              static: sounds fixed in the world
```

The partition is load-bearing and implicit. Ambient levels come from the leaf the
listener is in ([`bspfile.h`](bspfile.h.md)), so those four slots are addressed by the
contents type directly. Static sounds are placed once at level load and never freed, so
`total_channels` grows during loading and never shrinks. Dynamic sounds are the only
ones subject to replacement, which is why there are only eight of them and why
[`SND_PickChannel`](snd_dma.c.md#snd_pickchannel) can afford a linear scan.

`paintedtime` is the mixer's clock and it counts **mono samples ever mixed**, not
samples in the buffer. A channel's end time is in the same units, which is how "has
this sound finished" becomes an integer comparison with no wraparound concern for any
plausible session length.

## The device seam

### `SNDDMA_Init`

**Contract** — opens the device, negotiates a format, and fills in the device record
including a working read cursor. Returns whether it succeeded. Honours command-line
requests for a rate, a sample width and a channel count, but the record it fills is
authoritative — the engine reads back what it actually got.

### `SNDDMA_GetDMAPos`

**Contract** — returns the device's current read position in mono samples, modulo the
ring length, and also stores it in the record. Called once per mix.

**Invariants** — must advance. The mixer computes how much to write from this value
alone; a device that returns a constant produces silence or a stuck buffer, and a device
that jumps backwards produces a loop.

### `SNDDMA_Submit`

**Contract** — tells the device that the engine has written up to the current mix
cursor. For a device with direct memory access this is empty; for one that needs an
explicit hand-off it copies or queues.

### `SNDDMA_Shutdown`

**Contract** — stops the device and releases it.

## Starting and stopping sounds

### `S_StartSound`

**Contract** — takes an entity number, one of that entity's channel numbers, a sound, a
world position, a volume and an attenuation; allocates a channel and begins playing.
A non-zero channel number **replaces** any sound already playing on that entity's
channel — which is how a monster's pain sound cuts off its idle sound rather than
layering. Channel number zero always allocates a new slot.

**Invariants** — the entity-and-channel replacement rule is the engine's whole sound
priority model, and it is decided by the *caller* choosing a channel number. The game
logic assigns fixed channel numbers to fixed roles ([`qw-qc/`](../qw-qc/README.md)),
so the rule is really a convention between the game and the engine.

### `S_StaticSound`

**Contract** — takes a sound, a position, a volume and an attenuation; permanently adds
it above the dynamic and ambient slots. Never freed, never replaced. Used for ambient
loops placed in the map.

### `S_StopSound`

**Contract** — takes an entity and a channel number; silences that slot if it is
playing.

### `S_StopAllSounds`

**Contract** — silences every channel, optionally also zeroing the device buffer. Called
at level change and when the engine is about to stall.

### `S_ClearBuffer`

**Contract** — writes silence over the whole device ring. The correct silence value
depends on the sample width — zero for 16-bit, mid-scale for 8-bit unsigned — which is a
real trap.

## Per-frame driving

### `S_Update`

**Contract** — takes the listener's position and basis; respatializes every active
channel, drops finished ones, applies the ambient levels for the listener's current
leaf, then mixes forward. Called once per frame.

### `S_ExtraUpdate`

**Contract** — mixes forward without respatializing. Called from inside slow operations
— level loading, model loading — so that the ring does not run dry and audibly stutter.

**Notes** — this is the consequence of having no audio thread: every operation that
might take longer than the mixer's lookahead has to remember to call this. A rebuild
with a callback-driven device deletes it, and deletes the scattered calls with it.

### `S_PaintChannels`

**Contract** — takes a target time in mono samples; mixes every active channel into the
accumulator and converts the accumulator into the device's format, up to that target.
The engine's only synthesis path.

## Loading and precaching

### `S_PrecacheSound`

**Contract** — takes a name; returns a sound record, registering it if new. Does not
necessarily load the samples — that is deferred.

### `S_TouchSound`

**Contract** — takes a name; marks its cache entry as recently used, so that a sound
belonging to the next level is not evicted while that level loads.

### `S_LoadSound`

**Contract** — takes a sound record; returns its decoded samples, loading and resampling
them if the cache entry is absent. Returns nothing if the file cannot be read.

### `S_ClearPrecache`, `S_BeginPrecaching`, `S_EndPrecaching`

**Contract** — bracket a level's loading so that the cache can distinguish sounds this
level needs from leftovers.

### `GetWavinfo`

**Contract** — takes a name and a raw file; extracts rate, width, channel count, sample
count and the loop point. See [`snd_mem.c`](snd_mem.c.md).

## Spatialization

### `SND_PickChannel`

**Contract** — takes an entity and a channel number; returns the slot to use, applying
the replacement rule and, when every dynamic slot is busy, evicting the one that will
finish soonest.

### `SND_Spatialize`

**Contract** — takes a channel; sets its two ear volumes from its position, the
listener's basis and its attenuation. A channel belonging to the listener's own entity
is centred.

### `SND_InitScaletable`

**Contract** — builds the table that turns a volume in 0..255 and a sample value into a
scaled contribution, so that the mixer's inner loop is a table lookup rather than a
multiply.

## Volumes and the fake device

```text
VARIABLE volume, bgmvolume, loadas8bit : Cvar
VARIABLE snd_initialized : bool
VARIABLE snd_blocked     : int     # a nesting count; non-zero means silent
VARIABLE fakedma         : bool    # pretend a device is consuming samples
VARIABLE fakedma_updates : int     # how many times per second
```

**Notes** — the fake device exists so the renderer can be profiled with the mixer's cost
included but no hardware involved. It is genuinely useful and worth keeping in a
rebuild: it decouples a timing measurement from the audio device's behaviour.

`snd_blocked` is a count, not a flag, because the engine blocks sound around several
nested operations — losing window focus, entering a modal dialogue — and the innermost
unblock must not re-enable it.
