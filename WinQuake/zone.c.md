# WinQuake/zone.c

> Implements the four allocators: a double-ended stack, a first-fit heap, and a compacting least-recently-used cache that lives in whatever space the stack leaves.

**Needs** — [`zone.h`](zone.h.md) · [`quakedef.h`](quakedef.h.md) · [`common.h`](common.h.md) (argument parsing, byte moves) · [`cmd.h`](cmd.h.md) (registers the flush command) · [`console.h`](console.h.md) (diagnostic output) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`host.c`](host.c.md) initializes it; almost every other module allocates from it
**Tier floor** — T1 as written, because the cache computes free space by subtracting addresses of live objects. A T2 rebuild that keeps the arena as a byte array with integer offsets meets every contract here.

## Purpose

One contiguous block arrives at startup and this file divides it. Three
independent allocators share the block and must cooperate: the hunk grows from
both ends, and the cache occupies the middle and *gets out of the way* when either
end grows. That cooperation — not any one allocator's internals — is what makes
this file worth reading.

## State

```text
RECORD Hunk
  base       : pointer        # the single block
  size       : int            # its length in bytes
  low_used   : int            # bytes consumed from the bottom
  high_used  : int            # bytes consumed from the top
  temp_active: bool           # a temporary high allocation is outstanding
  temp_mark  : int            # the high mark to restore when it is dropped
  # invariant: low_used + high_used <= size
  # invariant: every live cache system lies entirely within
  #            [base+low_used, base+size-high_used)

RECORD HunkBlockHeader         # precedes every hunk allocation
  sentinel : int (32-bit)      # fixed magic; any other value means corruption
  size     : int               # including this header, always a multiple of 16
  name     : text[8]           # for the diagnostic dump; not null-terminated

RECORD Zone
  size      : int
  blocklist : ZoneBlock        # a sentinel node that is both head and tail
  rover     : pointer to ZoneBlock   # where the next search starts

RECORD ZoneBlock               # precedes every zone allocation
  size : int                   # including this header and any tiny tail
  tag  : int                   # 0 means free; any non-zero value means in use
  id   : int                   # fixed magic
  prev, next : pointer to ZoneBlock
  # invariant: blocks tile the zone with no gaps —
  #            address_of(b) + b.size == address_of(b.next)
  # invariant: no two adjacent blocks are both free
  # invariant: the four bytes ending an in-use block hold the magic, as a
  #            trash detector

RECORD CacheSystem             # precedes every cache entry
  size     : int               # including this header, multiple of 16
  user     : pointer to CacheHandle    # back-reference, so eviction can
                                       # clear the holder
  name     : text[16]
  prev, next         : pointer to CacheSystem   # address order, ascending
  lru_prev, lru_next : pointer to CacheSystem   # use order, most recent first
  # invariant: the address-order list is sorted by address and its members
  #            never overlap
  # invariant: a system is in both lists or in neither
```

The two cache lists over the same nodes are the design. The address-order list is
how the allocator finds a gap and how it knows which entries block a growing hunk;
the use-order list is how it chooses a victim. Neither alone suffices.

## `Memory_Init`

**Contract** — takes the block's base and size; sets both hunk marks to zero,
initializes the cache's two circular lists to empty, registers the console
command that flushes the cache, then allocates the zone out of the low hunk and
formats it as one free block. Honours a command-line override giving the zone size
in kilobytes; the override without a following value is a fatal error. Called once.

```text
FUNCTION memory_init(buf, size)
  hunk.base = buf ;  hunk.size = size
  hunk.low_used = 0 ;  hunk.high_used = 0
  cache_init()                                  # both lists point at the sentinel
  zonesize = 0xc000                             # 49,152 bytes
  IF command line has "-zone"
    IF a value follows  zonesize = value * 1024
    ELSE                FAIL WITH "must specify a size in KB after -zone"
  zone = hunk_alloc_named(zonesize, "zone")
  zone_clear(zone, zonesize)
```

**Notes** — the zone is the *first* hunk allocation, which is why it sits at the
very bottom of the map and can never be freed.

---

## The hunk

### `Hunk_AllocName`

**Contract** — takes a size and a name; rounds the size up to a multiple of 16,
adds a header, and carves it off the low end. Zero-fills the result. Evicts
whatever cache entries now overlap the new low mark. Fatal error on a negative
size or on insufficient free space; there is no failure return.

