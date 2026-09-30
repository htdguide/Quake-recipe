# WinQuake/cl_demo.c

> Demo recording and playback: a stream of length-prefixed server messages each stamped with the view angles at capture, played back at the recorded pace or as fast as the machine allows.

**Needs** — [`client.h`](client.h.md) · [`net.h`](net.h.md) · [`common.h`](common.h.md) · [`protocol.h`](protocol.h.md) · [`console.h`](console.h.md) · [`cmd.h`](cmd.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`cl_main.c`](cl_main.c.md) reads messages through it; [`host_cmd.c`](host_cmd.c.md) drives the playlist
**Tier floor** — none

## Purpose

A demo is the **server's own message stream**, captured verbatim. Playback substitutes the file for the network,
so every decoder, loader and renderer runs exactly as it did live. That is why a demo is the best conformance
test a rebuild has ([`SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md#6-conformance)): it exercises the whole
client against known-good bytes.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## The file format

```text
# A text line: the CD track to play, or -1.
"<track>\n"
# Then, repeatedly:
RECORD DemoMessage
  length     : int (32-bit, little-endian)
  viewangles : real[3]        # the client's angles AT CAPTURE
  data       : byte[length]   # one server message, verbatim
```

**Invariants** — **the view angles are recorded alongside each message** because they are the client's own state
and are not in the server's stream ([`client.h`](client.h.md)). On playback they are interpolated between
successive records, which is what makes a recorded demo's camera move smoothly.

The track number is a text line rather than a binary field, which is why the file begins with ASCII.

## `CL_WriteDemoMessage`

**Contract** — writes the current incoming message to the file, prefixed by its length and the client's current
view angles.

## `CL_GetMessage`

**Contract** — the substitute for the network read. During playback, returns the next demo message unless the
client's clock has not yet reached it; during recording, reads from the network and also writes to the file.
Returns 1 for a reliable message, 2 for an unreliable one, 0 for nothing yet, and stops playback at end of file.

```text
FUNCTION cl_get_message() -> int
  IF playing back a demo
    # Deliver at most one message per frame, and only once the client's clock
    # has caught up with the recorded timestamp.
    IF the previous message was unreliable AND the recorded time is still ahead
      RETURN 0                                  # not yet
    read the length and the three angles
    IF at end of file  cl_stop_playback() ;  RETURN 0
    IF the length exceeds the buffer  FAIL WITH "Demo message > MAX_MSGLEN"
    cl.mviewangles[1] = cl.mviewangles[0]
    cl.mviewangles[0] = the recorded angles
    read the message body
    RETURN 1
  # Live: read from the network, and record if recording.
  r = get a network message
  IF recording AND r is a message  cl_write_demo_message()
  RETURN r
```

**Invariants** — three things.

**Playback is paced by the client's clock against the message's own timestamp**, read out of the message's time
field, so a demo plays at the rate it was recorded regardless of the machine's frame rate. That pacing is
bypassed by the benchmark mode.

**The angles are shifted down before the new pair is stored**, giving the interpolator its two samples — the
same double-buffering the entity positions get.

**Recording captures the message *after* it is read**, so exactly the bytes the client processed are written.

## `CL_Record_f`

**Contract** — the `record` command; opens a demo file and, when given a map name, starts that map first.
Refuses while already recording, while connected without a fresh map, or with the wrong argument count. Writes
the track number line.

**Invariants** — recording must start **before** the level is entered, which is why the state lives in the
per-process record ([`client.h`](client.h.md)) and why the command can take a map name: the signon block must be
captured or playback has no baselines.

## `CL_PlayDemo_f`, `CL_StopPlayback`, `CL_Stop_f`

**Contract** — open a demo and begin playback, reading the track line; stop playback and close the file; stop
recording and close the file.

**Invariants** — stopping playback also disconnects, because the client's state came from the file.

## `CL_TimeDemo_f`, `CL_FinishTimeDemo`

**Contract** — play a demo with the pacing disabled, counting frames, and report the frame count, elapsed time
and frame rate at the end.

**Invariants** — this is the engine's second benchmark, alongside the renderer's own
([`r_misc.c`](r_misc.c.md#r_timerefresh_f)), and the more representative of the two because it exercises the
whole client. A rebuild should reproduce it and compare.

The frame count starts on the *second* frame, so the first frame's loading cost is excluded.
