# QW/client/cl_demo.c

> Recording and replaying a session: every packet written with the time it arrived and the command that preceded it, replayed on the recorded timeline instead of the network.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`net_chan.c`](net_chan.c.md) · [`cl_parse.c`](cl_parse.c.md)
**Used by** — [`cl_main.c`](cl_main.c.md) reads every packet through it
**Tier floor** — none

## Purpose

Read [`cl_demo.c`](../../WinQuake/cl_demo.c.md) for the idea: a recording is the server's byte stream plus timing, and replaying it
means substituting the file for the socket, so the whole client renders a recording with no special path.

The changes here follow from the packet-per-frame design.

## State

As [`cl_demo.c`](../../WinQuake/cl_demo.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**Each recorded packet carries the time it was received *and* the view angles at that moment.** The original stores the time and the
angles too, but here the angles matter more, because the client's own view is predicted
([`cl_pred.c`](cl_pred.c.md)) and a replay cannot predict — it must be told where the camera looked.

**The commands the client sent are recorded as well as the packets it received.** That is new, and it is what makes a recording
replayable *through the prediction path*: the replay feeds the recorded commands to the prediction the way the live client fed its own
([`cl_input.c`](cl_input.c.md)). Without them a recording would have to store the interpolated result instead of reproducing it.

**Delta compression is disabled while recording** ([`cl_input.c`](cl_input.c.md)), so every snapshot in a recording is absolute. A
delta chain is unreplayable from an arbitrary point and un-seekable; making the recording self-contained costs bandwidth during
recording only. **That is the right trade and it is the general rule: a stream meant to be stored must not depend on state the reader
does not have.**

**A recording can be re-recorded from another recording**, which is how the delta-free form is produced from a live session that used
deltas.

**Reading a packet is a single operation that returns either the next network packet or the next recorded one**, so nothing above it
knows which it is. That unification is what keeps the replay path from diverging.

**Invariants** —

- **The replay advances on the recorded timeline, not the machine's**: the next packet is delivered when the recorded time is reached.
  So a recording plays at its original speed regardless of frame rate, and a timed replay — used as the engine's benchmark — can
  deliberately ignore the timing and run as fast as possible.
- **Downloads are refused while recording or replaying** ([`cl_parse.c`](cl_parse.c.md)), because a recording must be self-contained
  and a replay has no server to ask.
- The file is written **as it arrives**, not buffered to the end, so a crash leaves a usable partial recording.

**Notes** — the two decisions worth carrying: **record the inputs as well as the outputs** when the client's state is derived rather
than received, and **make the recorded stream reference-free** even at a cost. Both are what separate a recording you can watch from
a recording you can seek, trim and re-encode.
