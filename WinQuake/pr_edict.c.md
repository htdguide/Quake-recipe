# WinQuake/pr_edict.c

> Loads and validates the compiled game logic, allocates and frees entities, and serializes an entity as name-keyed text so that a save game survives a change to the field layout.

**Needs** — [`progs.h`](progs.h.md) · [`pr_comp.h`](pr_comp.h.md) · [`progdefs.h`](progdefs.h.md) · [`quakedef.h`](quakedef.h.md) · [`common.h`](common.h.md) (the token parser, byte-order accessors, file loader) · [`crc.h`](crc.h.md) · [`zone.h`](zone.h.md) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md) · [`server.h`](server.h.md) · [`world.h`](world.h.md) (unlinking a freed entity) · [`pr_exec.c`](pr_exec.c.md) (spawn functions)
**Used by** — [`host.c`](host.c.md) and [`sv_main.c`](sv_main.c.md) (loading a level) · [`host_cmd.c`](host_cmd.c.md) (save and restore) · [`pr_cmds.c`](pr_cmds.c.md) (entity lifecycle host functions) · [`sv_main.c`](sv_main.c.md) · [`pr_exec.c`](pr_exec.c.md) (diagnostics)
**Tier floor** — T1 as written; the field-by-offset access and in-place byte swapping are what bind it, and neither is load-bearing

## Purpose

Three jobs, and the interesting one is third. It loads the compiled game logic and checks
that its idea of an entity's layout matches the engine's. It manages the entity array,
whose slot reuse policy is subtler than it looks. And it serializes an entity as
**name-keyed text**, which is the decision that makes save games and map entity
descriptions the same format and makes both survive a game modification adding fields.

## State

```text
VARIABLE progs            : ProgramHeader   # the loaded image, on the hunk
VARIABLE pr_functions, pr_strings, pr_globaldefs, pr_fielddefs,
         pr_statements, pr_global_struct, pr_globals   # views into the image
VARIABLE pr_edict_size    : int             # bytes per entity
VARIABLE pr_crc           : int (16-bit)    # checksum of the whole image
VARIABLE type_size        : int[8]          # slots per ValueType

# a two-entry lookup cache for resolving a field name at run time
CONSTANT gefv_cachesize = 2
CONSTANT max_field_len  = 64
VARIABLE gefvCache : (field name, resolved Def)[2]
```

**Invariants** — every table is a *view into the loaded file*, not a copy, and the file is
byte-swapped in place at load. So the image is mutable, and the profile counters
([`pr_comp.h`](pr_comp.h.md)) are written into it during play.

The type-size table gives one slot for everything except vectors, which get three. Its
entries are computed from the host's type widths in the source, which makes them
accidentally correct; a rebuild should write `1,1,1,3,1,1,1,1` literally.

## `PR_LoadProgs`

**Contract** — loads the compiled game logic onto the hunk; checksums the entire file;
byte-swaps the header and then every table; validates the version and the layout checksum;
computes the entity stride; and establishes every table view. Fatal error on a missing
file, a wrong version, a layout mismatch, or a field definition carrying the
save-global flag.

```text
FUNCTION pr_load_progs()
  clear the field-name cache
  progs = load "progs.dat" onto the hunk
  IF progs IS nothing  FAIL WITH "couldn't load progs.dat"

  pr_crc = CCITT-16 OVER the WHOLE file          # before any byte swapping
  byte-swap every 32-bit word of the header

  IF progs.version != 6
    FAIL WITH "wrong version number (<n> should be 6)"
  IF progs.crc != the engine's compiled-in layout checksum
    FAIL WITH "system vars have been modified, progdefs.h is out of date"

  establish views: functions, strings, global defs, field defs, statements,
                   globals (both as a named record and as a slot array)

  pr_edict_size = progs.entityfields * 4
                + size_of(Edict) - size_of(EntVars)      # the engine's head

  byte-swap each statement's four 16-bit fields
  byte-swap each function's six 32-bit fields
  byte-swap each global def's type, offset and name
  FOR EACH field def
    byte-swap its type
    IF the save-global flag IS SET  FAIL WITH "fielddefs type & DEF_SAVEGLOBAL"
    byte-swap its offset and name
  byte-swap every global slot as a 32-bit word
```

