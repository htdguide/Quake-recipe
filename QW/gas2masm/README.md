# QW/gas2masm — the assembly syntax translator, carried forward

> The same program as in the original tree, and here it is slightly beside the point.

[`gas2masm.c`](gas2masm.c.md) — unchanged; read
[`../../WinQuake/gas2masm/gas2masm.c`](../../WinQuake/gas2masm/gas2masm.c.md) for the substance.
[`gas2masm.001`](gas2masm.001.md) — a project-file backup.

**The oddity worth recording.** This tree checks in *both* the translator and its output: the hand-maintained translated assembly sits
in [`../client/`](../client/README.md) as `.asm` files beside the `.s` originals
([`../client/d_draw.asm`](../client/d_draw.asm.md) and fifteen siblings). So the duplication the translator exists to prevent is present
anyway, and the two copies must be kept in step by hand.

A rebuild using the portable inner loops
([Seam: Vectorized inner loops](../../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)) deletes the translator, both copies of the
assembly, and this whole question.

**Not twinned**: the editor and project files.
