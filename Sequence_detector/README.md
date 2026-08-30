# Sequence Detector — FSM Design, Simulation & Synthesis

A Mealy-machine sequence detector (target sequence `0010001`) taken
through RTL analysis, pre-synthesis simulation, synthesis, and
post-synthesis gate-level simulation (GLS) — completed as a timed
assessment.

## Design Overview

- **Module:** `sequence_detector` — 7-state FSM (`STATE_W = 3`,
  `NUM_STATES = 7`), detects the bit pattern `0010001` on a serial input `din`.
- **Testbench:** drives a pseudo-random bit stream into `din`, counts
  detection pulses via `detection_count`.

## Flow Summary

| Stage | Purpose | Details |
|---|---|---|
| [1. RTL Analysis](./01_rtl_analysis/) | Read FSM structure statically | Target sequence, state storage, Moore/Mealy classification, clock/reset timing |
| [2. Pre-Synthesis Simulation](./02_pre_synth_sim/) | Run RTL simulation, verify detection behavior | Waveform, first detection time, total count |
| [3. Synthesis](./03_synthesis/) | RTL → Sky130 gate-level netlist | Cell stats, state register identification, schematic |
| [4. GLS & Comparison](./04_gls_comparison/) | Verify netlist matches RTL behavior | RTL vs GLS timing comparison, final conclusion |

## Key Result

Total detection count matched exactly between RTL and gate-level
simulation (4 detections in both), but the **timing** of the first
detection differed — RTL detected first at 1,771,000 ns, GLS at
991,000 ns — traced to a missing reset connection on the synthesized
state register. Full analysis in
[04_gls_comparison](./04_gls_comparison/).

## Tools Used
Icarus Verilog, GTKWave, Yosys, Sky130 PDK (`sky130_fd_sc_hd`)

# Stage 1 — RTL Analysis (Static, No Simulation)

These answers come purely from reading the RTL and testbench source —
no simulation required. This mirrors real RTL design practice: predicting
behavior from code before ever running a simulator.

## Target Sequence

Traced via the FSM's state transition path — the only path that reaches
`next_detected = 1'b1` (state 6, `din == 1`):

Bits consumed in order: **`0010001`** — confirmed by the RTL's own
comment (`// Target sequence: 0010001`).

## State Storage

```verilog
reg [STATE_W-1:0] state;
```
Updated synchronously: `state <= next_state;` inside
`always @(posedge clk)`.

## Output Dependency — Moore or Mealy?

**Mealy.** `next_detected = 1'b1` is only set inside state 6's
`din == 1'b1` branch — the condition depends on both the *current
state* and the *current input*, not state alone.

## Clock Period

From `tb.v`: `always #6 clk = ~clk;`
- Half-period = 6 ns
- Full period = **12 ns**
- Frequency ≈ **83.33 MHz**

## Reset Timing

- `reset` initialized to `1'b1` at t=0.
- Held for 2 clock cycles: `repeat(2) @(posedge clk);`
  - First posedge at t=6ns, second at t=18ns.
- Released at the next negedge: **t=24 ns**.

## Note on Manual Prediction (Q6–Q8)

The assessment also asked for a manual, by-hand prediction of the first
sequence occurrence, total detection count, and first detection cycle —
*before* running simulation. Due to time constraints during the timed
assessment, these predictions were not fully completed by hand. The
actual, simulated results are documented in
[02_pre_synth_sim](../02_pre_synth_sim/), and the comparison against
what should have been predicted is discussed in
[04_gls_comparison](../04_gls_comparison/).

# Stage 2 — Pre-Synthesis Simulation

## Commands

```bash
cd ~/VLSI/assesments/24eg104c03
iverilog -o sim.out rtl/sequence_detector.v tb/tb.v
vvp sim.out
gtkwave dump.vcd
```

## Result

- **First detection:** `TIME = 1,771,000 ns`
- **Total detections:** `FINAL_DETECTION_COUNT = 4`

The testbench prints every bit driven and the `detected` output live via
`$display`, so both values were read directly from simulation output
rather than the waveform — faster and less error-prone than visually
scanning GTKWave.

