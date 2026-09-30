# WinQuake/conproc.c

> A helper thread that lets a dedicated server's console be driven by a parent process through shared memory and events.

**Needs** — [`conproc.h`](conproc.h.md) · the platform's threading, shared memory and console interfaces
**Used by** — [`sys_wind.c`](sys_wind.c.md), the dedicated-server platform backend
**Tier floor** — T2 as written; it needs a thread and shared memory

## Purpose

The only inter-process machinery in the engine. A dedicated server launched by a controlling program needs its
console readable and writable from outside; this provides that by running a thread that waits on an event, reads
a command out of a shared buffer, acts on the platform's console, and signals back.

## State

```text
VARIABLE the shared buffer handle, and two event handles — one signalled by the
         parent, one by this process
```

## The protocol

```text
CONSTANT ccom_write_text    = 0x2      # the buffer holds text to write
CONSTANT ccom_get_text      = 0x3      # read the console's contents into it
CONSTANT ccom_get_scr_lines = 0x4      # report the console's line count
CONSTANT ccom_set_scr_lines = 0x5      # set it
```

**Invariants** — a five-value command set over one shared buffer and a pair of events: the parent writes a
command and signals; the thread acts and signals back. That is the whole interface.

## `InitConProc`, `DeinitConProc`

**Contract** — take the buffer and event handles; start the servicing thread; and stop it.

## `RequestProc`

**Contract** — the thread body: waits for either the parent's event or a shutdown signal, dispatches the command
in the buffer, and signals completion.

## `GetMappedBuffer`, `ReleaseMappedBuffer`, `GetScreenBufferLines`, `SetScreenBufferLines`, `ReadText`, `WriteText`, `CharToCode`, `SetConsoleCXCY`

**Contract** — map and unmap the shared buffer; read and set the console's line count; read a range of console
lines into the buffer; write text to the console by synthesizing key events; translate a character into a key
code; and resize the console.

**Invariants** — **writing text works by synthesizing keyboard input events** into the console, because that
platform's console had no direct write path from another process. That is the file's one genuinely surprising
mechanism, and it means the written text must be translated character by character into key codes — which is what
the translation routine is for, and why it handles only a limited character set.

**Notes** — entirely platform-specific and entirely optional: the dedicated server works without it, losing only
external console control. A rebuild should use a socket or a pipe.
