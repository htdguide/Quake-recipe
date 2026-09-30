# WinQuake/wqreadme.txt

> Data: the Windows software build's release note — what changed from the original, the command-line options, and the known problems.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The user-facing description of the Windows software port. Its recipe value is the **command-line options list**, which is
the fullest statement anywhere in the tree of what the engine can be told at startup.

## State

Data; no run-time state.

## What it records

- The option set: heap size, resolution and mode, whether to use a window, which sound and network drivers to attempt, the
  content directory and the add-on directory, dedicated-server and listen-server modes, and the diagnostic switches. These
  are read in [`common.c`](common.c.md), [`host.c`](host.c.md) and each backend's initialization, and a rebuild that wants
  to accept the same invocations should take the list from here rather than by grepping.
- That the window can be **resized and the game keeps running**, which is the requirement behind the mode-change machinery
  in [`vid_win.c`](vid_win.c.md).
- The known problems, each of which corresponds to a platform workaround in the video or sound backend.
- The content requirements: which files must be present and where ([`common.c`](common.c.md)'s search path).

**Notes** — the option list and the content layout are the two things worth extracting. Both are contracts with the user
that a rebuild should honour if it wants to run against existing installations.
