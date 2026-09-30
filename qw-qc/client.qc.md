# qw-qc/client.qc

> The player's lifecycle: joining, choosing a spawn point, spawning, the two per-frame hooks, dying, respawning, the rules that end a level, and the intermission.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md) · [`combat.qc`](combat.qc.md) · [`items.qc`](items.qc.md) · [`weapons.qc`](weapons.qc.md)
**Used by** — the engine calls its callbacks directly ([`progdefs.h`](../QW/server/progdefs.h.md))
**Tier floor** — none

## Purpose

Everything the engine calls per player, and therefore the file that defines what being a player *is*. Read
[`defs.qc`](defs.qc.md)'s callback list first; this file implements it.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## `ClientConnect`, `ClientDisconnect`, `PutClientInServer`

**Contract** — announce a player joining and give them their initial state; announce them leaving and free their body; and place a
player in the world: decode their persistent parameters into health, armour, weapons and ammunition, pick a spawn point, place them
there, give them the standard body size and view height, clear their death state, spawn the teleport effect, and hand them their weapon.

```text
FUNCTION put_client_in_server()
  clear every field that should not survive a respawn
  health, armour, weapons, ammunition = decode_level_parms()
  origin = select_spawn_point()
  angles = that spawn point's angles ;  fixangle = true
  model, size, view offset = the standard player values
  movetype = walk ;  solid = slidebox ;  takedamage = yes
  think = the spawn-effect handler ;  nextthink = time + a frame
  set the current weapon and its model
```

**Invariants** —

