# WinQuake/cmd.c

> The command buffer and the dispatcher: everything the engine does on request arrives here as text, is queued, and is resolved against commands, aliases and variables in that order.

**Needs** — [`cmd.h`](cmd.h.md) · [`quakedef.h`](quakedef.h.md) · [`common.h`](common.h.md) (the byte buffer and the tokenizer) · [`zone.h`](zone.h.md) · [`cvar.h`](cvar.h.md) · [`console.h`](console.h.md) · [`client.h`](client.h.md) (forwarding to the server) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`host.c`](host.c.md) drives it once per frame; [`keys.c`](keys.c.md), [`console.c`](console.c.md), [`cl_main.c`](cl_main.c.md), [`sv_user.c`](sv_user.c.md) and [`host_cmd.c`](host_cmd.c.md) feed it; every module registers commands with it
**Tier floor** — none

## Purpose

One text buffer, consumed once per frame, is the engine's only work queue. This file
implements it, implements the aliasing and script-file expansion that make it a small
scripting language, and implements the dispatcher that turns a line into a call.

Two properties are worth naming up front because they shape everything downstream.
First, execution is **synchronous and re-entrant by insertion**: a command that wants
to run other commands does not call them, it inserts their text at the front of the
buffer and returns. Second, every handler can ask **where its line came from**, and
that question is the engine's entire authorization model.

## State

```text
VARIABLE cmd_text : SizeBuf           # the queue; 8192 bytes, does not grow
VARIABLE cmd_wait : bool              # stop consuming after the current line

RECORD CommandFunction
  name     : text                     # retained by reference, never copied
  function : handler
  next     : optional<CommandFunction>
VARIABLE cmd_functions : optional<CommandFunction>   # newest first

RECORD Alias
  name  : text[32]
  value : text                        # the expansion, always newline-terminated
  next  : optional<Alias>
VARIABLE cmd_alias : optional<Alias>

VARIABLE cmd_argc   : int             # tokens in the line now running
VARIABLE cmd_argv   : list<text>[80]  # each token, separately allocated
VARIABLE cmd_args   : optional<text>  # the raw text from the second token on
VARIABLE cmd_source : CommandSource
```

**Invariants** — the argument tokens are individually allocated from the zone and
freed by the *next* tokenize, so they are valid exactly for the duration of one
handler. The raw remainder points into the caller's line buffer, not into a copy, so
it is valid only while that buffer is. Both are why a handler may not stash its
arguments.

## The buffer

### `Cbuf_Init`

**Contract** — allocates 8192 bytes of hunk storage for the queue.

### `Cbuf_AddText`

**Contract** — appends text to the end. If it would not fit, prints an overflow
diagnostic and **drops the text entirely** rather than truncating.

```text
FUNCTION cbuf_add_text(text)
  IF cmd_text.cursize + length(text) >= cmd_text.maxsize
    print "Cbuf_AddText: overflow" ;  RETURN      # the text is lost
  append text to cmd_text                          # without a terminator
```

