# Stage 1 — Pre-Synthesis Simulation

## Purpose

Before any synthesis happens, the RTL (behavioral Verilog) is simulated
directly to verify the design logic is correct as written. This is a check
on design intent — no gates exist yet at this stage, only the RTL source
(`vsdbabysoc.v`, `rvmyth.v`, `clk_gate.v`, etc.) and the testbench.

## Commands

Run from the project root:

```bash
cd ~/VLSI/BabySoC_Simulation

# Clean any old build artifacts
rm -f pre_synth_sim.out pre_synth_sim.vcd

# Compile the RTL testbench
iverilog -o ./pre_synth_sim.out -DPRE_SYNTH_SIM src/module/testbench.v \
    -I src/include -I src/module/

# Run the simulation (generates pre_synth_sim.vcd)
vvp pre_synth_sim.out

# View the waveform
gtkwave pre_synth_sim.vcd
```

## What `-DPRE_SYNTH_SIM` does

The testbench uses this macro to select the RTL-based `` `include`` block
(instantiating `vsdbabysoc.v`, `rvmyth.v`, `clk_gate.v`, `avsddac.v`,
`avsdpll.v` directly), as opposed to the post-synthesis netlist used later.

## Result

The waveform below shows `CLK`, `reset`, `OUT`, and `RV_TO_DAC[9:0]`
behaving as expected — `reset` deasserts early in the simulation, `CLK`
toggles steadily, and `RV_TO_DAC` shows the expected nested toggling
pattern (LSBs switching fastest, MSBs slower) consistent with a counting/
DAC-code-generation pattern from the RVMYTH core.

<img width="1917" height="1078" alt="Screenshot 2026-08-30 181804" src="https://github.com/user-attachments/assets/cbaed13c-53d2-46c7-a279-d9001515dd63" />

