# QW/docs — the release notes

> Read as a specification of observable behaviour: everything in them is something a player can check.

Four documents, and for a rebuilder their value is that they describe the shipped system from outside the code — which makes them the
closest thing the recipe has to acceptance criteria for the client.

[`qwcl-readme.txt`](qwcl-readme.txt.md) — the long form: **the complete setting and command reference**, and a
symptom-to-cause table for connection problems. That table is the useful part: four network failures that look alike to a player, each
with a distinct cause.
[`readme.qwcl`](readme.qwcl.md) — the short form, and the clearest statement of **the three settings a player must tune**: bandwidth
rate, display lag, and whether to extrapolate other players. Those three are the honest cost of the design.
[`readme.qwsv`](readme.qwsv.md) — how to run a server, and the fact that **its entire configuration is a file of console commands**.
[`glqwcl-readme.txt`](glqwcl-readme.txt.md) — the hardware client: identical networking, visual differences only, and the warning that
removing the frame limit floods your own uplink.

**What to take from this directory.** Two things. The symptom table, because a rebuild that cannot distinguish those four failures in
its own diagnostics will not be able to support its users. And the three settings, because a rebuild on a modern network should be able
to derive all three automatically — the original could not, and that is a reason to try rather than a reason to copy.