```text
FUNCTION hunk_alloc_named(size, name) -> pointer
  IF size < 0  FAIL WITH "bad size"
  size = size_of(HunkBlockHeader) + round_up_to_multiple_of_16(size)
  IF hunk.size - hunk.low_used - hunk.high_used < size
    FAIL WITH "failed on <size> bytes"

  h = hunk.base + hunk.low_used
  hunk.low_used = hunk.low_used + size
  cache_free_low(hunk.low_used)          # make the cache yield the space
  zero_fill(h, size)
  h.size = size ;  h.sentinel = MAGIC ;  h.name = first 8 bytes of name
  RETURN h + size_of(HunkBlockHeader)
```

**Invariants** — the returned address is 16-byte aligned provided the block base
is, which the platform backend guarantees. The alignment is load-bearing: the
software renderer's structures are aligned to a cache line on purpose and the
assembly inner loops assume it.

**Notes** — note the order: the mark advances *before* the cache is told to move
out of the way. Growing the hunk into cache territory is legal; the cache is then
obliged to relocate or die.

### `Hunk_Alloc`

**Contract** — `Hunk_AllocName` with the name "unknown".

### `Hunk_LowMark`, `Hunk_FreeToLowMark`

**Contract** — read and restore the low mark. Restoring zeroes the reclaimed
range. A mark outside [0, current] is a fatal error.

**Notes** — the zeroing is not hygiene, it is relied upon: the next
`Hunk_AllocName` also zeroes, so the only purpose here is that a stale pointer
into freed level data reads zeros rather than plausible garbage. A rebuild may
skip it and will be harder to debug.

### `Hunk_HighAllocName`

**Contract** — as the low-end version but from the top, and it *returns nothing*
rather than aborting when space runs short, after printing a diagnostic. Discards
any outstanding temporary allocation first. Evicts cache entries overlapping the
new high mark.

**Notes** — the failure return exists for exactly one caller pattern: a video
backend trying a resolution, failing to fit its framebuffer, and falling back to
the base mode. That is the whole reason the high end is a separate discipline.

### `Hunk_HighMark`, `Hunk_FreeToHighMark`

**Contract** — read and restore the high mark. **Reading it has a side effect**:
an outstanding temporary allocation is discarded first, so that the returned value
does not include it. Restoring zeroes the reclaimed range and also discards a
temporary allocation first.

```text
FUNCTION hunk_high_mark() -> int
  IF hunk.temp_active
    hunk.temp_active = false
    hunk_free_to_high_mark(hunk.temp_mark)
  RETURN hunk.high_used
```

**Notes** — a read with a side effect is a trap for a rebuild. It is there because
`Hunk_TempAlloc` implements itself in terms of these two, and without the discard
a temporary allocation would be captured into somebody's saved mark and leak for
the rest of the process's life. A rebuild should separate the discard into its own
operation and call it explicitly; the observable behaviour is identical.

### `Hunk_TempAlloc`

**Contract** — takes a size; returns high-end space valid until the next call to
itself. Not zero-filled. Discards the previous temporary allocation.

```text
FUNCTION hunk_temp_alloc(size) -> pointer
  size = round_up_to_multiple_of_16(size)
  IF hunk.temp_active
    hunk.temp_active = false
    hunk_free_to_high_mark(hunk.temp_mark)
  hunk.temp_mark = hunk_high_mark()
  buf = hunk_high_alloc_named(size, "temp")
  hunk.temp_active = true
  RETURN buf
```

**Notes** — only one is live at a time, and this is the engine's whole-file read
buffer ([`common.c`](common.c.md)). The consequence a rebuild must respect: a
loader may not hold a temporary buffer across a call that itself loads a file. The
original satisfies this by call ordering and nothing checks it.

### `Hunk_Check`, `Hunk_Print`

**Contract** — walk the low-end blocks (and, for the print, the high-end ones
too) verifying each header's magic and that each size is at least 16 and stays
within the block; abort on violation. The print totals consecutive allocations
sharing a name unless asked for every block individually.

**Notes** — the walk relies on every hunk block being contiguous and
self-describing, which is why the size field includes the header. This is the
diagnostic that tells a rebuilder where their memory went, and it is worth
reproducing.

---

## The zone

### `Z_ClearZone`

**Contract** — formats a region as a single free block bracketed by a sentinel
node that acts as both list head and tail. The sentinel is marked in-use with size
zero, so the coalescing logic can never merge across it.

**Notes** — marking the sentinel in-use is the trick that removes every
end-of-list special case from free and allocate. A rebuild using a real sentinel
node should do the same.

### `Z_TagMalloc`

**Contract** — takes a size and a non-zero tag; returns a block of at least that
size, or nothing if no free block fits. Does not zero-fill. A zero tag is a fatal
error. Search is first-fit starting from a rover that remembers where the last
allocation ended, so repeated allocation walks forward rather than rescanning from
the front.

