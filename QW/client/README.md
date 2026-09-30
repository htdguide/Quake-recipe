# QW/client — the client program

> Sends one command per frame, predicts its own movement from the commands the server has not answered yet, decompresses everyone else's world from the last snapshot it confirmed, and draws the result a fraction of a second in the past.

Read [`../README.md`](../README.md) for the three ideas, and
[`../../WinQuake/README.md`](../../WinQuake/README.md) sections 6–9 for the renderers, the sound and the interface, which are almost
unchanged. Of the 183 twins here, about thirty carry new content and the rest are pointers with a **What differs** section.

This directory also holds the files the *server* compiles: the channel, the transport, the mover and the foundation
([`../server/makefile`](../server/makefile.md) lists them). That is why they live here rather than in a shared directory, and it is the
one piece of the source's organization a rebuild should change.

## Read in this order

**1. The shared contract**
[`bothdefs.h`](bothdefs.h.md) — the constants both programs must agree on, and **the 1450-byte message limit** chosen so a packet is
never fragmented by the network. Every other networking decision follows from that number.
[`protocol.h`](protocol.h.md) — version 28. What was added, what was retired, and why a retired message number can never be reused.
[`common.h`](common.h.md) · [`common.c`](common.c.md) — the foundation, plus **the information dictionary** (unordered,
absent-means-default, a reserved namespace for the authority's own assertions, a hard length limit) and the command delta encoder.
[`quakedef.h`](quakedef.h.md) — the client's umbrella, whose include list **cannot reach the server's data structures.** That absence is
what made the split real.

**2. The channel**
[`net.h`](net.h.md) — eleven files became three, because the transports the original abstracted no longer exist.
[`net_chan.c`](net_chan.c.md) — **the most important file in the chapter.** One packet per frame, a reliable stream acknowledged by one
alternating bit, a byte budget as a virtual clock, a connection identifier that survives address translation, and an out-of-band form
that makes connecting stateless.
[`net_udp.c`](net_udp.c.md) · [`net_wins.c`](net_wins.c.md) — the transport, reduced.

**3. The shared mover** — the one piece of simulation the client holds.
[`pmove.h`](pmove.h.md) — the interface whose whole design principle is that **movement is a pure function of a state, a world and a
command**. A flat list of at most 32 solid things, an opaque tag per entry, and the tunables as a record the server sends.
[`pmove.c`](pmove.c.md) — acceleration under friction, the slide move, the stair step chosen by whichever went further, immersion by
depth, and **the air-acceleration clamp on the projected component** that produced a movement technique nobody designed. Also the grid
snap that makes prediction stable.
[`pmovetst.c`](pmovetst.c.md) — the three collision queries, over the map's own precompiled hulls.

**4. The client's state and its loop**
[`client.h`](client.h.md) — **one ring indexed by packet sequence**, holding the command sent and the state received. Prediction, delta
decompression, latency measurement and the diagnostic display all read it. Design that ring first and the rest falls out.
[`cl_input.c`](cl_input.c.md) — build one command per frame, **quantize it to the wire before storing it**, and send it with the two
before it plus a checksum.
[`cl_pred.c`](cl_pred.c.md) — replay every unacknowledged command from the last confirmed state, then interpolate to the displayed
moment. Two hundred lines, and **no explicit reconciliation step**.
[`cl_ents.c`](cl_ents.c.md) — apply each snapshot as a difference, carry unmentioned entities forward, stamp each player's state with
*when it was valid*, and **under-extrapolate deliberately** so corrections read as forward motion.
[`cl_parse.c`](cl_parse.c.md) — the staged, resumable, index-driven join, and the block-by-block download with its path check.
[`cl_main.c`](cl_main.c.md) — the frame loop (**the frame limiter is a network control**), the idempotent connection retry, and a fatal
error that returns to the console instead of exiting.
[`cl_demo.c`](cl_demo.c.md) — recording: **record the inputs as well as the outputs**, and make the stream reference-free.

**5. What QuakeWorld added to the client**
[`cl_cam.c`](cl_cam.c.md) — the spectator camera: sample-and-score placement using the mover's own collision queries, and the fact that
spectating properly needs the server to know who is watching whom.
[`skin.c`](skin.c.md) — content named by a peer, fetched from the server, shared by name, cached evictably, **sanitized at the
boundary**, one item at a time.
[`md4.c`](md4.c.md) — a digest. ***Do not use it in a rebuild***; the twin says why.
[`gl_ngraph.c`](gl_ngraph.c.md) — the packet graph, which distinguishes *lost* from *choked*. Keep that distinction.
[`buildnum.c`](buildnum.c.md) — a version number that cannot be forgotten.

**6. The author's notes**
[`docs.txt`](docs.txt.md) — the connection sequence explained, and **a list of known defects written by the author**. Worth more than
any amount of inference: each item marks where the shipped behaviour is not the intended behaviour.
[`notes.txt`](notes.txt.md) — a threaded sender and receiver, **designed and abandoned**. Read it for why tying packets to frames is
load-bearing.
[`exitscrn.txt`](exitscrn.txt.md) — and the reminder that this shipped as an unsupported release.

## The renderers, the interface and the platform — pointers to the original

Unchanged in substance. Read [`../../WinQuake/README.md`](../../WinQuake/README.md) sections 6–9 and 12–13, then these twins for the
deltas — which are almost entirely *the view comes from the prediction* and *the entity list is rebuilt each frame*.

**Software renderer** — [`render.h`](render.h.md) (the client-renderer contract, with the fields the networking added) ·
[`r_main.c`](r_main.c.md) · [`r_bsp.c`](r_bsp.c.md) · [`r_draw.c`](r_draw.c.md) · [`r_edge.c`](r_edge.c.md) ·
[`r_surf.c`](r_surf.c.md) · [`r_alias.c`](r_alias.c.md) · [`r_aclip.c`](r_aclip.c.md) · [`r_sprite.c`](r_sprite.c.md) ·
[`r_part.c`](r_part.c.md) (which also carries the software build's packet graph) · [`r_light.c`](r_light.c.md) ·
[`r_sky.c`](r_sky.c.md) · [`r_efrag.c`](r_efrag.c.md) · [`r_misc.c`](r_misc.c.md) · [`r_local.h`](r_local.h.md) ·
[`r_shared.h`](r_shared.h.md) · [`r_vars.c`](r_vars.c.md) · and the rasterizer:
[`d_scan.c`](d_scan.c.md) · [`d_edge.c`](d_edge.c.md) · [`d_surf.c`](d_surf.c.md) · [`d_polyse.c`](d_polyse.c.md) ·
[`d_sprite.c`](d_sprite.c.md) · [`d_part.c`](d_part.c.md) · [`d_sky.c`](d_sky.c.md) · [`d_init.c`](d_init.c.md) ·
[`d_fill.c`](d_fill.c.md) · [`d_zpoint.c`](d_zpoint.c.md) · [`d_modech.c`](d_modech.c.md) · [`d_copy.s`](d_copy.s.md) ·
[`d_vars.c`](d_vars.c.md) · [`d_local.h`](d_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`block8.h`](block8.h.md) ·
[`block16.h`](block16.h.md).

**Hardware renderer** — [`glquake.h`](glquake.h.md) · [`gl_rmain.c`](gl_rmain.c.md) · [`gl_rsurf.c`](gl_rsurf.c.md) ·
[`gl_draw.c`](gl_draw.c.md) · [`gl_screen.c`](gl_screen.c.md) · [`gl_model.c`](gl_model.c.md) · [`gl_model.h`](gl_model.h.md) ·
[`gl_mesh.c`](gl_mesh.c.md) · [`gl_warp.c`](gl_warp.c.md) · [`gl_rlight.c`](gl_rlight.c.md) · [`gl_rmisc.c`](gl_rmisc.c.md) ·
[`gl_refrag.c`](gl_refrag.c.md) · [`gl_warp_sin.h`](gl_warp_sin.h.md) · [`glquake2.h`](glquake2.h.md) ·
[`gl_test.c`](gl_test.c.md).

**Model and formats** — [`model.h`](model.h.md) · [`model.c`](model.c.md) · [`bspfile.h`](bspfile.h.md) ·
[`modelgen.h`](modelgen.h.md) · [`spritegn.h`](spritegn.h.md) · [`wad.h`](wad.h.md) · [`wad.c`](wad.c.md).

**Interface** — [`vid.h`](vid.h.md) · [`draw.h`](draw.h.md) · [`draw.c`](draw.c.md) · [`screen.h`](screen.h.md) ·
[`screen.c`](screen.c.md) · [`console.h`](console.h.md) · [`console.c`](console.c.md) · [`keys.h`](keys.h.md) ·
[`keys.c`](keys.c.md) · [`menu.h`](menu.h.md) · [`menu.c`](menu.c.md) (**half of it deleted with the networking it configured**) ·
[`sbar.h`](sbar.h.md) · [`sbar.c`](sbar.c.md) (the scoreboards, built entirely from data the networking already kept) ·
[`view.h`](view.h.md) · [`view.c`](view.c.md) (**anything that decays predictably should be triggered, not transmitted**) ·
[`cl_tent.c`](cl_tent.c.md) · [`input.h`](input.h.md).

**Foundation** — [`cmd.h`](cmd.h.md) · [`cmd.c`](cmd.c.md) (**a command declares where it may be issued from** — the boundary that stops
a remote party reaching an engine command) · [`cvar.h`](cvar.h.md) · [`cvar.c`](cvar.c.md) (settings projected into the dictionaries) ·
[`crc.h`](crc.h.md) · [`crc.c`](crc.c.md) · [`zone.h`](zone.h.md) · [`zone.c`](zone.c.md) (**the memory model survived the rewrite
untouched**) · [`mathlib.h`](mathlib.h.md) · [`mathlib.c`](mathlib.c.md) · [`nonintel.c`](nonintel.c.md) ·
[`adivtab.h`](adivtab.h.md) · [`anorms.h`](anorms.h.md) · [`anorm_dots.h`](anorm_dots.h.md).

**Sound** — [`sound.h`](sound.h.md) · [`snd_dma.c`](snd_dma.c.md) · [`snd_mix.c`](snd_mix.c.md) · [`snd_mem.c`](snd_mem.c.md) ·
[`snd_win.c`](snd_win.c.md) · [`snd_linux.c`](snd_linux.c.md) · [`cdaudio.h`](cdaudio.h.md) · [`cd_win.c`](cd_win.c.md) ·
[`cd_linux.c`](cd_linux.c.md) · [`cd_audio.c`](cd_audio.c.md) · [`cd_null.c`](cd_null.c.md).

**Platform** — [`sys.h`](sys.h.md) · [`sys_win.c`](sys_win.c.md) · [`sys_linux.c`](sys_linux.c.md) ·
[`sys_null.c`](sys_null.c.md) · [`winquake.h`](winquake.h.md) · [`resource.h`](resource.h.md) · [`winquake.rc`](winquake.rc.md) ·
[`vid_win.c`](vid_win.c.md) · [`vid_x.c`](vid_x.c.md) · [`vid_svgalib.c`](vid_svgalib.c.md) · [`vid_null.c`](vid_null.c.md) ·
[`gl_vidnt.c`](gl_vidnt.c.md) · [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md) · [`gl_vidlinux.c`](gl_vidlinux.c.md) ·
[`gl_vidlinux_x11.c`](gl_vidlinux_x11.c.md) · [`gl_vidlinux_svga.c`](gl_vidlinux_svga.c.md) (the last two are checked-in renames) ·
[`in_win.c`](in_win.c.md) · [`in_null.c`](in_null.c.md).

**Assembly** — the layout headers [`asm_i386.h`](asm_i386.h.md) · [`asm_draw.h`](asm_draw.h.md) · [`d_ifacea.h`](d_ifacea.h.md) ·
[`quakeasm.h`](quakeasm.h.md), the routines [`d_draw.s`](d_draw.s.md) and its siblings, **and a checked-in hand-maintained copy of each
in the other assembler's syntax** ([`d_draw.asm`](d_draw.asm.md) and fifteen more) — which is the duplication
[`../gas2masm/`](../gas2masm/README.md) exists to prevent.

**Build** — [`makefile.svgalib`](makefile.svgalib.md), whose object list is the definition of the client.

## Cycles

- **[`cl_main.c`](cl_main.c.md) ↔ [`cl_parse.c`](cl_parse.c.md) ↔ [`cl_ents.c`](cl_ents.c.md).** The loop reads packets, the parser
  fills the ring, the ring feeds the loop. Broken by reading [`client.h`](client.h.md) first.
- **[`cl_pred.c`](cl_pred.c.md) ↔ [`cl_ents.c`](cl_ents.c.md).** Prediction needs the other players made solid; making them solid needs
  them extrapolated; extrapolating them runs the mover prediction uses. The source resolves it by calling the extrapolation twice, once
  without predicting — the ordering is stated in both twins.
- **[`model.c`](model.c.md) ↔ the renderer**, as in the original.

## Not twinned

The editor and project files (`qwcl.dsp`, `qwcl.dsw`, `qwcl.mak`, `qwcl.mdp`, `qwcl.plg`, `winquake.aps`), the batch file
(`q.bat`), and the binary assets (`quakeworld.bmp`, `qwcl2.ico`, `qe3.ico`).
