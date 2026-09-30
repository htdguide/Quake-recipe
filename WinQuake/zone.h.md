# WinQuake/zone.h

> Declares the engine's four allocators and, in the comment that is most of the file, the memory map they carve out of one block.

**Needs** — nothing; this is a leaf
**Used by** — [`zone.c`](zone.c.md) · [`common.c`](common.c.md) · [`model.c`](model.c.md) · [`host.c`](host.c.md) · [`snd_mem.c`](snd_mem.c.md) · [`d_surf.c`](d_surf.c.md) · every video backend · and roughly forty other files
**Tier floor** — T1 for the layout guarantees as written; a T2 rebuild keeps the allocators as arena objects and loses nothing

## Purpose

The engine is handed one contiguous block of memory at startup and never asks for
another. This file declares the four disciplines it divides that block under, and
which is appropriate for what. Choosing the wrong one is not a leak — it is a
crash three levels later when the surface cache cannot find room.

## State

The memory map, top to bottom, is the design:

```text
+----------------------------- top of block ------------------------------+
| high hunk allocations         (stack, grows down)                       |
|   <-- high reset point held by the video backend                        |
|   video framebuffer                                                     |
|   depth buffer                                                          |
|   surface cache                                                         |
|   <-- high-water mark                                                   |
|                                                                         |
|   ... free space: this is where the evictable cache lives ...            |
|                                                                         |
|   <-- low-water mark                                                    |
| client and server level data   (stack, grows up)                        |
|   <-- low reset point held by the host loop                             |
| startup allocations            (never freed)                            |
| zone block                     (about 48 KB, general-purpose heap)      |
+---------------------------- bottom of block ----------------------------+
```

Two stack pointers, one from each end, and a doubly linked list of evictable
objects occupying whatever they leave between them. There is no other allocator
and nothing is ever returned to the operating system.

## The four disciplines

**Hunk, low end** — a stack. Allocate by bumping a pointer up, free by resetting
the pointer to a previously recorded mark. Every allocation is 16-byte aligned,
zero-filled, and tagged with an 8-byte name for the diagnostic dump. This is where
level data goes: the host records a mark before loading a map and resets to it
when the map changes, and that one assignment frees the entire previous level.

**Hunk, high end** — the same stack from the other end, for things whose lifetime
is tied to the *video mode* rather than to the level. The framebuffer, depth
buffer and surface cache live here specifically so that switching to a
higher-resolution mode — which needs more of them — does not leave a hole
underneath the level data.

**Zone** — a conventional first-fit heap with coalescing, sized at about 48 KB by
default, allocated once out of the bottom of the hunk. For small, short-lived,
individually-freed things: console input strings, console variable values, command
buffer text. Anything large belongs on the hunk.

**Cache** — an evictable least-recently-used store occupying the gap between the
two hunk marks. For assets that are expensive to load, useful across level
changes, and *reconstructible*: model geometry, sound samples, surface texture
caches. A cache entry can vanish at any allocation, so every holder must re-check
it before each use and be prepared to reload.

## `Memory_Init`

**Contract** — takes the base address and size of the one block; establishes both
hunk pointers at the ends, initializes the cache list, and carves the zone out of
the low hunk. Reads a command-line override for the zone size in kilobytes.
Called once, before anything else allocates.

## `Hunk_Alloc`, `Hunk_AllocName`

**Contract** — takes a size and an optional name of up to eight characters;
returns zero-filled, 16-byte-aligned memory from the low end. Fatal error on a
negative size or on insufficient space — there is no failure return, because
every caller would only abort anyway. May evict cache entries to make room.

## `Hunk_LowMark`, `Hunk_FreeToLowMark`

**Contract** — read and restore the low-end stack pointer. Restoring to a mark
frees, in one operation, everything allocated after that mark, and zeroes it.
Restoring to a mark above the current pointer or below zero is a fatal error.

## `Hunk_HighAllocName`

**Contract** — as `Hunk_AllocName` but from the high end, and it *returns nothing*
on failure rather than aborting, because its callers — the video backends — have a
fallback: drop to a smaller resolution and try again.

## `Hunk_HighMark`, `Hunk_FreeToHighMark`

**Contract** — read and restore the high-end stack pointer. Reading the mark has a
side effect: it first discards any outstanding temporary allocation (below), so
that the value returned is stable.

## `Hunk_TempAlloc`

**Contract** — returns space from the high end that is valid only until the next
call to itself. Used for whole-file reads, where the caller parses the buffer into
hunk or cache memory and drops the raw bytes. Not zero-filled.

**Notes** — exactly one temporary allocation exists at a time, and the second call
silently invalidates the first. A rebuild will find several call sites that hold a
temporary buffer across a call that itself allocates temporarily; the original gets
away with it by ordering, not by design.

## `Cache_Alloc`

**Contract** — takes a caller-owned handle, a size and a name; finds or makes room
in the gap between the hunk marks, evicting least-recently-used entries until it
fits, and records the address in the handle. Fatal error only when evicting
*everything* still leaves too little room. On return the entry is the most
recently used.

## `Cache_Check`

**Contract** — takes a handle; returns the cached data and promotes the entry to
most-recently-used, or returns nothing if it has been evicted. **Every use of a
cached object must go through this**, not through a remembered address: the point
of the cache is that entries disappear.

## `Cache_Free`

**Contract** — takes a handle; releases the entry and clears the handle. Fatal
error if the handle holds nothing.

## `Cache_Flush`

**Contract** — discards every cache entry, so everything reloads on demand. Bound
to a console command, and used at level change.

## `Z_Malloc`, `Z_TagMalloc`, `Z_Free`

**Contract** — a small zero-filling heap. `Z_Malloc` aborts on exhaustion;
`Z_TagMalloc` takes a non-zero tag, returns nothing on failure, and is the form
used where failure is handled. `Z_Free` aborts on a null pointer, on a pointer
whose header magic is wrong, and on a double free.

## `Z_CheckHeap`, `Z_DumpHeap`, `Z_FreeMemory`, `Hunk_Check`

**Contract** — diagnostics: walk the structures verifying the invariants and
either abort on violation or print. `Z_FreeMemory` totals the zone's free blocks.

**Notes** — [`zone.c`](zone.c.md) calls the zone consistency check on *every*
allocation, unconditionally, not under a debug switch. That is a deliberate 1996
trade and a rebuild should make it conditional.

## Handle type

```text
RECORD CacheHandle            # the caller embeds one of these
  data : optional<pointer>    # nothing means "evicted or never allocated"
```

**Notes** — the handle is a one-field record rather than a bare pointer so that
the cache can find every holder of an entry and clear it on eviction. That
back-reference is the load-bearing part: a rebuild needs some way for the cache to
invalidate its holders, whether that is a handle indirection, a generation counter,
or weak references.
