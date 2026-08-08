# VLSI ALU Project — Sklansky Adder

A 4-bit ALU (`Y = A + B - C`) built full-custom in Cadence Virtuoso around a
4-bit **Sklansky parallel-prefix adder**, carried end-to-end from schematic
capture through physical layout, DRC/LVS signoff, and post-layout timing
characterization in Spectre. A second, independent exercise studies a CMOS
ring oscillator's frequency sensitivity to inverter sizing and loading.

**At a glance:** DRC clean · LVS clean · **4.33 GHz** f<sub>max</sub> @ 1.2 V ·
**2.05 GHz** f<sub>max</sub> @ 0.9 V · 52.29 µm² adder cell area · all
signal rise/fall times < 50 ps at both voltage corners.

📄 **[Read the full report](VLSI_ALU_Sklansky_Adder_Report.pdf)**

## What's here

- **`report.tex` / `VLSI_ALU_Sklansky_Adder_Report.pdf`** — the full
  writeup: Sklansky adder theory, gate-level floor plan and area budget,
  schematic capture (adder, subtractor, PG/Gray/Black cells, registers),
  physical layout, DRC/LVS signoff, functional verification (including a
  signed-overflow boundary test), critical-path and setup-time
  characterization at 1.2 V and 0.9 V, and the Part 2 ring-oscillator study.
- **`assets/`** — every schematic, layout, waveform, and measurement
  screenshot referenced in the report, extracted from the team's original
  Cadence session captures.

## Design summary

The datapath latches three 4-bit two's-complement inputs (`A`, `B`, `C`),
adds `A + B` with the Sklansky adder into a 5-bit register `X`, then
subtracts `C` from `X` (reusing the same adder via two's-complement
inversion) into a final 6-bit register `Y`. The Sklansky adder's
parallel-prefix carry tree (PG → Gray/Black cells) computes every carry in
$O(\log n)$ time rather than rippling bit-by-bit, at the cost of a bit more
wiring than a Kogge-Stone tree.

Signoff and characterization were done per-corner at the nominal 1.2 V
supply and a reduced 0.9 V corner, tracing the critical register-to-register
path through the subtractor's carry-out to determine setup time and
maximum clock frequency at each.

## Team

Amit Damari · Tomer Zahuri · Tal Franco · Israel Elmaliach

## Note on this writeup

This report was re-typeset from the group's original course submission for
portfolio use: a cycle-time arithmetic error at the 0.9 V corner was found
and corrected, figures were numbered/captioned throughout, and the
underlying schematic, layout, and simulation work is unchanged from the
original submission — see the report's Executive Summary for details.
