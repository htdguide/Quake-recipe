# WinQuake/cvar.c

> Keeps the registered console variables in one list, and makes assignment the single point where the text and numeric forms are brought back into agreement.

**Needs** — [`cvar.h`](cvar.h.md) · [`quakedef.h`](quakedef.h.md) · [`zone.h`](zone.h.md) · [`cmd.h`](cmd.h.md) (name-collision check, argument access) · [`console.h`](console.h.md) · [`server.h`](server.h.md) (broadcasting a server-flagged change)
**Used by** — every module that declares a tunable; [`cmd.c`](cmd.c.md) delegates unrecognized console lines here; [`pr_cmds.c`](pr_cmds.c.md) exposes read and write to game logic; [`host_cmd.c`](host_cmd.c.md) writes the archive section
**Tier floor** — none

## Purpose

Small, and almost all of its weight is in one rule: every assignment replaces the
string and re-derives the number, so the two can never disagree. Everything else is
a linear search over a singly linked list.

## State

```text
VARIABLE cvar_vars : optional<Cvar>   # head of the registration list,
                                      # most recently registered first
CONSTANT empty_string : text = ""     # returned for an unknown name
```

**Invariants** — a registered variable's string lives in zone memory and is owned by
this file; the record itself is owned by whoever declared it and is never freed. The
list is never unlinked from — registration is one-way and lasts the life of the
process.

## `Cvar_FindVar`

**Contract** — takes a name; returns the record whose name matches exactly, or
nothing. Case sensitive, linear.

**Notes** — the linear search runs on every console line, every game-logic variable
read, and every assignment. With a few hundred variables that is fine in 1996 and a
rebuild should still use a map, because the *command* dispatcher's equivalent search
([`cmd.c`](cmd.c.md)) is on the same path and both add up.

The case sensitivity here is inconsistent with the dispatcher, which matches
commands and aliases case-insensitively. So `r_draworder` typed as `R_DrawOrder` is
not found as a variable but *would* be found as a command. A rebuild should pick one
rule; matching the dispatcher's case-insensitive one accepts strictly more input.

## `Cvar_VariableValue`

**Contract** — takes a name; returns the number parsed from the found variable's
string, or zero when the name is unknown.

**Notes** — it re-parses the string rather than returning the cached number. The two
agree, so this is only a cost; but it means the parse rules of
[`Q_atof`](common.c.md#q_atof) apply here too, including the lack of exponent
notation.

## `Cvar_VariableString`

**Contract** — takes a name; returns the found variable's string, or the empty
string. Never returns nothing, so callers need no guard.

## `Cvar_CompleteVariable`

**Contract** — takes a partial name; returns the first registered variable's name
that begins with it, or nothing. An empty partial returns nothing.

**Notes** — "first in the list" means "most recently registered", which is an
arbitrary order from the user's point of view. Completion therefore proposes an
unpredictable candidate among several matches. A rebuild should sort or offer all
matches.

## `Cvar_Set`

**Contract** — takes a name and a new value; replaces the variable's string with a
fresh copy and re-derives the number. An unknown name prints a diagnostic and
returns. When the variable is server-flagged, the value actually differs, and a
server is running, announces the change to every connected player.

```text
FUNCTION cvar_set(name, value)
  var = cvar_find_var(name)
  IF var IS nothing
    print "Cvar_Set: variable <name> not found"     # a bug in the caller
    RETURN
  changed = (var.string != value)                   # compared BEFORE assigning
  release var.string
  var.string = a fresh copy OF value
  var.value  = parse_number(var.string)
  IF var.server AND changed AND a server is running
    broadcast "<name> changed to <value>"
```

**Invariants** — the change test compares the old and new *text*, not the derived
numbers, so setting `1` over `1.0` counts as a change and is announced. The old
string is released before the new one is taken, so the caller must not pass a
pointer into the variable's own string — assigning a variable to itself would read
freed memory. Nothing in the engine does; a rebuild that copies first avoids the
hazard entirely.

## `Cvar_SetValue`

**Contract** — takes a name and a number; formats it into text and assigns.

**Notes** — the formatting is the default decimal form with six fractional digits,
into a 32-byte buffer. That is observable: a value set numerically to 1 is stored as
the string `1.000000`, and that is what a configuration file written afterwards
contains. It also means a value set numerically and the "same" value typed at the
console have different strings and therefore compare as a change. A rebuild should
format minimally and gain both compactness and a stable comparison.

## `Cvar_RegisterVariable`

**Contract** — takes a record already holding a name and a default string; refuses
if the name is already a variable or already a command, printing in either case.
Otherwise copies the default string into managed memory, derives the number, and
pushes the record onto the front of the list.

```text
FUNCTION cvar_register_variable(var)
  IF cvar_find_var(var.name) EXISTS
    print "Can't register variable <name>, already defined" ;  RETURN
  IF a command named var.name EXISTS
    print "<name> is a command" ;  RETURN
  # The default string is a literal; copy it so that later assignment,
  # which releases the old string, has something it may release.
  var.string = a fresh copy OF var.string
  var.value  = parse_number(var.string)
  push var onto the front OF cvar_vars
```

**Invariants** — the copy is not hygiene. Assignment releases the previous string,
and releasing a compiled-in literal would be fatal; copying at registration is what
makes the assignment path uniform. A rebuild whose strings are managed values gets
this for free.

The refusals print rather than abort, so a duplicate registration leaves the *first*
one in force and the second record silently unregistered — reading it gives zero.
That is a quiet failure mode a rebuild should make loud.

## `Cvar_Command`

**Contract** — called with the console line already tokenized. Looks the first token
up as a variable name. Not a variable: reports "not handled". A variable with no
further tokens: prints `"name" is "value"` and reports handled. A variable with a
second token: assigns it and reports handled. Tokens past the second are ignored.

```text
FUNCTION cvar_command() -> bool
  var = cvar_find_var(argument 0)
  IF var IS nothing  RETURN false
  IF argument count == 1
    print "\"<name>\" is \"<value>\""
    RETURN true
  cvar_set(var.name, argument 1)
  RETURN true
```

**Notes** — ignoring extra tokens means `sensitivity 3 5` silently assigns 3. And
because the value is one token, assigning a value with a space requires quoting,
which the tokenizer ([`common.c`](common.c.md#com_parse)) supports.

## `Cvar_WriteVariables`

**Contract** — takes an output stream; writes `name "value"` for every
archive-flagged variable, one per line, in list order.

**Notes** — the value is quoted and the name is not, which is exactly the syntax the
console dispatcher accepts, so the file replays through the normal path. No escaping:
a value containing a quote produces a file that parses wrongly. Nothing can set such
a value from the console, because the tokenizer strips quotes, but game logic can.
List order means "most recently registered first", so the file's line order changes
whenever registration order does.
