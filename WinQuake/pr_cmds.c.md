# WinQuake/pr_cmds.c

> The host functions: the eighty-odd engine services the game logic calls out to, indexed by number, which together define what the interpreted language can actually do.

**Needs** — [`progs.h`](progs.h.md) · [`pr_comp.h`](pr_comp.h.md) · [`quakedef.h`](quakedef.h.md) · [`server.h`](server.h.md) · [`world.h`](world.h.md) (tracing and linking) · [`model.h`](model.h.md) (visibility queries) · [`mathlib.h`](mathlib.h.md) · [`common.h`](common.h.md) (message writing) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md) · [`client.h`](client.h.md) (the client record for per-player messages) · [`pr_exec.c`](pr_exec.c.md) · [`pr_edict.c`](pr_edict.c.md) · [`sv_move.c`](sv_move.c.md)
**Used by** — [`pr_exec.c`](pr_exec.c.md) dispatches into the table; nothing else calls these directly
**Tier floor** — none of its own

## Purpose

The interpreter ([`pr_exec.c`](pr_exec.c.md)) can compute, branch and call. It cannot
move an entity, trace a line, play a sound or send a message. Everything the game
*does* to the world it does through this table, and the table is therefore the real
definition of the game logic's power.

The indices are the interface. A game function declares itself as `= #17` and the
compiler emits a negative first-statement of −17; the engine supplies slot 17. So
**the numbering is a permanent contract** with every published game modification, and
the gaps in it — eleven entries that raise an error — are numbers that were once
something and must stay unusable.

## State

```text
VARIABLE pr_builtin      : list<handler>   # the table, indexed by number
VARIABLE pr_numbuiltins  : int
VARIABLE pr_string_temp  : text[128]       # ONE shared buffer for every
                                           # number-to-string conversion
VARIABLE checkpvs        : bytes           # a copy of one entity's visibility
                                           # bitset, held between frames
VARIABLE sv_aim          : Cvar = 0.93     # the auto-aim cone, as a cosine
```

**Invariants** — a host function takes no arguments and returns nothing in the host
language. It reads argument *n* from global slot `4 + n*3` and writes its result to global
slot 1 ([`pr_comp.h`](pr_comp.h.md)). The argument count is in `pr_argc`, so a function
may be variadic, and three of them are.

The single shared conversion buffer means **only one converted string is live at a time**.
Game logic that formats two numbers into one message gets the second value twice. This is a
real, observable limitation that published game logic works around, and a rebuild must
either reproduce it or accept that some modifications behave differently.

## Argument access

```text
FUNCTION var_string(first) -> text
  # Concatenate arguments `first` onward, as strings, into a 256-byte buffer.
  out = ""
  FOR i FROM first TO pr_argc-1
    append the string at argument slot i TO out        # NO bound check
  RETURN out
```

**Invariants** — no bounds check on the 256-byte result. Game logic printing long text
overruns it. The variadic functions are the print family and the two error functions.

---

## Errors

### `PF_error` — `error(string...)` = #10

**Contract** — prints the message and the current entity, then raises a host error that
**takes down the server**. The game's way of saying the level cannot continue.

### `PF_objerror` — `objerror(string...)` = #11

**Contract** — prints the message and the current entity, **frees the current entity**,
then raises a host error. The level may be reloaded; the object is gone.

### `PF_Fixme`

**Contract** — occupies every reserved or removed table slot; raises an interpreter error
naming the call as unimplemented.

**Invariants** — eleven slots hold it in this build: #5 (a set-absolute-size function that
was removed), #33, #39, #42, #49, and #52 through #58 (the sequel's trigonometry, pitch
turning, toss tracing, entity-to-string and water handling), plus #72. Those numbers are
**burned**: a rebuild must not reuse them, because a modification compiled against a
different engine may call them and must get an error rather than the wrong function.

### `PF_break` — `break()` = #6

**Contract** — prints a message and then **deliberately writes to an invalid address** to
drop into a debugger.

