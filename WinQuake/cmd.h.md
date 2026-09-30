# WinQuake/cmd.h

> Declares the console command system: a text buffer that queues work from every source, and a dispatcher that resolves a line to a command, an alias or a variable.

**Needs** — nothing; this is a leaf
**Used by** — [`cmd.c`](cmd.c.md) · [`cvar.c`](cvar.c.md) · [`host.c`](host.c.md) · [`host_cmd.c`](host_cmd.c.md) · [`keys.c`](keys.c.md) · [`console.c`](console.c.md) · [`sv_user.c`](sv_user.c.md) · [`cl_main.c`](cl_main.c.md) · and every module that registers a command
**Tier floor** — none

## Purpose

Almost nothing in the engine is invoked directly. A key press, a line typed at the
console, a line in a configuration file, a command-line argument, and a string sent
by a remote server all turn into *text*, land in one buffer, and are executed from
it once per frame. This file declares that buffer and the dispatcher behind it.

The reason to route everything through text is uniformity in both directions: a key
binding is a string, so any command can be bound; a configuration file is a script,
so any state reachable by command can be saved and restored; and a server can ask a
client to do something by sending the same string a user would type. The cost is that
the dispatcher is the engine's largest attack surface, which is why every handler is
told where its line came from.

## State

```text
ENUM CommandSource
  from_client      # arrived over the network as a string command from a player;
                   # the sending player is the current one while it runs
  from_console     # from the command buffer: typed, bound, scripted, or
                   # command line

VARIABLE cmd_source : CommandSource   # which of the two produced the line
                                      # now running
```

**Invariants** — the source is set immediately before a handler runs and is valid
only during it. Handlers that change the world, kick players or write files check it;
the check is the engine's authorization model, and it is the whole of it. A rebuild
that passes the source as an argument rather than reading a global is doing the same
thing more safely.

## The command buffer

### `Cbuf_Init`

**Contract** — allocates the text buffer. It does not grow: the declaration in this
file promises growth, and the implementation does not deliver it — see
[`cmd.c`](cmd.c.md#cbuf_addtext).

### `Cbuf_AddText`

**Contract** — appends text to the end of the buffer. Work queued this way runs after
everything already queued.

### `Cbuf_InsertText`

**Contract** — inserts text at the *front*, ahead of anything not yet executed. This
is how a command that expands into other commands — running a script file, invoking
an alias — gets its expansion executed before the rest of the line's siblings, which
is what makes an alias behave like the text it stands for rather than like a
deferred job.

### `Cbuf_Execute`

**Contract** — repeatedly takes one line off the front of the buffer and executes it,
until the buffer is empty. Called once per frame, and explicitly at a few startup
points. **Must not be called from inside a command handler**: a handler runs while
the buffer is mid-consumption, and re-entering would execute the rest of the buffer
before the handler returns.

## The dispatcher

### `Cmd_Init`

**Contract** — registers the six commands this file implements.

### `Cmd_AddCommand`

**Contract** — takes a name and a handler; registers it. The name is retained by
reference, not copied, so it must outlive the process — a literal, not a temporary.

### `Cmd_Exists`

**Contract** — takes a name; reports whether a command by that name is registered.
Used by variable registration to refuse a colliding name.

### `Cmd_CompleteCommand`

**Contract** — takes a partial name; returns the first registered command name
beginning with it, or nothing.

### `Cmd_ExecuteString`

**Contract** — takes one line and a source; tokenizes it, then resolves the first
token against commands, then aliases, then variables, in that order, and reports
"unknown command" if none match. The resolution order is the priority order, and it
is why a variable may not share a name with a command.

### `Cmd_TokenizeString`

**Contract** — takes a line, not necessarily newline-terminated; splits it into
tokens that the argument accessors then expose. Stops at the first newline.

### `Cmd_Argc`, `Cmd_Argv`, `Cmd_Args`

**Contract** — the handler's view of its line. `Cmd_Argc` is the token count
including the command name; `Cmd_Argv` returns the token at an index, or **the empty
string** for any out-of-range index, so a handler may read arguments it may not have
without a bounds check; `Cmd_Args` is the raw remainder of the line after the command
name, unsplit, for handlers that want the text verbatim.

**Invariants** — that out-of-range reads yield an empty string rather than nothing is
depended on by dozens of handlers. A rebuild must preserve it or audit every handler.

### `Cmd_CheckParm`

**Contract** — takes a token; returns its 1-based position among the arguments, or 0
if absent. Case-insensitive. A null token is a fatal error.

### `Cmd_ForwardToServer`

**Contract** — sends the current line to the connected server as a string command, so
that a command the client cannot honour itself is honoured by the server. Registered
under the name `cmd` as well, so `cmd <anything>` forwards verbatim.

### `Cmd_Print`

**Contract** — declared here for handlers that must send their output to whoever
invoked them — the local console for a locally typed line, the originating player for
a forwarded one. **Not implemented in this build**; it exists in the QuakeWorld
server ([`QW/server/sv_ccmds.c`](../QW/server/sv_ccmds.c.md)), where one binary serves
both. A rebuild of this engine should omit it.
