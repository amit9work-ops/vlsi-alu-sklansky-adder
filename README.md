# VLSI ALU Project · Sklansky Adder

A 4-bit ALU (`Y = A + B - C`) built full-custom in Cadence Virtuoso around a
4-bit **Sklansky parallel-prefix adder**, carried end-to-end from schematic
capture through physical layout, DRC/LVS signoff, and post-layout timing
characterization in Spectre across two supply corners. A second, independent
exercise studies a CMOS ring oscillator's frequency sensitivity to inverter
sizing and loading.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Tool](https://img.shields.io/badge/Tool-Cadence%20Virtuoso-0068B5)](https://www.cadence.com/en_US/home/tools/custom-ic-analog-rf-design/layout-design/virtuoso-layout-suite.html)
[![Simulator](https://img.shields.io/badge/Simulator-Spectre-0068B5)]()
[![DRC](https://img.shields.io/badge/DRC-clean-1A7A4A)]()
[![LVS](https://img.shields.io/badge/LVS-clean-1A7A4A)]()
[![fmax @ 1.2V](https://img.shields.io/badge/fmax%20%40%201.2V-4.27%20GHz-E67E22)]()
[![fmax @ 0.9V](https://img.shields.io/badge/fmax%20%40%200.9V-2.04%20GHz-E67E22)]()

<p align="center">
  <img src="assets/layout_zoom2.png" alt="Colorful standard-cell layout detail of the 4-bit Sklansky adder in Cadence Virtuoso, showing metal routing and vias across the PG/Gray/Black cell rows" width="880">
</p>

📄 **[Read the full report (PDF)](VLSI_ALU_Sklansky_Adder_Report.pdf)**: 31 pages, fully captioned, with every schematic, layout, and measurement screenshot.

## At a Glance

**4.27 GHz** f<sub>max</sub> @ 1.2 V · **2.04 GHz** f<sub>max</sub> @ 0.9 V ·
**52.29 µm²** adder cell area · **0** DRC violations · **0** LVS mismatches ·
all signal rise/fall times **< 50 ps** at both corners

The datapath latches three 4-bit two's-complement inputs, adds two of them
with a parallel-prefix adder, then subtracts the third by reusing that exact
same adder hardware. It's a small design, but it still went through a full
signoff flow: schematic, layout, DRC/LVS, and post-layout timing, at two
voltage corners.

## Architecture

<p align="center">
  <img src="assets/diagram_alu_dataflow.png" alt="ALU dataflow: A and B registers feed the Sklansky adder into a 5-bit X register; C register and X feed the subtractor into the final 6-bit Y register" width="820">
</p>

`A` and `B` are latched and added by the Sklansky adder into a 5-bit register
`X`; `C` is latched and subtracted from `X` by the *same* adder hardware
(reused via two's-complement inversion), producing the final 6-bit result `Y`.

| Block | Role |
|---|---|
| `4bit_register` | Input latches for `A`, `B`, `C` |
| `Adder` | 4-bit Sklansky parallel-prefix adder |
| `5bit_register` | Latches the partial sum `X = A + B` |
| `SUBTRACTOR` | Reuses the adder via two's-complement inversion for `X - C` |
| `6bit_register` | Latches the final signed result `Y` |

<p align="center">
  <img src="assets/diagram_subtractor_reuse.png" alt="The subtraction trick: C's bits are inverted through a bank of XOR-with-1 gates and fed into the same adder used for A+B, with the carry-in tied high to complete two's-complement negation, computing X - C with zero extra adder logic" width="720">
</p>

The subtractor is a clean example of hardware reuse. Rather than building a
second arithmetic unit, `C`'s bits are inverted and fed into the *exact same*
adder used for `A + B`, with the carry-in tied high. That gives
`X + C̄ + 1 = X − C` for free, with zero extra adder logic.

Both registers between the adder and subtractor are sized one bit wider than
the strict minimum, so intermediate results never wrap silently: the largest
possible sum (`0111 + 0111 = 14`) needs 5 bits, and the full `A + B − C`
range (`−23` to `+22`) needs 6 bits.

## The Sklansky Adder

<p align="center">
  <img src="assets/diagram_sklansky_tree.png" alt="4-bit Sklansky parallel-prefix tree with every wire labeled by signal: C1 comes directly from PG0 with no combining cell, C2 from a single Gray cell, and C3/C4 (carry-out) each from a second-stage Gray cell fed by the first stage's fan-out" width="760">
</p>

A **parallel prefix tree** computes every carry $C_i$ in $O(\log n)$ time
instead of rippling bit-by-bit through the word. The diagram above is traced
against the group's own gate-level netlist (`out_gray_1`, `out_gray_2`,
`out_gray_3` in [`assets/adder_schematic.png`](assets/adder_schematic.png)).
Worth noticing: $C_1$ needs **no cell at all**, since bit 0's generate signal
*is* the carry into bit 1. So the tree's worst-case depth for a 4-bit word
is really 2 stages (for $C_3$ and $C_4$), not a uniform 2 stages for every
carry. The fan-out from the level-1 Gray cell into *both* level-2 cells is
what keeps that depth logarithmic instead of linear.

- **PG cell** — generates $P_i = A_i \oplus B_i$ and $G_i = A_i \cdot B_i$ per bit.
- **Black cell** — combines two adjacent group signals: $G_{group} = G_{left} + (P_{left} \cdot G_{right})$, $P_{group} = P_{left} \cdot P_{right}$.
- **Gray cell** — an optimized black cell producing only the group *generate* signal, trimming area and wiring where the group propagate is never needed downstream.

| PG cell | Gray cell | Black cell |
|---|---|---|
| <img src="assets/pg_cell.png" width="240"> | <img src="assets/gray_cell.png" width="240"> | <img src="assets/black_cell.png" width="240"> |
| 1× XOR + 1× AND | 1× AND + 1× OR | 2× AND + 1× OR |

For 4 bits this tree is only **2 levels deep**: faster than a ripple-carry
adder's $O(n)$ delay, with less wiring than a Kogge-Stone tree of the same
width.

**Gate-level area budget** (measured from the actual standard cells used):

| Cell | Count | Area each (µm²) | Total (µm²) |
|---|---|---|---|
| AND | 9 | 2.128 | 19.152 |
| OR | 4 | 1.748 | 6.992 |
| XOR | 8 | 3.268 | 26.144 |
| **Total** | **21** | — | **52.288** |

<p align="center">
  <img src="assets/chart_area_breakdown.png" alt="Proportional area breakdown: XOR gates take half the total adder area at 50%, AND gates 37%, OR gates 13%, despite AND having more individual gate instances than XOR" width="720">
</p>

XOR gates dominate the area at half the total, despite AND contributing more
individual instances (9 vs. 8): every PG cell needs one, and so does every
final sum bit.

## Physical Layout

<p align="center">
  <img src="assets/adder_layout_screenshot.jpeg" alt="Full 4-bit Sklansky adder layout open in Virtuoso Layout Suite XL" width="820">
</p>
<p align="center">
  <img src="assets/layout_zoom3.png" alt="Transistor-level zoom of the adder layout, showing the out_gray and out_black internal routing nets" width="820">
</p>

Two rows of `4bit_adder_pg` / `4bit_adder_gray_cell` / `4bit_adder_black_Cell`
cells feed a final `XOR2X1` row that produces the sum bits. The layout
mirrors the gate-level floor plan directly.

The initial floor plan padded the raw 52.29 µm² gate sum by ~20% (to
~62.75 µm²) to leave room for inter-block routing. Once the layout was
actually routed, the real cell area came in at **~50.96 µm² (~97.5% of the
unpadded estimate)**, under the raw number entirely, because several VSS
power rails ended up shared between adjacent blocks instead of duplicated
per block.

## Verification: DRC & LVS

<p align="center">
  <img src="assets/lvs_check.jpeg" alt="LVS run confirming schematic and layout netlists match with zero errors" width="500">
  <img src="assets/drc_check2.jpeg" alt="DRC run: Design Rule Check Finished Normally, zero total DRC results" width="500">
</p>

Design Rule Checking confirms every wire and device satisfies the
fabrication process's geometric rules; Layout-Versus-Schematic extracts the
transistor-level netlist from the layout and diffs it against the original
schematic. **Both passed clean: 0 violations, 0 netlist mismatches.**

## Functional Verification

<p align="center">
  <img src="assets/overflow_check_annotated.png" alt="Overflow boundary test waveform: three annotated clock edges mark A/B latching, the partial sum X latching, and the final result Y latching" width="820">
</p>

The adder and subtractor were verified independently against dedicated
testbenches before exercising the full ALU end-to-end, including a
signed-overflow boundary test at the design's largest and smallest
representable results (−23 and +22). The three annotated clock edges above
mark: **1st** — `A`/`B` latched into their input registers; **2nd** — the
partial sum latched into `X`; **3rd** — the final result, after subtracting
`C`, latched into `Y`.

Overflow itself is flagged by comparing the carry into the MSB against the
carry out of it: if the two disagree, the result doesn't fit in the target
width. Equivalently, it's whenever two same-sign operands produce a
result of the opposite sign.

## Timing: Critical Path & Voltage Impact

<p align="center">
  <img src="assets/chart_fmax_vs_vdd.png" alt="Stacked bar chart showing the cycle-time budget at each voltage corner: setup time plus combined logic delay and clock-to-Q, stacked to total 234ps at 1.2V (4.27 GHz) and 489ps at 0.9V (2.04 GHz)" width="520">
</p>

Each bar above is literally $T_{cyc} = T_{setup} + (t_{cq} + t_{sub,max})$,
stacked to show where the time actually goes. Both components grow at the
lower supply, not just the logic delay: reduced drive current slows the
flip-flops themselves too. That combined growth is what nearly doubles the
cycle time.

Setup time wasn't taken from a library value. It was measured directly in
Spectre by sweeping the data-to-clock delay until the flip-flop just barely
stopped capturing correctly: a coarse 0–200 ps sweep at 1.2 V, refined to
0–50 ps, then to single-picosecond resolution between 15–25 ps. The 0.9 V
corner was located the same way, by bisection between −30 and −50 ps.

| Corner | Setup time | Combined logic + t<sub>CQ</sub> | Cycle time | f<sub>max</sub> |
|---|---|---|---|---|
| 1.2 V | ≈20 ps | 0.2143 ns | 0.2343 ns | **4.27 GHz** |
| 0.9 V | ≈44 ps | 0.4453 ns | 0.4893 ns | **2.04 GHz** |

The logic-delay figure in each row is the *worst case* of the subtractor's
rising and falling carry-out transitions, not their average: correctness on
every cycle can't depend on which transition happens to occur, so the
slower one sets the budget.

The voltage sensitivity follows directly from the familiar delay relation
$t = C \Delta V / I_{AVG}$: a lower $V_{DD}$ reduces the average drive
current available to charge and discharge the same node capacitance, so
every gate slows down and $f_{max}$ drops with it.

Every signal's rise and fall time (10–90%) was also measured at both
corners. The slowest signal in the design ($X[4{:}0]$ at 0.9 V, ~31 ps)
still clears the 50 ps requirement with about 19 ps (38%) to spare.

## Part 2 — Ring Oscillator Clock Generation

<p align="center">
  <img src="assets/ring_osc_ring_schematic.png" alt="Five-stage INXV1 ring oscillator: each stage is an inverter followed by a tunable series resistor and shunt capacitor to ground" width="700">
</p>

An independent exercise in the same report: a 5-stage inverter ring
oscillator, exploring how frequency trades off against inverter sizing and
output loading.

| Question | Finding |
|---|---|
| Bigger inverters? | Higher drive strength → lower per-stage delay → **higher** $f_{osc}$ |
| Added RC chains? | More load per stage → **lower** $f_{osc}$ |
| Output capacitor added? | $f_{osc}$: 2.5 MHz → 2.46 MHz; rail-to-rail swing narrowed, edges softened |
| Switched INXV1 → INXV2? | Full swing restored, sharp edges, $f_{osc}$ **increased** above the loaded baseline |

## Repository Structure

```
├── report.tex                          LaTeX source for the full report
├── VLSI_ALU_Sklansky_Adder_Report.pdf  Compiled 31-page report
├── assets/                             Every schematic, layout, and waveform figure
└── README.md
```

## Team

Amit Damari · Tomer Zahuri · Tal Franco · Israel Elmaliach

## License

MIT — see [LICENSE](LICENSE).