**Notes** — the buffer does not grow, contrary to the promise in
[`cmd.h`](cmd.h.md#cbuf_init). 8192 bytes is enough for a configuration file and a
frame of key bindings, and dropping on overflow is safer than truncating a line
mid-word. A rebuild should grow the buffer; nothing depends on the limit.

No newline is appended. The caller supplies one, and every caller does.

### `Cbuf_InsertText`

**Contract** — inserts text ahead of everything not yet executed, by copying the
remaining queue aside, appending the new text, and appending the remainder back.

```text
FUNCTION cbuf_insert_text(text)
  saved = a copy OF the entire remaining queue
  clear the queue
  cbuf_add_text(text)
  append saved back
```

**Notes** — the copy is why the source's own comment asks for a better structure. A
rebuild with a deque inserts at the front in constant time. What is load-bearing is
only the *ordering*: inserted text runs before anything already queued.

### `Cbuf_Execute`

**Contract** — repeatedly extracts one line and executes it. A line ends at a newline
or at a semicolon that is not inside double quotes; the terminator is consumed and not
part of the line. Each line is removed from the queue *before* it is executed, so a
handler that inserts text sees a consistent queue. Stops early, leaving the remainder
for the next frame, when a handler sets the wait flag.

```text
FUNCTION cbuf_execute()
  WHILE the queue is not empty
    # find the end of the first line
    quotes = 0
    FOR i FROM 0 TO queue length - 1
      IF byte i IS '"'                       quotes = quotes + 1
      IF quotes IS EVEN AND byte i IS ';'    BREAK
      IF byte i IS newline                   BREAK
    line = the first i bytes, zero-terminated

    IF i == queue length                     # no terminator found
      empty the queue
    ELSE
      drop the first i+1 bytes                # the line and its terminator
      # (the remaining bytes move down; see the note below)

    cmd_execute_string(line, from_console)

    IF cmd_wait
      cmd_wait = false
      BREAK                                   # resume next frame
```

**Invariants** — the semicolon is a statement separator only outside quotes, which is
what lets `alias x "say hi; say bye"` hold two statements as one value. The quote
count is a running parity, not a nesting depth, so an unbalanced quote makes every
following semicolon in the buffer literal until another quote appears.

The line is copied into a fixed 1024-byte buffer with **no bounds check**: a longer
line overruns it. Reachable from a configuration file, so a rebuild must add the
check.

Removing the line before executing is not an optimization — it is required, because a
handler may insert text at the front and the insertion must not collide with the line
still being read.

**Notes** — the wait flag exists for one reason, given in the source: a key binding
that needs a press and a release in separate frames, as in
`bind g "impulse 5 ; +attack ; wait ; -attack"`. Without it, both halves of a
weapon-fire sequence would land in the same server frame and the shot would not
register.

## The script commands

### `Cmd_Wait_f`

**Contract** — the `wait` command; sets the flag that stops consumption after the
current line.

### `Cmd_Exec_f`

**Contract** — the `exec` command; loads a file through the virtual filesystem and
inserts its whole text at the front of the queue. Prints usage for a wrong argument
count, and a diagnostic if the file is missing.

```text
FUNCTION cmd_exec_f()
  IF argument count != 2  print usage ;  RETURN
  mark = hunk low mark
  text = load argument 1 as a hunk file       # zero-terminated by the loader
  IF text IS nothing  print "couldn't exec <name>" ;  RETURN
  print "execing <name>"
  cbuf_insert_text(text)                       # copies the text into the queue
  restore the hunk to mark
```

**Invariants** — the file's text is copied into the queue by the insertion, so
releasing the hunk immediately afterwards is safe. Note the leak on the failure path:
the mark is taken but not restored. Harmless, because the load allocated nothing.

Inserting at the front means a script file's commands run before the rest of the line
that invoked it — so `exec a.cfg; echo done` prints `done` after `a.cfg` has fully
run. That is the behaviour scripts are written against.

### `Cmd_Echo_f`

**Contract** — the `echo` command; prints its arguments separated by spaces, then a
newline.

### `Cmd_Alias_f`

**Contract** — the `alias` command. With no arguments, lists every alias. Otherwise
takes a name of fewer than 32 characters and binds it to the remaining arguments
rejoined with spaces and terminated by a newline. Rebinding an existing name replaces
its value.

```text
FUNCTION cmd_alias_f()
  IF argument count == 1
    print every alias as "<name> : <value>" ;  RETURN
  name = argument 1
  IF length(name) >= 32  print "Alias name is too long" ;  RETURN
  existing = the alias named `name`
  IF existing EXISTS  release existing.value
  ELSE                existing = a new alias pushed onto the front of the list
  existing.name = name
  existing.value = arguments 2.. joined by single spaces, then a newline
```

**Invariants** — the value is reassembled from *tokens*, so the original spacing and
quoting are lost: `alias x "say  hello"` stores `say hello` with one space. And
because the tokenizer strips quotes, an alias value containing a semicolon must have
been quoted when defined, and is then stored unquoted — which is exactly what makes
it expand into two statements when invoked. That round trip is the mechanism, not an
accident.

The reassembly uses a fixed 1024-byte buffer with no bounds check.

**Notes** — the source's loop over the arguments has an off-by-one in its separator
test that appends a trailing space before the newline. Harmless; the tokenizer
discards it.

### `Cmd_StuffCmds_f`

**Contract** — the `stuffcmds` command; scans the process arguments for runs
introduced by a plus sign and inserts each as a line at the front of the queue. A run
ends at the next plus, the next hyphen, or the end. Prints usage if given arguments.

```text
FUNCTION cmd_stuff_cmds_f()
  IF argument count != 1  print usage ;  RETURN
  text = every process argument from 1 onward, joined by single spaces
  build = empty
  FOR EACH position i IN text
    IF text[i] IS '+'
      j = the first position after i holding '+', '-' or the terminator
      append text[i+1 .. j-1] AND a newline TO build
      i = j - 1                                  # rescan from the delimiter
  IF build is not empty  cbuf_insert_text(build)
```

**Invariants** — a hyphen ends a run, which is what lets `quake -nosound +map e1m1`
work: the flags are consumed by the code that reads them directly, and only the
plus-introduced runs become commands. The runs are inserted as one block, so they
execute in command-line order.

**Notes** — this is how the command line reaches the console, and it is invoked from
the startup script rather than from code, so the point at which command-line commands
run is itself configurable.

## The dispatcher

### `Cmd_Init`

**Contract** — registers `stuffcmds`, `exec`, `echo`, `alias`, `wait`, and `cmd` —
the last bound to the forward-to-server handler.

### `Cmd_TokenizeString`

**Contract** — takes a line; releases the previous line's tokens, then splits this one
using the shared token parser. Stops at the first newline or at the terminator. At
most 80 tokens are kept; further tokens are parsed and discarded. Records the position
of the raw text beginning at the second token.

```text
FUNCTION cmd_tokenize_string(text)
  release every token from the previous line
  cmd_argc = 0 ;  cmd_args = nothing
  LOOP
    skip spaces and control characters, but NOT a newline
    IF the current character IS a newline   consume it ;  BREAK
    IF at the terminator                    RETURN
    IF cmd_argc == 1
      cmd_args = the current position       # raw text from the second token on
    text = com_parse(text)                  # token left in the shared buffer
    IF text IS nothing  RETURN
    IF cmd_argc < 80
      cmd_argv[cmd_argc] = a fresh copy OF the parsed token
      cmd_argc = cmd_argc + 1
```

**Invariants** — the raw remainder is captured *before* the second token is parsed, so
it includes that token and everything after it with original spacing and quoting
intact. Handlers that must not lose quoting — the say commands, notably — use it
instead of the split tokens.

Tokens past the eightieth are parsed and thrown away rather than ending the line, so
a very long line silently loses its tail rather than erroring.

### `Cmd_AddCommand`

**Contract** — takes a name and a handler. Fatal error if called after host
initialization has finished, because the record is hunk-allocated and the host has
already taken the mark that later resets would restore past. Refuses, with a
diagnostic, a name already held by a variable or by another command. Otherwise
allocates the record and pushes it onto the front of the list, **retaining the name by
reference**.

**Invariants** — the name is not copied, so it must outlive the process. Every caller
passes a literal.

The registration-time-only restriction is a consequence of using the hunk for
registrations; a rebuild with ordinary allocation can register at any time and should
say so, because a plug-in architecture would need it.

### `Cmd_Exists`, `Cmd_CompleteCommand`

**Contract** — exact case-sensitive membership test, and first-prefix-match lookup
over the command list.

**Notes** — both are case *sensitive*, while [`Cmd_ExecuteString`](#cmd_executestring)
matches case-insensitively. So a command can be executed under a spelling under which
it does not "exist", and a variable can be registered under a name that collides in
practice. A rebuild should use one rule everywhere.

### `Cmd_Argc`, `Cmd_Argv`, `Cmd_Args`

**Contract** — the token count; the token at an index, or the empty string when the
index is at or above the count; and the raw remainder, which is nothing when the line
had fewer than two tokens.

**Invariants** — the index test is performed on an unsigned reading of the index, so a
negative index is also out of range and yields the empty string rather than reading
backwards.

### `Cmd_ExecuteString`

**Contract** — takes a line and a source; records the source, tokenizes, and resolves
the first token against commands, then aliases, then variables. An empty line does
nothing. A matching command is called. A matching alias has its value inserted at the
front of the queue. Otherwise the variable handler is offered the line, and if it
declines, "unknown command" is printed.

```text
FUNCTION cmd_execute_string(text, source)
  cmd_source = source
  cmd_tokenize_string(text)
  IF cmd_argc == 0  RETURN

  FOR EACH cmd IN cmd_functions
    IF cmd.name MATCHES argument 0, ignoring case
      CALL cmd.function ;  RETURN

  FOR EACH a IN cmd_alias
    IF a.name MATCHES argument 0, ignoring case
      cbuf_insert_text(a.value) ;  RETURN         # expands, does not call

  IF NOT cvar_command()
    print "Unknown command \"<argument 0>\""
```

**Invariants** — the order is the priority, and it is why a variable may not be
registered under a command's name: the command would always win and the variable
would be unreachable. Commands and aliases match case-insensitively; variables match
case-sensitively, inside the variable handler.

An alias is *expanded*, not called, which is what makes an alias whose value invokes
another alias work, and what makes a self-referential alias an infinite loop that
fills the buffer and then silently drops text.

**Notes** — the resolution is two linear searches over lists of a few hundred
entries, per line, and the variable handler adds a third. On a frame that executes a
configuration file this is the dominant cost. The source's own comment asks for a
lookup table. A rebuild should use one map for all three kinds and resolve in one
pass, keeping the priority order.

### `Cmd_ForwardToServer`

**Contract** — sends the current line to the connected server as a string command.
Prints a diagnostic and does nothing if not connected. Does nothing at all during demo
playback. The command name is included unless it is the literal forwarding command
`cmd`, in which case only the arguments are sent.

```text
FUNCTION cmd_forward_to_server()
  IF NOT connected           print "Can't \"<arg0>\", not connected" ;  RETURN
  IF playing back a demo     RETURN
  write the string-command tag to the client's reliable message
  IF argument 0 IS NOT "cmd"
    append argument 0 AND a space
  IF there are arguments
    append the raw remainder                 # original spacing and quoting
  ELSE
    append a newline
```

**Invariants** — appending uses the accumulating string write
([`common.c`](common.c.md#sz_write-sz_print)), so the pieces form **one** zero-terminated
string on the wire rather than several. The raw remainder rather than the tokens is
what makes `say hello   world` reach other players with its spacing.

Omitting the name for `cmd` is what makes `cmd <anything>` a transparent forwarder:
without it the server would receive `cmd <anything>` and have to strip the prefix.

### `Cmd_CheckParm`

**Contract** — takes a token; returns its 1-based position among the arguments, or 0.
Case-insensitive. A null token is a fatal error.

## `CopyString`

**Contract** — allocates a zone copy of a string. A local helper, used by the alias
command.

## Dead code

Two unused integers declared near the top of the file are leftovers from a memory
corruption hunt. A rebuild omits them.
