# WinQuake/progs.h

> The entity: an interpreter record with an engine-owned head, addressed by byte offset because its size is decided by the loaded game logic.

**Needs** — [`pr_comp.h`](pr_comp.h.md) · [`progdefs.h`](progdefs.h.md) · [`common.h`](common.h.md) (the intrusive list) · [`quakedef.h`](quakedef.h.md) (the network entity state)
**Used by** — [`pr_edict.c`](pr_edict.c.md) · [`pr_exec.c`](pr_exec.c.md) · [`pr_cmds.c`](pr_cmds.c.md) · [`world.c`](world.c.md) · [`sv_main.c`](sv_main.c.md) · [`sv_phys.c`](sv_phys.c.md) · [`sv_move.c`](sv_move.c.md) · [`sv_user.c`](sv_user.c.md) · [`host_cmd.c`](host_cmd.c.md)
**Tier floor** — T1 as written: an entity handle is a byte offset into one array, produced by pointer subtraction, and the interpreter's field opcodes add a field offset to it directly

## Purpose

The single most consequential structural decision in the engine is here: **an entity's
size is not known at compile time.** The loaded game logic declares how many slots an
entity has, so the engine allocates one flat array with a runtime stride and reaches an
entity by multiplying. Everything else in this file follows from that.

The second decision is that **an entity handle, as the bytecode sees it, is a byte
offset** into that array rather than an index. That makes the field-load opcode a single
addition — base plus entity offset plus field offset — with no multiply, which is why the
interpreter's hottest arm is three instructions. It also means an entity handle in
bytecode is only meaningful relative to one array, and it is why a save game cannot store
raw handles ([`pr_edict.c`](pr_edict.c.md)).

## State

```text
CONSTANT max_ent_leafs = 16

RECORD Edict                       # one entity
  # --- the engine's own head, invisible to the game logic ---
  free      : bool                 # this slot is available
  area      : Link                 # splices into the collision system's
                                   # spatial lists
  num_leafs : int
  leafnums  : int (16-bit)[16]     # which map leaves this entity touches,
                                   # for visibility
  baseline  : EntityState          # the state clients were told at spawn;
                                   # every update is a delta against THIS,
                                   # not against what a client last received
  freetime  : real                 # the server clock when this slot was freed

  # --- the interpreter's storage begins here ---
  v         : EntVars              # the fields the engine knows by name
  # ... and any further fields the game declared, reachable only by offset

VARIABLE pr_edict_size : int       # bytes per entity; from the loaded program
```

**Invariants** — the record is a *prefix*. The engine's head, then the named fields, then
an opaque tail of whatever else the game declared. The stride comes from the loaded
program and every access uses it.

