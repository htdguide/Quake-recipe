# QW/client/bothdefs.h

> The definitions the client and the server must agree on: the limits, the axis order, the statistic and item numbering, the message importance levels, and the two build switches.

**Needs** — nothing
**Used by** — [`quakedef.h`](quakedef.h.md) and [`qwsvdef.h`](../server/qwsvdef.h.md) — every file in both programs
**Tier floor** — none

## Purpose

Extracted from the original's single umbrella header ([`quakedef.h`](../../WinQuake/quakedef.h.md)) so that two programs can
include exactly the same constants. It exists because there are now two programs, and **everything in it is a shared contract**:
change a value here and the client and the server must be rebuilt together.

For a rebuilder it is the shortest useful summary of the game's dimensions.

## State

```text
CONSTANT the protocol and build version numbers
CONSTANT whether the target has the hand-written assembly available
CONSTANT whether unaligned memory access is permitted
CONSTANT cache_size = 32            # the alignment key records are padded to
CONSTANT minimum_memory             # refused below this at startup
CONSTANT pitch = 0, yaw = 1, roll = 2
CONSTANT max_scoreboard = 16        # players
CONSTANT sound_channels = 8
CONSTANT max_qpath = 64, max_ospath = 128
CONSTANT on_epsilon = 0.1
CONSTANT max_msglen = 1450, max_datagram = 1450
CONSTANT max_edicts = 768, max_lightstyles = 64
CONSTANT max_models = 256, max_sounds = 256
CONSTANT max_cl_stats = 32, and the numbering of each statistic
CONSTANT the item bit flags
CONSTANT the print importance levels: low, medium, high, chat
```

**Invariants** —

- **The message size is 1450 bytes, down from the original's 8000.** That single number is the most consequential constant in
  QuakeWorld: it is chosen to fit inside one datagram on a link that does not fragment, so **a packet is never split by the
  network**. Fragmentation would mean a single lost fragment discards the whole packet, which for a snapshot stream is a large
  loss for a small cause. The original's 8000-byte messages are fragmented by its own reliability layer
  ([`net_dgrm.c`](../../WinQuake/net_dgrm.c.md)); QuakeWorld deletes fragmentation entirely by choosing this number, and every
  other decision in [`net_chan.c`](net_chan.c.md) follows from it.
- **The entity limit is 768 and the source is unhappy about it** — the comment is an audible wince. It is a protocol limit
  ([`sv_ents.c`](../server/sv_ents.c.md) can address 512 in a snapshot) and a memory limit, and it constrains what a
  modification can do.
- **Model and sound counts are capped at 256 because they travel as single bytes**, which the comment states. That is the general
  rule the whole protocol is built on: *a limit that exists because of an encoding must be documented at the encoding, not at the
  limit.*
- **The alignment constant is duplicated in the assembly's header** and the comment says so. Two hand-synchronized copies of one
  number ([`d_ifacea.h`](d_ifacea.h.md)).
- **The assembly is disabled for the server build** by an explicit override, which is the build switch that makes the server
  portable.
- Statistics are **numbered and some numbers are retired** — commented out and not reused. The numbering is in the protocol, so a
  retired number can never be recycled.
- The importance levels are what let a client filter chat from critical messages
  ([`sv_send.c`](../server/sv_send.c.md)).

**Notes** — the item flags are game content in an engine header, which is a layering mistake the original also makes. They are
here because the status bar draws from them ([`sbar.c`](sbar.c.md)) and the engine's view code reads two of them
([`gl_rmain.c`](gl_rmain.c.md) hides the weapon when invisible). A rebuild should pass those two facts through the protocol
instead and keep the item set entirely in the game logic.