```text
FUNCTION zone_alloc_tagged(size, tag) -> optional<pointer>
  IF tag == 0  FAIL WITH "tried to use a 0 tag"
  size = size + size_of(ZoneBlock)
  size = size + 4                        # room for the trailing trash marker
  size = round_up_to_multiple_of_8(size)

  base = rover = zone.rover
  start = base.prev                      # one full lap ends here
  REPEAT
    IF rover == start  RETURN nothing     # scanned the whole list, no fit
    IF rover.tag != 0                     # in use: restart the candidate run
      base = rover = rover.next
    ELSE
      rover = rover.next                  # free: extend the candidate run
  WHILE base.tag != 0 OR base.size < size

  extra = base.size - size
  IF extra > MINFRAGMENT                 # 64 bytes
    split off the tail as a new free block, linked after base
    base.size = size
  # else: the tail is too small to be worth a header, so it stays inside
  #       the allocation and is silently wasted

  base.tag = tag
  base.id = MAGIC
  zone.rover = base.next                 # next search starts past this one
  write MAGIC into the last 4 bytes of the block   # trash detector
  RETURN base + size_of(ZoneBlock)
```

**Invariants** — a block is never split when the remainder would be 64 bytes or
less, so the heap never accumulates headers larger than the space they describe. A
returned block may therefore be up to 64 bytes larger than requested, and the
trailing magic sits at its true end, not at the requested end.

**Notes** — the loop reads oddly because `base` tracks the start of a run of
adjacent free blocks while `rover` scans; but the invariant that no two free
blocks are ever adjacent means the run is always length one, so the loop is a
plain first-fit over free blocks. A rebuild should write the plain version.

The rover is a deliberate policy, not an optimization artifact: it spreads
allocations across the heap so that short-lived strings from the console do not
repeatedly fragment the same region.

### `Z_Malloc`

**Contract** — `Z_TagMalloc` with tag 1, zero-filling the result, and a fatal
error rather than a failure return. Runs the full heap consistency check first, on
every call.

**Notes** — the unconditional check is quadratic in the number of live blocks and
a rebuild should put it behind a switch. It is left in because the zone holds only
a few dozen blocks in practice.

### `Z_Free`

**Contract** — takes a pointer previously returned by the zone; marks the block
free and merges it with an adjacent free neighbour on either side. Fatal error on
a null pointer, on a block whose magic is wrong (so: not a zone pointer, or
overwritten), and on a block already free.

```text
FUNCTION zone_free(ptr)
  IF ptr IS nothing              FAIL WITH "NULL pointer"
  block = ptr - size_of(ZoneBlock)
  IF block.id != MAGIC           FAIL WITH "freed a pointer without ZONEID"
  IF block.tag == 0              FAIL WITH "freed a freed pointer"
  block.tag = 0

  IF block.prev IS free                     # merge backwards
    block.prev absorbs block
    IF zone.rover == block  zone.rover = block.prev
    block = block.prev
  IF block.next IS free                     # merge forwards
    block absorbs block.next
    IF zone.rover == the absorbed block  zone.rover = block
```

**Invariants** — after coalescing, no two adjacent blocks are free. The rover must
be repaired whenever it points at a block that has just been absorbed; missing
that is the classic bug in this shape of allocator and the source handles both
directions.

### `Z_CheckHeap`, `Z_Print`

**Contract** — walk the block list asserting the three structural invariants —
blocks tile without gaps, back links agree with forward links, no two adjacent
free blocks — and abort (check) or print (dump) on violation.

---

## The cache

The cache occupies the gap between the two hunk marks. It has no space of its own:
every allocation searches the gaps between existing entries, and a growing hunk
forces entries to relocate or die.

### `Cache_TryAlloc`

**Contract** — takes an already-rounded size and a flag saying whether the very
bottom of the gap may be used; returns a new cache system linked into both lists,
or nothing if no gap is large enough. Searches address order from the bottom up,
then tries the space at the very top.

```text
FUNCTION cache_try_alloc(size, nobottom) -> optional<CacheSystem>
  IF NOT nobottom AND the cache is empty
    IF gap between hunk marks < size  FAIL WITH "greater than free hunk"
    place the system at hunk.base + hunk.low_used
    make it the only member of both lists
    RETURN it

  candidate = hunk.base + hunk.low_used          # bottom of the gap
  FOR EACH cs IN the address-ordered list, ascending
    skip this gap IF nobottom AND cs IS the first entry
    IF address_of(cs) - candidate >= size
      place the system at candidate, link it before cs in address order,
      and at the head of the use-ordered list
      RETURN it
    candidate = address_of(cs) + cs.size         # next gap starts after cs

  IF (hunk.base + hunk.size - hunk.high_used) - candidate >= size
    place the system at candidate, link it at the end of address order
    and at the head of use order
    RETURN it
  RETURN nothing
```

