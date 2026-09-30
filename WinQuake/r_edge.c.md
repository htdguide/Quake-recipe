# WinQuake/r_edge.c

> The edge sorter: walks the screen one scanline at a time, maintaining a sorted active edge list and a depth-sorted active surface stack, and emits a span each time the topmost surface changes. This is the software renderer's central algorithm.

**Needs** — [`r_local.h`](r_local.h.md) · [`r_shared.h`](r_shared.h.md) · [`d_iface.h`](d_iface.h.md) · [`quakedef.h`](quakedef.h.md) · [`vid.h`](vid.h.md) (the surface lock) · [`sound.h`](sound.h.md) (the mid-frame mixer top-up) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) (four routines have assembly twins in [`r_edgea.s`](r_edgea.s.md))
**Used by** — [`r_main.c`](r_main.c.md) drives it; [`r_draw.c`](r_draw.c.md) fills its input; [`d_edge.c`](d_edge.c.md) consumes its output
**Tier floor** — T1 as written; the algorithm is tier-free but depends on surfaces being comparable by address

## Purpose

This is the file the whole software renderer exists to serve. Everything upstream — the tree walk, the
clipping, the edge emission — produces its input; everything downstream consumes its output.

The algorithm: every visible polygon has contributed its edges to a per-scanline bucket. For each
scanline, insert that scanline's new edges into a list kept sorted by screen x; walk that list left to
right maintaining a stack of surfaces currently "open"; and whenever the *nearest* open surface changes,
emit a span for the one that was on top. Then step every edge's x by its per-scanline slope, remove the
edges that ended, and move down a line.

The result is a set of spans covering the screen exactly once, with no overdraw and **no depth buffer**.
Two properties make it work: surfaces arrive in back-to-front tree order so their pool addresses encode
depth, and a polygon is planar so its depth gradient is linear and two surfaces can be compared at a
point with three multiplies.

## State

```text
VARIABLE r_edges, edge_p, edge_max   : Edge      # the edge pool and its cursor
VARIABLE auxedges                    : Edge      # a second pool for sub-models
VARIABLE surfaces, surface_p, surf_max : Surf    # the surface pool and cursor
VARIABLE newedges    : Edge[1024]     # edges starting on each scanline
VARIABLE removeedges : Edge[1024]     # edges ending on each scanline
VARIABLE span_p, max_span_p : ESpan
VARIABLE r_currentkey : int           # the next surface's sort key
VARIABLE current_iv  : int ;  fv : real     # the scanline being processed
VARIABLE edge_head, edge_tail, edge_aftertail, edge_sentinel : Edge
VARIABLE edge_head_u_shift20, edge_tail_u_shift20 : int
VARIABLE pdrawfunc : handler          # which span generator is in use
```

**Invariants** — **surfaces[0] is a dummy and surfaces[1] is the background.** Index 0 must be unusable
because an edge stores its two surface indices as 16-bit values and zero means "no surface". Index 1 is
the surface that is always behind everything, and it doubles as the active stack's circular sentinel.

**A higher surface address means nearer the viewer**, because the tree walk allocates surfaces back to
front. That is stated in [`r_shared.h`](r_shared.h.md) and it is what makes the stack ordering a
comparison of *keys* — which are assigned in allocation order — rather than of geometry.

The four sentinel edges bracket the active list so that no walk needs an end test; their construction is
the subtlest part of the setup and is detailed below.

## `R_BeginEdgeFrame`

**Contract** — resets the edge and surface pools, establishes the background surface, selects the span
generator, and clears both per-scanline bucket arrays over the view's vertical range.

```text
FUNCTION r_begin_edge_frame()
  edge_p = the start of the edge pool ;  edge_max = its end
  surface_p = &surfaces[2]              # 0 is a dummy, 1 is the background
  surfaces[1].spans = nothing
  surfaces[1].flags = the draw-background flag

  IF the reverse-order diagnostic is enabled
    pdrawfunc = the BACKWARD span generator
    surfaces[1].key = 0 ;  r_currentkey = 1        # keys ASCEND
  ELSE
    pdrawfunc = the forward span generator
    surfaces[1].key = 0x7FFFFFFF ;  r_currentkey = 0   # keys ascend, background
                                                       # is maximal = farthest
  FOR EACH scanline v IN the view's vertical range
    newedges[v] = removeedges[v] = nothing
```

