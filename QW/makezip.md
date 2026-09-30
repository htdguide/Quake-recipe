# QW/makezip

> Data: a one-line packaging command.

**Needs** — nothing
**Used by** — the release process
**Tier floor** — none

## Purpose

One line: archive the release's files. No content beyond the file list it names.

## State

Data; no run-time state.

**Notes** — skip it. The installation layout that matters is in the packaging templates
([`qwcl.spec.sh`](qwcl.spec.sh.md) and its siblings) and in the search-path rules
([`client/common.c`](client/common.c.md)).
