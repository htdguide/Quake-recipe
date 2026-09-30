# QW/fixskins.sh

> Data: an eight-line script that renames downloaded skin files into the layout the client expects.

**Needs** — nothing
**Used by** — run by hand, once, after upgrading
**Tier floor** — none

## Purpose

A migration script. An earlier version stored player skins in a different directory from the one the shipped client uses
([`client/cl_parse.c`](client/cl_parse.c.md) writes skins to a fixed shared directory, everything else to the current content directory),
and this moves them.

## State

Data; no run-time state.

## What it records

The two directory layouts and the mapping between them.

**Notes** — worth one line in the recipe for what it implies: **the location of downloaded content is a decision with a migration cost.**
The client hard-codes it, so changing it strands every file a player already has, and this script is what that cost looked like.
