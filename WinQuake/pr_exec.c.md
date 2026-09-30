# WinQuake/pr_exec.c

> The interpreter: one switch over sixty-two opcodes, a call convention that saves and restores a run of globals, and a runaway counter that turns an infinite loop in game logic into an error rather than a hang.

**Needs** — [`progs.h`](progs.h.md) · [`pr_comp.h`](pr_comp.h.md) · [`quakedef.h`](quakedef.h.md) · [`server.h`](server.h.md) (the entity array and the server's state) · [`console.h`](console.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`sv_phys.c`](sv_phys.c.md) (think and touch callbacks) · [`sv_main.c`](sv_main.c.md) (the per-frame entry points) · [`sv_user.c`](sv_user.c.md) (player callbacks) · [`host_cmd.c`](host_cmd.c.md) · [`pr_cmds.c`](pr_cmds.c.md) (host functions call back into it) · [`world.c`](world.c.md)
**Tier floor** — none beyond what [`progs.h`](progs.h.md) imposes; the interpreter itself is tier-free

## Purpose

Six hundred lines, and it is the whole of [Seam: Game logic
interpreter](../SYSTEM-REQUIREMENTS.md#seam-game-logic-interpreter). Everything the game
*is* — every monster, every weapon, every door — runs through the switch in this file.

Three things in it are worth a rebuilder's full attention, and the rest is mechanical.
The **call convention** saves a run of global slots rather than pushing a frame, because
the bytecode addresses its locals absolutely. The **runaway counter** bounds execution, so
a game modification with a bug degrades the session instead of killing the process. And
the **state opcode** couples an animation frame to a scheduled callback in one instruction,
which is the primitive the entire game's behaviour is built from.

## State

```text
RECORD StackFrame
  s : int                   # the statement to resume at
  f : Function              # the function to resume in

CONSTANT max_stack_depth = 32
CONSTANT localstack_size = 2048

VARIABLE pr_stack        : StackFrame[32]
VARIABLE pr_depth        : int          # frames in use
VARIABLE localstack      : int[2048]    # saved copies of callees' local slots
VARIABLE localstack_used : int
VARIABLE pr_argc         : int          # the current call's argument count
VARIABLE pr_trace        : bool
VARIABLE pr_xfunction    : Function     # for diagnostics
VARIABLE pr_xstatement   : int
VARIABLE pr_opnames      : text[66]     # for the disassembler
```

**Invariants** — 32 frames of recursion and 2048 slots of saved locals, both fixed, both
fatal on exhaustion. The two limits are independent: a deeply recursive function with few
locals exhausts the first, a shallow chain of large functions the second.

The saved-locals stack holds **copies of global slots**, not a frame layout. Its depth
tracks the call depth exactly, so it is not a general allocator.

## Why locals are saved rather than pushed

This is the file's central mechanism and it is easy to misread.

A bytecode function's parameters and locals live at *fixed absolute slot indices* in the
one global pool — the compiler assigned them at compile time
([`pr_comp.h`](pr_comp.h.md)). So two invocations of the same function want the same
slots, and recursion would clobber them. The interpreter's answer is to copy those slots
aside on entry and copy them back on exit:

```text
FUNCTION pr_enter_function(f) -> int         # returns the resume point
  pr_stack[pr_depth].s = pr_xstatement
  pr_stack[pr_depth].f = pr_xfunction
  pr_depth = pr_depth + 1
  IF pr_depth >= 32  FAIL WITH "stack overflow"

  # Save the slots this function is about to overwrite
  count = f.locals
  IF localstack_used + count > 2048
    FAIL WITH "locals stack overflow"
  FOR i FROM 0 TO count-1
    localstack[localstack_used + i] = globals[f.parm_start + i]
  localstack_used = localstack_used + count

  # Copy the arguments out of the fixed argument slots into this function's
  # own parameter slots. parm_size says 1 for a scalar, 3 for a vector.
  dest = f.parm_start
  FOR i FROM 0 TO f.numparms-1
    FOR j FROM 0 TO f.parm_size[i]-1
      globals[dest] = globals[ofs_parm0 + i*3 + j]
      dest = dest + 1

  pr_xfunction = f
  RETURN f.first_statement - 1        # the loop increments before fetching

FUNCTION pr_leave_function() -> int
  IF pr_depth <= 0  FAIL WITH "prog stack underflow"
  count = pr_xfunction.locals
  localstack_used = localstack_used - count
  IF localstack_used < 0  FAIL WITH "locals stack underflow"
  FOR i FROM 0 TO count-1
    globals[pr_xfunction.parm_start + i] = localstack[localstack_used + i]
  pr_depth = pr_depth - 1
  pr_xfunction = pr_stack[pr_depth].f
  RETURN pr_stack[pr_depth].s
```

**Invariants** — the save covers `locals` slots starting at `parm_start`, and `locals`
counts parameters *plus* locals, so the parameter slots are saved before the arguments
are copied in. That ordering is required: without it a recursive call's arguments would
overwrite the caller's parameters before they were saved.

The argument copy steps by the *declared* parameter size but reads from a fixed
three-slot stride in the argument area. So a scalar argument occupies one slot in the
callee and three in the argument area, and the two strides differ. A rebuild that uses one
stride for both breaks every function with a mix of scalar and vector parameters.

The entry point returns `first_statement - 1` because the main loop increments the
counter *before* fetching. Every branch in the switch has the same off-by-one for the same
reason, and a rebuild that increments after fetching must remove all of them together.

**Notes** — the underflow check on leaving is a fatal system error rather than an
interpreter error, because reaching it means the interpreter itself is broken rather than
the program.

## `PR_ExecuteProgram`

**Contract** — takes a function index; runs it and everything it calls to completion, then
returns. A zero or out-of-range index prints the current entity and raises a host error.
Nested invocation is supported and is the normal case: a host function may call back into
this, and the nesting is bounded by the same frame limit.

```text
FUNCTION pr_execute_program(fnum)
  IF fnum == 0 OR fnum >= function count
    IF the global `self` is set  print that entity
    FAIL WITH "PR_ExecuteProgram: NULL function"

  f = functions[fnum]
  runaway = 100000
  pr_trace = false
  exitdepth = pr_depth                  # where this invocation began
  s = pr_enter_function(f)

  LOOP
    s = s + 1
    st = statements[s]
    a = slot st.a ;  b = slot st.b ;  c = slot st.c     # resolved eagerly

    runaway = runaway - 1
    IF runaway == 0  FAIL WITH "runaway loop error"

    pr_xfunction.profile = pr_xfunction.profile + 1
    pr_xstatement = s
    IF pr_trace  print the disassembled statement

    EXECUTE st.op                       # see the opcode table below
    # RETURN and DONE break out when pr_depth returns to exitdepth
```

**Invariants** — the runaway budget is **100,000 statements per outermost invocation**,
not per frame and not per function. It is reset on entry, so a host function calling back
in gets a fresh budget. Exceeding it is an error that unwinds to the host's error handler,
drops the player to the console, and leaves the server running.

The three operand slots are resolved **before** the opcode is examined, so every operand
of every instruction is dereferenced whether the opcode uses it or not. With a 16-bit
signed operand and no range check, a corrupt program reads outside the global pool. A
rebuild should bound-check the operands at load time, once, rather than per instruction.

`pr_trace` is cleared on entry, which means the trace console command only takes effect
for the *rest* of a function already running — it cannot be armed from outside. That is a
bug; a rebuild should not reset it.

The profile counter is incremented per statement and attributed to the enclosing function.

## The opcodes

Each arm below states the computation. The three-address shape is "read the slots at `a`
and `b`, write the slot at `c`", except where noted. Every comparison and logical result
is a float, 0 or 1.

### Arithmetic

```text
ADD_F   c = a + b                        (floats)
SUB_F   c = a - b
MUL_F   c = a * b
DIV_F   c = a / b                        # NO zero check; produces an infinity
ADD_V   c[i] = a[i] + b[i]               (three components)
SUB_V   c[i] = a[i] - b[i]
MUL_V   c = a[0]*b[0] + a[1]*b[1] + a[2]*b[2]     # a DOT PRODUCT, giving a float
MUL_FV  c[i] = a * b[i]                  # scalar times vector
MUL_VF  c[i] = b * a[i]                  # vector times scalar
BITAND  c = (truncate a) BITAND (truncate b), converted back to a float
BITOR   c = (truncate a) BITOR  (truncate b), converted back to a float
```

**Invariants** — division has no guard. A divide by zero in game logic yields an infinity
that propagates into an entity's position and then into the physics, where it is caught by
the not-a-number test ([`mathlib.h`](mathlib.h.md#is_nan)) several frames later. A rebuild
may add the guard; doing so changes behaviour that published game logic has been tested
against, so it should report rather than silently substitute.

There is no vector division, no cross product, no negation opcode (the compiler emits a
subtraction from zero), and no integer arithmetic at all.

### Comparison and logic

```text
GE  LE  GT  LT     c = (a ? b) as a float           (floats)
AND                c = (a != 0) AND (b != 0)        # NOT short-circuiting
OR                 c = (a != 0) OR  (b != 0)        # NOT short-circuiting
EQ_F   / NE_F      float equality / inequality
EQ_V   / NE_V      all three components equal / any component differs
EQ_S   / NE_S      STRING CONTENT comparison, through the string table
EQ_E   / NE_E      entity handle identity
EQ_FNC / NE_FNC    function handle identity
NOT_F              c = (a == 0)
NOT_V              c = all three components are zero
NOT_S              c = the string handle is 0 OR the string is empty
NOT_FNC            c = the function handle is 0
NOT_ENT            c = the entity IS THE WORLD (handle 0)
```

**Invariants** — string comparison compares **content**, not handles, so two separately
allocated equal strings compare equal. That is the one place the interpreter looks inside a
value's representation, and it is why the string table must be one contiguous region.

`NOT_ENT` treating the world as falsehood is the language's idiom: game logic writes
`if (!self.enemy)` and means "has no enemy", and "no entity" is handle 0, which is the
world. A rebuild must make the same identification or the game's conditionals invert.

Neither logical operator short-circuits, because both operands are already-evaluated slots.
The compiler emits explicit branches where short-circuiting matters, so a rebuild must not
"fix" this.

### Assignment

```text
STORE_F / STORE_S / STORE_ENT / STORE_FLD / STORE_FNC
    b = a                               # the DESTINATION IS OPERAND b
STORE_V
    b[i] = a[i]

STOREP_F / STOREP_S / STOREP_ENT / STOREP_FLD / STOREP_FNC
    # b holds a field POINTER: a byte offset into the entity array
    the slot at (entity array base + b) = a
STOREP_V
    the three slots at (entity array base + b) = a[0..2]
```

**Invariants** — the plain stores write into operand **b**, breaking the
three-address pattern. The pointer stores add operand `b` to the base of the entity array
with **no range check whatsoever**: a bad pointer writes anywhere in the entity array, and
past its end. That is a real memory-safety hole reachable from game logic, and a rebuild
should bound the offset.

### Field access

```text
LOAD_F / LOAD_S / LOAD_ENT / LOAD_FLD / LOAD_FNC
    entity = the entity at handle a
    c = the slot at (entity's named-field region + b)
LOAD_V
    c[i] = the three slots there

ADDRESS
    entity = the entity at handle a
    IF entity IS the world AND the server is running
      FAIL WITH "assignment to world entity"
    c = (address of that entity's field b) - (entity array base)
```

**Invariants** — a field offset is added to the *named-field region*, so it can never
reach the engine's private head ([`progs.h`](progs.h.md)). The range of the offset itself
is unchecked, so a large offset reads into the next entity.

The world-entity guard is on `ADDRESS` only — the operation that *forms* a writable
pointer. Direct stores to the world's fields through the plain store opcodes are not
guarded, because the compiler always routes an assignment to a field through `ADDRESS`. The
guard is also suppressed before the level is running, because the spawn code legitimately
writes the world's fields.

### Control flow

```text
IFNOT    IF the slot at a IS zero (as an integer)  s = s + b - 1
IF       IF the slot at a IS NOT zero              s = s + b - 1
GOTO     s = s + a - 1
```

**Invariants** — the offsets are relative to the current statement and carry the
compensating minus one. The condition is tested on the slot's **integer** reading, not its
float reading, which matters: negative zero as a float is a non-zero integer and therefore
true. Game logic does not produce it, but a rebuild testing the float reading behaves
differently on it.

### Calls

```text
CALL0 .. CALL8
    pr_argc = the opcode's index, i.e. how many arguments were placed
    IF the function handle at a IS 0  FAIL WITH "NULL function"
    callee = functions[handle]
    IF callee.first_statement < 0
        # a HOST function: the negation is its index in the engine's table
        index = -callee.first_statement
        IF index >= builtin count  FAIL WITH "Bad builtin call number"
        CALL builtins[index]                  # runs immediately, no frame
        BREAK
    s = pr_enter_function(callee)
```

**Invariants** — the argument count lives in the opcode, so the callee never has to be
consulted to know it. A host function gets **no interpreter frame**: it runs inside the
current one, reads the argument slots directly, and may call back into the interpreter,
which is how the think and touch callbacks work.

The sign test on the first statement is the whole foreign-function interface.

### Return

```text
DONE / RETURN
    return slots 0..2 = the three slots at a
    s = pr_leave_function()
    IF pr_depth == exitdepth  RETURN from this invocation
```

**Invariants** — **three slots are always copied**, even for a scalar or void return, so
the two slots after the return value are clobbered on every return. Nothing reads them, but
a rebuild that copies only what the type needs is not bit-compatible with a program that
(incorrectly) relies on the clobber.

`DONE` and `RETURN` are identical here. The compiler emits `DONE` as a function's implicit
final statement, where operand `a` is slot 0 and the return value is therefore zero.

Returning to the depth at which this invocation began — rather than to depth zero — is what
makes nesting work.

### `STATE`

```text
STATE
    entity = the entity in the global `self` slot
    entity.nextthink = the global `time` + 0.1        # or 0.05 under a switch
    IF the slot at a != entity.frame
        entity.frame = the slot at a
    entity.think = the function at b
```

**Invariants** — the interval is **hard-coded in the engine**, not in the bytecode, and it
is 0.1 seconds. Every animation in the game therefore runs at ten frames per second, and
the game's frame sequences are authored against that. The build switch to 0.05 exists and
is not enabled; enabling it doubles every animation's speed.

The entity is taken from the global `self` slot rather than from an operand, making this
the only opcode with an implicit operand.

The conditional around the frame assignment is pointless — assigning an equal value is
harmless — and a rebuild should drop it.

**Notes** — this opcode is why the game's code reads as a chain of frame functions each
ending in a state transition. It is the language's coroutine, and a rebuild that omits it
cannot run any published game logic.

### Unknown opcodes

```text
default   FAIL WITH "Bad opcode <n>"
```

## `PR_RunError`

**Contract** — takes a formatted message; prints the current statement disassembled, the
call stack, and the message; clears the frame depth so that the host's error path can
unwind; then raises a host error, which drops to the console without stopping the server.

**Invariants** — clearing the depth before raising is required, because the host error
unwinds through a non-local jump and the frames would otherwise stay counted forever. A
rebuild using exceptions must do the equivalent in a handler.

## `PR_StackTrace`

**Contract** — prints the call stack, innermost first, as source file and function name for
each frame. A frame with no function prints a placeholder; an empty stack prints one.

**Notes** — it writes the current function into the array slot one past the live depth
before walking, so that the innermost frame appears. That write is into a valid slot only
because the depth limit is checked *after* incrementing, leaving one slot of headroom. A
rebuild should not rely on that coincidence.

## `PR_PrintStatement`

**Contract** — disassembles one statement: the opcode's name padded to ten columns, then
its operands rendered with their names and current values. The branch opcodes print a
relative target; the store opcodes print their destination without a value, because reading
the destination before the store is meaningless; everything else prints each non-zero
operand.

**Invariants** — the name table is shorter than the opcode range in one respect: the six
field-load opcodes all share the name `INDIRECT`, so a disassembly does not distinguish
them. Cosmetic.

## `PR_Profile_f`

**Contract** — the `profile` console command; repeatedly finds the function with the
highest statement count, prints the top ten, and **zeroes each counter as it goes**, so the
command both reports and resets.

**Notes** — the selection loop is quadratic in the function count and runs to completion
even after the tenth line, purely to clear every counter. Harmless at a few thousand
functions; a rebuild should sort.
