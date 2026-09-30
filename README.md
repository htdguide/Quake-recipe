# Quake — a recipe

> Recipe of [`id-Software/Quake`](https://github.com/id-Software/Quake) at `bf4ac42`, 2012-01-31.
> A mirrored tree in which every source file is replaced by a page saying what it decides, promises and computes — enough to rebuild the
> engine in a language this recipe never mentions.

This is not documentation of the source. It is the source with the code taken out and the reasoning left in. Read it as a book: this page
is the argument and the table of contents, [`SYSTEM-REQUIREMENTS.md`](SYSTEM-REQUIREMENTS.md) is the preface, each directory's
`README.md` opens a chapter, and each twin is a page. **491 twins**, ten chapter openings, and these three pages.

---

## The argument

Quake is two engines in one repository, built two years apart, solving the same problem twice.

**The first** ([`WinQuake/`](WinQuake/README.md)) is one process holding a server, a client and two renderers, which talk to each other
through a network interface whose first driver is a pair of byte rings. Single player is a network game against a local server. That one
decision — *there is only one game loop to get right* — is the engine's most consequential and least visible, and everything from the
loopback driver to the null video backend exists to protect it.

Its technical centre is a **software rasterizer with no depth buffer for the world**: surfaces are clipped into edges, edges are sorted per
scanline into spans, and each span is textured by dividing once every eight pixels. Around that sit a surface cache that bakes lighting
into mipped copies of each surface and evicts them when a light moves, three precompiled collision hulls at hard-coded body sizes, a
62-operation interpreter for all game rules, and one heap divided among three allocators that never returns memory to the operating
system.

**The second** ([`QW/`](QW/README.md)) is the same game rebuilt for a link with latency and loss. Almost none of the simulation changed;
almost all of the networking did, and it did so through three ideas that every networked game since has used:

1. **One packet per frame, and nothing ever waits.** A packet carries the current unreliable state; the reliable stream rides along when
   there is room; a lost packet is superseded rather than retransmitted; acknowledgement is one bit in the next header. A byte budget paces
   the sender to the link. ([`QW/client/net_chan.c`](QW/client/net_chan.c.md))
2. **Every update is a difference from the last snapshot the receiver confirmed.** Not from a fixed baseline — from something the client
   acknowledged. A stationary entity then costs nothing and no update needs to be reliable at all.
   ([`QW/server/sv_ents.c`](QW/server/sv_ents.c.md))
3. **Player movement is a pure function, compiled into both programs.** So the client replays its own unacknowledged commands from the
   last confirmed state, and a correction needs no reconciliation step — the replay simply starts later.
   ([`QW/client/pmove.c`](QW/client/pmove.c.md), [`QW/client/cl_pred.c`](QW/client/cl_pred.c.md))

And **the rules of the game itself** ([`qw-qc/`](qw-qc/README.md)) are ten thousand lines in a language with arithmetic, comparison and
one timer per entity — which is why animation and behaviour are the same mechanism there, and why every delayed action is a spawned
entity.

**Why the pairing is worth reading as one book.** The two engines are a controlled experiment: same simulation, same content, same
renderers, one variable changed. Reading the first tells you how a 1996 game worked; reading the delta tells you what the internet cost,
and the delta is small, specific, and still correct thirty years later.

---

## Where to start

You do not have to read 491 twins. Four routes:

**If you want to rebuild the engine** — [`SYSTEM-REQUIREMENTS.md`](SYSTEM-REQUIREMENTS.md), then the build order below in full.

**If you want the networking** — [`QW/README.md`](QW/README.md), then
[`QW/client/bothdefs.h`](QW/client/bothdefs.h.md) (the 1450-byte message that everything follows from),
[`QW/client/net_chan.c`](QW/client/net_chan.c.md), [`QW/server/sv_ents.c`](QW/server/sv_ents.c.md),
[`QW/client/pmove.h`](QW/client/pmove.h.md), [`QW/client/cl_pred.c`](QW/client/cl_pred.c.md),
[`QW/client/cl_ents.c`](QW/client/cl_ents.c.md) — and then
[`QW/server/newnet.txt`](QW/server/newnet.txt.md), the author's own design notes, which show the order the problems arrive in.

**If you want the renderer** — [`WinQuake/README.md`](WinQuake/README.md) section 6, then
[`WinQuake/r_edge.c`](WinQuake/r_edge.c.md), [`WinQuake/r_surf.c`](WinQuake/r_surf.c.md),
[`WinQuake/d_scan.c`](WinQuake/d_scan.c.md), [`WinQuake/d_surf.c`](WinQuake/d_surf.c.md) — then section 7 for how the same game looks
when a graphics library takes over, especially [`WinQuake/gl_rsurf.c`](WinQuake/gl_rsurf.c.md) and
[`WinQuake/gl_rmain.c`](WinQuake/gl_rmain.c.md).

**If you want the game design** — [`qw-qc/README.md`](qw-qc/README.md), then
[`qw-qc/combat.qc`](qw-qc/combat.qc.md) and [`qw-qc/items.qc`](qw-qc/items.qc.md). Those two files are the deathmatch.

Terms are defined once in [`GLOSSARY.md`](GLOSSARY.md) so the twins can be terse.

---

## The tree

```
Quake-recipe/
├── README.md                    this page
├── SYSTEM-REQUIREMENTS.md       tier, seams, platform, formats, conformance, build order
├── GLOSSARY.md                  the vocabulary, once
├── readme.txt.md  gnu.txt.md    the source release's own notes — 2
├── WinQuake/                    the original engine — 230
│   └── gas2masm/                the assembly syntax translator — 1
├── QW/                          QuakeWorld — 14 build, changelog and release-note twins
│   ├── client/                  the client program — 183
│   ├── server/                  the dedicated server — 32
│   ├── progs/                   a build copy; points at qw-qc/
│   ├── qwfwd/                   a datagram forwarder — 3
│   ├── gas2masm/                the translator, carried forward — 2
│   └── docs/                    the release notes — 4
└── qw-qc/                       the game logic — 20
```

---

## Build order

The chapters in dependency order. Each is a link to its opening page.

| | Chapter | Role |
|---|---|---|
| 1 | [`WinQuake/`](WinQuake/README.md) §1–2 | **Foundation and formats** — one heap and three allocators, the wire encoders, the command interpreter, and the four file formats the content already is |
| 2 | [`WinQuake/`](WinQuake/README.md) §3 | **The model loader** — three precompiled collision hulls at hard-coded body sizes, visibility, texture animation |
| 3 | [`WinQuake/`](WinQuake/README.md) §4 | **The server** — the game interpreter, collision, physics, authority |
| 4 | [`WinQuake/`](WinQuake/README.md) §5–6 | **The client and the software renderer** — the chapter's technical core |
| 5 | [`WinQuake/`](WinQuake/README.md) §7 | **The hardware renderer** — a drop-in substitution for chapter 4's renderer |
| 6 | [`WinQuake/`](WinQuake/README.md) §8–13 | **Interface, sound, networking, host, platform backends, assembly** |
| 7 | [`QW/server/`](QW/server/README.md) | **The dedicated server** — snapshots, pacing, admission control |
| 8 | [`QW/client/`](QW/client/README.md) | **The client** — prediction, delta decompression, the staged join |
| 9 | [`qw-qc/`](qw-qc/README.md) | **The game logic** — what the rules actually are |
| 10 | [`QW/qwfwd/`](QW/qwfwd/README.md), [`QW/docs/`](QW/docs/README.md), [`QW/gas2masm/`](QW/gas2masm/README.md) | **Auxiliaries** — a relay, the release notes read as acceptance criteria, the translator |

A rebuilder who wants a playable result soonest can take 1 → 2 → 3 → 4 and stop: that is a single-player engine. Adding 7 → 8 makes it
networked. Chapter 5 is optional and chapter 6 is mostly seams.

---

## Cycles

Dependency cycles are facts about the design, not defects in the recipe. Four cross chapters; each chapter README lists its own.

- **The host ↔ the client ↔ the server** ([`WinQuake/host.c`](WinQuake/host.c.md)). The host drives both halves and both call back into it
  for printing, errors and timing. Broken by reading the host last.
- **The loader ↔ the renderer** ([`WinQuake/model.c`](WinQuake/model.c.md)). The loader fills records the renderer defines. Broken at
  [`WinQuake/model.h`](WinQuake/model.h.md).
- **Physics ↔ the engine operation table ↔ the game logic**
  ([`WinQuake/sv_phys.c`](WinQuake/sv_phys.c.md) ↔ [`WinQuake/pr_cmds.c`](WinQuake/pr_cmds.c.md) ↔ [`qw-qc/`](qw-qc/README.md)). Physics
  calls the game's handlers; the game asks the engine to move things. Broken at the interpreter.
- **Prediction ↔ entity reception** ([`QW/client/cl_pred.c`](QW/client/cl_pred.c.md) ↔
  [`QW/client/cl_ents.c`](QW/client/cl_ents.c.md)). Prediction needs other players made solid; making them solid needs them extrapolated;
  extrapolating them runs the mover prediction uses. The source resolves it by extrapolating twice, once without predicting.

One cycle is worth singling out because a rebuild will meet it early and it is not obvious:

- **The video backend sizes the surface cache using a rule the renderer owns**
  ([`WinQuake/vid_win.c`](WinQuake/vid_win.c.md) ↔ [`WinQuake/d_surf.c`](WinQuake/d_surf.c.md)), because a display mode must be refused
  *before* it is entered if its three buffers will not fit. Broken at
  [`WinQuake/vid_dos.h`](WinQuake/vid_dos.h.md), which declares the rule.

---

## What this recipe is honest about

Four things a rebuilder should not inherit, each recorded at the page where it lives:

- **Remote administration sends its password in the clear** on every command
  ([`QW/server/sv_main.c`](QW/server/sv_main.c.md)), and the client stores it in a settings file
  ([`QW/client/cl_main.c`](QW/client/cl_main.c.md)). Redesign it.
- **The channel is unauthenticated** ([`QW/client/net_chan.c`](QW/client/net_chan.c.md)); the source-address and sequence-window checks
  stop spoofing by a stranger and nothing by an observer.
- **The digest function is broken** ([`QW/client/md4.c`](QW/client/md4.c.md)), so the content comparison built on it can be defeated
  deliberately. The self-reported model checksums ([`QW/server/sv_init.c`](QW/server/sv_init.c.md)) and the movement checksum
  ([`QW/server/sv_user.c`](QW/server/sv_user.c.md)) deter casual tampering and prove nothing.
- **A server can run console commands on a connected client** ([`QW/client/cl_main.c`](QW/client/cl_main.c.md)). In 1996 that was a
  feature; scope it deliberately.

And three places where the source itself says the shipped behaviour is not the intended behaviour —
[`QW/client/docs.txt`](QW/client/docs.txt.md) lists the known defects,
[`QW/server/newnet.txt`](QW/server/newnet.txt.md) lists what was planned and not done, and
[`QW/server/notes.txt`](QW/server/notes.txt.md) describes a scheme that was designed and never built. Those three pages are worth more
than any inference from the code.

---

## What is not twinned

Of the source tree's **626 files, 135 are skipped** and the remaining 491 each have a twin at the same path with `.md` appended.
(The one exception to the path rule: `QW/progs/*` is twinned under [`qw-qc/`](qw-qc/README.md), because the sources are a chapter of their
own and that directory is only a build copy.)

The skip list, by category:

- **Vendored third-party trees** — `WinQuake/dxsdk/`, `WinQuake/scitech/`.
- **The packaging kit** — `WinQuake/kit/`.
- **Shipped content and documentation** — `WinQuake/data/`, `WinQuake/docs/`.
- **Editor and project files** — `*.dsp`, `*.dsw`, `*.mdp`, `*.mak`, `*.ncb`, `*.opt`, `*.plg`, `*.aps`, and the `.001` backups (two of
  which are recorded anyway, as curiosities).
- **Compiler-invoking batch files** — `*.bat`.
- **Binary assets** — icons, bitmaps, `cwsdpmi.exe`.
- **One compiled binary** — `QW/progs/qwprogs.dat`, whose format is described by
  [`QW/server/pr_comp.h`](QW/server/pr_comp.h.md) and [`QW/server/progs.h`](QW/server/progs.h.md).

`files.dat` is *not* skipped despite its extension: it is a generated text manifest and
[`qw-qc/files.dat`](qw-qc/files.dat.md) records why it is worth a page.
