# QW/progs — the game logic's build directory

> A build copy. The sources are twinned in [`../../qw-qc/`](../../qw-qc/README.md).

This directory holds the compiled game logic's sources and its compiled output, placed where the server loads them from. The recipe
twins the sources once, as their own chapter, because they are a language and a subject of their own rather than part of the engine:

**→ [`../../qw-qc/`](../../qw-qc/README.md)**

The mapping is one to one: every `.qc` file here, plus `progs.src`, `files.dat` and `progdefs.h`, has its twin there under the same
name.

**Not twinned**: `qwprogs.dat`, the compiled output. It is a binary produced from the sources, and its format is described by
[`pr_comp.h`](../server/pr_comp.h.md) and [`progs.h`](../server/progs.h.md).

**Why the sources are a separate chapter.** They are written in a different language, compiled by a different tool, and they answer a
different question: the engine chapters say how the world is simulated and transmitted, and
[`qw-qc/`](../../qw-qc/README.md) says what the game's rules *are*. A rebuilder replacing the engine keeps them; a rebuilder making a
different game replaces them and keeps the engine.