**Notes** — the `nobottom` flag is the subtle part. It means "do not use the space
at the very bottom of the gap", and it exists for exactly one caller: the
relocation path, which is *making room* at the bottom and would otherwise
immediately place the relocated copy in the space it was trying to clear.

### `Cache_Move`

**Contract** — takes a live entry; tries to allocate an equally-sized entry
elsewhere (with the bottom excluded), copies the payload, transfers the holder's
handle to the new location, and frees the old. If no room exists anywhere else,
frees the entry outright — the data is reconstructible, so losing it is legal.

```text
FUNCTION cache_move(c)
  new = cache_try_alloc(c.size, nobottom = true)
  IF new EXISTS
    copy the payload bytes from c to new
    new.user = c.user ;  new.name = c.name
    cache_free(c.user)                 # unlinks the old entry
    new.user.data = payload address of new
  ELSE
    cache_free(c.user)                 # no room: drop it
```

**Notes** — "tough luck", as the source puts it. That the cache may silently
discard anything at any allocation is the contract every cache user is written
against, and it is why [`Cache_Check`](#cache_check) exists.

### `Cache_FreeLow`, `Cache_FreeHigh`

**Contract** — take a proposed new hunk mark; relocate or discard entries until
nothing lies beyond it. `Cache_FreeLow` works from the lowest-addressed entry up;
`Cache_FreeHigh` from the highest down, and if an entry fails to move out of the
way twice in a row it is discarded rather than retried.

```text
FUNCTION cache_free_low(new_low)
  WHILE the cache is not empty
    c = lowest-addressed entry
    IF address_of(c) >= hunk.base + new_low  RETURN     # gap is clear
    cache_move(c)

FUNCTION cache_free_high(new_high)
  previous = nothing
  WHILE the cache is not empty
    c = highest-addressed entry
    IF address_of(c) + c.size <= hunk.base + hunk.size - new_high  RETURN
    IF c IS previous
      cache_free(c.user)      # it was asked to move once and did not; drop it
    ELSE
      cache_move(c) ;  previous = c
```

**Invariants** — both loops terminate. `Cache_FreeLow` terminates because each
iteration either returns or removes the lowest entry from the region below the new
mark (relocation is forbidden from using the bottom). `Cache_FreeHigh` needs the
extra `previous` guard because relocation may legitimately leave an entry where it
was, and without the guard the loop would spin.

**Notes** — the asymmetry between the two is real and a rebuild must keep it.

### `Cache_Alloc`

**Contract** — takes a handle, a size and a name; rounds the size up past a header
to a multiple of 16 and repeatedly tries to place it, evicting the
least-recently-used entry each time it fails. Fatal error only when the cache is
empty and it still does not fit. A handle that already holds data, or a
non-positive size, is a fatal error. On return the entry is most-recently-used.

```text
FUNCTION cache_alloc(handle, size, name) -> pointer
  IF handle.data EXISTS  FAIL WITH "already allocated"
  IF size <= 0           FAIL WITH "size <size>"
  size = round_up_to_multiple_of_16(size + size_of(CacheSystem))
  LOOP
    cs = cache_try_alloc(size, nobottom = false)
    IF cs EXISTS
      cs.name = name ;  cs.user = handle ;  handle.data = payload of cs
      BREAK
    IF the use-ordered list is empty  FAIL WITH "out of memory"
    cache_free(least recently used entry's handle)
  RETURN cache_check(handle)            # also promotes it to most-recent
```

### `Cache_Check`

**Contract** — takes a handle; returns nothing if the entry has been evicted,
otherwise returns the payload address and promotes the entry to most-recently-used
by unlinking and relinking it in the use-ordered list.

**Notes** — this is the *only* legitimate way to reach a cached object. The
promotion is why it is not a plain field read.

### `Cache_Free`

**Contract** — takes a handle; unlinks the entry from both lists and clears the
handle. Fatal error if the handle holds nothing.

### `Cache_Flush`

**Contract** — frees every entry, oldest-addressed first, so that everything
reloads on demand. Registered as the console command `flush`; also used at level
change.

### `Cache_Init`, `Cache_Report`, `Cache_Print`, `Cache_Compact`

**Contract** — `Cache_Init` empties both lists and registers the flush command.
`Cache_Report` prints the current gap size in megabytes. `Cache_Print` lists
entries in address order with their sizes. `Cache_Compact` does nothing.

**Notes** — the empty compaction routine is a stub that was never written, and a
rebuild should omit it rather than reproduce it. Its absence is why
[`Cache_Move`](#cache_move) has to relocate one entry at a time during hunk growth
rather than defragmenting once.
