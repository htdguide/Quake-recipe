# WinQuake/net_vcr.c

> The replay driver: substitutes a recorded log of network operations for the network, so a session can be re-run exactly.

**Needs** — [`net.h`](net.h.md) · [`net_vcr.h`](net_vcr.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`net_main.c`](net_main.c.md) registers it in place of every other driver during playback
**Tier floor** — none

## Purpose

The engine's determinism tool. With recording enabled, every network operation's *result* is logged alongside the
command line ([`host.c`](host.c.md#host_initvcr)); with playback enabled, this driver replaces the network entirely
and returns the logged results in order. So a session that exhibited a bug can be replayed under a debugger.

That is the closest thing this engine has to a test harness, and it is worth reproducing: the alternative for a
networked bug is to reproduce it live.

## State

```text
RECORD LogEntry                     # one recorded operation
  time    : real
  op      : int                     # which operation
  session : int                     # which connection
VARIABLE next : LogEntry            # read one ahead
VARIABLE vcrFile : file
CONSTANT vcr_op_cansendmessage = 4 ;  vcr_max_message = 4
```

**Invariants** — each entry carries the **time** at which the operation happened, and playback sets the network
clock from it — so the replay advances on the recorded timeline rather than the machine's. That is what makes it
deterministic.

## `VCR_ReadNext`

**Contract** — reads the next log entry, substituting a sentinel operation at end of file.

## `VCR_GetMessage`

**Contract** — verifies that the next logged operation is a message read for the expected connection at the expected
time, then returns the logged result, copying the logged bytes into the incoming buffer when a message was returned.
A mismatch is a fatal error.

**Invariants** — **a mismatch is fatal**, and that is the point: it means the engine has diverged from the recorded
run, so the operation sequence doubles as a divergence detector. That is why the log records the operation kind and
the connection as well as the result.

## `VCR_SendMessage`, `VCR_CanSendMessage`

**Contract** — verify the logged operation matches and return its logged result. Nothing is actually sent.

## `VCR_Connect`, `VCR_CheckNewConnections`, `VCR_SearchForHosts`, `VCR_Listen`, `VCR_Init`, `VCR_Shutdown`, `VCR_Close`

**Contract** — the remaining driver operations, each returning its logged result or doing nothing.

**Notes** — the recording side is not here: the *other* drivers write the log as they run, so recording is a
property of the real drivers and playback is this one. A rebuild should keep the asymmetry, because the thing
recorded must be the real driver's behaviour.