**Notes** — a debugging trap from 1996 that crashes the process. A rebuild should raise an
interpreter error instead; the commented-out line in the source does exactly that.

---

## Entity placement

### `PF_setorigin` — `setorigin(entity, vector)` = #2

**Contract** — sets an entity's position and relinks it into the collision system. **The
only legitimate way to move an entity without physics**: assigning the position field
directly leaves the spatial links stale and breaks collision for that entity and for
anything testing against it.

```text
FUNCTION pf_setorigin()
  e = entity argument 0
  e.origin = vector argument 1
  sv_link_edict(e, touch_triggers = false)
```

**Invariants** — the relink does *not* trigger touch functions. So teleporting an entity
into a trigger does not fire it this frame; the game uses the retouch countdown
([`progdefs.q1`](progdefs.q1.md)) when it needs that.

### `SetMinMaxSize`

**Contract** — takes an entity and a bounding box; validates that the minimum is no
greater than the maximum on every axis, stores the box and its size, and relinks. Fatal
interpreter error on a reversed box.

```text
FUNCTION set_min_max_size(e, min, max, rotate)
  FOR EACH axis i
    IF min[i] > max[i]  FAIL WITH "backwards mins/maxs"
  rotate = false                       # forced; see the note
  e.mins = min ;  e.maxs = max ;  e.size = max - min
  sv_link_edict(e, touch_triggers = false)
```

**Invariants** — **the rotation parameter is forced off** at the top of the function, so
the axis-aligned-box-under-rotation code beneath it never runs. The source's comment says
to implement rotation properly again. The dead code computes the axis-aligned bound of a
box rotated about the vertical axis by brute force over its eight corners; a rebuild
wanting rotating brush entities needs exactly that and should write it correctly.

The consequence of the forcing is visible in the game: a rotated model's collision box is
its unrotated one.

### `PF_setsize` — `setsize(entity, vector, vector)` = #4

**Contract** — sets an entity's bounding box, unrotated.

### `PF_setmodel` — `setmodel(entity, string)` = #3

**Contract** — sets an entity's model by name; requires the name to have been precached,
and adopts the model's own bounding box. Interpreter error on a name not in the precache
list.

```text
FUNCTION pf_setmodel()
  e = entity argument 0 ;  name = string argument 1
  i = the index OF name IN sv.model_precache        # a linear scan
  IF not found  FAIL WITH "no precache: <name>"
  e.model = name ;  e.modelindex = i
  mod = sv.models[i]
  set_min_max_size(e, mod.mins, mod.maxs, rotate = true)   # rotate is forced off
  # a model that failed to load yields a zero-size box
```

**Invariants** — the model *index* is the position in the precache list, and that index is
what travels on the wire ([`protocol.h`](protocol.h.md)). So the precache order determines
the protocol's model numbering, and the client's list must be identical — which it is,
because the server sends the list in the server-information message.

Requiring a precache is not bureaucracy: the client only knows the models it was told
about at connect time, so setting an unprecached model would produce an entity the client
cannot draw.

---

## Printing

### `PF_bprint` — `bprint(string...)` = #23

**Contract** — prints to every connected player's console.

### `PF_sprint` — `sprint(entity, string...)` = #24

**Contract** — prints to one player's console, reliably. A non-player entity prints a
diagnostic locally and does nothing.

### `PF_centerprint` — `centerprint(entity, string...)` = #73

**Contract** — as above, to the centre of one player's screen.

### `PF_dprint` — `dprint(string...)` = #25

**Contract** — prints locally, and only when the developer variable is set.

**Notes** — the two per-player functions range-check the entity number against the player
count and print "tried to sprint to a non-client" — the same message from both, a copy
slip. They return quietly rather than erroring, so game logic may print to an arbitrary
entity harmlessly.

---

## Arithmetic the language lacks

### `PF_normalize` — `normalize(vector) -> vector` = #9

**Contract** — returns the unit vector in the same direction; a zero vector returns zero
rather than a not-a-number.

### `PF_vlen` — `vlen(vector) -> float` = #12

