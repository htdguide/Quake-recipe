# WinQuake/pr_comp.h

> The bytecode format: sixty-two three-address opcodes over a flat pool of 32-bit slots, and the file layout the compiler emits.

**Needs** — nothing; this is a leaf
**Used by** — [`progs.h`](progs.h.md) · [`pr_exec.c`](pr_exec.c.md) (the interpreter) · [`pr_edict.c`](pr_edict.c.md) (the loader) · and by the game-logic compiler, which is not in this tree
**Tier floor** — none; a byte layout and an instruction set

## Purpose

This file is the interface between two programs: the engine and the compiler that
produces the game logic. It is shared source, and everything in it is a contract.

The instruction set's character comes from one decision: **every value is a 32-bit
slot in one flat pool, and the type is carried by the instruction, not by the slot.**
There is no boxing, no tag, and no runtime type. A slot holds a float, or an integer,
or an offset into the string table, or an entity's byte offset, or a field offset, or a
function index, and the only thing that knows which is the opcode that touches it. A
vector is three consecutive slots and nothing marks the grouping.

The second decision is that **the instruction set is fully typed at the opcode level**:
there is no generic add, there is add-float and add-vector; no generic compare, but
five equality opcodes, one per type. That is what lets the interpreter be a switch with
no type dispatch inside any arm.

## State

A file format and an instruction encoding. No runtime state.

## Types

```text
TYPE func   = int      # an index into the function table
TYPE string = int      # a byte offset into the string table

ENUM ValueType
  void = 0, string, float, vector, entity, field, function, pointer
```

**Invariants** — the type enumeration is used by the loader to size and print values and
is stored in the definition tables. Its *order* is load-bearing: the engine indexes a
size table by it ([`pr_edict.c`](pr_edict.c.md)), where every type is one slot except
vector, which is three.

## The global slot pool and the call convention

```text
CONSTANT ofs_null      = 0     # slot 0 is permanently zero
CONSTANT ofs_return    = 1     # the return value: 3 slots, 1..3
CONSTANT ofs_parm0     = 4     # argument 0: 3 slots
CONSTANT ofs_parm1     = 7
CONSTANT ofs_parm2     = 10
CONSTANT ofs_parm3     = 13
CONSTANT ofs_parm4     = 16
CONSTANT ofs_parm5     = 19
CONSTANT ofs_parm6     = 22
CONSTANT ofs_parm7     = 25
CONSTANT reserved_ofs  = 28    # compiled globals begin here
CONSTANT max_parms      = 8
```

**Invariants** — the first 28 slots are the call frame, at fixed addresses, and they are
*global*. Eight argument positions, each three slots wide so that a vector argument fits
without a separate convention. A caller writes into these slots and a callee copies out
of them.

This is the whole calling convention and its consequences are everywhere:

- Arguments are passed through fixed global slots, so **a function call inside an
  argument expression would clobber the outer call's arguments**. The compiler must
  therefore evaluate nested calls into temporaries first, and it does.
- At most eight arguments, and the opcode itself says how many — there are nine distinct
  call opcodes.
- The return value is three slots even for a scalar return, and every return writes all
  three ([`pr_exec.c`](pr_exec.c.md)).
- Slot zero is the null value, read as a zero float, an empty string offset, and the
  world entity, all at once. That triple meaning is depended on by the game logic, which
  compares an entity against `world` and a string against the empty string using the
  same literal.

## The instruction set

```text
RECORD Statement                # one instruction, 8 bytes
  op      : int (16-bit unsigned)
  a, b, c : int (16-bit signed)
```

**Invariants** — three operands, each a **16-bit signed** slot index, so the global pool
is limited to 32767 slots. The operands are almost always "read a, read b, write c", with
the branch and call opcodes as exceptions.

The opcodes in numeric order, which is also the order the interpreter's switch takes them
and the order a rebuild must preserve:

| # | Opcode | Meaning |
|---|---|---|
| 0 | `DONE` | return; identical to `RETURN` in this interpreter |
| 1–4 | `MUL_F`, `MUL_V`, `MUL_FV`, `MUL_VF` | float product; **vector dot product**; scalar times vector; vector times scalar |
| 5 | `DIV_F` | float quotient |
| 6–7 | `ADD_F`, `ADD_V` | sums |
| 8–9 | `SUB_F`, `SUB_V` | differences |
| 10–14 | `EQ_F`, `EQ_V`, `EQ_S`, `EQ_E`, `EQ_FNC` | equality, per type; result is a float 0 or 1 |
| 15–19 | `NE_*` | the five inequalities |
| 20–23 | `LE`, `GE`, `LT`, `GT` | float ordering |
| 24–29 | `LOAD_F`, `LOAD_V`, `LOAD_S`, `LOAD_ENT`, `LOAD_FLD`, `LOAD_FNC` | read a field: operand a is an entity, b is a field offset |
| 30 | `ADDRESS` | form a pointer to a field of an entity |
| 31–36 | `STORE_*` | copy slot a into slot b — note: **into b, not c** |
| 37–42 | `STOREP_*` | store through a pointer: write a to the location b points at |
| 43 | `RETURN` | copy three slots from a into the return slots and leave |
| 44–48 | `NOT_F`, `NOT_V`, `NOT_S`, `NOT_ENT`, `NOT_FNC` | logical negation, per type |
| 49–50 | `IF`, `IFNOT` | branch by a **relative statement offset** in operand b |
| 51–59 | `CALL0` … `CALL8` | call, with the argument count in the opcode |
| 60 | `STATE` | set this entity's animation frame and think function, and schedule the next think |
| 61 | `GOTO` | unconditional relative branch, offset in operand a |
| 62–63 | `AND`, `OR` | logical, on floats |
| 64–65 | `BITAND`, `BITOR` | bitwise, on floats converted to integers and back |

