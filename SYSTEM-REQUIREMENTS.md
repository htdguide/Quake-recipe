# System requirements

What the world must provide before a line of this is written.

---

## 1. What this builds

A first-person shooter engine and the game that runs on it. A player launches one
program; it reads a game directory of packed asset archives, loads a map, and
renders a three-dimensional world at whatever frame rate the machine allows,
while a simulation advances the world twice as often as the display updates and a
separate mixer keeps positional audio in step with the player's head.

The same program is three programs wearing one coat. A **server** owns the
authoritative world: it advances physics, runs game logic written in a small
interpreted language, and broadcasts state. A **client** samples input, predicts
or receives the server's view of the world, and draws it. A **host** loop drives
both, so one process can run a solo game with no network at all, a listen server
with the local player plus guests, or a headless dedicated server.

The tree holds two generations of that program. The first — `WinQuake/` — is the
original release: single-player and small-LAN multiplayer, server and client in
one binary, lock-step unreliable-datagram netcode that assumes a millisecond
round trip. The second — `QW/` — is the same engine re-cut for the open internet:
client and server split into separate builds, client-side movement prediction,
delta-compressed entity snapshots, and a rate-limited reliable channel. Both are
carried here, because the delta between them is the most instructive netcode
lesson in the tree.

---

## 2. Tier

A recipe is read without the tool that wrote it, so the ladder travels with the
file.

| Tier | Name | The repo demands | Typically |
|---|---|---|---|
| **T0** | Metal | No runtime, no allocator you did not write; bytes at fixed addresses; interrupts, MMIO, boot | asm, C, Zig, Rust `no_std` |
| **T1** | Manual | Deterministic destruction, explicit layout, hard latency or memory budgets, FFI as a first-class concern | C, C++, Rust, Zig |
| **T2** | Managed native | Compiled or JIT with a GC; throughput and data layout still matter; concurrency is explicit | Go, Java, C#, Swift, Kotlin |
| **T3** | Dynamic | Iteration speed over throughput; the library ecosystem carries most of the weight | Python, TypeScript, Ruby, Elixir |
| **T4** | Glue | Orchestration, config and text; spawning processes is the primary abstraction | Shell, Make, Nix, small Python |

T0 is the most demanding tier and T4 the most abstract. Place a repo as close to
T4 as its hard requirements allow.