**Invariants** — the checksum covers the file **as loaded**, before byte swapping, so it is
the same on any host. It is sent to clients in the server-information message
([`protocol.h`](protocol.h.md)), which is why it must be host-independent.

The stride is the game's declared slot count, times four, plus the engine's private head.
So the head's size is subtracted out of the compiled record size and added back, which is
the code's way of saying "the named fields overlay the game's slots". A rebuild that keeps
the head in a parallel array computes the stride as the slot count alone and is simpler.

Byte-swapping the globals as 32-bit words swaps **floats as integers**, which is correct
because a swap is a byte permutation regardless of interpretation.

The save-global flag being illegal on a field definition is a sanity check on the
compiler: the flag means "record this in a save game" and only globals are recorded that
way.

**Notes** — the header swap loops over `size_of(header)/4` words, which assumes the header
is exactly a run of 32-bit fields. It is. A rebuild should swap named fields.

Nothing validates the table offsets against the file's length. A truncated or crafted
image reads outside the buffer. A rebuild should validate all six offsets and counts before
establishing any view.

## `PR_Init`

**Contract** — registers four console commands — print one entity, print all, count them,
and the profiler — and eleven console variables: one to suppress monsters, one game
configuration value, four scratch values, and five persisted values.

**Notes** — the scratch and saved variables exist purely as storage the game logic can
reach from the interpreted side, with the saved ones surviving into the configuration
file. They are a general-purpose escape hatch for game modifications and carry no engine
meaning.

## Entity lifecycle

### `ED_ClearEdict`

**Contract** — zeroes an entity's whole interpreter region — the declared slot count, not
just the named fields — and marks the slot in use.

### `ED_Alloc`

**Contract** — returns a cleared entity. Prefers a slot that has been free long enough,
scanning from just past the player slots; otherwise extends the array. Fatal error when the
array is full.

```text
FUNCTION ed_alloc() -> Edict
  FOR i FROM maxclients+1 TO num_edicts-1
    e = entity i
    # A slot is reusable if it is free AND either the server is still in its
    # first two seconds or it has been free for at least half a second.
    IF e.free AND (e.freetime < 2 OR sv.time - e.freetime > 0.5)
      ed_clear_edict(e) ;  RETURN e
  IF i == MAX_EDICTS  FAIL WITH "no free edicts"
  num_edicts = num_edicts + 1
  e = entity i ;  ed_clear_edict(e) ;  RETURN e
```

**Invariants** — the **half-second delay before reuse** is the interesting part, and the
source explains it: a client interpolates an entity's position between frames by entity
number, so if number 42 is a rocket that explodes and number 42 immediately becomes a gib
across the room, the client draws the gib streaking from the rocket's position. Delaying
reuse makes the client see a removal and a separate creation. A rebuild that reuses slots
eagerly gets exactly that visual artifact, and it is subtle enough to be blamed on the
renderer.

The relaxation during the first two seconds is because level spawn allocates and frees
heavily, and the delay would exhaust the array.

The scan starts past the player slots, because those are reserved by index: player *n*
is always entity *n+1* ([`sv_main.c`](sv_main.c.md)). And it stops at the current high
water mark rather than the maximum, so the array grows monotonically and never shrinks
within a level.

**Notes** — the loop variable is compared against the maximum *after* the loop, but the
loop bound is the current count; when the count is below the maximum and no slot is
reusable, the check passes and the array grows. When the count has reached the maximum the
check fires. It works, but a rebuild should test the count directly.

### `ED_Free`

**Contract** — unlinks the entity from the collision system, marks the slot free, and
clears the handful of fields that would otherwise keep it visible or interactive: model,
damage acceptance, model index, colour map, skin, frame, position, angles, solidity; sets
the next think time to a value that never fires; and records the current server time.

