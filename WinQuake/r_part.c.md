# WinQuake/r_part.c

> The particle system: one fixed pool, nine effect constructors each with its own velocity distribution and colour ramp, and a simulation whose motion type is stored per particle.

**Needs** — [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`client.h`](client.h.md) · [`render.h`](render.h.md) · [`common.h`](common.h.md) (the message reader) · [`anorms.h`](anorms.h.md) · [`zone.h`](zone.h.md)
**Used by** — [`cl_parse.c`](cl_parse.c.md) and [`cl_tent.c`](cl_tent.c.md) trigger effects; [`cl_main.c`](cl_main.c.md) runs trails; [`r_main.c`](r_main.c.md) draws; [`d_part.c`](d_part.c.md) rasterizes
**Tier floor** — none

## Purpose

Every visual effect in this game that is not a sprite is particles, and all of it is here. The design is one
fixed pool with a free list, eight motion types, and nine constructors that differ only in how many particles
they spawn and with what initial state.

**Particles are entirely client-side.** The server sends an event; the client invents the particles. So two
players watching one explosion see different particles, and a recorded demo replays the event
([`protocol.h`](protocol.h.md)).

## State

```text
CONSTANT max_particles = 2048             # default; a command line can change it
CONSTANT absolute_min_particles = 512
VARIABLE active_particles, free_particles : Particle
VARIABLE particles : Particle[]           # the pool, hunk-allocated
VARIABLE ramp1, ramp2, ramp3 : int[8]     # the three colour ramps
VARIABLE avelocities : vec3[162]          # per-normal angular velocities, for
                                          # the entity sparkle field
```

**Invariants** — the pool is fixed and **spawning fails silently when it is empty**. So a busy firefight
progressively loses particle detail rather than slowing down, which is the correct trade for a real-time
renderer.

The three colour ramps are eight-entry palette-index sequences that a particle walks as it ages: one for
smoke, one for fire, one for blood. Which ramp and how fast is the motion type's business.

## `R_InitParticles`, `R_ClearParticles`

**Contract** — allocate the pool from the command line or the default, never below the absolute minimum; and
reset it to all free.

## The motion types

```text
ENUM ParticleType
  static      # no motion at all
  grav        # full gravity
  slowgrav    # a fraction of gravity
  fire        # rises, walks the fire ramp, dies when the ramp ends
  explode     # accelerates outward, walks the first ramp
  explode2    # accelerates outward, walks the second ramp
  blob        # a tarbaby's inward-collapsing motion
  blob2       # the flattened variant
```

**Invariants** — the type is stored per particle and interpreted by the simulation, so one pool holds every
effect simultaneously. That is what lets an explosion and a rocket trail coexist without separate systems.

## The nine constructors

**Contract** — each takes a position and effect-specific parameters and spawns a burst. Each chooses a count,
a lifetime, a motion type, a colour or ramp start, and a velocity distribution.

| Constructor | Spawns | Character |
|---|---|---|
| `R_ParticleExplosion` | 1024 | half fire, half explode; random velocities in a 32-unit cube |
| `R_ParticleExplosion2` | 512 | a caller-given palette *range*, so the game can colour it |
| `R_BlobExplosion` | 1024 | half each blob type; a tarbaby's implosion |
| `R_RunParticleEffect` | a caller-given count | the generic burst the protocol carries |
| `R_LavaSplash` | 1024 | arranged on a ring, thrown upward and outward |
| `R_TeleportSplash` | a lattice | a column of particles on a regular 4-unit grid |
| `R_RocketTrail` | along a segment | one of five trail kinds, spaced every 3 units |
| `R_EntityParticles` | 162 | one per model normal, orbiting the entity |
| `R_ParseParticleEffect` | from the wire | decodes a position, direction, count and colour |

**Invariants** — five things.

**The teleport splash is a lattice, not a random cloud**: particles are placed on a regular grid through the
entity's volume with small random offsets. That regularity is why a teleport reads as a structured effect.

**The lava splash is arranged on a ring** by iterating two angular indices, which is why it forms an expanding
circle rather than a sphere.

**The entity sparkle field places one particle per entry of the model normal table**
([`anorms.h`](anorms.h.md)), each orbiting at its own randomly-chosen angular velocity — so the field is a
sphere of 162 points and the velocities are generated once and reused for every such entity.

**A rocket trail is spaced by distance, not by time**: the segment from the entity's last position to its
current one is walked in 3-unit steps and one particle is spawned per step. So a fast projectile leaves the
same trail density as a slow one, and a projectile that teleports leaves a line across the map — which is why
the protocol has a no-interpolate flag ([`protocol.h`](protocol.h.md)).

**The wire-carried effect clamps a count of 255 to a much larger number**, which is how the game requests a
large burst through a byte field.

## `R_ReadPointFile_f`

**Contract** — the `pointfile` command; reads a text file of coordinates produced by the map compiler when a
map leaks, and spawns a permanent particle at each so the leak's path is visible in the game.

**Notes** — a map-authoring tool built into the engine. Worth keeping: it is the only way to find a leak
visually.

## `R_DrawParticles`

**Contract** — advances every live particle by the frame's elapsed time according to its motion type, frees
those whose lifetime has expired, and hands each survivor to the rasterizer.

```text
FUNCTION r_draw_particles()
  grav = cl.frametime * sv_gravity * 0.05
  d_start_particles()
  FOR EACH active particle p, unlinking any whose die time has passed
    d_draw_particle(p)
    p.org = p.org + p.vel * frametime
    SELECT p.type
      static     nothing
      fire       p.ramp = p.ramp + frametime*5
                 IF the ramp has run out  p.die = now
                 ELSE p.color = ramp3[truncate(p.ramp)]
                 p.vel[2] = p.vel[2] + grav                # rises
      explode    p.ramp = p.ramp + frametime*10
                 IF exhausted  die  ELSE p.color = ramp1[truncate(p.ramp)]
                 p.vel = p.vel + p.vel * frametime * 4     # ACCELERATES
                 p.vel[2] = p.vel[2] - grav
      explode2   as explode, with ramp2 and a slower acceleration
      blob       p.vel = p.vel + p.vel * frametime * 4     # all three axes
                 p.vel[2] = p.vel[2] - grav
      blob2      the same, but only the horizontal axes accelerate
      grav       p.vel[2] = p.vel[2] - grav
      slowgrav   p.vel[2] = p.vel[2] - grav                # see the note
  d_end_particles()
```

**Invariants** — four things.

**Gravity is the server's gravity times 0.05**, so particles fall at a twentieth of an entity's rate. That is
what makes smoke drift rather than drop.

**The exploding types accelerate along their own velocity**, multiplying it by a factor each frame, which is
what makes an explosion expand rather than merely drift. That is not physical and it is the effect's whole
character.

**A particle dies when its colour ramp runs out**, not on a timer, for the three ramped types. So the ramp's
length *is* the lifetime, and the ramp advance rate sets it.

**The slow-gravity case applies the same gravity as the full one.** The two motion types are therefore
identical, which is a defect: one of them was meant to scale it. A rebuild reproducing the original's look
should keep them identical.
