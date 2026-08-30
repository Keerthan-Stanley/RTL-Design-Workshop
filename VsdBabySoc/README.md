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

PLL — avsdpll
1. Function: Phase-Locked Loop — generates a stable, higher-frequency system clock (CLK) from a lower-frequency reference input (REF)

2. Inputs: ENb_CP (charge pump enable, active-low), ENb_VCO (VCO enable, active-low), REF (reference clock), VCO_IN
- Output: CLK — feeds the RVMYTH core

3. Why it's needed: the core needs a clock of a specific frequency/quality; rather than relying on an external clock source directly, the PLL locks onto the reference and multiplies/stabilizes it on-chip

4. In synthesis: treated as a black-box cell (behavioral model from .lib), not decomposed into gates — it's analog IP, not synthesizable digital logic


Core — rvmyth
1. Function: a RISC-V CPU core (RV32I-class), the actual programmable compute block of the SoC
Inputs: CLK (from PLL), reset

2. Output: OUT (renamed RV_TO_DAC on the wire) — a digital value the core computes/updates each cycle, fed to the DAC

3. Why it's the largest block: contains the register file, ALU, control logic, pipeline/sequencing logic — this is why it dominates cell count (~5,000 of the SoC's 5,285 total cells; 1,144+ flip-flops alone for registers/pipeline state)

4. In synthesis: fully decomposed into real Sky130 gates (NAND, NOR, MUX, DFF, etc.) — this is genuine synthesizable RTL, unlike the PLL/DAC


DAC — avsddac
1. Function: Digital-to-Analog Converter — takes the core's digital output and converts it to an analog-equivalent signal

2. Inputs: D (digital input, from RV_TO_DAC), VREFH (reference high voltage
 
3. Output: OUT — the final SoC output, an analog-equivalent value (visible in your waveform as a real type, e.g. 0.1674, not a digital bit pattern)

4. Why it's needed: it's the SoC's actual output interface — converting the CPU's internal digital computation into something that could drive an external analog circuit

5. In synthesis: same as PLL — black-box behavioral model, not decomposed into gates


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