- **A player's state is reconstructed from sixteen numbers on every spawn**
  ([`DecodeLevelParms`](#decodelevelparms-setchangeparms-setnewparms)), never carried in fields, because fields do not survive a level
  change.
- **The forced-angle flag is set**, which tells the engine to send the angle to the client rather than accept the client's
  ([`sv_ents.c`](../QW/server/sv_ents.c.md)). It is the only time the server overrides the player's aim, and
  [`docs.txt`](../QW/client/docs.txt.md) records that it travels unreliably and can be lost — so a teleport occasionally leaves a player
  facing wrong.
- **The body size is the movement model's fixed box** ([`pmove.c`](../QW/client/pmove.c.md)) and the map's precompiled hull
  ([`model.c`](../QW/server/model.c.md)). The game may change it — crouching, gibbing — and the engine then shifts the origin to
  compensate ([`sv_user.c`](../QW/server/sv_user.c.md)).
- Everything that must not survive a respawn is cleared **explicitly**, one field at a time. A missed field is a cheat: a player
  respawning with a powerup still active.

## `SelectSpawnPoint`, `CheckSpawnPoint`

**Contract** — choose where a player appears: in a team or cooperative game, prefer a point associated with their team; otherwise pick
from the deathmatch points, avoiding the one just used and any occupied by another player.

```text
FUNCTION select_spawn_point()
  IF coop or team play  try the points of that kind first
  collect the deathmatch spawn points
  pick one at random, skipping the last one used
  IF a player is standing in it  try the next
  IF none is free  use it anyway, which will telefrag the occupant
```

**Invariants** —

- **Spawning into an occupied point kills the occupant**, rather than failing or stacking. That is a design decision with real
  consequences — a crowded server produces spawn kills — and the alternatives (waiting, or stacking players) were each worse.
- **The point just used is avoided**, which is the whole of the anti-spawn-camping measure. It is weak and it is what shipped.
- Points are declared by the map as entities of several classes
  ([`info_player_*`](#validateuser-player_pain-player_stand1-spawn_tfog-spawn_tdeath-info_player_start-info_player_start2-info_player_deathmatch-info_player_coop)), so the choice of where players
  appear is the level designer's.

## `PlayerPreThink`, `PlayerPostThink`

**Contract** — before the engine runs a player's movement: handle their death state, check the level's end conditions, apply the
liquid effects, run the weapon animation and the jump; after it: apply the landing sound and damage, and expire the powerups.

**Invariants** —

- **The split is around the movement**, which the engine runs between them
  ([`sv_user.c`](../QW/server/sv_user.c.md)). So anything that must influence the movement — the jump, the water handling — is before,
  and anything that reacts to it — the fall damage, the landing sound — is after. **That ordering is the whole meaning of the two
  callbacks** and getting a rule on the wrong side of it is a subtle behavioural bug.
- **The jump is handled here and *also* in the movement model** ([`pmove.c`](../QW/client/pmove.c.md)): the model produces the upward
  velocity and this produces the sound and the animation. The split exists because the client predicts the first and cannot know the
  second.
- Fall damage is computed from the **velocity the player had before landing**, which the post-think can still see because the engine
  records the ground contact rather than zeroing the velocity.

## `PlayerDie`, `PlayerDeathThink`, `respawn`, `set_suicide_frame`, `ClientKill`

**Contract** — die: drop the backpack, clear the powerups, choose a death animation or burst into pieces if the damage was large enough,
and enter the waiting-to-respawn state; then, once a key is pressed and a delay has passed, respawn.

**Invariants** —

- **A death with enough damage produces pieces instead of a body**, which is a threshold comparison and a different model.
- **The dead player's body is moved to a queue of recent corpses** ([`world.qc`](world.qc.md)) so that it persists after they respawn,
  because a body that vanishes at respawn reads as the player having escaped.
- **Respawning requires a key press after a minimum delay**, so a player is not returned to the fight before they have seen what
  happened.
- A player killing themselves is credited as such ([`combat.qc`](combat.qc.md)).

## `DecodeLevelParms`, `SetChangeParms`, `SetNewParms`

**Contract** — unpack the sixteen persistent numbers into the player's state; pack the current state back into them for a level change,
capping health and stripping keys; and set them to the starting values for a new player.

```text
FUNCTION set_change_parms()
  IF the player is dead  set the new-player values instead
  strip the keys and the powerups
  health = MAX(50, health)                # never arrive nearly dead
  parm1 = items ;  parm2 = health ;  parm3 = armour value
  parm4..7 = the four ammunition counts
  parm8 = the current weapon ;  parm9 = the armour type
```

**Invariants** —

- **Sixteen numbers are the entire persistence across a level** ([`defs.qc`](defs.qc.md)), so the packing is the game's save format and
  it is hand-written. Adding a persistent property means finding a spare number or encoding two into one.
- **Health is floored on a level change**, so a player never begins a level unable to survive it. A design decision, and the kind of
  thing a rebuild will omit and then wonder why its campaign is unfair.
- **Keys are stripped**, because a key belongs to one level.

## `CheckRules`, `NextLevel`, `GotoNextMap`, `execute_changelevel`, `changelevel_touch`, `trigger_changelevel`, `FindIntermission`, `IntermissionThink`, `info_intermission`

**Contract** — end the level when the score or time limit is reached; choose the next level from the rotation or the map's own
declaration; run the change, telling the engine; and the intermission: freeze the players, move the camera to a declared viewpoint,
show the scoreboard, and proceed when everyone is ready.

**Invariants** —

- **The level's end is checked in the pre-think, once per player per frame**, which is wasteful and harmless. There is no other place a
  periodic check could live, because the frame callback ([`world.qc`](world.qc.md)) runs before the players.
- **The intermission camera is a map-declared entity**, so a level designer chooses the final view. Its position and angles are sent as
  a message ([`protocol.h`](../QW/client/protocol.h.md)) rather than by moving the player, which is why the protocol's intermission
  message carries a position where the original's carried music.
- **A level change does not disconnect anyone** ([`sv_init.c`](../QW/server/sv_init.c.md)), which is the QuakeWorld difference and the
  reason this code differs most from the original's.

## `WaterMove`, `CheckWaterJump`, `PlayerJump`, `CheckPowerups`

**Contract** — drowning, liquid damage and the bubbles; the ledge-exit impulse; the jump's sound and animation; and expiring the timed
powerups with their warning sounds.

**Invariants** —

- **Drowning is on a timer that resets when the head leaves the water**, and the damage accelerates. The numbers are the design.
- **The water-jump and the jump itself duplicate logic the movement model also has**
  ([`pmove.c`](../QW/client/pmove.c.md)) — the model does the physics, this does the presentation. The duplication is inherent to
  predicting movement and not predicting game logic, and a rebuild inherits it.
- **A powerup warns before it expires**, by sound, because an invulnerability that ends without notice is unfair.

## `ValidateUser`, `player_pain`, `player_stand1`, `spawn_tfog`, `spawn_tdeath`, `info_player_start`, `info_player_start2`, `info_player_deathmatch`, `info_player_coop`

**Contract** — the remaining handlers and the spawn-point classes.

**Notes** — the two things to take from this file are the **pre-think/post-think ordering rule** — what influences movement versus what
reacts to it — and the **sixteen-number persistence**, which is the honest shape of "carry state across a level" when the only storage is
the engine's fixed globals. Both are constraints a rebuild can lift, and both are worth understanding before lifting them.