**Invariants** — several details here are easy to get wrong and are observable:

- **`MUL_V` is a dot product**, producing a float from two vectors. There is no
  component-wise vector multiply.
- The store opcodes write into operand **b**, not the usual **c**. The interpreter's own
  printing code special-cases them for this reason.
- Branches are relative to the *current* statement, and the interpreter increments the
  counter after each instruction, so the stored offset is one more than the displacement
  — see [`pr_exec.c`](pr_exec.c.md#pr_executeprogram).
- The bitwise operations convert their float operands to integers, operate, and convert
  back, so they are exact only within the 24-bit mantissa. The game logic's item sets use
  bits up to 31 ([`quakedef.h`](quakedef.h.md)), which is why the highest item bits
  behave oddly in the original game.
- `AND` and `OR` are **not** short-circuiting: both operands are already evaluated, since
  they are slot reads. The compiler emits branches where short-circuiting is needed.
- There are no integer arithmetic opcodes at all. Every number in the language is a
  float.

## `STATE`

**Contract** — takes a frame number and a function; sets the *current* entity's animation
frame, sets its think function, and schedules its next think one animation interval into
the future.

**Invariants** — the interval is a **compile-time constant in the engine**, 0.1 seconds,
with a build switch for 0.05. It is not in the bytecode. So every animation in the game
runs at ten frames per second because of a constant in the interpreter, and the game
logic's frame sequences are written assuming it.

The entity acted upon is the one in the global `self` slot, not an operand. So this one
opcode reaches outside its operands into the global state, which is why it exists as an
opcode rather than as three stores: it is the language's animation primitive and the game
logic's entire state machine is built from it.

## The definition tables

```text
RECORD Def                      # one named global or field
  type : int (16-bit unsigned)  # a ValueType, with the high bit as a flag
  ofs  : int (16-bit unsigned)  # slot index, or field offset
  s_name : string               # into the string table

CONSTANT def_saveglobal = 1 SHIFTED LEFT 15   # include this global in save games
```

**Invariants** — the high bit of the type field is a flag, so the type is the low fifteen
bits. The flag selects which globals a save game records
([`pr_edict.c`](pr_edict.c.md#ed_writeglobals)), which is how the game's cross-level
state — the episode sigils, the skill — survives a save without saving the whole pool.

The same record shape describes globals and fields; for a field the offset is a slot
index *within an entity*, which is why a field offset is added directly to an entity's
base in the load and store opcodes.

## The function table

```text
RECORD Function
  first_statement : int     # NEGATIVE means a host function; see below
  parm_start      : int     # the first slot this function's parameters and
                            # locals occupy
  locals          : int     # total slots of parameters plus locals
  profile         : int     # a runtime counter, written into the loaded image
  s_name, s_file  : string
  numparms        : int
  parm_size       : byte[8] # slots per parameter: 1 for scalars, 3 for vectors
```

**Invariants** — a **negative first statement is a host function**, and its absolute
value is the index into the engine's table of built-ins
([`pr_cmds.c`](pr_cmds.c.md)). That single sign test is the entire foreign-function
interface: the compiler declares a host function by writing `= #17`, and the engine
supplies slot 17.

`parm_start` and `locals` describe a contiguous run of *global* slots that this function
uses as its frame. There is no stack in the bytecode: recursion works because the
interpreter saves and restores that run of globals around every call
([`pr_exec.c`](pr_exec.c.md)). A rebuild must do the same or must
rewrite the compiler's output, because the bytecode addresses its locals absolutely.

The profile counter lives in the loaded image, so profiling mutates the program. Benign;
a rebuild should keep it beside the image.

## The file

```text
CONSTANT prog_version = 6

RECORD ProgramHeader
  version        : int         # 6
  crc            : int         # a checksum of the field declarations the
                               # compiler saw; the engine compares it to its own
  ofs_statements, numstatements : int   # statement 0 is an error trap
  ofs_globaldefs, numglobaldefs : int
  ofs_fielddefs,  numfielddefs  : int
  ofs_functions,  numfunctions  : int   # function 0 is empty
  ofs_strings,    numstrings    : int   # the first string is empty
  ofs_globals,    numglobals    : int
  entityfields   : int         # slots per entity, as the compiler laid them out
```

**Invariants** — statement 0, function 0 and string 0 are all reserved sentinels, so
index zero means "none" for all three and needs no separate representation.

The checksum is compared against a value compiled into the engine
([`progdefs.q1`](progdefs.q1.md)) and a mismatch refuses the load. It is not a
tamper check — it is a *layout* check: the engine reads the first several dozen fields of
an entity as a native record, so the compiler and the engine must agree on their order
and offsets exactly. See [`progdefs.q1`](progdefs.q1.md).

`entityfields` is the compiler's count of slots per entity, and the engine allocates its
entity array from it. So the *size of an entity is determined by the loaded game logic*,
not by the engine — which is why the engine can never use a fixed-size entity record and
why every entity access in [`progs.h`](progs.h.md) goes through a runtime stride.
