# WinQuake/net_comx.c

> The serial hardware layer: two ring buffers per port filled and drained by an interrupt handler, plus the modem dialling conversation.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`dosisms.h`](dosisms.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — included textually by [`net_ser.c`](net_ser.c.md)
**Tier floor** — T1: an interrupt handler shares ring buffers with the main loop, and the hardware is programmed through port registers

## Purpose

Turns a serial chip into the byte-at-a-time read and write that [`net_ser.c`](net_ser.c.md) wants. It is the deepest
platform code in the engine outside the video drivers, and almost none of it survives a rebuild — a modern platform
hands you a byte stream. What survives is the shape of the interface and two decisions inside it.

## State

```text
RECORD ComPort
  uart          : port base address
  irq, baud     : int
  inputQueue, outputQueue : byte ring        # shared with the interrupt handler
  useModem      : bool
  dialType, clear, startup, shutdown : text  # modem command strings
  enabled, modemInitialized, modemConnected : bool
  statusUpdated : bool
VARIABLE portConfig[ports]
CONSTANT error codes: no data, line status, connection lost
```

**Invariants** — the rings are the **only** thing shared between the interrupt handler and the main loop, and they are
single-producer single-consumer in each direction: the handler writes the input ring and reads the output ring, the
main loop the reverse. That is what makes the sharing safe without a lock, and a rebuild using a thread instead of an
interrupt inherits both the structure and the reason.

## `ISR_8250`, `ISR_16550`

**Contract** — the interrupt handler, one variant per chip generation: drain every pending cause — a received byte
into the input ring, a free transmit register filled from the output ring, a line-status or modem-status change
recorded — then acknowledge the interrupt.

**Invariants** — **the loop must handle every pending cause before returning**, because the chip raises one interrupt
for a set of causes and an unserviced cause leaves it latched and the line dead. The second variant exists because the
later chip has a sixteen-byte hardware buffer, so it fills several bytes per interrupt rather than one.

An overrun — the input ring full — **drops the byte and records the error**, which surfaces as a lost message and is
recovered by the framing in [`net_ser.c`](net_ser.c.md). Blocking in an interrupt handler is not an option.

## `ComPort_Enable`, `ComPort_Disable`

**Contract** — claim the interrupt vector, program the divisor for the configured speed, the frame format and the
interrupt mask, enable the chip's own buffer if present, and unmask the interrupt line; or undo all of it in reverse.

**Invariants** — **the vector is restored on disable and at shutdown**, including on a fatal error path, because an
interrupt into freed code is an unrecoverable machine state. This is the concrete reason
[`sys_dos.c`](sys_dos.c.md)'s error path runs cleanup before printing.

## `TTY_Open`, `TTY_Close`, `TTY_Enable`, `TTY_ReadByte`, `TTY_WriteByte`, `TTY_Flush`, `TTY_OutputQueueIsEmpty`, `TTY_IsEnabled`, `TTY_IsModem`

**Contract** — the interface the driver above uses: take a byte from the input ring or report no-data; put a byte in
the output ring, priming the transmitter if it is idle; report whether the output ring has drained; and query a port's
configuration.

**Invariants** — **writing must prime the transmitter when the ring was empty**, because the interrupt that refills it
only fires after a byte has been sent. Forgetting this is the classic stall: data sits in the ring and nothing ever
starts it moving.

## `TTY_Connect`, `TTY_Disconnect`, `TTY_CheckForConnection`, `Modem_Init`, `Modem_Command`, `Modem_Response`, `Modem_Hangup`

**Contract** — the modem conversation: send an initialization string, then a dial string with the number, then read
responses until one indicates a connection, a failure, or a timeout; answer an incoming ring; and hang up by the
escape-then-command sequence with its required pauses.

**Invariants** — the hangup is **split across several procedures driven by the timer queue**
([`net_main.c`](net_main.c.md#net_poll-schedulepollprocedure)) because it needs second-long pauses between steps and the frame
loop cannot block for them. That is the one structural idea worth carrying: a slow device conversation belongs on a
timer, not in a blocking call.

The command strings are **player-configurable** ([`menu.c`](menu.c.md)) rather than compiled in, because every modem
wanted different ones.

## `TTY_GetComPortConfig`, `TTY_SetComPortConfig`, `TTY_GetModemConfig`, `TTY_SetModemConfig`, `TTY_Init`, `TTY_Shutdown`, `ResetComPortConfig`, `CheckStatus`, `Com_f`

**Contract** — read and write a port's address, interrupt number, speed and modem flag, and the four modem strings;
initialize the table to the conventional defaults; report line status changes to the player; and the console command
that prints or changes any of it.

**Notes** — the defaults encode the standard address and interrupt pairs of the platform. They are historical trivia,
not decisions, and a rebuild discards them along with everything else in this file except the ring discipline, the
prime-on-write rule, and the timer-driven hangup.