> **Tier: T1 (Manual).** Floor constraint: the software renderer writes one
> byte per pixel into a framebuffer through a hand-maintained span list, and the
> whole design rests on being able to take the address of a structure field, cast
> a byte range to a record, and walk raw memory with an index that the compiler
> is not allowed to bounds-check. The surface cache, the edge list, the span
> buffer and the particle array are all fixed-size arenas carved out of one
> allocation at startup, with recycling driven by a frame counter rather than by
> reachability — there is no moment at which a collector could run. A 320×200
> frame is 64,000 pixel writes plus per-span perspective correction and
> per-surface lightmap rebuild, inside a budget that must hold at 30 Hz on a
> 1996 machine.
>
> **T2 is viable and is the right target for a modern rebuild**, on one
> condition: the span-filling inner loops must sit behind an interface that a
> rebuild can implement either with native buffer writes or by handing the whole
> job to the GPU, and the fixed arenas must stay arenas rather than becoming
> collected object graphs. Nothing in the simulation, the interpreter, the
> physics, the netcode, or the asset loaders needs manual destruction — they need
> *stable identity* (an entity is addressed by its index in one array, and that
> index travels over the wire), which any tier provides.
>
> **T3 is viable for the server alone.** A dedicated server never rasterizes
> anything; its hot loops are a bytecode interpreter and a BSP hull trace, both
> of which are a few thousand operations per frame at 20 Hz. A T3 rebuild of
> `QW/server` is a reasonable weekend and the recipe supports it.
>
> The original is C with hand-written x86 assembly for roughly forty inner loops
> (see [Seam: Vectorized inner loops](#seam-vectorized-inner-loops)). The
> assembly is an optimization, not a requirement: every assembly routine has a C
> twin selected by one compile-time switch, and the source's own release notes
> say the software renderer loses "almost half its speed" without it and the
> hardware-accelerated renderer loses almost nothing.

---

## 3. Seams

### Seam: Framebuffer surface

**Verdict** — given

**Must provide**
- A rectangular writable buffer of 8-bit palette indices, at least 320×200, with
  a row stride that may exceed the width
- The stride, width and height readable by the engine before it draws
- A way to push an arbitrary set of sub-rectangles of that buffer to the display
  in one call, tearing-free or not
- A 256-entry palette settable at runtime, changeable per frame without
  reallocating the buffer (used for pain flashes and underwater tint)
- An enumerable list of available resolutions with a human-readable description
  of each, and a way to switch between them while the program runs
- Optionally, a second buffer address that writes go straight to the display —
  the engine will use it for the loading plaque if offered and fall back if not

**Surface used** — initialize with a palette, shut down, set palette, shift
palette, set mode, flush a rectangle list, lock/unlock the buffer, notify of
pause. Eight entry points; see [`WinQuake/vid.h.md`](WinQuake/vid.h.md).

**Known-good substitutes** — SDL2/SDL3 with a streaming 8-bit surface blitted
through a palette texture; a raw Linux framebuffer or DRM dumb buffer; a
platform window with a client-side bitmap (`DIBSection`, `XImage`, `CGImage`); a
canvas element with a per-frame palette expansion in the browser. The original
ships nine implementations of exactly this: mode-13h and ModeX VGA, VESA linear
framebuffer, SciTech MGL, SVGAlib, X11 shared-memory images, Sun XIL, a Win32
DIB, and a null sink for the dedicated server.

**If you build it** — do not. On any modern platform a palette framebuffer is one
texture upload; the value in this seam is knowing the engine only wants eight
calls, not that you wrote them.

---

### Seam: Hardware 3D rasterizer

**Verdict** — given

**Must provide**
- Triangle and quad rasterization with per-vertex two-dimensional texture
  coordinates and perspective-correct interpolation
- Two texture units, or one unit plus two passes, with a multiply blend between
  them — the second carries the lightmap
- Bilinear and nearest filtering, selectable per texture
- Mipmapping with runtime selection of the minification filter
- Depth testing with a writable and maskable depth buffer, and depth-test-only
  passes
- Alpha blending with source-alpha and additive modes, and alpha testing against
  a threshold
- Sub-image texture upload into an already-allocated texture, called several
  times per frame as lightmaps change
- Backface culling with a settable winding order, and a scissor or viewport
  restricted to a sub-rectangle of the window
- A projection that can be set from a field of view and a near/far pair, plus a
  two-dimensional orthographic mode for the console and status bar
- One-byte-per-texel upload is *not* required; the engine expands its palette to
  32-bit before upload

**Surface used** — about forty calls: texture bind/upload/parameters, immediate
vertex submission with texture coordinates, matrix load, blend/depth/cull state,
viewport, clear, finish. See [`WinQuake/glquake.h.md`](WinQuake/glquake.h.md).

**Known-good substitutes** — OpenGL 1.1 with `GL_ARB_multitexture` (what the
original targets); any modern OpenGL, Vulkan, Metal, Direct3D or WebGPU wrapper,
which will want the immediate-mode submission replaced by a per-frame vertex
buffer; a software rasterizer, which is what the other renderer in this tree is.

**If you build it** — building this is building the other half of the repo. The
tree already contains a complete software rasterizer
([`WinQuake/d_scan.c.md`](WinQuake/d_scan.c.md) and neighbours), so a rebuilder
who wants no graphics dependency should implement that path and skip this seam
entirely.

---

### Seam: Audio output

**Verdict** — given

**Must provide**
- A ring buffer of interleaved signed PCM samples that the device consumes
  continuously and never stops consuming
- A readable play position within that ring, in samples, monotonically
  increasing modulo the buffer length — the mixer is driven entirely by how far
  the hardware has advanced, so an implementation that cannot report position
  cannot host this engine unchanged
- A negotiated format the engine reads back after opening: sample rate (8000 to
  44100 is exercised), 8 or 16 bit, mono or stereo
- Either direct memory access to the ring, or an explicit submit call the engine
  invokes after writing — both shapes are supported
- Latency small enough that the mixer's lookahead, which defaults to a tenth of
  a second, stays ahead of the read cursor

**Surface used** — open and negotiate, read play position, submit, close. Four
entry points; see [`WinQuake/snd_dma.c.md`](WinQuake/snd_dma.c.md).

**Known-good substitutes** — SDL audio with a callback that reports the consumed
count; ALSA or PulseAudio in mmap mode; WASAPI or DirectSound; `AudioQueue`;
`AudioWorklet` with a `SharedArrayBuffer` in the browser. The original uses DMA
to a SoundBlaster or Gravis card, Win32 `WaveOut` or DirectSound, OSS `mmap`, and
Sun `/dev/audio`.

**If you build it** — the mixer is in the repo and is not this seam; what you
would be building is a device driver. Use a library.

---

### Seam: Keyboard and mouse input

**Verdict** — given

**Must provide**
- Key press and release delivered as separate events, in order, each carrying a
  stable identifier for the physical key — not a translated character, because
  bindings are per key and movement keys must repeat by being held
  down rather than by an auto-repeat timer
- Text entry for the console and menus, which the engine derives from the same
  key identifiers plus a shift state, so a raw scancode stream is sufficient
- Relative mouse motion, unclamped by screen edges, readable once per frame, with
  a way to enter and leave that mode (windowed play needs the cursor back)
- Mouse buttons as three or more keys in the same event stream as the keyboard
- A mouse wheel, delivered as a press/release pair rather than as an axis
- Optionally, a joystick with up to six analog axes and a hat, which the engine
  maps to axes by name rather than by index

**Surface used** — initialize, shut down, drain the event queue into the
engine's key handler, sample the mouse and accumulate into the move command,
activate/deactivate exclusive mouse mode. See
[`WinQuake/input.h.md`](WinQuake/input.h.md).

**Known-good substitutes** — SDL's event queue plus relative mouse mode; GLFW
with a raw-input callback; X11 `XSelectInput` with pointer warping; Win32 raw
input or DirectInput; browser `KeyboardEvent.code` plus Pointer Lock. The
original reads the DOS keyboard interrupt directly, uses Win32 messages plus
DirectInput, or uses SVGAlib's raw keyboard mode.

---

### Seam: Redbook CD audio

**Verdict** — given

**Must provide**
- Enumerate tracks on an inserted audio disc and report whether each is audio
- Play a track by number, with and without looping
- Stop, pause and resume
- Report whether playback is still running, polled once per frame so the engine
  can restart a looping track
- Set output volume, or accept that the engine's music volume control does
  nothing

**Surface used** — initialize, shut down, play track, stop, pause, resume, poll,
set volume. See [`WinQuake/cdaudio.h.md`](WinQuake/cdaudio.h.md).

**Known-good substitutes** — nothing modern; physical CD audio is gone. The
honest rebuild replaces this seam with a music player over files: the engine's
demand is "start looping track *n*, stop it, tell me when it ended", and a
directory of numbered audio files satisfies it exactly. The repo already ships a
do-nothing implementation
([`WinQuake/cd_null.c.md`](WinQuake/cd_null.c.md)) and the engine runs fine
against it, so this seam is genuinely optional.

---

### Seam: Operating system services

**Verdict** — given

**Must provide**
- Blocking byte-stream file access: open for read reporting length, open for
  write truncating, seek to an absolute offset, read, write, close, stat a path
  for a modification timestamp, create a directory
- A monotonic clock with better than millisecond resolution, returned as a
  floating-point count of seconds from an arbitrary origin — the whole engine
  times itself off differences of this value
- Write a line to a console that exists even when the graphics surface has taken
  over the screen
- Read a line from that console without blocking, used by the dedicated server
- Terminate the process with a message the user will see after the graphics
  surface is released
- Yield the remainder of a scheduling quantum
- One allocation of contiguous memory of a size given on the command line,
  defaulting to eight megabytes, which is never grown and never freed
- Optionally: make a code page writable (the original patches its own assembly at
  startup on some platforms), and set the floating-point rounding mode and
  precision

**Surface used** — see [`WinQuake/sys.h.md`](WinQuake/sys.h.md); every platform
backend in the tree implements exactly this list.

**Known-good substitutes** — any standard library. The one item worth checking
is the monotonic clock: the original's DOS and early Win32 backends drift and
wrap, and the code contains workarounds for both that a rebuild on a sane clock
should delete rather than reproduce.

---

### Seam: Unreliable datagram transport

**Verdict** — given

**Must provide**
- Send and receive discrete datagrams up to 1450 bytes (`QW`) or 1024 bytes plus
  an 8-byte header (`WinQuake`) with no delivery, ordering or duplicate
  guarantee — all three are the engine's job and it implements them itself
- Bind to a specific port, and open an ephemeral port
- A non-blocking or pollable read that reports the sender's address
- Broadcast to the local segment, used for server discovery
- Compare two addresses for equality, format one as text, parse one from text,
  and read or replace the port within one
- Resolve a host name to an address

**Surface used** — a fifteen-function driver record, listed in full at
[`WinQuake/net.h.md`](WinQuake/net.h.md), which the engine fills in once per
available transport and then selects among at run time.

**Known-good substitutes** — UDP over IP, which is what every surviving
implementation uses. The original additionally ships IPX, a direct serial-cable
and modem driver with its own link-layer framing, and a loopback driver used for
the single-player case; those are historical and a rebuild needs only UDP plus
loopback.

**If you build it** — the *reliability layer* on top is in the repo and is
load-bearing ([`WinQuake/net_dgrm.c.md`](WinQuake/net_dgrm.c.md) and
[`QW/client/net_chan.c.md`](QW/client/net_chan.c.md)); do not confuse it with
this seam. The transport itself is a socket.

---

### Seam: Vectorized inner loops

**Verdict** — pluggable

**Must provide** — for each of roughly forty routines, an implementation
satisfying the contract in the corresponding twin. The repo defines the
interface, ships a portable implementation of each, and ships a hand-written x86
implementation selected by one compile-time switch. The routines cluster into:
- Span fill: draw one horizontal run of texels with perspective-corrected
  texture coordinates, 8-bit and 16-bit destinations, with and without a
  lightmap
- Edge-list stepping: advance and sort the active edge table for one scanline
- Alias model rendering: transform, light and rasterize one gouraud-shaded
  triangle from a vertex-compressed mesh
- Particle plotting, sky warping, sprite blitting, screen-to-screen copy
- Vector and matrix primitives, and float-to-int rounding with a known tie rule
- Audio mixing: accumulate one sound into the paint buffer with per-ear volume

**Surface used** — each routine is called from exactly one place in the C code
and the switch is per routine, so a rebuild can accelerate any subset.

**Known-good substitutes** — the portable implementation already in the tree; any
SIMD intrinsic set; a GPU compute pass for the span fillers. A rebuild on a
modern machine should start with the portable path and measure before touching
this seam at all — the 1996 arithmetic is tuned for a machine whose integer
multiply and floating-point divide costs no longer resemble anything current.

**If you build it** — the twins carry the arithmetic, including the fixed-point
formats and the exact rounding, because the rounding is observable: the software
and hardware renderers must agree closely enough that recorded demos play back
identically on both.

---

### Seam: Game logic interpreter

**Verdict** — pluggable

**Must provide** — a virtual machine for the bytecode described in
[`WinQuake/pr_comp.h.md`](WinQuake/pr_comp.h.md), plus the ninety-odd host
functions the bytecode calls out to, described in
[`WinQuake/pr_cmds.c.md`](WinQuake/pr_cmds.c.md). Specifically:
- A flat global value pool addressed by 32-bit slot index, where a three-slot run
  is a vector and any slot may be read as float, integer, string handle, entity
  handle, field offset or function handle
- An entity table of fixed stride, where an entity is identified by its byte
  offset into that table and that offset is what a field-pointer operation
  produces
- 62 opcodes, each a three-address instruction over 16-bit signed operands
- A call convention with eight parameter slots, one return slot, and callee-side
  local save/restore over a shared stack
- A state instruction that couples "set my animation frame and next-think time"
  into one operation, because the game logic's entire animation system is built
  on it

**Known-good substitutes** — the interpreter in this tree; any of the
recompiling implementations that grew up around it later. A rebuild may replace
the bytecode with native code in the host language and skip this seam, but then
it cannot load third-party game modifications, which are most of what this engine
is remembered for.

**If you build it** — do. It is about six hundred lines, the opcode semantics are
fully specified in the twins, and the alternative is throwing away the tree's
entire game layer ([`qw-qc/`](qw-qc/README.md)).

---

### Seam: Palette-indexed image decoding

**Verdict** — buildable

**Must provide** — nothing external. Every image format the engine reads is
defined inside the repo — the archive format, the texture lump format with its
four mip levels, the sprite format, the model skin format — and all of them are
raw 8-bit palette indices with a small header. There is no PNG, no JPEG, no
zlib.

**If you build it** — the only compression anywhere in the asset path is a
run-length scheme for visibility bitsets
([`WinQuake/model.c.md`](WinQuake/model.c.md)) and an unused LZSS flag in the
archive header. Both are a dozen lines. Screenshot writing produces uncompressed
targa or a 1990s LBM variant; a rebuild should write PNG and move on.

---

## 4. Platform assumptions

Each item below is something the code believes without checking. A rebuild that
violates one gets a subtle bug, not a compile error.

**Byte order** — little-endian throughout. Every on-disk and on-wire integer is
little-endian. The code does contain byte-swapping helpers chosen at startup by
probing a union, and it *does* route file and network reads through them, so a
big-endian rebuild is possible; but the assembly, the `UNALIGNED_OK` fast paths,
and several `(int *)` casts over byte buffers assume little-endian anyway.

**Word size** — 32-bit. Pointers are assumed to fit in an integer in several
places; the interpreter's entity handles are byte offsets stored as 32-bit
integers; a coordinate is a 32-bit float everywhere, and the network protocol
quantizes it to a 16-bit fixed-point value with 1/8 unit resolution, so the world
is bounded to ±4096 units on each axis.

**Unaligned access** — assumed free on x86 and guarded by a switch elsewhere.

**Floating point** — IEEE 754 single precision, with the control word explicitly
set to single precision and round-to-nearest on entry, because the fixed-point
conversions in the rasterizer depend on the rounding mode. Denormals are never
relied on. `1.0/0.0` is avoided by epsilon guards, not by trapping.

**Filesystem** — case-insensitive lookup is *not* assumed, but all asset names
inside archives are lowercase and the code lowercases nothing, so a
case-sensitive host works only if the archives are well-formed. Paths use forward
slashes internally and are joined by string concatenation; the code converts to
backslashes nowhere. Maximum in-game path 64 bytes, maximum host path 128 bytes.
No atomic rename, no locking, no symlink awareness. Save games are written in
place with no temporary file.

**Threading** — none. There is exactly one thread of execution. The audio mixer
is driven synchronously from the frame loop and from a "give the sound card more
data" call sprinkled through the slow parts of level loading; the network is
polled. A rebuild is free to add threads but must then invent the
synchronization, because none exists to copy.

**Memory** — one contiguous block, sized from the command line, default eight
megabytes, minimum about 5.5. Out of it: a double-ended stack allocator for
permanent and temporary data, a linked-list cache for evictable assets, and a
hunk-allocated zone for small dynamic objects. Nothing is ever returned to the
operating system. The renderer's working set is carved out of the same block
after the mode is known, which is why changing video mode reloads the level.

**Clock** — a monotonic source with sub-millisecond resolution. The engine
clamps a frame's elapsed time to 0.1 seconds and refuses to advance a frame at
all if less than one server tick has passed in dedicated mode. Wall-clock time is
read once, for the savegame comment.

**Network reachability** — the engine is happy with none. Loopback transport is
always available, so single-player never touches a socket.

**Locale and encoding** — no locale awareness. Text is 8-bit bytes indexed into
a bitmap font; the high bit selects an alternate "glowing" glyph set, so byte
values 128–255 are *not* an extended character set, they are the same 128 glyphs
in a different colour. Uppercasing and comparison are ASCII-only.

**Process model** — one process. The dedicated server does not fork. On some
platforms it installs a helper that pumps the parent's console, which is the only
inter-process machinery anywhere.

---

## 5. Data and persistence

### Must match exactly — third parties hold these bytes

**Asset archive (`.pak`)** — magic `PACK`, a directory offset and length, then
entries of a 56-byte name, an offset and a length. Uncompressed, stored in load
order. Third-party content depends on this format and on the lookup rule: later
archives shadow earlier ones, and a loose file on disk shadows every archive
unless the engine was told otherwise. See
[`WinQuake/common.c.md`](WinQuake/common.c.md).

**Map (`.bsp`)** — version 29, a 15-lump directory, little-endian throughout.
Carries the entity text, the plane array, embedded textures with four mip levels,
vertices, the run-length-encoded visibility bitsets, the draw BSP, texture
mapping vectors, faces, lightmaps, three separate collision BSPs at different
hull sizes, leaves, mark-surface indices, edges, surface-edge indices, and
sub-models. Every published map for this engine is this format. See
[`WinQuake/bspfile.h.md`](WinQuake/bspfile.h.md).

**Animated model (`.mdl`)** — magic `IDPO`, version 6. A shared vertex set
quantized to one byte per axis with a per-model scale and origin, a per-vertex
index into a fixed table of 162 normals, per-frame vertex arrays, texture
coordinates with a seam flag, triangles with a front/back flag, and one or more
skins that may themselves be animation groups. See
[`WinQuake/modelgen.h.md`](WinQuake/modelgen.h.md).

**Sprite (`.spr`)** — magic `IDSP`, version 1. Frames or frame groups of
palette-indexed bitmaps with an origin offset and an orientation mode. See
[`WinQuake/spritegn.h.md`](WinQuake/spritegn.h.md).

**Texture archive (`.wad`)** — magic `WAD2`, a flat directory of typed lumps.
Only the type used for wall textures and the type used for interface graphics are
read by the engine. See [`WinQuake/wad.h.md`](WinQuake/wad.h.md).

**Compiled game logic (`progs.dat`)** — version 6, with a CRC of the field
declarations that the engine checks against its own compiled-in copy and refuses
to load on mismatch. See [`WinQuake/pr_comp.h.md`](WinQuake/pr_comp.h.md).

**Sound (`.wav`)** — RIFF with a `cue` chunk optionally marking a loop point and
an `LTXT` chunk optionally giving the loop length. 8 or 16 bit, mono, any rate;
resampled on load. See [`WinQuake/snd_mem.c.md`](WinQuake/snd_mem.c.md).

**Game protocol** — version 15 for `WinQuake`, version 28 for `QW`. Both are a
byte-tagged message stream over datagrams with the engine's own reliability
layer. Coordinates are 16-bit fixed point at 1/8 unit; angles are one byte per
axis at 360/256 degrees, except where the protocol says otherwise. The exact
message inventory and field order are in
[`WinQuake/protocol.h.md`](WinQuake/protocol.h.md) and
[`QW/client/protocol.h.md`](QW/client/protocol.h.md).

**Connection protocol** — version 3 for `WinQuake`: a four-request,
five-response handshake over broadcast, used for both server discovery and
connection setup. `QW` replaces it with a connectionless text-command layer
prefixed by four `0xFF` bytes. See [`WinQuake/net.h.md`](WinQuake/net.h.md).

**Demo (`.dem`)** — a text line of camera angles, then a stream of
length-prefixed server messages each preceded by the view angles at the time of
capture. Playing a recorded demo back is the single best conformance test a
rebuild has, because it exercises the entire message decoder, the model loaders,
the renderer and the interpolation, and any divergence is visible.

**Save game (`.sav`)** — a text file: version, a comment, the 16 spawn
parameters, skill level, map name, elapsed time, the light style strings, then
every interpreter global and every entity printed as key/value text. Because it
is text keyed by field name, it survives changes to the field layout, which is
the reason it is text.

**Configuration (`config.cfg`)** — a console script of bind and set commands,
written on shutdown and executed on startup.

### Internal — choose freely

The in-memory forms of everything above. The loaders deliberately rewrite the
on-disk structures into different shapes: BSP nodes gain parent pointers and
pointer-ified children, faces gain cached lighting state and a polygon, models
gain a resolved texture list, sounds gain a resampled fixed-rate buffer. Nothing
outside the process sees any of it. The surface cache, the edge and span lists,
the lightmap atlas, the particle pool, the entity visibility list and the
interpreter's runtime stack are all free-form.

---

## 6. Conformance

The repo ships no test suite. Everything below is an assertion the code itself
makes, a constant a third party already depends on, or an externally checkable
behaviour — assembled so a rebuilder has an acceptance list despite there being
no tests to port.

### Golden inputs a rebuild must consume

1. A shipped `.bsp` at version 29 loads, and its three collision hulls trace
   identically to the draw hull's contents for a point-sized probe.
2. A shipped `.mdl` loads, and frame interpolation between two frames of a group
   produces the same vertex positions the original does to within the one-byte
   vertex quantization.
3. A shipped `.pak` enumerates in order, and a loose file of the same name
   shadows the archived one.
4. A recorded `.dem` from the original plays to completion with no unknown
   message byte, no buffer overrun, and the same final score.
5. A `progs.dat` at version 6 loads, its field CRC matches, and the compiled
   game runs a full level from spawn to level change.
6. A `config.cfg` written by the original is executed without error.
7. A `.sav` written by the original restores to a playable state.

### Invariants the code checks at runtime and a rebuild should too

- A message read past its end is an error, not a wrap: the reader sets a flag and
  returns −1 forever after.
- A write into a fixed buffer that would overflow is a fatal error in debug and a
  silent truncation nowhere — the code always either grows or aborts.
- Interpreter stack depth is bounded (32 frames, 64 locals levels); exceeding
  either is a fatal error with a printed call stack.
- Interpreter execution is bounded: after 100,000 instructions without returning,
  the engine declares an infinite loop, prints the call stack, and aborts the
  frame rather than hanging.
- An entity handle of zero is the world; assigning to a field of the world from
  game code is an error the interpreter reports by name.
- The BSP hull trace's recursion asserts that a ray split by a plane yields two
  segments whose union is the original; a failure prints
  `SV_RecursiveHullCheck: backup past 0` or an equivalent.
- The edge list, span list, surface cache and particle pool each have a fixed
  capacity and each has a defined overflow behaviour that is *not* an error:
  edges and surfaces drop the frame's excess, particles refuse to spawn. A
  rebuild that instead grows these silently changes observable behaviour under
  load.
- The surface cache is invalidated by a monotonic frame counter; a cached surface
  whose stamp is older than the current frame's lightstyle change is rebuilt. A
  rebuild that caches without the stamp gets stale lighting.
- Server frames advance at a fixed rate (default 1/72 s in `WinQuake`, 1/77 s
  ceiling in `QW`) independent of render rate; physics must not be coupled to
  frame time, and the original's clamping of frame time to 0.1 s is what keeps a
  stalled machine from tunnelling players through walls.

### Wire and format constants a rebuild cannot choose

| Thing | `WinQuake` | `QW` |
|---|---|---|
| Game protocol version | 15 | 28 |
| Connection protocol version | 3 | text handshake |
| Program bytecode version | 6 | 6 |
| BSP version | 29 | 29 |
| Model version / magic | 6 / `IDPO` | 6 / `IDPO` |
| Sprite version / magic | 1 / `IDSP` | 1 / `IDSP` |
| Archive magic | `PACK` | `PACK` |
| Max reliable message | 8000 bytes | 1450 bytes |
| Max unreliable message | 1024 bytes | 1450 bytes |
| Max entities per level | 600 | 768 |
| Max models / sounds per level | 256 / 256 | 256 / 256 |
| Max players | 16 | 32 |
| Client stat slots | 32 | 32 |
| Default server port | 26000 | 27500 |
| Snapshot history depth | none | 64 frames |
| Coordinate quantization | 1/8 unit, 16-bit | 1/8 unit, 16-bit |
| Angle quantization | 360/256 per byte | 360/256 per byte, 16-bit for view |

### Benchmarks with numbers

- The software renderer without the assembly inner loops runs at roughly half the
  speed of the version with them (the release notes state this as "almost half").
  A rebuild has met the bar if its portable path is within a small factor of its
  accelerated path, because the accelerated path is optional.
- The hardware renderer is "not effected very much" by losing the assembly, which
  tells a rebuilder where the time actually goes: in the hardware path, almost
  none of it is in these loops.
- The engine ships a timed-demo command that reports frames and seconds; it is
  the original's own benchmark and the natural one to compare against.

---

## 7. Build order

Read in this order. Each chapter's vocabulary is defined before the chapter that
uses it.

| # | Chapter | Role |
|---|---|---|
| 1 | [`WinQuake/`](WinQuake/README.md) | The original engine, complete: foundation, asset formats, interpreter, server, client, both renderers, and every platform backend. |
| 2 | [`WinQuake/gas2masm/`](WinQuake/gas2masm/README.md) | The build-time tool that lets one assembly source serve three assemblers. |
| 3 | [`QW/server/`](QW/server/README.md) | The dedicated server re-cut for the internet: delta snapshots, per-client rate limiting, authoritative movement. |
| 4 | [`QW/client/`](QW/client/README.md) | The matching client: prediction, snapshot interpolation, a reliable channel, and the same two renderers. |
| 5 | [`QW/qwfwd/`](QW/qwfwd/README.md) | A datagram forwarder for servers behind a firewall. |
| 6 | [`QW/gas2masm/`](QW/gas2masm/README.md) | The same build tool, forked. |
| 7 | [`qw-qc/`](qw-qc/README.md) | The game itself, in the interpreted language — weapons, monsters, doors, items, rules. |
| 8 | [`QW/progs/`](QW/progs/README.md) | The build copy of chapter 7. |

Within [`WinQuake/`](WinQuake/README.md) and [`QW/client/`](QW/client/README.md)
the source tree is flat — two hundred files in one directory — so each chapter
README imposes the reading order the directory does not: foundation, then
formats, then interpreter, then world and server, then network, then models, then
the software renderer, then the hardware renderer, then the client, then sound,
then the platform backends, then the host loop that drives all of it.

---

See [`GLOSSARY.md`](GLOSSARY.md) for the domain vocabulary the twins use bare,
and [`README.md`](README.md) for the argument and the table of contents.
