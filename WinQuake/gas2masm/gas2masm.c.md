# WinQuake/gas2masm/gas2masm.c

> A one-purpose translator: converts the engine's assembly sources from the syntax one assembler accepts into the syntax the other does, by table-driven token matching over the small subset of instructions actually used.

**Needs** — nothing beyond standard input and output
**Used by** — the Windows build, to assemble the `.s` files with the Microsoft assembler
**Tier floor** — none

## Purpose

The engine's assembly ([`d_draw.s`](../d_draw.s.md) and its forty siblings) is written for one assembler's syntax, and the
Windows build uses another. Rather than maintain two copies, the build translates. This file is that translator.

It is in the recipe because it records a **decision with a consequence**: there is exactly one copy of each assembly
routine, and the price is this program. A rebuild that keeps the assembly must either translate as well or accept the
duplication; a rebuild that uses the portable inner loops
([Seam: Vectorized inner loops](../../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)) deletes both.

## State

```text
CONSTANT max_tokens = 100 ;  max_token_length = 1024
VARIABLE tokens : text[100][1025]  ;  tokennum
VARIABLE inline, outline : line counters, for error messages
VARIABLE currentseg : none | data | text

RECORD RegisterDescription   text (as written) -> emit (as wanted), and its width
RECORD ParseRule             text, emit, expected token count, handler
```

**Invariants** — the translator handles **only what the engine's sources contain**. It is not an assembler front end: an
instruction not in the table is an error, and that is deliberate. Bounding the problem to the actual input is what makes a
thousand lines sufficient.

## The tables

**Contract** — three tables drive everything: registers with their emitted spelling and operand width; instructions with the
number of operands to expect and the handler that emits them; and directives.

**Invariants** —

- **The operand order is reversed** between the two syntaxes — one writes source then destination, the other the reverse —
  so every two-operand handler swaps. This is the single most important fact about the translation and the one a reader of
  the assembly twins needs, because it means the `.s` sources read in the opposite order from what the Windows build
  assembles.
- **Operand size is part of the instruction name** in the input syntax and implicit in the operands in the output, so the
  handlers are grouped by width: separate handlers for byte, word and long forms. That is why there are three variants of
  each emitter.
- A handful of instructions need **special handlers** because the two syntaxes disagree about which operand a
  non-commutative floating-point operation subtracts from. Getting one of these backwards produces code that assembles
  cleanly and computes the wrong thing — the worst possible failure mode, and the reason those handlers are written out
  individually rather than generated.

## `gettoken`, `whitespace`, `parseline`

**Contract** — split a line into tokens, treating the input syntax's separators and comment forms; then match the first token
against the tables and dispatch to its handler, which consumes the expected number of operands.

```text
FUNCTION parse_line()
  tokenize the line
  IF the line is empty or a comment  DONE
  IF the first token is a label      emit it in the output's label form
  IF it is a directive               dispatch to the directive handler
  ELSE
    find the instruction in the table
    IF not found  ERROR with the line number
    IF the token count does not match what the rule expects  ERROR
    call the rule's handler, which emits the mnemonic and the operands
```

**Invariants** — the line number of the **input** is tracked and reported on error, which matters because the output is
generated and nobody reads it. Any translator should do this; it is the difference between a usable tool and a wall of
unexplained assembler errors.

## `emitanoperand` and the operand emitters

**Contract** — render one operand in the output syntax: a register by table lookup, an immediate with its marker removed, a
memory reference with its base, index, scale and displacement rearranged into the output's bracket form, and a symbol
possibly with an offset.

**Invariants** — the memory-reference form is where the two syntaxes differ most, and the rearrangement is purely
mechanical. What is *not* mechanical is that the output syntax needs an explicit size for a memory operand where the input
carries it in the mnemonic — which is why the operand emitter must be told the width by its caller.

## `datasegstart`, `textsegstart`, `emitdata`, `emitonedata`, `emitonecalldata`, `emitonejumpdata`, `emit_multiple_data`, `emitexterndef`

**Contract** — emit segment declarations, data definitions of each width, a table entry holding a routine address, a jump
table entry, a repeated value, and an external declaration.

**Invariants** — **jump tables are recognized and emitted specially**, because the assembly uses computed jumps over tables of
label addresses ([`d_draw.s`](../d_draw.s.md)'s span loops do this) and the two syntaxes declare a code address in data
differently. A rebuild would not have jump tables in data at all; the recipe records that the original does, because it is
part of why those inner loops are fast.

## `main`, `errorexit`

**Contract** — read the input file, write the output file, translating line by line; abort with the input's line number on any
unrecognized construct.

**Notes** — the whole program is incidental: it exists because of a toolchain difference, and the problem it solves is "one
source, two assemblers". A rebuild's equivalent problem is "one algorithm, several vector instruction sets", and the answer
is the same shape — write it once in a portable form and let the target decide — which is exactly what the portable twins of
the assembly routines already are.
