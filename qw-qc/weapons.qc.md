# qw-qc/weapons.qc

> The weapons: what each fires, how the shot is traced or launched, the accumulation that makes a spread of pellets one damage event, and the animation and selection machinery around them.

**Needs** — [`defs.qc`](defs.qc.md) · [`combat.qc`](combat.qc.md) · [`items.qc`](items.qc.md) · [`subs.qc`](subs.qc.md)
**Used by** — [`client.qc`](client.qc.md) through the weapon frame; [`items.qc`](items.qc.md) for pickups
**Tier floor** — none

## Purpose

The largest file in the chapter and the one that most directly *is* the game. Each weapon is a firing function, an ammunition cost, an
animation sequence and a set of numbers, and the numbers are the balance.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## The two firing models

**Contract** — a weapon either **traces instantly** to what it would hit, or **launches a projectile entity** that the engine simulates.

```text
instant:   trace from the eye along the aim; damage what was hit
projectile: spawn an entity with a velocity, an owner and a touch handler;
            the engine moves it (sv_phys.c) and calls the handler on contact
```

**Invariants** —

- **A projectile carries its owner**, so it does not collide with whoever fired it on its first frame
  ([`world.h`](../QW/server/world.h.md)) and so a kill is credited correctly
  ([`combat.qc`](combat.qc.md)).
- **A projectile fired this frame moves this frame**, because the game sets the engine's projectile signal
  ([`defs.qc`](defs.qc.md), [`sv_phys.c`](../QW/server/sv_phys.c.md)). Without that, a point-blank rocket behaves differently from a
  distant one.
- **An instant weapon's trace starts at the eye, not the weapon model**, so what the player aims at is what is hit. The visible muzzle
  and the actual origin differ, and that is deliberate.

## `ClearMultiDamage`, `AddMultiDamage`, `ApplyMultiDamage`, `Multi_Finish`

**Contract** — accumulate damage against one target across several traces and apply it as a single event.

```text
FUNCTION fire_bullets(count, direction, spread)
  clear_multi_damage()
  REPEAT count times
    trace along direction perturbed randomly within spread
    IF something was hit  add_multi_damage(it, per-pellet damage)
  apply_multi_damage()

FUNCTION add_multi_damage(hit, amount)
  IF hit is not the entity already accumulating
    apply what has accumulated ;  start accumulating against `hit`
  add amount
```

**Invariants** — **a shotgun's pellets must become one damage event per target.** If each pellet called the damage function
independently, the target would be knocked back once per pellet — sending them across the room — and would play its pain sound eight
times. The accumulator is the fix and it is the only reason this machinery exists. *Any weapon that fires several projections at once
needs it.*

The accumulation is against **one target at a time**, flushed when the target changes, which is correct because traces are processed in
order and pellets rarely alternate between targets.

## `W_FireShotgun`, `W_FireSuperShotgun`, `FireBullets`, `TraceAttack`

**Contract** — the two instant weapons: a number of pellets within a spread, each tracing and contributing damage, with blood or spark
effects at the impact.

**Invariants** — **the spread is a random perturbation of the aim direction per pellet**, in the plane perpendicular to it, which is why
the basis vectors are needed. The spread values and the pellet counts are the weapons' identity.

## `W_FireRocket`, `W_FireGrenade`, `GrenadeTouch`, `GrenadeExplode`, `T_MissileTouch`

**Contract** — launch a rocket that explodes on contact, and a grenade that bounces and explodes on a timer or on contact with a
creature.

**Invariants** —

- **A grenade uses the engine's bouncing movement type and a rocket uses the flying one**
  ([`sv_phys.c`](../QW/server/sv_phys.c.md)), so the arc and the bounce are the engine's, not the game's. The game supplies a velocity
  and a handler.
- **A grenade explodes on a timer regardless**, which is what makes it usable around corners and unusable at close range.
- Both do direct damage plus radius damage ([`combat.qc`](combat.qc.md)).

## `W_FireSpikes`, `W_FireSuperSpikes`, `launch_spike`, `spike_touch`, `superspike_touch`

**Contract** — launch the rapid projectiles, alternating the muzzle position, with a variant carrying more damage.

**Invariants** — **these are the projectiles the protocol special-cases** ([`sv_ents.c`](../QW/server/sv_ents.c.md)): the engine
recognizes them by model and encodes them in six bytes. So the game's choice of model here is coupled to the network encoding — a
coupling by name, with nothing enforcing it, and one of the few places where changing the game logic's content silently changes the
protocol's behaviour.

## `W_FireLightning`, `LightningDamage`, `LightningHit`

**Contract** — trace a beam along the aim, damage everything it passes through, and tell the client to draw it.

**Invariants** — **damage is applied to everything along the beam, found by repeated tracing**, not to the first thing hit. And the
beam is drawn by the client from a temporary-entity message ([`cl_tent.c`](../QW/client/cl_tent.c.md)) re-aimed each frame from the
owner's predicted position — so the visual and the damage are computed in different places and need not agree exactly.

**Firing it in water damages everything in the water**, which is the discharge rule. It is one of the game's best-known interactions
and it is a few lines here.

## `W_FireAxe`, `SpawnMeatSpray`, `SpawnBlood`, `spawn_touchblood`

**Contract** — the melee attack, and the impact effects.

## `W_WeaponFrame`, `W_Attack`, `W_ChangeWeapon`, `W_SetCurrentAmmo`, `W_BestWeapon`, `W_CheckNoAmmo`, `W_Precache`, and the animation functions

**Contract** — the per-frame weapon state machine: advance the animation, fire when the button is pressed and the animation allows,
change weapons on request, select the best available weapon when the current one runs out, and set the model and ammunition counter for
the current weapon.

**Invariants** —

- **Firing is gated by the animation, not by a timer.** The animation sequence's length is the rate of fire, so changing a weapon's
  animation changes its damage per second. That coupling is economical and surprising, and a rebuild separating them will find the
  weapons feel different.
- **Running out of ammunition switches to the best remaining weapon**, by a fixed preference order, which is a usability decision with a
  tactical consequence.
- **The animation runs on the server and the frame number is sent to the client**
  ([`sv_ents.c`](../QW/server/sv_ents.c.md)), for the player themselves and for whoever spectates them. So the weapon animation is
  authoritative and unpredicted, which is why it lags slightly on a poor link.

## `ImpulseCommands`, `CycleWeaponCommand`, `CycleWeaponReverseCommand`, `CheatCommand`, `ServerflagsCommand`

**Contract** — dispatch the numbered impulse the client sent: select a weapon, cycle forward or back, or one of the developer commands.

**Invariants** — **a client's requests arrive as a single numbered impulse per command**
([`protocol.h`](../QW/client/protocol.h.md)), which is one byte and dispatches here. That is the whole of the client-to-game
vocabulary beyond movement, and the numbering is a contract with every player's key bindings.

**The cheat commands check whether cheats are enabled**, and that check is here in the game logic rather than the engine — so a
modification can get it wrong. A rebuild should put the check where it cannot be omitted.

**Notes** — the two reusable ideas are the **damage accumulator** (several projections, one damage event) and the
**animation-as-rate-of-fire** coupling. The first is necessary; the second is a choice worth making consciously.
