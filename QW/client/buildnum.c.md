# QW/client/buildnum.c

> Derives a monotonically increasing build number from the compiler's own date.

**Needs** — nothing
**Used by** — [`cl_main.c`](cl_main.c.md) and [`sv_main.c`](../server/sv_main.c.md) report it
**Tier floor** — none

## Purpose

A build identifier that needs no build system support. It reads the date the compiler stamps into the translation unit and converts
it to a count of days since a fixed origin.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## `build_number`

**Contract** — parses the compiler's date string into a month, day and year, and returns the number of days from a fixed starting
date. Computes once and remembers.

```text
FUNCTION build_number() -> int
  IF already computed  RETURN it
  parse the three-letter month name, the day and the year from the date string
  days = (year - origin_year) * 365 + accumulated days of the months before
       + the day
  add one for each leap year passed
  RETURN days - a fixed offset
```

**Invariants** — **it is monotonic, which is the only property required.** Two builds can be compared, and a player reporting a
problem can say which build they have. The leap-year handling exists only so that monotonicity holds.

**Notes** — worth reproducing in spirit: a version number that cannot be forgotten, because it comes from the act of compiling.
Any rebuild with a build system should use it instead, but the reason the facility exists — *the reported version must not depend
on anyone remembering to update it* — is a good one.