The free flag plus the free time implement a delay: a freed slot is not reused for a
short interval, so that a stale handle held by game logic refers to a *free* entity
rather than to a different live one. See [`pr_edict.c`](pr_edict.c.md#ed_alloc).

The leaf list caps at 16. An entity spanning more leaves than that is marked as spanning
*too many* and is then treated as always visible, because partial visibility information
would make it flicker.

The baseline being the delta reference — rather than per-client history — is this
protocol's defining limitation and is what [`QW`](../QW/server/sv_ents.c.md) replaces.

## Types

```text
UNION Eval                # one interpreter slot, read as whatever the opcode says
  string   : string       # a byte offset into the string table
  _float   : real
  vector   : real[3]      # reading three slots as one
  function : func
  _int     : int
  edict    : int          # a byte offset into the entity array
```

**Notes** — the union is the absence of a type system made concrete. A rebuild in a
language with tagged values must not add a tag: the bytecode's correctness depends on the
same 32 bits being readable as any of these, and the compiler relies on it (a stored
entity handle and a stored integer are the same opcode). A byte array with typed accessors
is the honest translation.

## Addressing an entity

```text
FUNCTION edict_num(n)      -> Edict      # the nth entity
  RETURN sv.edicts + n * pr_edict_size

FUNCTION num_for_edict(e)  -> int        # its index
  RETURN (address_of(e) - sv.edicts) / pr_edict_size

FUNCTION next_edict(e)     -> Edict      # the following slot
  RETURN address_of(e) + pr_edict_size

FUNCTION edict_to_prog(e)  -> int        # the handle the bytecode uses
  RETURN address_of(e) - sv.edicts

FUNCTION prog_to_edict(h)  -> Edict      # and back
  RETURN sv.edicts + h
```

**Invariants** — the bytecode handle is a **byte offset**, so handle 0 is entity 0, which
is the world; and the null slot of the global pool
([`pr_comp.h`](pr_comp.h.md)) reads as handle 0, which is why "no entity" and "the world"
are the same value in the game logic.

The index conversion is a division by a runtime value, so it is not free; the engine
prefers the offset form on hot paths and converts only where a number is needed — for the
wire, for a diagnostic, for an array index.

**Notes** — the header carries both a macro form and a function form of the index
conversions, with the macros commented out. The functions exist so that they can validate
the range ([`pr_edict.c`](pr_edict.c.md#edict_num-num_for_edict)); the macros are what the assembly-era
version used. A rebuild should validate.

## Reaching a global slot

```text
# `o` is a slot index from an instruction operand.
FUNCTION g_float(o)    -> real     RETURN globals[o]
FUNCTION g_int(o)      -> int      RETURN globals[o] READ AS an integer
FUNCTION g_vector(o)   -> vec3     RETURN globals[o .. o+2]
FUNCTION g_string(o)   -> text     RETURN string_table AT globals[o]
FUNCTION g_function(o) -> func     RETURN globals[o] READ AS a function index
FUNCTION g_edict(o)    -> Edict    RETURN sv.edicts + (globals[o] READ AS int)
FUNCTION g_edictnum(o) -> int      RETURN num_for_edict(g_edict(o))
```

## Reaching a field of an entity

```text
# `o` is a field offset in SLOTS, from a definition table entry.
FUNCTION e_float(e, o)  -> real    RETURN (e.v AS a slot array)[o]
FUNCTION e_int(e, o)    -> int     ... READ AS an integer
FUNCTION e_vector(e, o) -> vec3    ... three slots
FUNCTION e_string(e, o) -> text    RETURN string_table AT that slot
```

**Invariants** — the field offset is relative to the start of the *named-fields* region,
not to the start of the entity record. So the engine's own head is invisible to the
bytecode and cannot be reached by any field offset — which is the only thing protecting
the collision list links and the baseline from game logic.

## The loaded program

```text
VARIABLE progs            : ProgramHeader
VARIABLE pr_functions     : list<Function>
VARIABLE pr_strings       : bytes            # the string table
VARIABLE pr_globaldefs    : list<Def>
VARIABLE pr_fielddefs     : list<Def>
VARIABLE pr_statements    : list<Statement>
VARIABLE pr_global_struct : GlobalVars       # the named view
VARIABLE pr_globals       : list<real>       # the same memory, as slots
VARIABLE pr_crc           : int (16-bit)     # the loaded program's field checksum
VARIABLE type_size        : int[8]           # slots per ValueType
```

**Invariants** — the named view and the slot array are the **same memory**, aliased. The
engine writes `pr_global_struct.time` and the bytecode reads it as a slot. A rebuild that
keeps two copies must synchronize them every frame, which is worse than accessing the
slot pool by resolved offset.

## The host-function table

```text
TYPE builtin = handler                    # takes nothing, reads and writes the
                                          # argument and return slots directly
VARIABLE pr_builtins    : list<builtin>
VARIABLE pr_numbuiltins : int
VARIABLE pr_argc        : int             # how many arguments this call passed
```

**Invariants** — a host function takes no parameters and returns nothing in the host
language. It reads its arguments out of the fixed argument slots and writes its result
into the fixed return slots. The argument count is available because the call opcode
encodes it. See [`pr_cmds.c`](pr_cmds.c.md).

## Interpreter diagnostics

```text
VARIABLE pr_trace      : bool        # print every statement as it executes
VARIABLE pr_xfunction  : Function    # the function now running
VARIABLE pr_xstatement : int         # the statement now running
```

**Contract** — `PR_RunError` prints the current statement, a call stack and a message,
then abandons the frame through the host's error path. It is the interpreter's only
failure mode and every error check in the interpreter and in every host function goes
through it.

## Entry points

**Contract** — `PR_Init` registers the interpreter's console commands.
`PR_LoadProgs` reads and validates the compiled game logic. `PR_ExecuteProgram` runs one
function to completion. `PR_Profile_f` reports the busiest functions.

## Entity lifecycle

**Contract** — `ED_Alloc` finds or makes a free slot; `ED_Free` releases one.
`ED_NewString` copies a string into the interpreter's string space, which the engine grows
but never reclaims.

## Text serialization

**Contract** — `ED_Print` and `ED_Write` render an entity as key/value text;
`ED_ParseEdict` reads that text back into one entity; `ED_WriteGlobals` and
`ED_ParseGlobals` do the same for the flagged globals; `ED_LoadFromFile` spawns every
entity described by a map's entity text.

**Invariants** — the save format is text keyed by *field name*, which is what makes a save
game survive a change to the field layout. That is a deliberate trade of size for
robustness, and the reason is worth keeping: the alternative is that adding a field to a
game modification invalidates every existing save.

## `GetEdictFieldValue`

**Contract** — takes an entity and a field name; returns that field's slot, or nothing if
the game logic declared no such field. This is how the engine reaches a field it was not
compiled to know about.

**Notes** — the only place the engine reads a field by name at run time, and it is used
for optional fields a game modification may or may not declare. A rebuild that resolves
*all* fields this way ([`progdefs.q1`](progdefs.q1.md#why-the-file-is-generated)) gets one
uniform path and no generated header.
