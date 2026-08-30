# BabySoC — RTL to Gate-Level Synthesis & Verification

This project documents the complete flow for BabySoC — a small SoC combining
a PLL (`avsdpll`), a RISC-V core (`rvmyth`), and a DAC (`avsddac`) — from
RTL simulation through synthesis to post-synthesis gate-level simulation (GLS).

## Design Overview

BabySoC integrates three blocks:
- **PLL (avsdpll)** — generates the system clock from a reference input
- **Core (rvmyth)** — a RISC-V core, clocked by the PLL output
- **DAC (avsddac)** — converts the core's digital output to an analog-equivalent signal

<img width="937" height="227" alt="Screenshot 2026-08-26 193734" src="https://github.com/user-attachments/assets/38368b9d-26a5-4703-8acf-f133192ccf26" />


## Flow Summary

| Stage | Purpose | Details |
|---|---|---|
| [1. Pre-Synthesis Simulation](./01_pre_synth_sim/) | Verify RTL behavioral correctness | RTL simulated directly, no gates involved |
| [2. Synthesis](./02_synthesis/) | Convert RTL into a gate-level netlist | Yosys + Sky130 standard cell library |
| [3. Post-Synthesis Simulation (GLS)](./03_post_synth_sim/) | Verify the synthesized netlist behaves correctly | Same testbench, now simulating real gates |
| [4. Pre vs Post Comparison](./04_pre_post_comparison/) | Confirm functional equivalence | Side-by-side waveform comparison |

## Key Result

Post-synthesis simulation matches pre-synthesis simulation on all top-level
signals (`CLK`, `reset`, `OUT`, `RV_TO_DAC[9:0]`), confirming synthesis
preserved the RTL's intended functional behavior. See
[04_pre_post_comparison](./04_pre_post_comparison/) for details.

## Tools Used
- **Icarus Verilog** (`iverilog`, `vvp`) — simulation
- **GTKWave** — waveform viewing
- **Yosys** — RTL synthesis
- **Sky130 PDK** (`sky130_fd_sc_hd`) — standard cell library
