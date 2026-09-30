# QW/server/notes.txt

> Data: a five-line sketch of a challenge-response scheme by which a directory server would verify that a listed game server is genuine.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

A design note for something the shipped code does not contain. It is worth one page because it is the only place in the tree
where authentication is designed at all, and because it names the problem the announcement mechanism
([`sv_ccmds.c`](sv_ccmds.c.md)) leaves open: anyone can announce anything to the directory.

## State

Data; no run-time state.

## What it records

```text
The server holds a random token of its own.
The directory, on seeing a new server, challenges it with a digest of
  (its own secret, the server's token).
The server replies with a digest of (that challenge, its own token).
The directory verifies the reply.
The directory's secret changes per server, randomly.
```

**Invariants** — the scheme's property is that **the reply proves the server received the challenge**, and the per-server secret
means one server's exchange reveals nothing about another's. It is a challenge-response over a shared secret, and the digest
function is the one in [`md4.c`](../client/md4.c.md).

**Notes** — recorded as a **design that was not built**. The first line — *registered servers will auto-kick unregistered ones*
— describes an intended policy with no implementation anywhere in the tree.

For a rebuilder the useful content is the problem statement, not the scheme: a modern rebuild should not use this construction or
this digest function ([`md4.c`](../client/md4.c.md) records why), but it faces the same question, and the answer — challenge,
response, per-party secret — is the right shape.
