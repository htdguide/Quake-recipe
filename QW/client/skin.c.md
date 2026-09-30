# QW/client/skin.c

> Player appearance as downloadable content: each player names a skin, the client finds it or fetches it from the server, and shares one loaded copy among everyone using it.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`common.h`](common.h.md) · [Seam: Palette-indexed image decoding](../../SYSTEM-REQUIREMENTS.md#seam-palette-indexed-image-decoding)
**Used by** — [`cl_parse.c`](cl_parse.c.md) on a player's appearance change; [`gl_rmisc.c`](gl_rmisc.c.md) and [`r_alias.c`](r_alias.c.md) to draw
**Tier floor** — none

## Purpose

A facility the original engine has no counterpart for: **players supply their own appearance, and the client obtains it from the
server if it does not have it.** The recipe records it because it establishes a pattern — client-named, server-served, shared
content — and because its caching rules are the interesting part.

## State

```text
RECORD Skin
  name   : text                   # as the player declared it
  failedload : bool               # do not keep retrying
  cache  : a cache entry holding the decoded image
VARIABLE skins : Skin[limit] ;  numskins
VARIABLE noskin_name, baseskin_name : the fallbacks
```

**Invariants** —

- **Skins are shared by name, not per player.** Twenty players naming the same skin cost one loaded image. The player record holds
  a reference to the shared entry ([`client.h`](client.h.md)).
- **A failed load is remembered**, so a missing skin is attempted once rather than every frame. Without that flag a player with a
  misspelled skin name costs a file-system miss per frame per viewer.
- The entry lives in the **evictable cache** ([`zone.h`](zone.h.md)), so skins are the first thing discarded under memory
  pressure, and a later draw reloads. That is the right classification: they are reconstructible.

## `Skin_Find`

**Contract** — takes a player; reads their declared skin name from their information dictionary; finds or creates the shared entry
for that name. An empty or unacceptable name falls back to the base skin.

**Invariants** — the name is **sanitized**: path separators and escapes are rejected, because the name comes from another player
and is used to open a file. ***A name supplied by a remote party that reaches the file system is an attack surface***, and this is
the check. A rebuild must not omit it — the same rule as the download path in
[`sv_user.c`](../server/sv_user.c.md).

## `Skin_Cache`

**Contract** — returns the decoded image for an entry, loading and decoding it on first use: reads the palette image file, checks
its dimensions against the expected skin size, and copies it into the cache — scaling down if it is larger than expected.

**Invariants** —

- **The image must be exactly the model's skin dimensions**, and a mismatched one is rejected or reduced rather than accepted,
  because the model's texture coordinates assume the size ([`modelgen.h`](modelgen.h.md)).
- Decoding is the palette-image path ([Seam: Palette-indexed image decoding](../../SYSTEM-REQUIREMENTS.md#seam-palette-indexed-image-decoding)) and is
  the only place the client reads an image file that did not come from the content archives.

## `Skin_NextDownload`

**Contract** — walks every player, ensuring each one's skin is present; requests the first missing one from the server and returns,
to be called again when that download completes. When none is missing, finishes the connection sequence.

```text
FUNCTION skin_next_download()
  FOR EACH player with a name
    find their skin entry
    IF its file is not present locally
      ask the server to send it ;  RETURN            # resume when it arrives
  FOR EACH player  translate their colours into their skin
  tell the server we are ready to begin
```

**Invariants** —

- **Downloading is a state machine driven by completion**, one item at a time, and it is the *last* stage of joining a server
  ([`cl_parse.c`](cl_parse.c.md) runs the same pattern for models and sounds). A player joins only once everything is present,
  which is why a first connection to a busy server is slow and subsequent ones are not.
- The **colour translation happens after all downloads**, because it rewrites each skin per player
  ([`gl_rmisc.c`](gl_rmisc.c.md)) and must see the final image.

**Notes** — the pattern worth carrying is the whole of it: **content named by a peer, fetched from the server, shared by name,
cached evictably, sanitized at the boundary, and fetched one item at a time driven by completion.** Every one of those five is a
decision a rebuild supporting user content has to make, and this file makes all five defensibly.
