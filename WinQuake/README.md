# WinQuake — the original engine

> One process containing a server, a client and two renderers, sharing a single heap and talking to itself through a loopback network driver.

This is the whole of the 1996 engine: a flat directory of about two hundred files that build, depending on which ones the build
selects, a software-rendered game, a hardware-rendered game, or a dedicated server. The directory is flat because the source is flat;
this page imposes the reading order it lacks.

**Read [`../SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md) first** — the tier, the eleven seams and the wire constants it fixes
change how every page below should be read.

The single most important structural fact: **the client and the server are separate subsystems in one process, connected by a network
interface whose first driver is a pair of byte rings** ([`net_loop.c`](net_loop.c.md)). Single player is a network game against a local
server. Everything else follows from that.

---

## 1. Foundation — the vocabulary everything else uses

Read these first and in this order. Nothing here knows about the game.

| | |
|---|---|
| [`bothdefs.h`](../QW/client/bothdefs.h.md) has no counterpart here; the constants live in [`quakedef.h`](quakedef.h.md) | the umbrella header, the limits, the axis order |
| [`sys.h`](sys.h.md) | the platform interface: files, the clock, printing, the fatal error |
| [`zone.h`](zone.h.md) · [`zone.c`](zone.c.md) | **one heap, three allocators**: a double-ended stack, a small first-fit heap, an evictable cache in the gap |
| [`common.h`](common.h.md) · [`common.c`](common.c.md) | byte order, the wire encoders, growable buffers, the tokenizer, the archive search path |
| [`crc.h`](crc.h.md) · [`crc.c`](crc.c.md) | the checksum |
| [`mathlib.h`](mathlib.h.md) · [`mathlib.c`](mathlib.c.md) | vectors, angles, the box-against-plane test |
| [`cvar.h`](cvar.h.md) · [`cvar.c`](cvar.c.md) | named settings |
| [`cmd.h`](cmd.h.md) · [`cmd.c`](cmd.c.md) | the deferred command interpreter — the engine's whole configuration language |
| [`wad.h`](wad.h.md) · [`wad.c`](wad.c.md) | the interface-image archive |
| [`nonintel.c`](nonintel.c.md) · [`adivtab.h`](adivtab.h.md) · [`anorms.h`](anorms.h.md) · [`anorm_dots.h`](anorm_dots.h.md) | portable arithmetic and three generated tables |

## 2. Data formats — what the content already is

The formats are fixed by the tools that produced the content, so they are constraints rather than decisions.

[`bspfile.h`](bspfile.h.md) — the map: planes, nodes, leaves, faces, **three precompiled collision hulls**, run-length-encoded
visibility, baked lightmaps on a sixteen-unit grid.
[`modelgen.h`](modelgen.h.md) — animated models, with normals quantized to 162 directions.
[`spritegn.h`](spritegn.h.md) — sprites and their four orientation types.
[`protocol.h`](protocol.h.md) — the wire: **version 15**, coordinates at an eighth of a unit, angles at a 256th of a turn.

## 3. The model loader — turning files into the runtime's records

[`model.h`](model.h.md) · [`model.c`](model.c.md) — the single most consequential file in the chapter after the renderer. It builds the
three collision hulls **at hard-coded body sizes**, fabricates the missing point-sized hull, decompresses visibility on demand, and
follows the naming convention that makes texture animation work. The hard-coded sizes are an unwritten contract with the map compiler.

## 4. The server — simulation and authority

Read in this order; each depends on the ones before.

[`progs.h`](progs.h.md) · [`pr_comp.h`](pr_comp.h.md) · [`progdefs.h`](progdefs.h.md) — the compiled game language's format and the
engine-game record contract.
[`pr_exec.c`](pr_exec.c.md) — **the interpreter**: 62 three-address operations over a flat slot pool, entity references as byte
offsets, a runaway instruction bound, and a hard-coded animation interval.
[`pr_edict.c`](pr_edict.c.md) — loading a compiled program, the entity pool, the map's entity text.
[`pr_cmds.c`](pr_cmds.c.md) — the numbered, append-only table of everything the game may ask the engine to do.
[`world.h`](world.h.md) · [`world.c`](world.c.md) — **collision**: a hand-built spatial tree, every box sweep reduced to a point sweep
by expanding the obstacle, a fabricated six-node box tree, and the back-off epsilon.
[`sv_phys.c`](sv_phys.c.md) — movement types, the slide move, pushers that carry riders, think scheduling.
[`sv_move.c`](sv_move.c.md) — monster locomotion.
[`sv_user.c`](sv_user.c.md) — the player's movement, inside the server and unreachable by the client. *This is what QuakeWorld
extracted.*
[`server.h`](server.h.md) · [`sv_main.c`](sv_main.c.md) — the server's state, the baselines, the per-frame entity updates, the client
connection.

## 5. The client — receiving and presenting

[`client.h`](client.h.md) · [`cl_main.c`](cl_main.c.md) — the client's state and its frame.
[`cl_parse.c`](cl_parse.c.md) — reading the server's stream.
[`cl_input.c`](cl_input.c.md) — the key-state machine and the command sent to the server.
[`cl_tent.c`](cl_tent.c.md) — temporary visual effects.
[`cl_demo.c`](cl_demo.c.md) — recording and replay, which is also the engine's benchmark harness.
[`view.c`](view.c.md) · [`view.h`](view.h.md) — the camera: bob, roll, damage tints, the view punch.
[`chase.c`](chase.c.md) — the third-person camera.
[`r_efrag.c`](r_efrag.c.md) — registering entities into the world's leaves so the renderer finds them while walking.

## 6. The software renderer — the chapter's technical core

This is the part of the engine that is genuinely unlike anything else, and it is worth reading in full and in order.

**The design in one sentence:** the world is drawn by sorting edges into spans and texturing each span with one division every eight
pixels, so there is no depth buffer for the world and no overdraw at all.

[`render.h`](render.h.md) — the interface between the client and any renderer.
[`r_local.h`](r_local.h.md) · [`r_shared.h`](r_shared.h.md) · [`d_iface.h`](d_iface.h.md) · [`d_local.h`](d_local.h.md) — the private
declarations, and the boundary between the renderer and its rasterizer.
[`r_main.c`](r_main.c.md) — the frame: the view basis, the frustum, the pass order.
[`r_misc.c`](r_misc.c.md) · [`r_vars.c`](r_vars.c.md) · [`d_vars.c`](d_vars.c.md) · [`d_modech.c`](d_modech.c.md) — per-frame and
per-mode derivations, and the instrumentation.
[`r_bsp.c`](r_bsp.c.md) — the front-to-back tree walk.
[`r_draw.c`](r_draw.c.md) · [`r_edge.c`](r_edge.c.md) — clipping surfaces into edges, and **the edge sorter**.
[`r_surf.c`](r_surf.c.md) · [`d_surf.c`](d_surf.c.md) — **the surface cache**: a lit, mipped copy of each surface, rover-allocated,
invalidated on light-style, dynamic-light and texture-animation changes, with thrash detection reported to the player.
[`d_edge.c`](d_edge.c.md) · [`d_scan.c`](d_scan.c.md) — spans, and **the inner loops**: perspective-correct texturing by dividing once
every eight pixels, sixteen for liquids, thirty-two for sky.
[`d_sky.c`](d_sky.c.md) · [`d_init.c`](d_init.c.md) · [`d_fill.c`](d_fill.c.md) · [`d_zpoint.c`](d_zpoint.c.md) ·
[`block8.h`](block8.h.md) · [`block16.h`](block16.h.md) — the remaining span cases and their generated loop bodies.
[`r_alias.c`](r_alias.c.md) · [`r_aclip.c`](r_aclip.c.md) · [`d_polyse.c`](d_polyse.c.md) — animated models: the quantized-normal
lighting tables and the triangle rasterizer.
[`r_sprite.c`](r_sprite.c.md) · [`d_sprite.c`](d_sprite.c.md) — sprites.
[`r_part.c`](r_part.c.md) · [`d_part.c`](d_part.c.md) — particles.
[`r_light.c`](r_light.c.md) · [`r_sky.c`](r_sky.c.md) — light styles, dynamic-light marking, the point light sample, and the sky's
scroll.

## 7. The hardware renderer — the same game through a graphics library

A drop-in replacement for section 6: the build substitutes these files and nothing else changes
([`Makefile.linuxi386`](Makefile.linuxi386.md) lists the substitution, which is the clearest statement of the renderer boundary).

[`glquake.h`](glquake.h.md) — the private vocabulary, and **thirty console switches**, each corresponding to a real observed hardware
failure ([`glqnotes.txt`](glqnotes.txt.md), [`3dfx.txt`](3dfx.txt.md)).
[`gl_model.h`](gl_model.h.md) · [`gl_model.c`](gl_model.c.md) — the same formats, prepared differently: vertex lists built at load,
textures uploaded, skins flood-filled so filtering has something correct to blend with.
[`gl_rsurf.c`](gl_rsurf.c.md) — **lightmaps packed into pages by a skyline allocator**, drawn in texture order, lit in one pass with two
texture units or two passes with a blend.
[`gl_rmain.c`](gl_rmain.c.md) — the frame, the **depth-range partitioning** that gives a mirror and a view model their own slices, and
the quantized-normal model lighting.
[`gl_mesh.c`](gl_mesh.c.md) — strips and fans built once per model and shared by every pose.
[`gl_warp.c`](gl_warp.c.md) — sky and liquid: **grid-snapped subdivision** so adjacent surfaces distort without tearing.
[`gl_draw.c`](gl_draw.c.md) — the texture manager, the packed sheets, and the **pixel-exact two-dimensional space** that makes the whole
interface renderer-independent.
[`gl_screen.c`](gl_screen.c.md) · [`gl_rlight.c`](gl_rlight.c.md) · [`gl_rmisc.c`](gl_rmisc.c.md) ·
[`gl_refrag.c`](gl_refrag.c.md) · [`gl_warp_sin.h`](gl_warp_sin.h.md) · [`glquake2.h`](glquake2.h.md) ·
[`gl_test.c`](gl_test.c.md) — the remainder, including two files that are dead.

## 8. The interface — shared by both renderers

Everything here draws in the pixel-exact two-dimensional space and is identical in both builds.

[`vid.h`](vid.h.md) — the video interface: a buffer, a stride, a palette, a present.
[`draw.h`](draw.h.md) · [`draw.c`](draw.c.md) — characters, images, fills, the tiling background.
[`screen.h`](screen.h.md) · [`screen.c`](screen.c.md) — the frame composer, the view sizing, the loading plaque.
[`console.h`](console.h.md) · [`console.c`](console.c.md) — the console and the notification overlay.
[`keys.h`](keys.h.md) · [`keys.c`](keys.c.md) — key names, bindings and input routing.
[`menu.h`](menu.h.md) · [`menu.c`](menu.c.md) — the menus, half of which configure networking.
[`sbar.h`](sbar.h.md) · [`sbar.c`](sbar.c.md) — the status bar.
[`input.h`](input.h.md) — the input interface: four operations.

## 9. Sound

[`sound.h`](sound.h.md) · [`snd_dma.c`](snd_dma.c.md) — channels, spatialization, and the ring buffer the mixer fills ahead of the
hardware.
[`snd_mix.c`](snd_mix.c.md) · [`snd_mem.c`](snd_mem.c.md) — the mixer and the sample loader.
[`cdaudio.h`](cdaudio.h.md) — the music interface, whose absence must be silent.

## 10. Networking

[`net.h`](net.h.md) — **a two-level driver abstraction**: message drivers over transport drivers, each a table of operations.
[`net_main.c`](net_main.c.md) — the dispatcher, and the timer queue that exists for one purpose.
[`net_loop.c`](net_loop.c.md) · [`net_loop.h`](net_loop.h.md) — **the loopback driver**, which is why single player is a network game.
[`net_dgrm.c`](net_dgrm.c.md) · [`net_dgrm.h`](net_dgrm.h.md) — the reliability layer: one outstanding reliable message per direction,
fragmentation, timer-driven retransmission. *This is what QuakeWorld replaced.*
[`net_ser.c`](net_ser.c.md) · [`net_ser.h`](net_ser.h.md) · [`net_comx.c`](net_comx.c.md) — the serial driver: the same service over a
byte stream, which is the framing problem stated cleanly.
[`net_udp.c`](net_udp.c.md) · [`net_udp.h`](net_udp.h.md) — **the reference transport**, including the partial-address rule.
[`net_wins.c`](net_wins.c.md) · [`net_wipx.c`](net_wipx.c.md) · [`net_ipx.c`](net_ipx.c.md) · [`net_bw.c`](net_bw.c.md) ·
[`net_mp.c`](net_mp.c.md) and their headers — five more transports over families that share nothing, which is the evidence the interface
is genuinely portable.
[`net_vcr.c`](net_vcr.c.md) · [`net_vcr.h`](net_vcr.h.md) — the replay driver, the engine's only determinism tool.
[`net_bsd.c`](net_bsd.c.md) · [`net_win.c`](net_win.c.md) · [`net_dos.c`](net_dos.c.md) · [`net_none.c`](net_none.c.md) — the driver
tables, which are the build's transport selection.
[`mpdosock.h`](mpdosock.h.md) · [`mplib.c`](mplib.c.md) · [`mplpc.c`](mplpc.c.md) — a commercial matchmaking service entering the engine
*as a transport driver*.
[`net_wso.c`](net_wso.c.md) — empty.

## 11. The host — what arbitrates between the two halves

[`host.c`](host.c.md) — the frame: run the server, then the client, then the renderer. The initialization order, the error recovery, the
frame-time clamp.
[`host_cmd.c`](host_cmd.c.md) — the commands: level changes, saved games, the connection.
[`conproc.h`](conproc.h.md) · [`conproc.c`](conproc.c.md) — the shared-memory channel an external program may drive a dedicated server
through.

## 12. Platform backends — the seams, filled

Each group is one interface filled several times. Read one of each and skim the rest.

**System** — [`sys_win.c`](sys_win.c.md) (the reference) · [`sys_dos.c`](sys_dos.c.md) (no operating system) ·
[`sys_linux.c`](sys_linux.c.md) (the shortest) · [`sys_sun.c`](sys_sun.c.md) · [`sys_wind.c`](sys_wind.c.md) (dedicated) ·
[`sys_null.c`](sys_null.c.md) (**the checklist**).

**Video, software** — [`vid_win.c`](vid_win.c.md) (the reference: lock semantics, deferred palette and mode changes, the
dirty-rectangle-versus-page-flip conflict) · [`vid_dos.c`](vid_dos.c.md) + [`vid_dos.h`](vid_dos.h.md) +
[`vid_vga.c`](vid_vga.c.md) + [`vid_ext.c`](vid_ext.c.md) + [`vgamodes.h`](vgamodes.h.md) +
[`vregset.c`](vregset.c.md) + [`vregset.h`](vregset.h.md) (the vertical-blank rule and bank switching) ·
[`vid_x.c`](vid_x.c.md) (**an indexed renderer on a true-colour display, via one lookup per pixel**) ·
[`vid_svgalib.c`](vid_svgalib.c.md) · [`vid_sunx.c`](vid_sunx.c.md) · [`vid_sunxil.c`](vid_sunxil.c.md) (the engine's only real
threading) · [`vid_null.c`](vid_null.c.md).

**Video, hardware** — [`gl_vidnt.c`](gl_vidnt.c.md) (the reference) · [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md) (the pointer grab and
its hazard) · [`gl_vidlinux.c`](gl_vidlinux.c.md).

**Input** — [`in_win.c`](in_win.c.md) (the reference: accumulate then consume, the pitch clamp, the axis naming) ·
[`in_dos.c`](in_dos.c.md) · [`in_sun.c`](in_sun.c.md) · [`in_null.c`](in_null.c.md).

**Sound** — [`snd_win.c`](snd_win.c.md) · [`snd_dos.c`](snd_dos.c.md) · [`snd_gus.c`](snd_gus.c.md) ·
[`snd_linux.c`](snd_linux.c.md) · [`snd_sun.c`](snd_sun.c.md) · [`snd_next.c`](snd_next.c.md) ·
[`snd_null.c`](snd_null.c.md).

**Music** — [`cd_win.c`](cd_win.c.md) · [`cd_audio.c`](cd_audio.c.md) · [`cd_linux.c`](cd_linux.c.md) ·
[`cd_null.c`](cd_null.c.md).

**Shared platform declarations** — [`winquake.h`](winquake.h.md) (**one shared focus flag with four consequences**) ·
[`dosisms.h`](dosisms.h.md) · [`dos_v2.c`](dos_v2.c.md) · [`resource.h`](resource.h.md) · [`winquake.rc`](winquake.rc.md).

## 13. The assembly — a pluggable seam, not a requirement

About forty hand-written routines, **every one of which has a portable twin the build can select instead**
([Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)). Read the three layout headers and skim the
rest.

**The layout headers freeze record sizes and offsets**: [`asm_i386.h`](asm_i386.h.md) · [`asm_draw.h`](asm_draw.h.md) ·
[`d_ifacea.h`](d_ifacea.h.md) · [`quakeasm.h`](quakeasm.h.md).

**The routines**: [`d_draw.s`](d_draw.s.md) · [`d_draw16.s`](d_draw16.s.md) · [`d_parta.s`](d_parta.s.md) ·
[`d_polysa.s`](d_polysa.s.md) · [`d_scana.s`](d_scana.s.md) · [`d_spr8.s`](d_spr8.s.md) · [`d_varsa.s`](d_varsa.s.md) ·
[`d_copy.s`](d_copy.s.md) · [`r_aclipa.s`](r_aclipa.s.md) · [`r_aliasa.s`](r_aliasa.s.md) · [`r_drawa.s`](r_drawa.s.md) ·
[`r_edgea.s`](r_edgea.s.md) · [`r_varsa.s`](r_varsa.s.md) · [`surf8.s`](surf8.s.md) · [`surf16.s`](surf16.s.md) ·
[`snd_mixa.s`](snd_mixa.s.md) · [`math.s`](math.s.md) · [`worlda.s`](worlda.s.md) · [`sys_dosa.s`](sys_dosa.s.md) ·
[`sys_wina.s`](sys_wina.s.md) · [`dosasm.s`](dosasm.s.md).

**The translator**: [`gas2masm/`](gas2masm/README.md) — one source, two assemblers.

## 14. Build and release notes — data twins

[`Makefile.linuxi386`](Makefile.linuxi386.md) — **the authoritative list of which files make which of the three engines.** Read it if
you read nothing else in this section.
[`Makefile.Solaris`](Makefile.Solaris.md) · [`README.Solaris`](README.Solaris.md) — the big-endian port, which is the conformance test
for every wire and file format in the recipe.
[`wqreadme.txt`](wqreadme.txt.md) — the command-line options and the content layout.
[`glqnotes.txt`](glqnotes.txt.md) · [`3dfx.txt`](3dfx.txt.md) — the hardware renderer's switches and what each works around.
[`quake.spec.sh`](quake.spec.sh.md) · [`quake-data.spec.sh`](quake-data.spec.sh.md) ·
[`quake-hipnotic.spec.sh`](quake-hipnotic.spec.sh.md) · [`quake-rogue.spec.sh`](quake-rogue.spec.sh.md) — the installation layout the
file system's search order depends on.

---

## What is not twinned

Skipped as the recipe's classification allows: the editor and project files (`*.dsp`, `*.dsw`, `*.mdp`, `*.ncb`, `*.opt`, `*.plg`,
`winquake.aps`), the batch files that invoke the compiler, the binary assets (`quake.ico`, `qe3.ico`, `quake.gif`,
`cwsdpmi.exe`), the vendored third-party trees (`dxsdk/`, `scitech/`), the packaging kit (`kit/`), the shipped text content
(`data/`), and the shipped documentation (`docs/`).

## Cycles

The chapter is not acyclic and the cycles are facts about the design:

- **[`host.c`](host.c.md) ↔ the client and the server.** The host drives both and both call back into it for printing, errors and
  timing. Broken by reading the host last.
- **[`model.c`](model.c.md) ↔ the renderer.** The loader fills records the renderer defines, and the renderer asks the loader for
  models. Broken at the record definitions ([`model.h`](model.h.md)).
- **[`sv_phys.c`](sv_phys.c.md) ↔ [`pr_cmds.c`](pr_cmds.c.md) ↔ the game logic.** Physics calls the game's handlers, the game asks the
  engine to move things. Broken at the interpreter ([`pr_exec.c`](pr_exec.c.md)).
- **[`vid_*.c`](vid_win.c.md) ↔ [`d_surf.c`](d_surf.c.md).** The video backend sizes the surface cache using a rule the renderer owns.
  Broken at [`vid_dos.h`](vid_dos.h.md)'s declaration of the rule.
