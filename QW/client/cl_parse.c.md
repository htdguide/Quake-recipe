# QW/client/cl_parse.c

> Receiving the server's stream: the staged join that fetches every model, sound and skin the client lacks before entering the game, the block-by-block file transfer, the per-player information dictionaries, and the loss measurement.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`protocol.h`](protocol.h.md) · [`cl_ents.c`](cl_ents.c.md) · [`skin.c`](skin.c.md) · [`net_chan.c`](net_chan.c.md) · [`sound.h`](sound.h.md) · [`cdaudio.h`](cdaudio.h.md)
**Used by** — [`cl_main.c`](cl_main.c.md) for every packet; [`cl_demo.c`](cl_demo.c.md) when replaying
**Tier floor** — none

## Purpose

Read [`cl_parse.c`](../../WinQuake/cl_parse.c.md) for the shared substance: the message dispatch, the sound and temporary-entity
handling, the light styles, the centre prints and the statistics. The three additions are what let a client join a server it has
never seen with content it does not have.

## State

As [`cl_parse.c`](../../WinQuake/cl_parse.c.md); the records are unchanged except where **What differs** says otherwise.

## `CL_ParseServerMessage`

**Contract** — dispatches every message in a packet by its leading byte, until the packet is exhausted. An unknown kind is a fatal
error for the session.

**Invariants** — the message set is the protocol ([`protocol.h`](protocol.h.md)) and is larger than the original's, adding the
snapshot messages, the nail message, the player-state message, the download blocks, the dictionary updates and the server
information. **An unknown message kind cannot be skipped**, because messages are not length-prefixed, so it ends the session — which
is why the message set is a hard version contract.

## The staged join

## `CL_ParseServerData`, `CL_ParseSoundlist`, `CL_ParseModellist`, `CL_RequestNextDownload`, `Model_NextDownload`, `Sound_NextDownload`

**Contract** — the client's side of the handshake ([`sv_user.c`](../server/sv_user.c.md)): read the protocol version, its own
player slot, the level name and the movement tunables; then the sound list and the model list in runs, asking for the next run each
time; then, once the lists are complete, walk them ensuring every item is present locally, requesting the first missing one and
resuming when it arrives; then load them all and tell the server to spawn.

```text
FUNCTION request_next_download()
  IF models are not all checked
    FOR EACH model in the list from where we left off
      IF check_or_download(its file) returned "not yet"  RETURN
    load every model ;  move on to sounds
  IF sounds are not all checked
    FOR EACH sound in the list from where we left off
      IF check_or_download(its file) returned "not yet"  RETURN
    load every sound
  ask the server to send the level description blocks
```

**Invariants** —

- **Nothing is loaded until everything is present.** The client cannot enter a level with a missing model, because model indices are
  positions in the list and a gap would misname every later one.
- **The walk resumes from an index, not from the start**, so a hundred-item list with one missing item does not re-check the
  ninety-nine. The index is per stage and is the whole state the resumption needs.
- **The lists arrive in runs because they exceed a reliable message**, and the client asks for the next run by index. The same
  request-response resumption as the downloads and the description blocks; see
  [`sv_user.c`](../server/sv_user.c.md).
- The movement tunables arrive here, and **prediction is impossible until they do**
  ([`pmove.h`](pmove.h.md)).

## `CL_CheckOrDownloadFile`, `CL_ParseDownload`

**Contract** — report whether a named file is present, and if not, ask the server for it and report "not yet"; and receive one block
of a file, appending it to a temporary file and asking for the next until the server reports completion, then renaming it into place.

```text
FUNCTION check_or_download(filename) -> bool
  IF the name contains a parent-directory escape  REFUSE, and report "present"
  IF the file exists locally  RETURN present
  IF recording or replaying  RETURN present            # cannot download then
  remember the name; derive a TEMPORARY name from it
  ask the server to send it
  RETURN not yet

FUNCTION parse_download()
  size = a 16-bit length ;  percent = a byte
  IF size says "not found"  report it ;  request the next item ;  RETURN
  IF no file is open  create the path and open the TEMPORARY file
  append `size` bytes from the packet
  IF percent is not complete
    record the progress ;  ask for the next block
  ELSE
    close ;  RENAME the temporary file to its real name
    request the next item
```

**Invariants** —

