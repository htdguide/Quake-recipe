# qw-qc/items.qc

> Everything that can be picked up: how an item is placed, what touching it does, how it returns after a delay, and the rules that differ between a cooperative game and a deathmatch.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md) · [`combat.qc`](combat.qc.md) · [`weapons.qc`](weapons.qc.md)
**Used by** — the map's entity list instantiates its classes; [`client.qc`](client.qc.md) for the backpack
**Tier floor** — none

## Purpose

The pickup system. Its structure is one idea applied twenty times — **an item is a touchable entity whose handler modifies the toucher
and then removes or hides itself** — plus the respawn machinery that makes a competitive game work.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## `StartItem`, `PlaceItem`

**Contract** — every item class ends by calling these: give the item its model and bounds, drop it to the floor, and make it a trigger
so a player passing through it touches it.

```text
FUNCTION place_item()
  movetype = none ;  solid = trigger
  origin += a small upward offset
  IF dropping it to the floor fails  report the item as stuck in the map
  keep the resting origin for the respawn

FUNCTION start_item()
  nextthink = time + a small delay        # let the map finish spawning
  think = place_item
```

**Invariants** —

- **Items are dropped to the floor at load rather than placed exactly**, so a level designer need not position them precisely, and an
  item over a hole is reported rather than silently falling forever. The report is a diagnostic a mapper depends on.
- **The resting position is remembered**, because a respawning item must return to where it started, not to where it was taken.
- **Placement is deferred by a frame**, because the floor may be a moving platform that has not spawned yet.

## `SUB_regen`

**Contract** — make a hidden item visible and solid again, and play the respawn sound.

**Invariants** — **an item that has been taken is hidden, not removed.** So it keeps its identity, its position and its targets, and
returning it is one function. Removing and re-spawning it would lose the map's wiring
([`subs.qc`](subs.qc.md)) and would give it a new entity slot — which matters, because entity slots are what the protocol addresses
([`sv_ents.c`](../QW/server/sv_ents.c.md)) and a changed slot makes the client treat it as a different object.

## The touch handlers

**Contract** — each item class has a handler: health adds to health up to a cap, or above it for the large one; armour sets both the
absorbed fraction and the capacity, but only if better than what the player has; a weapon grants the weapon and its ammunition;
ammunition adds to one counter up to its cap; a key sets a flag; a powerup sets a timer; the backpack grants everything it holds.

```text
FUNCTION item_touch()
  IF the toucher is not a living player  RETURN
  IF the toucher already has as much as this gives  RETURN     # do not waste it
  modify the toucher
  print the pickup message ;  play the sound
  fire this item's targets                        # items are triggers too
  IF this is a deathmatch  hide it and schedule its return
  ELSE                     remove it
```

**Invariants** —

- **An item refuses to be taken when it would be wasted**, so a player at full health walks over a health pack and it stays. That is a
  design decision with a tactical consequence — items are resources you leave for later — and it is the single most important rule in
  the file.
- **Armour is replaced only if better**, compared by the product of its fraction and its capacity, so a weak armour does not overwrite a
  strong one. The comparison is the armour design
  ([`combat.qc`](combat.qc.md) spends it).
- **An item fires its targets when taken**, so a map can make a pickup open a door.
- **Respawning happens in a deathmatch and removal in a cooperative game.** One flag, two entirely different games: a competitive game
  is about contested renewable resources and a cooperative one about depleting them. The design notes name that distinction directly
  ([`newnet.txt`](../QW/server/newnet.txt.md) sketches *consumption, renewable resources, depletable resources, rate of extraction*).
- **The respawn delay per item type is the pacing of a competitive match**, and those numbers are as much the game as the weapon damage
  values.

## `item_health`, `T_Heal`, `health_touch`, `item_megahealth_rot`

**Contract** — the health items, including the one that grants health above the normal cap and then decays it back down.

**Invariants** — **the over-cap health rots at a fixed rate and the item does not respawn until it has finished rotting.** That couples
the item's availability to the state of whoever took it, which is unusual and is what makes it a strategic pickup rather than a large
health pack.

## `armor_touch`, `item_armor1`, `item_armor2`, `item_armorInv`

**Contract** — the three armour grades, each with its own absorbed fraction and capacity.

## `weapon_touch`, `Deathmatch_Weapon`, `RankForWeapon`, `WeaponCode`, `bound_other_ammo`

**Contract** — grant a weapon; decide whether picking it up should also switch to it, by comparing the new weapon's rank to the
current one's; and clamp every ammunition counter to its maximum.

**Invariants** — **picking up a better weapon switches to it and a worse one does not**, by a fixed ranking. Switching unconditionally
would get players killed; never switching would make pickups feel inert.

## `powerup_touch` and the four powerup classes

**Contract** — grant a timed powerup: invulnerability, invisibility, the damage multiplier, or the liquid protection. Each sets a
deadline and a flag.

**Invariants** — **a powerup is a deadline plus an item flag**, expiring in the post-think
([`client.qc`](client.qc.md)) with a warning sound. The damage multiplier is read by the damage function on the *attacker*
([`combat.qc`](combat.qc.md)); invisibility is read by the renderer through the item flags
([`bothdefs.h`](../QW/client/bothdefs.h.md)), which is why those flags are in an engine header.

**In a deathmatch a powerup is dropped on death** rather than lost, so killing a player holding one is worth doing.

## `key_touch`, `key_setsounds`, `item_key1`, `item_key2`, `sigil_touch`, `item_sigil`

**Contract** — the keys, whose appearance and sound depend on which set of levels the map belongs to, and the episode tokens.

**Invariants** — a key's model is **chosen from the map's world entity**, because the same class must look like a silver key in one
setting and a rune in another. That indirection is content configuration living in the game logic.

## `BackpackTouch`, `DropBackpack`, `DropQuad`, `DropRing`, `q_touch`, `r_touch`

**Contract** — drop a pack holding the dead player's ammunition and weapon, and drop any powerup still running; and the handlers for
picking those up.

**Invariants** — **a dropped pack expires**, or a long match fills the map with them. And a dropped powerup keeps its *remaining*
time, not a fresh allocation — otherwise killing a player would refresh their powerup for whoever takes it.

**Notes** — the file's shape — every item is a touch handler plus a respawn rule — is worth copying wholesale. The two decisions worth
naming are **refuse a pickup that would be wasted** and **hide rather than remove**, and the second has a protocol consequence a
rebuild will not anticipate: an entity's slot is its network identity.
