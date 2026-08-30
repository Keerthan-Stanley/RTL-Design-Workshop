# Stage 3 — Post-Synthesis Simulation (GLS)

## Purpose

Now that synthesis has produced a gate-level netlist (`vsdbabysoc.synth.v`),
this stage re-simulates the design — but this time using the actual
synthesized gates instead of the original behavioral RTL. This verifies
that synthesis preserved the RTL's intended functional behavior, catching
any bugs introduced during the RTL-to-gates translation (e.g. unintended
latches, optimization side effects, clock-gating mismatches).

This is **functional GLS** — zero/unit-delay, checking pure logic
correctness. It does not check real timing (setup/hold), which requires
SDF-annotated GLS after place-and-route — a separate, later step.

## Required Files

The testbench's `` `ifdef POST_SYNTH_SIM `` block includes:
- `vsdbabysoc.synth.v` (or `baby_soc_netlist3.v`, per this testbench) — the synthesized netlist
- `avsddac.v`, `avsdpll.v` — analog IP behavioral models (`src/module/`)
- `primitives.v`, `sky130_fd_sc_hd.v` — Sky130 cell behavioral models (`src/gls_model/`)

## Commands

```bash
cd ~/VLSI/BabySoC_Simulation

iverilog -o ./post_synth_sim.out -DPOST_SYNTH_SIM -DFUNCTIONAL \
    -I src/include -I src/module -I src/gls_model \
    src/module/testbench.v

vvp post_synth_sim.out
gtkwave post_synth_sim.vcd
```

## Note on `-DFUNCTIONAL`

`sky130_fd_sc_hd.v` includes `specify` blocks encoding real gate delay and
timing-check information for timing-accurate simulation — but these
require SDF back-annotation from place-and-route, which isn't available
at this stage. `-DFUNCTIONAL` tells the cell library to skip these
timing/specify sections and simulate pure logical behavior only. This
flag was also required to work around an Icarus Verilog parsing
limitation on a UDP latch definition inside `sky130_fd_sc_hd.v`
(`sky130_fd_sc_hd__udp_dlatch$P_pp$PG$N`) that otherwise caused a syntax
error during compilation.

## Result

The waveform below shows `CLK`, `reset`, `OUT`, and `RV_TO_DAC[9:0]`
from the gate-level simulation — same signal set as the pre-synthesis
run, now driven by real Sky130 standard cells instead of behavioral RTL.


