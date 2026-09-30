# WinQuake/quake-hipnotic.spec.sh

> Data: a package description for the first official expansion, as an add-on directory.

**Needs** — nothing
**Used by** — [`Makefile.linuxi386`](Makefile.linuxi386.md) generates a package from it
**Tier floor** — none

## Purpose

A packaging template, and therefore a statement of **where an installation's files must live**. That is information the
engine only implies, through the search-path rules in [`common.c`](common.c.md).

## State

Data; no run-time state.

## What it records

- The install root and the directory layout beneath it.
- Which files belong to this package, and which other packages it requires.
- The launcher scripts it installs, and the options they pass.

**Invariants** — the layout must match what [`common.c`](common.c.md) searches: a base directory holding the shared
content, and an add-on directory per expansion that is searched **before** it, so an add-on can replace a base file without
deleting it. That override order is the load-bearing rule, and these files are its concrete expression.

**Notes** — the packaging system itself is a **given**. The layout is the content.
