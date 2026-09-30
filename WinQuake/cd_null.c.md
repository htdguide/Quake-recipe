# WinQuake/cd_null.c

> An empty music backend: no soundtrack, for builds with no drive or no need.

**Needs** — [`quakedef.h`](quakedef.h.md)
**Used by** — the dedicated-server build, and any platform without a music backend
**Tier floor** — none

## Purpose

Every operation of the music interface present and doing nothing, with initialization reporting failure so the rest of the
engine treats music as unavailable.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## The operations

**Contract** — play, stop, pause, resume, update, initialize (reporting failure), shut down: all do nothing.

**Invariants** — the interface is **seven operations wide** and entirely optional
([Seam: Redbook CD audio](../SYSTEM-REQUIREMENTS.md#seam-redbook-cd-audio)). A rebuild that wants music at all replaces this with a file
player and keeps the same seven names, because the map-load path calls them by name
([`cl_parse.c`](cl_parse.c.md)).