## Waveform

Signals shown: `clk`, `reset`, `din`, `detected`, `state[2:0]`.


`detected` pulses exactly 4 times across the run, consistent with the
`FINAL_DETECTION_COUNT` printed by the testbench's built-in counter.

# Stage 3 — Synthesis

## Commands

```bash
yosys
```
```tcl
read_verilog rtl/sequence_detector.v
synth -top sequence_detector
dfflibmap -liberty <sky130_liberty_file>
opt
abc -liberty <sky130_liberty_file>
stat
write_verilog -noattr synth_out.v
show sequence_detector
```

## Result — Cell Count = 25

| Category | Count |
|---|---|
| Sequential (`$_DFF_P_` × 7 + `$_SDFF_PP0_` × 1) | 8 |
| Combinational (`$_ANDNOT_`, `$_NOR_`, `$_ORNOT_`, `$_OR_`) | 17 |
| **Total** | **25** |

<img width="968" height="1078" alt="Screenshot 2026-08-29 111529" src="https://github.com/user-attachments/assets/26a966a3-6e47-4127-9dff-44f812dbc1ba" />


## State Register Identification

One flip-flop directly driving the `state` register:
- **Instance name:** `$308`
- **Cell type:** `$_DFF_P_`

(Any of `$308`, `$309`, `$310`, `$311` are valid — these four DFFs
together implement the 3-bit `state` register. `$302` (`$_SDFF_PP0_`)
is the `detected` output register, not part of state storage — notably,
it is the *only* flop with a reset (`R`) pin connected, which becomes
significant in the GLS comparison stage.)

## Why 3 Bits Is Sufficient for State Encoding

3 bits can represent up to 2³ = 8 distinct states. This FSM has
`NUM_STATES = 7` (states 0 through 6), so 3 bits is the minimum width
that can uniquely encode all states, with exactly one unused encoding
left spare.

## Schematic

<img width="960" height="1077" alt="Screenshot 2026-08-29 112033" src="https://github.com/user-attachments/assets/40851b84-43d4-4f73-9300-1b783078ac4a" />

# Stage 4 — Gate-Level Simulation & RTL vs GLS Comparison

## Commands

```bash
iverilog -o gls_sim.out synth_out.v tb/tb.v
vvp gls_sim.out
```

## Result

| Metric | RTL Simulation | GLS Simulation |
|---|---|---|
| First detection time | 1,771,000 ns | **991,000 ns** |
| Total detections | 4 | 4 |
| Target sequence match | 0010001 | 0010001 |

## Does GLS Match RTL?

**Not exactly.** Total detection count matches (4 in both), confirming
the core sequence-detection logic was synthesized correctly. However,
the *first detection time* differs substantially (991,000 ns vs
1,771,000 ns) — too large a gap to be simulator delta-cycle noise.

## Root Cause

From the synthesis stage: only the `detected` output register (`$302`,
`$_SDFF_PP0_`) has a reset (`R`) pin connected. The four flip-flops
holding the `state` register (`$308`–`$311`, all `$_DFF_P_`) have **no
reset connection** in the generic synthesized netlist. This means
`state` begins from an undefined value rather than being forced to 0 at
reset, causing the FSM to reach valid, stable state tracking at a
different point in simulation time than the RTL — shifting when the
target sequence is first detected.

## Final Conclusion

The synthesized netlist preserves the RTL's overall functional behavior
in terms of total detection count, confirming the core sequence-
detection logic was synthesized correctly. However, the first detection
event's timing differs between RTL and GLS due to the state register's
flip-flops lacking reset connectivity during generic synthesis. The
design is therefore **not yet fully sign-off ready** — reset
connectivity for the state register should be corrected (e.g. via
`dfflibmap` to a reset-capable standard cell library, or an explicit
reset branch in the RTL for the state register) before this netlist can
be considered fully equivalent to the RTL under all reset conditions.

Full supporting evidence, including waveform screenshots, is available
<img width="1512" height="806" alt="image" src="https://github.com/user-attachments/assets/c67c4dc4-75a0-4242-8a76-be9ded99e5b6" />

