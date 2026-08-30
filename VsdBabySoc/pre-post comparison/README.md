# Stage 4 — Pre vs Post-Synthesis Comparison

## Purpose

The final verification step: compare the pre-synthesis (RTL) waveform
against the post-synthesis (gate-level) waveform on the same top-level
signals, to confirm synthesis preserved functional correctness.

## Method

Both `.vcd` files were opened side-by-side in GTKWave, with the same
signal set added to each: `CLK`, `reset`, `OUT`, `RV_TO_DAC[9:0]`
(bit-expanded).

```bash
gtkwave pre_synth_sim.vcd &
gtkwave post_synth_sim.vcd &
```

## Result

<img width="957" height="1020" alt="Screenshot 2026-08-27 212725" src="https://github.com/user-attachments/assets/7d33e3c5-1ed1-478d-b133-6d68bd037bbf" />


`CLK`, `reset`, and `OUT` show identical timing and toggling in both
windows. `RV_TO_DAC[9:0]`, bit by bit, shows the same switching pattern
and structure in both — LSBs toggling fastest, MSBs slower, consistent
nested toggling — confirming the synthesized netlist behaves identically
to the RTL on all observable top-level signals.

## Conclusion

**Functional equivalence confirmed.** The gate-level netlist produced by
synthesis preserves the RTL's intended behavior for this testbench.
Internal signal names/hierarchy differ between the two runs (e.g. RTL
exposes signals like `w_CPU_rs1_a1` from the RVMYTH core that no longer
exist by that name post-synthesis, due to `flatten` and `rename
-enumerate`), which is expected — only top-level ports are guaranteed to
persist and match between pre- and post-synthesis representations.