- **A parent-directory escape in the name is refused.** The name comes from the server's list or from another player's skin
  declaration, so it is remote input reaching the file system. ***This check is security-critical*** and it is the same rule as the
  server's side ([`sv_user.c`](../server/sv_user.c.md)) and the skin name sanitizer
  ([`skin.c`](skin.c.md)). Note it is only a substring check — a rebuild should canonicalize the path and verify it stays within the
  content directory, which is strictly stronger.
- **The download goes to a temporary name and is renamed only on completion**, and the source says why: an interrupted download must
  not leave a truncated file that later looks present. That is the general pattern for any resumable fetch and it costs one rename.
- **Skins are written to a fixed shared directory and everything else to the current content directory**, so a skin downloaded on
  one server is available on the next.
- **The blocks ride in the reliable stream**, so they are ordered and complete, and they share the link with play rather than
  blocking it ([`net_chan.c`](net_chan.c.md)). A download therefore slows the game a little and never stops it.
- A file the server does not have is **reported and skipped**, not fatal — the client plays without it.
- Downloading is **disabled while recording or replaying**, because a recording must be self-contained.

## `CL_CalcNet`

**Contract** — classify each of the last several packets as received with a measured latency, lost, choked, or carrying an
unusable delta; return the loss percentage over the window.

```text
FUNCTION calc_net() -> percent
  FOR EACH of the last `window` outgoing sequences
    frame = the ring slot
    IF no reply was ever received   mark it LOST
    ELSE IF it was choked locally   mark it CHOKED
    ELSE IF its delta was unusable  mark it INVALID
    ELSE                            record its round trip
  RETURN (count of LOST) * 100 / window
```

**Invariants** — **four outcomes, distinguished**, and only the first counts as loss. This is the data the network display draws
([`gl_ngraph.c`](gl_ngraph.c.md)) and the loss figure the client reports to the server
([`cl_input.c`](cl_input.c.md)). Distinguishing *lost* from *choked* is the point: the first is the network's fault and the second is
the player's rate setting.

## `CL_ProcessUserInfo`, `CL_UpdateUserinfo`, `CL_SetInfo`, `CL_ServerInfo`, `CL_NewTranslation`

**Contract** — receive another player's information dictionary or a single changed entry, or the server's; extract the fields the
client needs — display name, colours, skin, spectator marker — and rebuild that player's colour translation.

**Invariants** — **each player's dictionary is held on the client**, so every client knows every other player's chosen appearance
and can ask for the skin ([`skin.c`](skin.c.md)). A single entry changing costs a few bytes rather than the whole dictionary.

The colour translation is rebuilt when the colours change and the result is uploaded to the renderer
([`gl_rmisc.c`](gl_rmisc.c.md)) or kept as a table ([`r_alias.c`](r_alias.c.md)).

## `CL_ParseClientdata`, `CL_SetStat`, `CL_MuzzleFlash`, `CL_ParseBaseline`, `CL_ParseStartSoundPacket`

**Contract** — read the player's own view state and item set; set one statistic; spawn a muzzle-flash light on a named player; read an
entity's baseline for the delta scheme; and read a sound event with its optional volume and attenuation.

**Invariants** — **the player's own state arrives every frame and is not predicted**: health, ammunition, items and the view punch
are authoritative and there is no attempt to guess them. Only position is predicted. That separation — *predict motion, trust
everything else* — is worth stating as the rule.

Baselines arrive in the level description and are the delta reference for newly visible entities
([`sv_init.c`](../server/sv_init.c.md)).

## `CL_StartUpload`, `CL_NextUpload`, `CL_StopUpload`, `CL_IsUploading`

**Contract** — begin sending a file to the server a block per packet, send the next block, stop, and report whether one is in
progress.

**Invariants** — used only for the screen-capture request ([`sv_ccmds.c`](../server/sv_ccmds.c.md)), which the player can refuse
([`sv_user.c`](../server/sv_user.c.md)).

**Notes** — the pattern to take from this file is the **staged, resumable, index-driven join**: every stage is a request the client
repeats with an index, so no stage holds state on the server, every stage survives loss and reordering, and the whole sequence
composes out of one idea. It is what makes joining a server with unfamiliar content work at all, and it is built entirely out of
request-response over a reliable stream.
