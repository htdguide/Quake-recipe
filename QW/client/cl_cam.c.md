# QW/client/cl_cam.c

> The spectator camera: follow a chosen player, either from their own eyes or from a position found by searching for an unobstructed view nearby, and choose automatically who is worth watching.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`pmove.h`](pmove.h.md) · [`cl_ents.c`](cl_ents.c.md)
**Used by** — [`cl_input.c`](cl_input.c.md) when spectating; [`view.c`](view.c.md) for the view origin; [`cl_main.c`](cl_main.c.md) to suppress the local player's own drawing
**Tier floor** — none

## Purpose

A facility with no counterpart in the original engine, and the reason spectating is a first-class mode rather than a free-flying
camera. A spectator follows a player in one of two modes — through their eyes, or from a third-person position behind and near them —
and the second mode is the interesting one, because a camera placed blindly behind a player spends most of its time inside a wall.

## State

```text
VARIABLE autocam : off | tracking | locked
VARIABLE spec_track : which player is followed
VARIABLE oldphysents ;  desired_position ;  locked, cam_forceview
VARIABLE cam_viewangles ;  cam_lastviewtime
CONSTANT the search radius, the height offsets, and the angle step
```

## `Cam_Track`, `Cam_FinishMove`, `Cam_SetView`

**Contract** — each frame, if following a player: place the camera either at that player's eyes or at the position the flyby search
chose, and set the view angles either from the player's own aim or from the direction toward them. Requests a re-search when the
current position becomes obstructed.

**Invariants** —

- **In first-person mode the camera takes the tracked player's own view angles**, taken from their reported command
  ([`cl_ents.c`](cl_ents.c.md)). That is only possible because the command travels — a spectator sees exactly what the player sees
  because the player's aim is in the protocol.
- **In third-person mode the view angles point at the player**, adjusted smoothly rather than snapped, so the camera turns rather
  than jumping.
- The position is **re-derived when the view becomes blocked**, not every frame, because the search is not free.

## `InitFlyby`, `Cam_TryFlyby`, `Cam_IsVisible`, `Cam_DoTrace`

**Contract** — find a camera position from which the tracked player is visible: sample a set of directions around the player at a
fixed radius, sweep from the player toward each candidate, score each by how far the sweep got and how well it faces the player's
direction of travel, and take the best.

```text
FUNCTION init_flyby(player, forced)
  best_score = worst
  FOR EACH sampled direction around the player          # a ring, plus above
    candidate = player position + direction * radius
    IF the player is not visible from the candidate  SKIP
    score = how unobstructed the sweep was,
            weighted by how much the candidate looks along the player's motion
    IF score beats the best  remember the candidate
  IF nothing scored  fall back to the player's own eyes
  desired_position = the best candidate

FUNCTION is_visible(from, player)
  sweep a point from `from` toward the player's eye position
  RETURN the sweep reached it
```

**Invariants** —

- **Visibility is tested with the movement model's own collision queries**
  ([`pmovetst.c`](pmovetst.c.md)), which the client has only because prediction needed them. So the spectator camera is a free
  consequence of the prediction work — a good illustration of a well-placed interface paying twice.
- **The candidate set is sampled, not solved.** A ring of directions at a fixed radius, scored and best-taken. That is the whole
  algorithm and it is the right level of effort: a camera that is *usually* well placed and never inside geometry beats an optimal
  one that costs real time.
- **Positions behind the player's direction of travel are preferred**, so the camera shows where the player is going rather than
  where they came from. Without that weighting the camera settles in front of a running player and shows nothing.
- The **fallback is the player's own eyes**, which is always valid. A camera search must have a guaranteed answer.
- The world list must be **saved and restored** around the search, because it borrows the movement model's shared record
  ([`pmove.h`](pmove.h.md)) — the same borrowing discipline as [`cl_pred.c`](cl_pred.c.md).

## `Cam_CheckHighTarget`, `Cam_Lock`, `Cam_Unlock`, `Cam_Reset`

**Contract** — choose the player with the highest score to follow when nobody is chosen; begin and stop following a player, telling
the server which one; and reset on disconnection.

**Invariants** — **the server is told who is being followed**, because it then sends that player's weapon animation and skips its
visibility filtering for them ([`sv_ents.c`](../server/sv_ents.c.md)). A spectator following someone needs to see what only they can
see, and that requires server cooperation — it cannot be done client-side.

The automatic choice is by score, which is a reasonable default and not a good one; a rebuild might prefer recent activity.

## `Cam_DrawViewModel`, `Cam_DrawPlayer`

**Contract** — report whether the weapon model should be drawn (only in first-person mode) and whether a given player's body should
be (not the one whose eyes we are using).

**Invariants** — the two questions have to be asked from the renderer
([`gl_rmain.c`](gl_rmain.c.md), [`cl_main.c`](cl_main.c.md)), so the camera exposes them as predicates rather than setting flags.
That keeps the camera's state in one place.

## `adjustang`, `vectoangles`, `vlen`, `CL_InitCam`

**Contract** — move an angle toward a target by a bounded step taking the shorter way around; convert a direction to angles; a
length; and register the settings.

**Invariants** — **turning must take the shorter way around and must wrap correctly**, or the camera spins the long way when a
player crosses the wrap point. That is the one arithmetic subtlety in the file and it is the classic angle bug.

**Notes** — the file is worth a page for two transferable things: the **sample-and-score camera placement**, which is the standard
answer for any third-person camera in geometry, and the observation that **spectating properly requires the server to know who is
watching whom**.
