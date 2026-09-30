# qw-qc/combat.qc

> Damage: how much of it armour absorbs, who gets the credit, what a blast reaches, and the sequence a death goes through.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md)
**Used by** — [`weapons.qc`](weapons.qc.md) · [`player.qc`](player.qc.md) · [`items.qc`](items.qc.md) · [`triggers.qc`](triggers.qc.md) · [`misc.qc`](misc.qc.md)
**Tier floor** — none

## Purpose

The rules of harm, in one file. It is the clearest example in the chapter of game design expressed as arithmetic, and the numbers are
the design.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## `T_Damage`

**Contract** — apply damage from an inflictor, credited to an attacker, against a target: reject it if the target cannot be hurt, halve
it if the target and attacker are on the same team and friendly fire is disabled, remove the armour's share, push the target along the
damage's direction in proportion to the damage, credit the attacker, and kill the target if its health reaches zero — otherwise call its
pain handler.

```text
FUNCTION damage(target, inflictor, attacker, amount)
  IF the target cannot take damage  RETURN
  IF the attacker holds the damage-multiplier item  amount *= 4
  save_damage = amount * the target's armour fraction
  IF that exceeds the armour's remaining value
    save_damage = the remaining value ;  the armour is destroyed
  the armour's value -= save_damage
  take = amount - save_damage
  IF the target is a player AND the attacker is a player
    IF they are on the same team AND friendly fire is off  take /= 2
  # knock the target back
  direction = normalize(target.origin - inflictor's centre)
  target.velocity += direction * take * 8
  target.health -= take
  IF the target's health <= 0
    killed(target, attacker) ;  RETURN
  set the target's pain handler's arguments and call it, unless it is
    within its pain-immunity window
```

**Invariants** —

- **Armour absorbs a *fraction* and has a *capacity*, and both are per armour type.** So better armour both absorbs more per hit and
  lasts longer; the two numbers together are the whole of the armour design
  ([`items.qc`](items.qc.md) sets them).
- **The knock-back is proportional to the damage taken after armour**, so armour reduces both harm and displacement. That coupling is
  what makes rocket jumping depend on not wearing armour, which players exploit.
- **The knock-back is applied along the direction from the inflictor's *centre*, not its origin**, which for a blast means the geometric
  middle. Using the origin makes an explosion at a player's feet push them sideways rather than up — and the upward push is the whole of
  rocket jumping.
- **Friendly fire halves rather than eliminating damage**, which is a design decision about whether a team can be blocked by its own
  members.
- **The pain handler has an immunity window**, so a rapid stream of hits does not interrupt an entity's behaviour every frame.
- The damage multiplier is checked on the **attacker**, not the target.

## `T_RadiusDamage`

**Contract** — damage everything within a radius of a point, falling off linearly with distance, skipping anything the point cannot
reach, and exempting a named entity.

```text
FUNCTION radius_damage(inflictor, attacker, damage, ignore)
  FOR EACH entity within the radius, found by the engine's sphere search
    IF it is the ignored one OR cannot take damage  SKIP
    distance = |its centre - inflictor.origin|
    points = 0.5 * distance                          # the falloff
    points = MAX(0, damage - points)
    IF points > 0 AND can_damage(it, inflictor)
      IF it is the attacker  points *= 0.5           # you hurt yourself less
      damage(it, inflictor, attacker, points)
```

**Invariants** —

- **The falloff is linear in distance with a fixed coefficient**, not inverse-square. So a blast has a hard edge at a predictable
  distance, which is what makes rocket splash learnable. Physical correctness would make it worse to play.
- **Self-damage is halved**, which is the number that decides whether rocket jumping is affordable. It is one multiplication and it
  defines a whole movement technique.
- **Reachability is tested**, so a wall stops a blast — see below.

## `CanDamage`

**Contract** — report whether a target is reachable from a point by a straight line, trying the target's centre and, if that fails, the
corners of its box.

**Invariants** — **the corners are tried after the centre**, because a target standing behind a low wall has its centre blocked and its
head exposed. Testing only the centre makes cover far more effective than it looks; testing the corners makes a blast reach anything
partly visible. Four extra traces, and the game is different.

## `Killed`, `ClientObituary`, `monster_death_use`

**Contract** — run a death: mark the entity dead, stop it taking further damage, credit the killer, run the entity's own death handler,
fire whatever it targets; and compose the message announcing a player's death, choosing the wording from what killed them and adjusting
the scores.

**Invariants** —

- **A dying entity fires its targets**, which is how a map makes a door open when a monster dies. The death is a trigger like any
  other ([`subs.qc`](subs.qc.md)).
- **The score adjustment distinguishes suicide, a fall, a hazard and another player**, with a penalty for the first three and a credit
  for the last. The obituary text and the score change are computed together, which is why they are one function — the same distinction
  drives both.
- **A player killed by a projectile is credited to whoever fired it**, not to the projectile, which is why every projectile carries its
  owner ([`weapons.qc`](weapons.qc.md), and the engine's owner field in [`world.h`](../QW/server/world.h.md)).
- Team kills are counted separately and announced differently.

## `T_MissileTouch`, `T_BeamDamage`

**Contract** — a projectile striking something: damage it, damage a radius around the impact, and remove the projectile; and a beam
damaging everything along its length.

**Invariants** — a projectile **damages its target directly and then the radius around it**, so a direct hit is worth more than a near
miss by exactly the direct damage. That two-part structure is the whole of the rocket's balance.

**Notes** — the numbers in this file are the game. A rebuild that changes the falloff coefficient, the knock-back multiplier, the
self-damage fraction or the armour fractions has built a different game — not necessarily a worse one, but the players will know
immediately. The recipe records them as decisions with names attached, because that is what they are.