**Contract** — the Euclidean length.

### `PF_vectoyaw` — `vectoyaw(vector) -> float` = #13

**Contract** — the direction's yaw in degrees, in [0, 360). A vector with no horizontal
component returns zero.

```text
FUNCTION pf_vectoyaw()
  v = vector argument 0
  IF v[0] == 0 AND v[1] == 0
    yaw = 0
  ELSE
    yaw = truncate_to_int(atan2(v[1], v[0]) * 180 / pi)   # TRUNCATED to whole
    IF yaw < 0  yaw = yaw + 360                           # degrees
  RETURN yaw
```

**Invariants** — the result is **truncated to a whole number of degrees**. That is not
rounding for display: the value goes into an entity's ideal yaw and is compared for
equality against the current yaw by [`PF_changeyaw`](#pf_changeyaw-changeyaw-49), and the quantization
is what makes that comparison settle. A rebuild returning a real angle makes monsters
rotate forever without ever arriving.

### `PF_vectoangles` — `vectoangles(vector) -> vector` = #51

**Contract** — the direction's pitch and yaw in degrees, with roll zero. Both truncated to
whole degrees and normalized into [0, 360). A vector with no horizontal component returns
a pitch of 90 or 270 by the sign of the vertical component.

**Invariants** — the pitch sign is such that straight up is 90 and straight down is 270,
which is the **opposite** of the pitch convention everywhere else in the engine
([`mathlib.c`](mathlib.c.md#anglevectors), where positive pitch looks down). The game
logic negates it at every call site. A rebuild must keep the inconsistency, because
published game logic compensates for it.

### `PF_rint`, `PF_floor`, `PF_ceil` — = #36, #37, #38

**Contract** — round to nearest away from zero; round down; round up.

```text
FUNCTION pf_rint()
  f = argument 0
  RETURN truncate_to_int(f + 0.5) IF f > 0 ELSE truncate_to_int(f - 0.5)
```

**Invariants** — rounding is away from zero at the half, not to even, and is asymmetric
about zero in that exactly zero rounds down through the second branch. Both matter only
where the game rounds damage.

### `PF_fabs` — `fabs(float) -> float` = #43

**Contract** — magnitude.

### `PF_random` — `random() -> float` = #7

**Contract** — returns a value in [0, 1), from the language's own generator masked to
fifteen bits and divided by that mask.

```text
FUNCTION pf_random()
  RETURN (next_random() BITAND 0x7FFF) / 32767.0
```

**Invariants** — **32767 distinct values**, and the divisor is the mask rather than the
mask plus one, so the value 1.0 *is* attainable when the masked value is the mask itself.
Game logic that indexes an array by `random() * n` can therefore index one past the end.
Published logic guards against it by comparison rather than indexing. A rebuild should
divide by 32768.

The generator is not seeded by the engine, so the sequence is identical on every run of the
process unless a platform backend seeds it. That is a real determinism property some
speedrunning depends on, and it is accidental.

---

## Sound

### `PF_sound` — `sound(entity, float, string, float, float)` = #8

**Contract** — plays a sound on one of an entity's eight channels, at a volume in [0, 1]
scaled to a byte and an attenuation in [0, 4]. A volume, attenuation or channel outside
those ranges is a **fatal system error**, not an interpreter error.

**Invariants** — channel 0 allocates a fresh slot; channels 1 through 7 replace whatever
is on that entity's channel. The header comment states the roles the game assigns —
voice, weapon, feet — and the replacement is what makes a monster's pain sound cut its
idle sound.

Failing fatally rather than recoverably on a bad argument is out of character for this
file and a rebuild should downgrade it.

### `PF_ambientsound` — `ambientsound(vector, string, float, float)` = #74

**Contract** — appends a permanent looping sound at a position to the **signon** stream, so
every client that ever connects receives it. Requires the sound to be precached; an
unprecached name prints and does nothing.

**Invariants** — it writes to the signon rather than to a datagram, which means it is only
meaningful during level load and that the sound cannot be stopped. The volume is scaled by
255 and the attenuation by 64 into single bytes.

---

## Tracing and the world

### `PF_traceline` — `traceline(vector, vector, float, entity)` = #16

**Contract** — traces a zero-sized line between two points, optionally ignoring monsters,
ignoring one named entity; writes eight result globals. Returns nothing.

```text
FUNCTION pf_traceline()
  trace = sv_move(from = argument 0, mins = zero, maxs = zero,
                  to = argument 1,
                  nomonsters = argument 2, ignore = argument 3)
  the global trace_allsolid    = trace.allsolid
  the global trace_startsolid  = trace.startsolid
  the global trace_fraction    = trace.fraction
  the global trace_inwater     = trace.inwater
  the global trace_inopen      = trace.inopen
  the global trace_endpos      = trace.endpos
  the global trace_plane_normal= trace.plane.normal
  the global trace_plane_dist  = trace.plane.dist
  the global trace_ent         = trace.ent, OR the world when nothing was hit
```

**Invariants** — the result is returned **through globals**, so a trace cannot be nested
and the game must consume the result before tracing again. And a trace that hit nothing
sets the hit entity to the *world*, not to nothing, because the language has no null
distinct from the world ([`progs.h`](progs.h.md)).

The declared signature in published game logic takes three parameters; the engine reads a
fourth. The compiler places arguments in fixed slots, so the fourth slot holds whatever the
previous call left there unless the game declares it. Published game logic does declare
it.

### `PF_pointcontents` — `pointcontents(vector) -> float` = #41

**Contract** — returns the world's contents value at a point: empty, solid, water, slime,
lava or sky ([`bspfile.h`](bspfile.h.md)). Ignores entities.

### `PF_checkbottom` — `checkbottom(entity) -> float` = #40

**Contract** — reports whether an entity's bounding box is fully supported from below, so
that a monster may step without falling. See [`sv_move.c`](sv_move.c.md#sv_checkbottom).

### `PF_droptofloor` — `droptofloor() -> float` = #34

**Contract** — traces the current entity's box downward up to 256 units; on a hit, moves it
there, marks it as on the ground and records what it is standing on, and returns success;
otherwise returns failure and leaves it where it was.

**Invariants** — 256 units is a fixed reach. An entity placed higher than that above the
floor by a map author stays in the air and returns failure, which every map entity's spawn
function checks. This is how every item in the game settles onto its floor.

### `PF_walkmove` — `walkmove(float, float) -> float` = #32

**Contract** — attempts to move the current entity a distance along a yaw, stepping up and
down as a walking monster does; returns whether it succeeded. Returns failure immediately
if the entity is neither on the ground, flying nor swimming. Saves and restores the
interpreter's current function and current entity around the move, because the move can
trigger touch functions that run game code.

```text
FUNCTION pf_walkmove()
  ent = the current entity
  IF ent.flags LACKS on-ground, flying AND swimming  RETURN 0
  yaw = argument 0 IN RADIANS ;  dist = argument 1
  move = (cos(yaw)*dist, sin(yaw)*dist, 0)
  saved_function = pr_xfunction ;  saved_self = the global self
  result = sv_movestep(ent, move, relink = true)     # may run game code
  pr_xfunction = saved_function ;  the global self = saved_self
  RETURN result
```

**Invariants** — the save-and-restore is required and is easy to miss. `sv_movestep`
relinks the entity, which fires touch functions, which call back into the interpreter,
which overwrites both the current function pointer and the current-entity global. Without
the restore, the caller resumes with the wrong `self` and the wrong diagnostic context.
Two other host functions in this file have the same hazard and do **not** restore, which is
a latent bug.

### `PF_checkclient` — `checkclient() -> entity` = #17

**Contract** — returns a player the current entity might be able to see, or the world if
none. Cycles through the players one per tenth of a second and caches that player's
visibility bitset; returns the cached player only if the current entity's own leaf is
marked visible in it.

```text
FUNCTION pf_checkclient()
  IF sv.time - sv.lastchecktime >= 0.1
    sv.lastcheck = next_check_client(sv.lastcheck)   # advance the cursor,
    sv.lastchecktime = sv.time                       # and cache its bitset

  ent = entity sv.lastcheck
  IF ent IS free OR ent.health <= 0  RETURN the world

  self = the current entity
  leaf = the map leaf containing self.origin + self.view_ofs
  l = index_of(leaf) - 1
  IF l < 0 OR bit l OF the cached bitset IS CLEAR  RETURN the world
  RETURN ent

FUNCTION next_check_client(check) -> int
  # Walk the player slots circularly from `check`+1, skipping players that are
  # free, dead, or flagged as not-a-target; stop after one full lap.
  clamp check INTO [1, maxclients]
  i = 1 IF check == maxclients ELSE check + 1
  LOOP
    IF i == maxclients+1  i = 1
    ent = entity i
    IF i == check  BREAK                    # came all the way round
    IF ent IS free OR ent.health <= 0 OR ent.flags HAS not-a-target
      i = i + 1 ;  CONTINUE
    BREAK
  # cache the chosen entity's visibility bitset for the next tenth of a second
  leaf = the map leaf containing ent.origin + ent.view_ofs
  checkpvs = a COPY of that leaf's visibility bitset
  RETURN i
```

**Invariants** — this is the engine's monster-awareness budget, and it is a deliberate
approximation. **One player's visibility is evaluated per tenth of a second**, shared by
every monster that asks in that interval. So with four players, a given monster's target
candidate cycles at 2.5 Hz, and a monster can fail to notice a player who is plainly
visible for up to four tenths of a second. That latency is part of the game's feel.

The leaf index is offset by one because leaf 0 is the solid leaf and carries no visibility
([`bspfile.h`](bspfile.h.md)); a negative result means the asking entity is inside a wall
and sees nothing.

The bitset is **copied** rather than referenced, because the model's decompression buffer
is reused. The copy is up to 1024 bytes and happens ten times a second.

**Notes** — two counters in the source tally visible and invisible results and are never
reported. Dead diagnostics.

### `PF_aim` — `aim(entity, float) -> vector` = #44

**Contract** — returns the direction a player's shot should take: the exact view direction
if it already hits something damageable, otherwise the direction to the best nearby
damageable entity within a cone, otherwise the view direction. Respects team play by
skipping teammates.

```text
FUNCTION pf_aim()
  ent = entity argument 0
  start = ent.origin, raised 20 units
  dir = the global v_forward

  # First: does looking straight ahead already hit something shootable?
  tr = trace a point FROM start ALONG dir FOR 2048 units, ignoring ent
  IF tr.ent takes aimed damage AND is not a teammate
    RETURN v_forward                              # no adjustment at all

  # Otherwise: find the shootable entity closest to the view direction,
  # within the cone, that is actually reachable.
  best_cosine = sv_aim                            # 0.93, about 21.6 degrees
  best = nothing
  FOR EACH entity check FROM 1 upward
    IF check does not take aimed damage  CONTINUE
    IF check IS ent                      CONTINUE
    IF team play AND check is a teammate CONTINUE
    target = the centre of check's bounding box
    d = normalize(target - start)
    cosine = dot(d, v_forward)
    IF cosine < best_cosine  CONTINUE             # outside the cone
    tr = trace a point FROM start TO target, ignoring ent
    IF tr.ent IS check                            # actually reachable
      best_cosine = cosine ;  best = check

  IF best EXISTS
    # Aim horizontally along the view direction but vertically at the target:
    d = best.origin - ent.origin
    horizontal = v_forward SCALED BY dot(d, v_forward)
    result = horizontal WITH its vertical component REPLACED BY d's
    RETURN normalize(result)
  RETURN v_forward
```

**Invariants** — the cone is a **cosine**, not an angle, and the variable's default of
0.93 is about 21.6 degrees of half-angle. Setting it to 1 disables auto-aim entirely, which
is what the value's existence is for.

The final direction is the strange part and it is deliberate: the horizontal direction
stays the player's own, and only the **vertical** component is taken from the target. So
auto-aim adjusts elevation, not bearing. That is what makes it feel like assistance rather
than like the game shooting for you, and a rebuild that aims fully at the target changes
the game's feel markedly.

The candidate loop does not skip free entities, so a freed entity whose damage field
survived ([`pr_edict.c`](pr_edict.c.md#ed_free) does not clear it) can be a candidate; the
reachability trace then fails and it is discarded. Accidental correctness.

### `PF_changeyaw` — `changeyaw()` = #49

**Contract** — turns the current entity's yaw toward its ideal yaw by at most its turn
speed, taking the shorter way round.

```text
FUNCTION pf_changeyaw()
  ent = the current entity
  current = anglemod(ent.angles[YAW])          # quantized to 360/65536
  ideal = ent.ideal_yaw ;  speed = ent.yaw_speed
  IF current == ideal  RETURN                  # exact comparison
  move = ideal - current
  IF ideal > current AND move >= 180   move = move - 360     # go the short way
  IF ideal <= current AND move <= -180 move = move + 360
  clamp move INTO [-speed, +speed]
  ent.angles[YAW] = anglemod(current + move)
```

**Invariants** — the equality test is exact, and it works only because both sides are
quantized: the current yaw through the angle reduction
([`mathlib.c`](mathlib.c.md#anglemod)) and the ideal yaw through
[`PF_vectoyaw`](#pf_vectoyaw-vectoyawvector---float-13)'s truncation to whole degrees. Break either quantization and
monsters never stop turning. This is the single best example in the engine of two
quantizations that exist to make an equality comparison terminate.

**Notes** — the source's comment says this was moved from the interpreted language into
the engine because it was a major time waster, which is the only host function in the table
that exists purely for speed.

---

## Entity lifecycle and queries

### `PF_Spawn` — `spawn() -> entity` = #14

**Contract** — allocates and returns a cleared entity.

### `PF_Remove` — `remove(entity)` = #15

**Contract** — frees an entity.

### `PF_Find` — `find(entity, .string, string) -> entity` = #18

**Contract** — takes a starting entity, a string-typed field, and a value; returns the next
entity after the start whose field equals the value, or the world when none. A null search
string is an interpreter error.

```text
FUNCTION pf_find()
  e = index of entity argument 0 ;  f = field argument 1 ;  s = string argument 2
  IF s IS nothing  FAIL WITH "PF_Find: bad search string"
  FOR e FROM e+1 TO num_edicts-1
    ed = entity e
    IF ed IS free  CONTINUE
    IF the string at ed's field f EQUALS s  RETURN ed
  RETURN the world
```

**Invariants** — returning the world at the end is what makes the game's idiom
`while (e) e = find(e, targetname, t)` terminate, because the world is falsehood
([`pr_exec.c`](pr_exec.c.md)). String comparison is by content.

**Notes** — the sequel build of this function instead threads *every* match onto the scratch
chain field and returns the first, which changes the calling convention. Not compiled here.

### `PF_findradius` — `findradius(vector, float) -> entity` = #22

**Contract** — returns a chain of every solid entity whose bounding-box centre lies within a
radius of a point, threaded through the scratch chain field, ending at the world.

```text
FUNCTION pf_findradius()
  chain = the world
  org = vector argument 0 ;  rad = argument 1
  FOR EACH entity ent FROM 1 upward
    IF ent IS free  CONTINUE
    IF ent.solid IS not-solid  CONTINUE
    centre = ent.origin + (ent.mins + ent.maxs) * 0.5
    IF length(org - centre) > rad  CONTINUE
    ent.chain = chain ;  chain = ent          # push onto the chain
  RETURN chain
```

**Invariants** — the distance is to the box's **centre**, not to the box, so a large
entity is missed by an explosion that overlaps it but not its centre. That is how splash
damage behaves in the original game and it is observable.

The chain is threaded through a *shared* field, so **only one such query can be live at a
time** and nesting two destroys the outer one. A rebuild returning a list is better and
still needs to expose the chain, because published game logic walks it.

Non-solid entities are skipped, which is why explosions do not damage items.

### `PF_nextent` — `nextent(entity) -> entity` = #47

**Contract** — returns the next non-free entity after the given one, or the world at the
end. The game's iteration primitive over all entities.

---

## Precaching

### `PF_precache_model` — `precache_model(string) -> string` = #20, also #75

### `PF_precache_sound` — `precache_sound(string) -> string` = #19, also #76

**Contract** — register an asset name in the level's list, loading the model immediately.
Returns the name unchanged so the call can be used inline. **Only legal while the level is
loading**; calling later is an interpreter error. A name beginning with a space or a
control character is an error. A repeated name is accepted and does nothing. Overflowing
the 256-entry list is an error.

```text
FUNCTION pf_precache_model()
  IF the server is not loading
    FAIL WITH "Precache can only be done in spawn functions"
  s = string argument 0
  return value = s                                  # so it can be used inline
  IF s[0] <= ' '  FAIL WITH "Bad string"
  FOR i FROM 0 TO 255
    IF sv.model_precache[i] IS empty
      sv.model_precache[i] = s
      sv.models[i] = load the model named s          # immediately
      RETURN
    IF sv.model_precache[i] == s  RETURN             # already present
  FAIL WITH "PF_precache_model: overflow"
```

**Invariants** — the load-time restriction exists because the list is sent to clients in
the server-information message, so adding to it afterwards would give the server a name the
client does not have. The index in this list is the model or sound number on the wire.

Returning the argument is what allows `self.model = precache_model("x")` as one expression.

Two numbers each map to the same function — #20 and #75 for models, #19 and #76 for
sounds. The source notes that the duplicate differs only inside the compiler, where it
suppresses a warning.

### `PF_precache_file` — `precache_file(string) -> string` = #68, also #77

**Contract** — returns its argument and does nothing else. The name exists so the compiler
can notice a file and copy it into a distribution.

---

## Console and variables

### `PF_stuffcmd` — `stuffcmd(entity, string)` = #21

**Contract** — sends console text to one player, to be executed on their machine. A
non-player entity is an interpreter error. Temporarily switches the engine's notion of the
current client around the send.

**Invariants** — this is the **server telling a client to run a command**, and it is the
protocol's largest trust boundary ([`protocol.h`](protocol.h.md)). The temporary switch of
the current-client global is required because the underlying send reads it.

### `PF_localcmd` — `localcmd(string)` = #46

**Contract** — appends console text to the *server's* own command queue, executed at the end
of the frame.

**Invariants** — appended, not inserted, so it runs after the current frame completes. The
game uses it for level changes and for anything needing the console.

### `PF_cvar` — `cvar(string) -> float` = #45

### `PF_cvar_set` — `cvar_set(string, string)` = #72

**Contract** — read and write a console variable by name. An unknown name reads as zero;
writing an unknown name prints a diagnostic and does nothing.

---

## Number formatting

### `PF_ftos` — `ftos(float) -> string` = #26

**Contract** — formats a number into the shared buffer: as an integer when it has no
fractional part, otherwise to one decimal place.

### `PF_vtos` — `vtos(vector) -> string` = #27

**Contract** — formats a vector as three values to one decimal place inside single quotes.

**Invariants** — both write the **same shared 128-byte buffer** and return an offset into
the string table computed from its address. Only one result is live. The returned "string
table offset" is actually the buffer's address minus the table's base, which works only
because both are in the same address space; a rebuild with a real string table must
allocate or must return a distinguishable handle.

One decimal place is a real precision limit that game logic printing coordinates lives
with.

---

## Light styles

### `PF_lightstyle` — `lightstyle(float, string)` = #35

**Contract** — sets a light style's animation string and, if the level is running, sends it
to every connected player.

```text
FUNCTION pf_lightstyle()
  style = argument 0 ;  val = string argument 1
  sv.lightstyles[style] = val
  IF the server is not active  RETURN               # the signon will carry it
  FOR EACH client that is active or spawned
    write the lightstyle message, the style and the string TO its reliable
    message
```

**Invariants** — no range check on the style index against the 64-entry table, so a bad
index from game logic writes outside it. Reachable; a rebuild should check.

The animation string is a sequence of letters where `a` is dark and `z` is bright, one per
tenth of a second, cycled. That interpretation lives in the renderer
([`r_light.c`](r_light.c.md)), not here.

---

## Message writing

### `WriteDest`

**Contract** — resolves the destination selector every write function's first argument
carries, into one of four buffers.

```text
FUNCTION write_dest() -> buffer
  SELECT argument 0
    0  broadcast: the server's UNRELIABLE datagram, to everyone
    1  one:       the reliable message of the client named by the msg_entity
                  global; a non-client is an interpreter error
    2  all:       the server's RELIABLE datagram, to everyone
    3  init:      the signon stream, delivered to every future connection
    otherwise     FAIL WITH "WriteDest: bad destination"
```

**Invariants** — four destinations with genuinely different delivery semantics, selected by
a number the game passes. The recipient for the single-client case comes from a *global*,
not from the argument, which is why game logic sets that global before a run of writes.

### `PF_WriteByte`, `PF_WriteChar`, `PF_WriteShort`, `PF_WriteLong`, `PF_WriteCoord`, `PF_WriteAngle`, `PF_WriteString`, `PF_WriteEntity` — = #52 through #59 in the numbering, filled at #59 onward here

**Contract** — each resolves the destination and appends one value in the protocol's
encoding ([`common.c`](common.c.md)). The entity form writes the entity's **number** as a
16-bit value.

**Invariants** — these give game logic the ability to compose arbitrary protocol messages,
which is how every temporary-effect message in the game is produced: the game writes the
tag and the payload itself. So the protocol's temporary-entity numbering
([`protocol.h`](protocol.h.md)) is a contract between the game logic and the *client*, with
the server merely forwarding bytes. A rebuild changing that numbering must change the game
logic too.

---

## Static entities

### `PF_makestatic` — `makestatic(entity)` = #69

**Contract** — writes an entity's model, frame, colour map, skin, position and angles into
the signon stream as a permanent non-updatable entity, then frees the entity.

**Invariants** — the entity is **destroyed**, and the client receives it once at connect.
So a static entity cannot move, animate or be removed, and it costs no per-frame network
traffic. That is how every torch and every decorative object in the game is placed.

Angles are written with the coarse angle encoding, and the write loop interleaves one
coordinate and one angle per axis — so the wire order is x, pitch, y, yaw, z, roll, not all
three coordinates then all three angles. A rebuild's reader must match.

---

## Level flow

### `PF_setspawnparms` — `setspawnparms(entity)` = #78

**Contract** — copies a player's sixteen stored carry values into the globals, so that the
game can restore their inventory. A non-player entity is an interpreter error.

### `PF_changelevel` — `changelevel(string)` = #70

**Contract** — queues a level change as a console command. Ignores every call after the
first, within one level.

**Invariants** — the once-only guard is load-bearing: two players touching the exit in the
same frame would otherwise queue two level changes and load the level twice.

---

## `SV_MoveToGoal` — = #67

**Contract** — the monster navigation step, implemented in
[`sv_move.c`](sv_move.c.md#sv_movetogoal) and placed directly in this table rather than
wrapped. The only entry that is not a function in this file.

---

## The table

**Contract** — the ordered list of handlers, indexed by the negation of a function's first
statement. Its length is the valid index bound, checked by the interpreter.

**Invariants** — the numbering is a permanent contract. The table's shape, with the eleven
error entries preserving burned numbers and two pairs of duplicate entries, is not
something a rebuild may tidy: a published game modification calling #75 must reach the
model precache, and one calling #42 must get an error.

The sequel's seven functions occupy #52 through #58 and are compiled out here, each
replaced by the error handler. A rebuild targeting only the original game should do the
same rather than renumbering.
