# WinQuake/cl_tent.c

> Temporary entities: decodes the fourteen one-shot effect messages into particles, sounds and lights, and maintains the lightning beams that persist for a fraction of a second.

**Needs** — [`client.h`](client.h.md) · [`protocol.h`](protocol.h.md) · [`common.h`](common.h.md) · [`render.h`](render.h.md) · [`model.h`](model.h.md) · [`sound.h`](sound.h.md) · [`mathlib.h`](mathlib.h.md)
**Used by** — [`cl_parse.c`](cl_parse.c.md) dispatches the effect message here; [`cl_main.c`](cl_main.c.md) updates the beams
**Tier floor** — none

## Purpose

The client half of the protocol's temporary-entity events
([`protocol.h`](protocol.h.md)). Each event is a few bytes on the wire and expands here into particles, a sound
and sometimes a light — so the effects are entirely client-side and cost the server almost nothing.

The beams are the interesting part: a lightning bolt is not a model, it is a **chain of model instances placed
along a segment**, rebuilt every frame from the two endpoints.

## State

```text
VARIABLE cl_beams : Beam[24]                       # client.h
VARIABLE cl_temp_entities : Entity[64]
VARIABLE cl_num_temp_entities : int
VARIABLE cl_sprite_explosion, cl_sprite_smoke, cl_bolt1..3 : Model
```

## `CL_InitTEnts`

**Contract** — loads the models the effects need: three lightning bolt models, the spike models, and the
explosion sprite.

**Invariants** — loaded once at startup rather than precached per level, so they are always available regardless
of what the map declared. That is a deliberate exception to the precache discipline
([`pr_cmds.c`](pr_cmds.c.md#precaching)) and it is why these effects work in any map.

## `CL_ParseTEnt`

**Contract** — reads an effect number and its own payload — the payloads differ per effect — and produces the
effect. An unknown effect number is a host error.

```text
FUNCTION cl_parse_tent()
  SELECT the effect byte
    spike           read a position ;  a bullet-impact particle burst
                    with a one-in-five chance play the ricochet sound, else
                    one of two impact sounds at random
    superspike      as above with more particles and a different sound set
    gunshot         read a position ;  a small particle burst, no sound
    explosion       read a position ;  a particle explosion, a dynamic light of
                    radius 350 decaying over half a second, the explosion
                    sprite, and the explosion sound
    tarexplosion    read a position ;  the blob explosion and its own sound
    lightning1/2/3  read an entity, a start and an end ;  cl_parse_beam with
                    the matching bolt model
    wizspike        a green particle burst and a sound
    knightspike     likewise
    lavasplash      read a position ;  the lava splash
    teleport        read a position ;  the teleport splash
    explosion2      read a position, a palette start and a length ;  a coloured
                    explosion, a light, and the explosion sound
    beam            read an entity and two points ;  a generic beam
    otherwise       FAIL WITH "CL_ParseTEnt: bad type"
```

**Invariants** — three things.

**The sound choice is randomized per impact**, from a small set, with a one-in-five chance of a distinct
ricochet. That randomization is what stops repeated fire sounding mechanical, and it is done on the client — so
two players hear different sounds for the same shot.

**An explosion spawns four separate things**: particles, a light, a sprite entity and a sound. Only the sprite
is an entity; the rest are transient.

**The payloads differ per effect and there is no length field**, so a reader must switch on the effect before it
knows how many bytes to consume ([`protocol.h`](protocol.h.md)). An unknown effect therefore cannot be skipped
and is fatal.

## `CL_ParseBeam`

**Contract** — reads a beam's owning entity and its two endpoints; reuses the existing beam for that entity if
there is one, otherwise takes a free or expired slot, and sets its model, endpoints and expiry.

**Invariants** — **keyed by the owning entity**, so a continuously-fired lightning gun updates one beam rather
than accumulating them — the same pattern as the dynamic-light allocator
([`cl_main.c`](cl_main.c.md#cl_allocdlight)).

The expiry is a fraction of a second in the future, so a beam whose owner stops firing fades on its own.

## `CL_UpdateTEnts`

**Contract** — once per frame: for each live beam, if it belongs to the player, moves its start to the player's
own position; then walks the segment in 30-unit steps placing one temporary entity per step, each oriented along
the segment, with the last segment's model rotated.

```text
FUNCTION cl_update_tents()
  cl_num_temp_entities = 0
  FOR EACH beam WITH a model and an unexpired end time
    IF the beam's entity IS the player  its start = the player's origin
    delta = end - start ;  d = length(delta)
    yaw, pitch = the direction's angles
    # Walk the segment in 30-unit steps, placing one model instance per step.
    WHILE d > 0
      ent = cl_new_temp_entity() ;  IF none, RETURN
      ent.origin = the current point
      ent.model = the beam's model
      ent.angles = (pitch, yaw, a RANDOM roll)
      advance the point by 30 units along the direction ;  d = d - 30
```

**Invariants** — three things.

**A lightning bolt is a chain of model instances 30 units apart**, not a stretched model, which is why it looks
segmented and why a long bolt costs more temporary entities. The 30 is the bolt model's own length.

**Each segment gets a random roll**, which is what makes the bolt crackle rather than look like a rigid pipe.
Re-randomized every frame.

**A beam owned by the player has its start moved to the player's position each frame**, so the bolt stays
attached to the weapon as the player moves — the server's recorded start would lag by the interpolation
interval.

## `CL_NewTempEntity`

**Contract** — takes a slot from the temporary-entity pool, also appending it to the renderer's visible list.
Returns nothing when either is full.

**Invariants** — a temporary entity is **not** in the persistent entity array; it exists for one frame and is
re-created next frame. So the pool is reset each frame and nothing is ever removed from it.
