# QW/qwcl.x11.spec.sh

> Data: a package description for the windowed software client.

**Needs** — nothing
**Used by** — [`Makefile.Linux`](Makefile.Linux.md) generates a package from it
**Tier floor** — none

## Purpose

A packaging template, and therefore a statement of **where an installation's files must live** — information the engine only implies,
through the search-path rules in [`client/common.c`](client/common.c.md).

## State

Data; no run-time state.

## What it records

- The install root and the directory layout beneath it.
- Which files this package contains, and which other packages it requires.
- The launcher script it installs and the options that script passes.

**Invariants** — the layout must match what [`client/common.c`](client/common.c.md) searches: a base directory holding shared content,
and a per-modification directory searched **before** it, so a modification can replace a base file without deleting it. That override
order is the load-bearing rule and these templates are its concrete form.

**The server's package carries the compiled game logic** ([`../qw-qc/`](../qw-qc/README.md) builds it) and the client's does not,
which is the packaging expression of the client-server split.

**Notes** — the packaging system itself is a **given**. The layout is the content.