**Invariants** — it clears *selected* fields, not all of them, so a freed entity retains
most of its data. That is deliberate: game logic holding a handle to a freed entity reads
plausible-but-inert values rather than zeros, and the engine's own delta encoding sees a
zeroed model index and stops sending the entity.

**Notes** — the source's own comment marks the real gap: nothing walks the other entities
nulling out references to this one. So a freed entity can remain some other entity's
enemy, owner or ground entity, and the game logic must tolerate it. Every published game
modification does, by checking the model or the health. A rebuild that fixes this changes
behaviour game logic depends on.

## Definition lookup

### `ED_GlobalAtOfs`, `ED_FieldAtOfs`, `ED_FindField`, `ED_FindGlobal`, `ED_FindFunction`

**Contract** — linear searches over the definition and function tables: by slot offset, or
by name. Each returns the entry or nothing.

**Notes** — five linear scans over tables of a few thousand entries. They run at load
time, at save time, and — through the next function — during play. A rebuild should build
maps at load.

### `GetEdictFieldValue`

**Contract** — takes an entity and a field name; returns that field's slot, or nothing when
the game declared no such field. Caches the two most recently requested names, including
negative results.

```text
FUNCTION get_edict_field_value(ed, field) -> optional<slot>
  FOR EACH entry IN the two-entry cache
    IF entry.field == field  def = entry.def ;  GOTO done
  def = ed_find_field(field)                     # a linear scan
  IF length(field) < 64
    overwrite the cache entry at `rep` with (field, def)
    rep = rep BITXOR 1                           # alternate between the two
done:
  IF def IS nothing  RETURN nothing
  RETURN the slot at (ed's named-field region + def.ofs * 4)
```

**Invariants** — the cache stores negative results too, so a repeated miss is cheap. It
holds exactly two entries and alternates, which is a two-way round robin rather than
least-recently-used.

