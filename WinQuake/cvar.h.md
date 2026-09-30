# WinQuake/cvar.h

> Declares the console variable: a named value reachable from C, from the console, from a configuration file and from game logic, all through one record.

**Needs** — nothing; this is a leaf
**Used by** — [`cvar.c`](cvar.c.md) · [`cmd.c`](cmd.c.md) · [`host.c`](host.c.md) · [`pr_cmds.c`](pr_cmds.c.md) · and roughly every module, each of which declares its own tunables
**Tier floor** — none

## Purpose

The engine has a few hundred tunable values. Rather than a settings object, each is
a free-standing record declared beside the code that reads it, linked into one
global list at startup. The effect is that the code reads a plain field —
no lookup, no string — while the console, the configuration file and the
interpreted game logic all reach the same value by name.

That one-record-per-tunable, declared-where-used shape is the load-bearing
decision. A rebuild is free to use a map from name to value, but it must keep three
properties: reading a value in hot code costs nothing, the set of names is
discovered at startup rather than declared centrally, and a name can be looked up
and set from a string at run time.

## State

```text
RECORD Cvar
  name    : text            # the console name; also the key
  string  : text            # the authoritative value
  archive : bool            # written to the configuration file on quit
  server  : bool            # a change is announced to every connected player
  value   : real            # the numeric reading of `string`, kept in step
  next    : optional<Cvar>  # the registration list

VARIABLE cvar_vars : optional<Cvar>    # head of the list
```

**Invariants** — the string is authoritative and the number is derived; every write
goes through the setter, which updates both. A variable that has not been registered
has a number of zero regardless of what its string says, because the derivation
happens at registration. Names never collide with command names — the registration
of either refuses a name the other holds — because the console resolves a bare word
by trying commands first and then variables, and an ambiguity would make that
resolution arbitrary.

## Declaring one

```text
# Minimum: a name and a default, as text.
CONSTANT r_draworder = Cvar { name: "r_draworder", string: "1" }

# Persisted across runs:
CONSTANT screensize  = Cvar { name: "screensize",  string: "1", archive: true }
```

The default is given as **text**, not as a number, so that the parse path is the
same on first use as on every later assignment. A rebuild should keep that: it is
what makes `"0.5"` from a configuration file and `0.5` from a default agree exactly.

## `Cvar_RegisterVariable`

**Contract** — takes an initialized record; copies its default string into managed
memory, derives the number, and links it into the list. Refuses and prints if the
name is already a variable or already a command. Must be called before any console
command executes — an unregistered variable reads as zero.

## `Cvar_Set`, `Cvar_SetValue`

**Contract** — take a name and a new value, as text or as a number, and assign it,
re-deriving the other representation. Setting a name that does not exist prints a
diagnostic and does nothing — the source's comment is explicit that reaching this
means a bug in the calling code, not bad user input.

## `Cvar_VariableValue`, `Cvar_VariableString`

**Contract** — look a name up and return the number or the string. An unknown name
yields zero and the empty string respectively; neither signals an error. Used by
game logic and by code that does not hold the record.

## `Cvar_FindVar`

**Contract** — takes a name; returns the record or nothing. Exact match, case
sensitive.

## `Cvar_CompleteVariable`

**Contract** — takes a partial name; returns the name of the first registered
variable that starts with it, or nothing. Used by console tab completion.

## `Cvar_Command`

**Contract** — called by the command dispatcher when the first token matches no
command and no alias. If the token names a variable, prints its value when there are
no further tokens and assigns the second token otherwise, and reports that it
handled the line. Otherwise reports that it did not, and the caller prints "unknown
command".

**Notes** — this is why the console has no `set` keyword: typing a variable's name
prints it, and typing a name and a value assigns it. The configuration file is
written in exactly that syntax, which is why it can be replayed through the same
dispatcher.

## `Cvar_WriteVariables`

**Contract** — takes an open output stream; writes one `name "value"` line for every
variable whose archive flag is set. This is the configuration file's variable
section.

## The `server` flag

**Contract** — when a flagged variable changes to a different value while a server
is running, every connected player is told. Used for rules a player is entitled to
know about — friendly fire, time limit, cheats.

**Notes** — the announcement fires on a *change*, compared as text before the
assignment, so re-assigning the same value is silent.
