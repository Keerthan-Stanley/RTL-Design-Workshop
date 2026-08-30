# Stage 2 — Synthesis

## Purpose

Synthesis is the mandatory transformation from behavioral RTL into a
gate-level netlist made of real, physical standard cells from the Sky130
PDK. This is not an optional verification step — it's how RTL code
actually becomes hardware. Every digital design must go through this
translation before it can be fabricated.

## Block Diagram (Post-Synthesis, Pre-Flatten)

<img width="937" height="227" alt="Screenshot 2026-08-26 193734" src="https://github.com/user-attachments/assets/d84662d4-92d3-4c26-a607-c5e71212bc9c" />


`vsdbabysoc` integrates three sub-blocks: the PLL (`avsdpll`) generates
`CLK` from a reference input, the RVMYTH core (`rvmyth`) is clocked by
that signal and produces `RV_TO_DAC`, and the DAC (`avsddac`) converts
that digital output into the final `OUT` signal.

## Synthesis Script

Run inside Yosys (`yosys` to launch, then paste the block below):

```tcl
read_verilog src/module/vsdbabysoc.v
read_verilog -I src/include src/module/rvmyth.v
read_verilog -I src/include src/module/clk_gate.v
read_liberty -lib src/lib/avsdpll.lib
read_liberty -lib src/lib/avsddac.lib
read_liberty -lib src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
synth -top vsdbabysoc
dfflibmap -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
opt
abc -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
flatten
setundef -zero
clean -purge
rename -enumerate
stat
write_verilog -noattr src/module/vsdbabysoc.synth.v
```

## What Each Pass Does

| Command | Purpose |
|---|---|
| `read_verilog` | Load RTL source files |
| `read_liberty` | Load Sky130 standard cell timing/behavior models |
| `synth -top` | Elaborate design hierarchy, generic logic synthesis |
| `dfflibmap` | Map generic flip-flops to real Sky130 sequential cells |
| `opt` | Logic optimization — simplify redundant/constant logic |
| `abc` | Technology mapping — map combinational logic to real Sky130 cells |
| `flatten` | Collapse module hierarchy (pll/core/dac) into one flat netlist |
| `clean -purge` | Remove unused wires and cells, including any exposed by flattening |
| `write_verilog` | Save the final gate-level netlist to disk |

## Result — Cell Count Summary


| Cell type | Count |
|---|---|
| `sky130_fd_sc_hd__dfxtp_1` (real DFF) | 1,144 |
| `sky130_fd_sc_hd__nand2_1` | 1,269 |
| `avsddac` (black-box instance) | 1 |
| `avsdpll` (black-box instance) | 1 |

## Bonus: Isolated RVMYTH Synthesis

To understand which sub-block drives the SoC's complexity, the RVMYTH
core was also synthesized standalone (outside the full SoC context):

```tcl
read_verilog -I src/include src/module/rvmyth.v
read_verilog -I src/include src/module/clk_gate.v
read_liberty -lib src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
synth -top rvmyth
dfflibmap -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
opt
abc -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
stat
write_verilog -noattr rvmyth_synth.v
```

**Interesting finding:** RVMYTH standalone maps to **1,273** real DFF cells
(`dfxtp_1`), but in the full `vsdbabysoc` context, only **1,144** DFF cells
remain. This confirms that `flatten` + `opt` + `clean -purge`, when run at
full-SoC scope, find additional cross-module optimization opportunities
that aren't visible when synthesizing a sub-block in isolation.

Confirms RVMYTH alone accounts for the vast majority of the SoC's total
cell count (~5,000 of 5,285 cells) — consistent with it being a full
RISC-V core, while the PLL and DAC remain simple black-box instances at
this level (their internals are described via `.lib` timing models, not
further synthesized into gates).
