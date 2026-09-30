# WinQuake/gas2masm — the assembly syntax translator

> A one-purpose program so that the forty hand-written routines exist in exactly one copy.

The engine's assembly is written for one assembler and the Windows build uses another. Rather than maintain two copies, the build
translates. That is the whole of this directory.

[`gas2masm.c`](gas2masm.c.md) — a table-driven translator over the small subset of instructions the engine actually uses, with the
**reversed operand order** and the **size-in-the-mnemonic** rule as its two substantive transformations, and hand-written handlers for
the non-commutative floating-point operations because getting one backwards produces code that assembles cleanly and computes the wrong
thing.

**Why it is in the recipe.** It records a decision with a consequence: *one source, two toolchains*. A rebuild keeping the assembly
faces the same choice — translate, or duplicate. A rebuild using the portable twins
([Seam: Vectorized inner loops](../../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)) deletes both this directory and the
assembly it serves.

Note what happened in the later tree: [`QW/client`](../../QW/client/README.md) checks in both the translator *and* its output, and the
output is maintained by hand. This directory is the tool that would have prevented that.

**Not twinned**: the editor and project files.