The cache is keyed by name only and is **not invalidated when a different game is
loaded** — except that [`PR_LoadProgs`](#pr_loadprogs) clears it explicitly, which is the
one thing making it safe.

**Notes** — two entries is enough because the engine only reaches optional fields from a
handful of call sites, each using one name. A rebuild resolving every field this way needs
a real map.

## Value rendering

### `PR_ValueString`, `PR_UglyValueString`

**Contract** — render one slot as text according to a type, into a shared static buffer.
The first is for human reading — entities as `entity 42`, functions as `name()`, fields as
`.name`, floats to one decimal, vectors quoted; the second is for the save file — bare
numbers and names, floats at full precision.

**Invariants** — the save form must be exactly what [`ED_ParseEpair`](#ed_parseepair) can
read back, and the two are the save format's two halves. Floats are written with six
fractional digits, which loses precision beyond that; the game's state tolerates it.

A shared static buffer means one live result. The callers format one value at a time.

**Notes** — the human form of a field value looks up the field's *name* from its offset by
linear scan, so printing an entity with many fields is quadratic. Only a diagnostic.

### `PR_GlobalString`, `PR_GlobalStringNoContents`

**Contract** — render a global slot for the disassembler as `offset(name)value`, padded to
twenty-one columns; or without the value, for a store's destination. An unknown offset
renders as `offset(???)`.

**Notes** — the padding is done by appending spaces to a 128-byte buffer with no bound
check, and the name plus a long vector value can exceed it. A diagnostic path only.

## Entity serialization

### `ED_Print`, `ED_PrintNum`, `ED_PrintEdicts`, `ED_PrintEdict_f`, `ED_Count`

**Contract** — print one entity's non-zero fields, or every entity, or a summary count.
The summary reports the array's high water mark, how many are in use, how many have a
model, how many are solid, and how many use stepped physics. A free entity prints as
`FREE`.

**Invariants** — fields whose name ends in an underscore followed by one character are
skipped, because the compiler emits `origin_x`, `origin_y` and `origin_z` alongside
`origin` so that the game can address a vector's components; printing all four would be
redundant. The test is on the second-to-last character being an underscore.

A field whose every slot is zero is skipped, which is what keeps the output and the save
file small.

**Notes** — the underscore test reads `name[length-2]`, which reads before the buffer for a
one-character field name. No such name exists. A rebuild should guard it.

### `ED_Write`

**Contract** — writes one entity as a brace-delimited block of quoted key/value pairs, one
per line, skipping zero-valued and component fields exactly as the print form does. A free
entity writes an empty block, which preserves entity *numbering* across a save.

**Invariants** — writing the empty block for a free slot is what makes entity numbers
stable across a save and restore, which matters because game logic stores handles in
fields.

### `ED_WriteGlobals`

**Contract** — writes the flagged globals as a brace-delimited block. Only globals carrying
the save flag, and only of string, float or entity type, are written.

**Invariants** — restricting to three types is a limitation, not a design: a flagged vector
global is silently dropped. The source's own comment at the top of the section admits the
mechanism does not really work, because the compiler flags constants as well as variables.

### `ED_ParseGlobals`

**Contract** — reads a brace-delimited block of key/value pairs into the global slots.
A name that is not a global is reported and skipped. A parse failure is a host error.
End of input without the closing brace is fatal.

### `ED_NewString`

**Contract** — takes a string; allocates a hunk copy, translating a backslash followed by
`n` into a newline and any other backslash pair into a single backslash. Returns the copy.

```text
FUNCTION ed_new_string(s) -> text
  out = allocate length(s)+1 bytes on the hunk
  FOR EACH position i IN s
    IF s[i] IS '\' AND a character follows
      i = i + 1
      emit newline IF s[i] IS 'n' ELSE emit '\'
    ELSE
      emit s[i]
  RETURN out
```

**Invariants** — this is the only escape processing anywhere in the engine's text formats,
and it exists so a map can put a multi-line message on a sign. The token parser
([`common.c`](common.c.md#com_parse)) does *not* process escapes, so a backslash survives
tokenization and is interpreted here.

Strings allocated this way live on the hunk for the level's life and are **never
reclaimed**. Game logic that builds strings at run time therefore leaks until the level
changes. Nothing in the shipped game does.

### `ED_ParseEpair`

**Contract** — takes a base address, a definition, and a text value; writes the parsed
value into the slot. Strings are copied into the string space and stored as an offset;
floats parsed directly; vectors as three space-separated numbers; entities as an index
converted to a handle; fields resolved by name to another field's offset; functions
resolved by name to an index. An unresolvable field or function name is reported and the
call fails. Any other type is silently ignored.

```text
FUNCTION ed_parse_epair(base, key, s) -> bool
  d = base + key.ofs * 4
  SELECT key.type WITH the save flag stripped
    string:   d = ed_new_string(s) AS an offset into the string table
    float:    d = parse_number(s)
    vector:   split s at spaces into exactly three numbers ;  d[0..2] = them
    entity:   d = handle_of(entity number parse_integer(s))
    field:    def = ed_find_field(s)
              IF def IS nothing  print "Can't find field <s>" ;  RETURN false
              d = the CURRENT CONTENTS of global slot def.ofs
    function: f = ed_find_function(s)
              IF f IS nothing  print "Can't find function <s>" ;  RETURN false
              d = index of f
  RETURN true
```

**Invariants** — a **field-valued** field is stored as the *contents of the global slot*
holding that field's offset, not as the offset itself. The compiler emits a global for
every field name whose value is the field's offset; this reads that global. It is
indirect and correct.

The vector parse writes a terminator into the value string at each space, so it mutates a
128-byte local copy. A vector with fewer than three components reads past the string's end
into whatever follows; a longer value string than 127 characters overflows the copy.
Both are reachable from a crafted map. A rebuild should parse properly.

The entity parse converts an index to a handle through the entity array, so it validates
the index ([`EDICT_NUM`](#edict_num-num_for_edict)) — the one place a save file's entity references are
range-checked.

### `ED_ParseEdict`

**Contract** — reads one brace-delimited block into an entity, clearing it first except
for entity zero. Returns the position after the block. A block with no pairs at all marks
the entity free. End of input without a closing brace is fatal, as is a closing brace
where a value was expected. Keys beginning with an underscore are discarded. Unknown keys
are reported and skipped.

Three input fix-ups apply:

```text
IF the key IS "angle"
  treat it as "angles" and wrap the value as "0 <value> 0"   # a scalar yaw
IF the key IS "light"
  treat it as "light_lev"                                    # name collision
strip trailing spaces from the key
```

**Invariants** — the scalar-angle fix-up exists because the map editor writes a single
number for an entity's facing, and the field is a vector; the value becomes a pure yaw.
Published maps rely on it.

The `light` rename exists because the map compiler's key collides with a game field name.
Also required by published maps.

The trailing-space strip exists because some map editors emitted keys with them.

Underscore-prefixed keys are editor comments and are dropped by design.

**Invariants** — not clearing entity zero is marked as a hack in the source, and it is
necessary: entity zero is the world, and the caller has already set fields on it before
parsing its block.

**Notes** — all three fix-ups are compatibility with tools, not engine design, and a
rebuild must keep all three to load published maps. The key buffer is 256 bytes with no
bound check.

### `ED_LoadFromFile`

**Contract** — takes a map's entity text; parses each block into an entity, filters it
against the current game mode and difficulty, then immediately calls the spawn function
named by its class. The first block becomes entity zero; each subsequent block allocates.
Reports how many entities were filtered out. A block whose first token is not an opening
brace is fatal. An entity with no class name, or with a class name naming no function, is
reported and freed.

```text
FUNCTION ed_load_from_file(data)
  ent = nothing ;  inhibit = 0
  the global `time` = sv.time
  LOOP
    parse a token
    IF at end of input  BREAK
    IF the token IS NOT '{'  FAIL WITH "found <token> when expecting {"

    ent = entity 0 ON THE FIRST ITERATION, ELSE ed_alloc()
    ed_parse_edict(into ent)

    # Filter by game mode and difficulty, using the entity's spawn flags
    IF deathmatch
      IF ent.spawnflags HAS the not-in-deathmatch bit
        ed_free(ent) ;  inhibit = inhibit + 1 ;  CONTINUE
    ELSE IF (skill == 0 AND the not-on-easy bit)
         OR (skill == 1 AND the not-on-medium bit)
         OR (skill >= 2 AND the not-on-hard bit)
      ed_free(ent) ;  inhibit = inhibit + 1 ;  CONTINUE

    IF ent.classname IS unset
      print "No classname for:" ;  print the entity ;  ed_free(ent) ;  CONTINUE
    func = ed_find_function(the class name)
    IF func IS nothing
      print "No spawn function for:" ;  print the entity ;  ed_free(ent)
      CONTINUE

    the global `self` = handle of ent
    run func to completion
  report "<inhibit> entities inhibited"
```

**Invariants** — entities are placed **directly** rather than through the allocator for the
first one, and the source explains: allocating would make entity numbers depend on how
far parsing got, so a partial load would renumber everything. The numbering must be
deterministic because saved games and the protocol both use it.

The difficulty filter reads the *snapshot* of the skill setting
([`quakedef.h`](quakedef.h.md)), not the live variable, so changing skill mid-level does
not change which entities exist.

The spawn function is found **by class name**: the map says `classname "monster_ogre"` and
the engine calls the game function of that name. That is the entire mechanism by which a
map's content is realized, and it is why adding a monster to the game requires only adding
a function with the right name. A rebuild must keep the name-based dispatch.

The spawn function runs *immediately*, inside the parse loop, so an entity's spawn code can
see every entity parsed before it and none after. Game logic that needs to wire entities
together therefore defers to a later frame.

## `EDICT_NUM`, `NUM_FOR_EDICT`

**Contract** — convert between an entity index and an entity, validating the range. An
index below zero or at or above the array's capacity is fatal; a pointer outside the live
range is fatal.

**Invariants** — the two use different bounds: the index form checks against the array's
*capacity*, the pointer form against the current *count*. So an entity beyond the high
water mark can be obtained by index and then fails to convert back. That asymmetry is
relied on by [`ED_Alloc`](#ed_alloc), which obtains the new slot by index before
incrementing the count.