**Invariants** — the background's key is the **maximum** in the normal path and **zero** in the reverse
path, because the two generators order their stacks oppositely. That one difference is the whole of the
reverse mode, which exists as a diagnostic for verifying the sorter.

## Active edge list maintenance

### `R_InsertNewEdges`

**Contract** — takes a scanline's new edges, already sorted by x, and merges them into the active list,
which is also sorted by x and terminated by a sentinel. Both lists are non-empty.

```text
FUNCTION r_insert_new_edges(to_add, list)
  FOR EACH edge IN to_add
    advance `list` until list.u >= edge.u          # a linear scan from where
                                                   # the previous insert landed
    splice edge in before list
```

**Invariants** — the scan does **not** restart from the head for each new edge, because both lists are
sorted; so the merge is linear in the sum of their lengths. That is the property the whole per-scanline
cost depends on.

**Notes** — the source unrolls the search four ways with explicit jumps. Incidental.

### `R_RemoveEdges`

**Contract** — unlinks every edge on a scanline's removal list.

### `R_StepActiveU`

**Contract** — advances every active edge's x by its per-scanline step and restores the list's sort order
by pushing each displaced edge backward to its correct position.

```text
FUNCTION r_step_active_u(first)
  edge = first
  LOOP
    edge.u = edge.u + edge.u_step
    IF edge.u >= edge.prev.u                       # still in order
      edge = edge.next ;  CONTINUE
    # Out of order: pull it out and walk BACKWARD to its place.
    IF edge IS the after-tail sentinel  RETURN
    next = edge.next
    unlink edge
    w = edge.prev.prev
    WHILE w.u > edge.u  w = w.prev
    relink edge after w
    edge = next
    IF edge IS the tail sentinel  RETURN
```

**Invariants** — this is an **insertion sort over a nearly-sorted list**, and it is linear in practice
because two edges rarely cross between adjacent scanlines. That assumption is the performance model of
the entire renderer: if it failed — if geometry were full of near-parallel crossing edges — the sorter
would become quadratic.

The backward walk needs no bound test because the head sentinel's x is smaller than any real edge's.

## Span emission

### `R_LeadingEdge`

**Contract** — called when the walk crosses an edge that *enters* a surface. Increments that surface's
span state, and on the transition into a span, inserts it into the active stack at its depth position —
emitting a span for whatever it obscures if it becomes the new top. Does nothing on a second entry
(an inverted span).

```text
FUNCTION r_leading_edge(edge)
  IF edge has no entering surface  RETURN
  surf = surfaces[edge.surfs[1]]
  IF ++surf.spanstate != 1  RETURN                 # an inverted span; ignore
  IF surf.insubmodel  r_bmodelactive = r_bmodelactive + 1

  top = the current top of the active stack
  IF surf.key < top.key  GOTO newtop               # strictly nearer

  # Same key means two surfaces on the same plane. The one already active is
  # in front — UNLESS both are sub-models, in which case compare depth.
  IF surf.insubmodel AND surf.key == top.key
    compare_depth_and_maybe(GOTO newtop)

  # Walk down the stack to the insertion point.
  REPEAT  top = top.next  WHILE surf.key > top.key
  IF surf.key == top.key
    IF NOT surf.insubmodel  CONTINUE the walk       # already-active wins
    compare_depth_and_maybe(GOTO gotposition)
    CONTINUE the walk
  GOTO gotposition

newtop:
  # This surface obscures the current top: close the top's span here.
  iu = edge.u SHIFTED RIGHT 20                      # to whole pixels
  IF iu > top.last_u
    emit a span for `top` FROM top.last_u OF length (iu - top.last_u)
  surf.last_u = iu                                  # the new top starts here

gotposition:
  insert surf into the stack before top
```

The depth comparison referred to above:

```text
FUNCTION compare_depth(surf, other, edge) -> bool     # is surf nearer?
  # Evaluate both surfaces' depth gradients at this exact screen point.
  fu = (edge.u - 0xFFFFF) * (1 / 0x100000)          # the edge's x, as a real
  mine   = surf.d_ziorigin  + fv*surf.d_zistepv  + fu*surf.d_zistepu
  theirs = other.d_ziorigin + fv*other.d_zistepv + fu*other.d_zistepu
  IF mine * 0.99 >= theirs  RETURN true              # clearly nearer
  IF mine * 1.01 >= theirs                           # within one percent:
    RETURN surf.d_zistepu >= other.d_zistepu         # break the tie by SLOPE
  RETURN false
```

**Invariants** — five load-bearing details.

**The one-percent tolerance band** is the interesting one. Two coplanar sub-model surfaces have
arithmetically equal depth and comparing them exactly would let floating-point noise flip the answer
between adjacent pixels — which produces the shimmering seam every renderer of this era had. Instead:
if one is more than one percent nearer it wins outright, and within the band the tie is broken by which
surface's depth is *increasing faster across the screen*, which is stable because it does not depend on
the current pixel. A rebuild must implement both halves; the tie-break alone or the tolerance alone
still shimmers.

**A surface with a lower key is nearer** in the forward generator, so the stack is ordered by ascending
key from the top.

**Equal keys mean coplanar, and the already-active surface wins** unless both are sub-models. That rule
is why a door flush against a wall does not fight with it: the wall was allocated first and stays in
front. It is also why two sub-models in the same leaf need the depth comparison — neither was allocated
by the tree walk in a meaningful order.

**The span is emitted for the surface being *obscured*, at the moment it is obscured**, from where it
last became visible up to this edge. So a surface accumulates a list of spans, one per uninterrupted
visible run.

**The span state is a counter, not a flag**, so an edge that enters a surface already open — which
happens after clipping produces an inverted span — is ignored rather than corrupting the stack.

### `R_TrailingEdge`

**Contract** — called when the walk crosses an edge that *leaves* a surface. Decrements the span state,
and on the transition out of a span, removes the surface from the stack — emitting its span if it was on
top, and handing the start position to the surface beneath.

```text
FUNCTION r_trailing_edge(surf, edge)
  IF --surf.spanstate != 0  RETURN                  # an inverted span
  IF surf.insubmodel  r_bmodelactive = r_bmodelactive - 1
  IF surf IS the current top
    iu = edge.u SHIFTED RIGHT 20
    IF iu > surf.last_u
      emit a span for surf FROM surf.last_u OF length (iu - surf.last_u)
    surf.next.last_u = iu                           # whatever is beneath now
                                                    # becomes visible from here
  unlink surf FROM the stack
```

**Invariants** — handing `last_u` to the surface beneath is what makes the span accounting complete: the
newly-exposed surface's span starts exactly where the departing one's ended, so the scanline is covered
with no gap and no overlap.

### `R_LeadingEdgeBackwards`

**Contract** — the reverse generator's entering-edge handler. Same structure with the key comparisons
inverted and **no depth comparison at all** — two coplanar sub-models are resolved arbitrarily, with the
source's comment noting they will never be farthest anyway.

### `R_CleanupSpan`

**Contract** — at the right edge of the screen, emits a span for whatever is still on top, then resets
every stacked surface's span state to zero.

**Invariants** — the reset is required because the stack is not unwound at the end of a scanline; the
sentinels ensure it is empty apart from the background, but the *states* must be cleared or the next
scanline's counters start wrong.

### `R_GenerateSpans`, `R_GenerateSpansBackward`

**Contract** — walk the active edge list once, left to right, calling the trailing handler for each
edge's leaving surface and the leading handler for its entering surface, then clean up.

```text
FUNCTION r_generate_spans()
  r_bmodelactive = 0
  the active stack = just the background, pointing at itself
  surfaces[1].last_u = the left edge, in whole pixels
  FOR EACH edge FROM the head's successor TO the tail
    IF edge has a leaving surface
      r_trailing_edge(that surface, edge)
      IF edge has no entering surface  CONTINUE
    r_leading_edge(edge)
  r_cleanup_span()
```

**Invariants** — an edge can both leave one surface and enter another — that is the common case where two
polygons share an edge — and the leaving is processed **first**. Reversing the order would emit a span
for a surface that has already been superseded.

## `R_ScanEdges`

