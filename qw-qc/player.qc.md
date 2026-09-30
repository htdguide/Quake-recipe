# qw-qc/player.qc

> The player's animations and the pieces of their death: every frame sequence named, and the rules that pick between them.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md) · [`combat.qc`](combat.qc.md)
**Used by** — [`client.qc`](client.qc.md) · [`weapons.qc`](weapons.qc.md)
**Tier floor** — none

## Purpose

Animation as game logic. In this engine an animation is **a chain of functions, one per frame**, each setting the model's frame number
and naming its successor, so the animation system *is* the scheduling system
([`defs.qc`](defs.qc.md)'s think field) and there is no animation data at all beyond the frame numbers.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## The animation idiom

```text
each frame of a sequence is a named function:
FUNCTION player_run1()  [frame 1, player_run2]   # the language's frame syntax
  set the model's frame number
  advance to the named successor next frame
```

**Invariants** —

- **The language has syntax for this**: a function can declare a frame number and a successor, and the compiler turns it into setting the
  frame field and the think field with a fixed interval. The interval is the hard-coded tenth of a second in the engine's animation
  operation ([`pr_exec.c`](../QW/server/pr_exec.c.md)). **So every animation in the game runs at exactly ten frames per second**, and that
  constant is in the interpreter, not in the content.
- **An animation occupies the entity's only timer**, so an entity cannot animate and think about anything else at once. Every behaviour in
  the game is therefore written *as* an animation with logic in its frames — which is why monster behaviour and animation are the same
  code.
- **Interrupting an animation means assigning a different successor**, which any frame may do conditionally. That is how a running player
  becomes a firing player mid-stride.

## The sequences

**Contract** — standing, running, and the death sequences, plus the pain reactions.

**Invariants** —

- **The weapon animations are separate** and live in [`weapons.qc`](weapons.qc.md), driven by a separate frame counter on the entity
  ([`defs.qc`](defs.qc.md)) — because the body and the weapon animate independently and the protocol carries both
  ([`sv_ents.c`](../QW/server/sv_ents.c.md)).
- **The body's animation is chosen from the player's speed and ground state** in the think callbacks
  ([`client.qc`](client.qc.md)), so other players can read intent from the animation.
- **There are several death sequences and one is chosen at random**, with a separate path for a death violent enough to produce pieces
  ([`client.qc`](client.qc.md)). The variety is cosmetic and cheap.
- **Pain has an immunity window** ([`combat.qc`](combat.qc.md)) so that a stream of hits does not restart the animation every frame,
  which would freeze the player in place.

## The pieces

**Contract** — on a violent death, spawn several independently tumbling fragments with the bouncing movement type, each with a random
velocity and spin, expiring after a time.

**Invariants** — **the fragments are ordinary entities using the engine's bouncing movement**
([`sv_phys.c`](../QW/server/sv_phys.c.md)), so they collide with the world correctly for free. They **expire**, because otherwise a long
match fills the entity pool ([`bothdefs.h`](../QW/client/bothdefs.h.md)) — the same bound that limits the corpse queue
([`world.qc`](world.qc.md)).

**Notes** — the transferable observation is uncomfortable and worth stating: **because an entity has one timer, animation and behaviour
are the same mechanism, and every piece of game logic is written inside an animation frame.** It works, and it is why the game logic reads
as it does. A rebuild with more than one timer per entity — or with any form of coroutine — separates the two and will find the resulting
code much shorter, but should understand that the original's structure is a consequence of the interpreter's single `nextthink` field and
not a stylistic choice.
