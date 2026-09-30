# readme.txt

> Data: the release note for the source publication itself — what is in the tree, what licence it is under, and what has been removed.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The top-level note accompanying the source release. Its value to the recipe is the inventory and the two caveats.

## State

Data; no run-time state.

## What it records

- **Two engines in one tree**: the original ([`WinQuake/`](WinQuake/README.md)) and QuakeWorld ([`QW/`](QW/README.md)), with the
  hardware-rendered variants built from the same directories.
- **The licence**: the release is under the terms in [`gnu.txt`](gnu.txt.md).
- **Content is not included.** The engine will not run without the game's own data files, which are not part of the release. That is why
  the recipe treats the content formats as **givens** ([`SYSTEM-REQUIREMENTS.md`](SYSTEM-REQUIREMENTS.md)) — a rebuilder can read them but
  cannot obtain them here.
- **The tools are not included**: no map compiler, no model compiler, no game-logic compiler. So the three formats those tools produce
  ([`bspfile.h`](WinQuake/bspfile.h.md), [`modelgen.h`](WinQuake/modelgen.h.md), [`pr_comp.h`](WinQuake/pr_comp.h.md)) are constraints the
  recipe can describe but not regenerate.
- **No support.** The release is as-is, which is consistent with what the tree contains: abandoned designs, unresolved defects and disabled
  experiments ([`QW/client/notes.txt`](QW/client/notes.txt.md), [`QW/client/docs.txt`](QW/client/docs.txt.md)).

**Notes** — the missing tools are the single most important practical fact in this file. A rebuild can play existing maps and models; it
cannot make new ones without either reimplementing three compilers or using a third party's.