**Contract** — processes every scanline of the view. Builds the four sentinel edges, allocates the span
pool on the stack, and for each scanline: inserts new edges, generates spans, flushes to the rasterizer
if the span pool is nearly full, removes ended edges, and steps the remaining ones. The last scanline
skips the step, sort and removal.

```text
FUNCTION r_scan_edges()
  # A stack span pool, aligned up to a cache line.
  basespans = 3000 span records plus alignment slack
  span_p = the aligned start
  max_span_p = &span_p[3000 - the view's width]     # the flush threshold

  # --- the four sentinels ---
  edge_head.u = the view's left edge, in 12.20 fixed point
  edge_head.u_step = 0 ;  edge_head.next = &edge_tail
  edge_head.surfs = (none, the background)          # entering the background
  edge_tail.u = the view's right edge + 0xFFFFF     # just past the last pixel
  edge_tail.u_step = 0 ;  edge_tail.next = &edge_aftertail
  edge_tail.surfs = (the background, none)          # leaving the background
  edge_aftertail.u = -1                             # forces the stepper to
                                                    # detect it
  edge_aftertail.next = &edge_sentinel
  edge_sentinel.u = 2000 << 24                      # nothing can sort past this

  FOR EACH scanline iv FROM the view's top TO its bottom-1
    current_iv = iv ;  fv = iv AS a real
    surfaces[1].spanstate = 1                       # the background is
                                                    # pre-opened
    IF newedges[iv] EXISTS  r_insert_new_edges(newedges[iv], edge_head.next)
    CALL pdrawfunc

    IF span_p >= max_span_p                          # the pool is nearly full
      unlock the surface ;  top up the sound mixer ;  lock the surface again
      hand every surface's spans to the rasterizer
      clear every surface's span list
      span_p = the pool's start

    IF removeedges[iv] EXISTS  r_remove_edges(removeedges[iv])
    IF the active list is not empty  r_step_active_u(edge_head.next)

  # The last scanline: generate spans, then flush whatever remains.
  ...as above, without the step, sort or removal
  hand every surface's spans to the rasterizer
```

**Invariants** — six things here matter.

**The head and tail sentinels are the background surface's own edges.** The head *enters* the background
at the view's left and the tail *leaves* it at the right, so the background is always open and every
scanline is covered even where no geometry is. That is why there is no "clear the screen" pass.

**The x coordinates are 12.20 fixed point**, and the conversion to whole pixels is a right shift by 20.
The tail's low twenty bits are all ones so that the shift lands exactly one pixel past the last drawn
one — which is what makes the final span's length come out right.

**The after-tail sentinel's x is −1**, which is less than any real edge's, so the stepper's backward-push
detection fires on it and the loop terminates there rather than running off the list.

**The span pool is flushed mid-frame when it nears exhaustion**, not grown. The threshold leaves room for
one more scanline's worth. A flush hands every accumulated span to the rasterizer and starts over — which
is correct only because a surface's spans are independent of each other.

**The sound mixer is topped up inside that flush**, with the surface unlocked around it. A frame that
overflows the span pool is a slow frame, and without this the audio ring would run dry
([`sound.h`](sound.h.md#s_extraupdate)). A rebuild with a callback-driven mixer deletes it.

**The span pool is a stack allocation of about 48 kilobytes**, aligned up to a cache line by hand. A
rebuild allocates it once.

## `R_DrawCulledPolys`

**Contract** — the alternative output path for a rasterizer that declared it wants clipped polygons
rather than spans ([`d_iface.h`](d_iface.h.md)): hands each surface that produced any spans to the
polygon renderer instead, in the order the rasterizer asked for.

**Invariants** — the spans are computed and then **discarded**, used only as a visibility test — which is
why this path is slower and is not what ships. It exists because the interface was designed to admit a
hardware rasterizer, and it is the clearest surviving evidence of that intent.

**Notes** — the file's opening block of disabled text is a list of the author's own doubts: that the
complex cases defeat span coherence, that spans are broken at every edge including hidden ones, and a
question about sentinels at both ends. All three are accurate criticisms of the algorithm and none was
acted on. A rebuild considering this architecture should read them as the author's own assessment of its
limits.
