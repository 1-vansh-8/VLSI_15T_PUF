# 16×16 Dual-Mode Self-Healing SRAM PUF (28 nm, Cadence Monte Carlo)

**Author:** Vansh Goel · **Course:** EE 617: VLSI Design · **Reference design:** Chen *et al.*, IEEE TVLSI 2024 [1]

Simulation study of a **256-cell, 15-transistor, dual-mode SRAM physical unclonable function (PUF)** with
**voltage-tilt preselection and self-healing**. Designed and simulated in Cadence Virtuoso/Spectre with a 28 nm
HPC CMOS PDK (`l28hpc_mm_v1201.lib.scs`), nominal **VDD = 0.9 V**, local mismatch on (one Monte Carlo instance = one cell).

Full report (IEEE two-column): [`report/16x16_SRAM_PUF_Technical_Report.pdf`](report/16x16_SRAM_PUF_Technical_Report.pdf)

All numbers below come from the executed notebook [`notebooks/VLSI_project_SRAM_PUF.ipynb`](notebooks/VLSI_project_SRAM_PUF.ipynb);
the tables are in [`results/`](results/).

---

## Contents
1. [Headline results](#1-headline-results)
2. [Motivation](#2-motivation)
3. [The dual-mode cell](#3-the-dual-mode-cell)
4. [Voltage-tilt preselection and self-healing](#4-voltage-tilt-preselection-and-self-healing)
5. [Peripheral circuits](#5-peripheral-circuits)
6. [Timing](#6-timing)
7. [Cadence server directory structure](#7-cadence-virtuoso-server-design--simulation-directory-structure)
8. [Results](#8-results)
9. [Points to check in the data](#9-points-to-check-in-the-data)
10. [Limitations and next steps](#10-limitations-and-next-steps)
11. [Repository layout and how to run](#11-repository-layout-and-how-to-run)
12. [References](#12-references)

---

## 1. Headline results

| Quantity | Value |
|---|---|
| Mode 1 unstable cells, 27 °C, 0.9 V, \|Vd\| = 50 / 80 / 120 mV | 5.47 / 8.20 / 11.72 % |
| Healing efficiency (unstable cells repaired by Mode 2), 256-cell runs | 93.3–100 % |
| 32×64 run, ±120 mV: unstable → repaired → discarded | 211 → 191 → 20 (valid yield **99.02 %**) |
| Healed 2028-bit key Hamming weight | 54.49 % |
| Mode 1 vs. Mode 2 golden keys (256 bits) differ in | 125 bits (48.8 %) |
| Raw → healed BER, 0.9 V, ±120 mV (healing chosen at ±80 mV) | 5.86 % → 1.76 % (3.3×) |
| Valid-bit yield at −100 / −40 / 0 / 27 / 75 °C | 100 / 100 / 99.61 / 99.61 / 96.88 % |
| Mean raw Mode 1 tilt BER, −100 → 75 °C | 0.26 % → 6.12 % |
| Mean healed tilt BER (unmasked), −100 → 75 °C | 0.00 % → 0.91 % |
| Healed 128-bit BLER at 27 °C / 75 °C | 15.4 % / 69.0 % (raw: 99.6 % / 99.97 %) |

---

## 2. Motivation

A PUF turns random manufacturing variation into a device-unique key, so no key has to be stored in non-volatile
memory. In an SRAM PUF the two inverters of a latch race from a balanced state and mismatch picks the winner.
Small satellites need such low-cost keys, but supply, temperature and radiation change which way weak cells
settle, so the key drifts and the bit error rate (BER) rises [1].

Existing fixes are costly [1]: ECC (area), majority voting (power, time, throughput), hot-carrier burn-in (ageing,
test time), unstable-bit masking (test time, discarded cells), dark-bit detection (NVM overhead), self-check and
heal (not shown under radiation), a radiation-hardened 8T cell (BER still 2.68 %).

The approach of [1]: give each cell **two independent responses**, find weak cells **on chip** with a voltage tilt,
**heal** a weak Mode 1 bit with the Mode 2 bit, and discard only cells weak in both modes.

## 3. The dual-mode cell

<p align="center"><img src="docs/schematics/fig01_15T_multimode_bitcell_annotated.png" width="560"><br>
<em>15T multimode cell [1, Fig. 2(a)], blocks colour-coded in the Phase 0 presentation [2]; W/L from [1].</em></p>

| Block | Transistors | Role |
|---|---|---|
| 1 Cross-coupled core | M1–M4 | Two inverters (M1+M3, M2+M4); pMOS M1/M2 shared by both modes; M3/M4 = Mode 1 pull-downs |
| 2 Mode 2 pull-downs | M11, M12 | Second nMOS pair on the same Q/QB → second, independent bit |
| 3 Readout | M5, M6, M9, M10 | M5/M6 sense Q/QB, WL-gated M9/M10 connect to BL/BLb; reading does not load Q/QB |
| 4 Precharge | M7, M8 | Pull Q and QB to VDD when EN0 = EN1 = 0 |
| 5 Mode switch | OR, M13–M15 | EN1 → M13 (M3/M4), EN0 → M14 (M11/M12), OR(EN0, EN1) → footer M15 and M7/M8 |

| Mode | EN0 | EN1 | Discharge path |
|---|---|---|---|
| Mode 1 | 0 | 1 | Q/QB → M3/M4 → M13 → M15 → GND |
| Mode 2 | 1 | 0 | Q/QB → M11/M12 → M14 → M15 → GND |
| Sleep / precharge | 0 | 0 | none; Q = QB = VDD |

<p align="center">
<img src="docs/schematics/fig04_mode1_current_path.png" width="300">
<img src="docs/schematics/fig05_mode2_current_path.png" width="300"><br>
<em>Mode 1 (left) and Mode 2 (right) current paths [2].</em></p>

**One evaluation:** precharge (Q = QB = VDD) → release (one enable high, M7/M8 off) → race (both nodes fall,
mismatch makes one faster) → lock (positive feedback drives one node to 0, the other to 1).

<p align="center"><img src="docs/schematics/fig02_QQB_voltage_discrimination.png" width="380"><br>
<em>Q/QB voltage discrimination [1, Fig. 2(b)].</em></p>

## 4. Voltage-tilt preselection and self-healing

<p align="center">
<img src="docs/schematics/fig06_split_supply_cell_VDDL_VDDR.png" width="260">
<img src="docs/schematics/fig07_voltage_tilt_preselection_procedure.png" width="260"><br>
<em>Cell with split supplies VDDL/VDDR, and the two-way tilt test [1, Fig. 3].</em></p>

The inverter on the higher supply has its switching point pushed up and tends to output 1. Tilt one way
(VDDL = VDD + Vd, VDDR = VDD − Vd), then the other; if the bit is the same both times, the cell's mismatch beats
the tilt and it is **stable**. Then:

<p align="center"><img src="figures/report_bw/bw_selfhealing_flow.png" width="340"></p>

* **Stable**: Mode 1 never flips → keep Mode 1.
* **Repaired**: Mode 1 flips, Mode 2 never flips → use Mode 2.
* **Discarded (dark)**: both modes flip → mask.

**In this project** the bit is the mean `/BL` voltage in a 6 ns window compared with VDD/2 (0.45 V at 0.9 V):
Mode 2 window **37–43 ns**, Mode 1 window **97–103 ns** of a 0–125 ns transient. The maps compare nominal,
+tilt and −tilt; tilts are ±50, ±80, ±120 mV around VDD = 0.8, 0.9 and 1.0 V.

## 5. Peripheral circuits

Reference-design peripherals [1] as presented in Phase 0 [2]. The 28 nm project schematics are on the server (§7).

<p align="center"><img src="docs/schematics/fig08_array_system_architecture.png" width="560"><br>
<em>Test-chip architecture: PUF array, mode control + precharge, power-supply control and WL driver, read and latch, stability decision [1, Fig. 5].</em></p>

| Block | Figure | Function |
|---|---|---|
| Self-healing system | <img src="docs/peripherals/fig01_self_healing_system_block_diagram.png" width="300"> | Digital control drives precharge and the DAC; PUF_valid decides enroll or mask |
| Power control (7-bit DAC) | <img src="docs/peripherals/fig03_power_control_7bit_DAC.png" width="300"> | Capacitor array C…nC switched to VDD/VSS by A0…An sets VDDL/VDDR by charge redistribution; PCH pull-down can ground the rail |
| Key readout | <img src="docs/peripherals/fig04_key_readout_sense_amplifier.png" width="300"> | SA compares BL with VREF on SAE; delay buffer and stacked structure filter race spikes; latched PUF OUT |
| Readout timing | <img src="docs/peripherals/fig05_key_readout_timing.png" width="300"> | Precharge (PCH) → WL enable (BL ramps) → readout (SAE) |
| Stability decision | <img src="docs/peripherals/fig02_stability_decision_circuit_and_timing.png" width="300"> | PUF OUT clocks two D flip-flops (one through an inverter); any edge between the two tilted reads sets a flip-flop and the NOR pulls STABLE low |

## 6. Timing

![Single-cell timing diagram](docs/timing/single_cell_timing_diagram.png)

Cadence ViVA, single-cell functional run, 0–500 ns (`/home/himani/simulation/sram15T_test/spectre/schematic/psf`,
dataset `tran-tran`). The sequence repeats every 120 ns:

| Time (ns) | Signals | What happens |
|---|---|---|
| 0–10 | EN0 = EN1 = 0 | Precharge: Q and Qbar both high, BL/BLb high |
| ≈10 | EN1 ↑ | Release through M3/M4: Qbar falls, Q stays high (latch resolves) |
| 40–60 | WL high, EN1 high | Read: `/BL` discharges (Qbar = 0) |
| 60 / 70 | EN1 ↓, EN0 ↑ | Qbar returns high; with EN0 the cell re-evaluates through M11/M12 and Q falls |
| 70–90 | WL high, EN0 high | Second evaluation on the bit lines |
| 160–180, 280–300, 400–420 | WL and EN1 high | In later cycles `/BLb` discharges instead |

Signal levels reach about 1.0–1.1 V in this run, so it may not be at 0.9 V. An eighth, unlabelled trace follows EN1.

In the **Monte Carlo testbench** (0–125 ns) each cell is evaluated twice and the bit lines are sampled at
**37–43 ns (Mode 2)** and **97–103 ns (Mode 1)**, after each read has settled.

> **Mode-label check.** In the run above the first read happens while EN1 is high, which [1] calls Mode 1, whereas
> the notebook calls the first MC window Mode 2. Please confirm in the MC testbench stimuli. This only changes which
> mode is called "raw"; the healing statistics are otherwise the same.

## 7. Cadence Virtuoso server design & simulation directory structure

Transcribed verbatim from the screenshots (the second path wraps after `UMC28` in the original; there is no space).

| Purpose (as written) | Path on server |
|---|---|
| All the circuits used in the project | `/home/himani/Work/UMC28_test/cds/project/` |
| All the testing done on the SRAM cell | `/home/himani/Work/UMC28_test/cds/project_testing/sram15T_test/` |
| All the CSV files generated for the MC simulations | `/home/himani/simulation/project_testing/sram15T_test/` |
| Single-cell transient results (ViVA status bar) | `/home/himani/simulation/sram15T_test/spectre/schematic/psf` (dataset `tran-tran`) |

| From the ViVA window tab | Value |
|---|---|
| Library | `project_testing` |
| Cell | `sram15T_test` |
| View | `schematic` (tab text truncated as `sche…`) |
| Probed nets | `/BL`, `/BLb`, `/EN1`, `/WL`, `/I0/Qbar`, `/I0/Q`, `/EN0` (+ one unlabelled) |
| PDK model file | `l28hpc_mm_v1201.lib.scs` |

```text
/home/himani/
├── Work/
│   └── UMC28_test/
│       └── cds/
│           ├── project/                         # all circuits used in the project
│           └── project_testing/
│               └── sram15T_test/                # SRAM cell testbench (lib project_testing, cell sram15T_test)
└── simulation/
    ├── project_testing/
    │   └── sram15T_test/                        # Monte Carlo CSV exports (mc_*.csv)
    └── sram15T_test/
        └── spectre/
            └── schematic/
                └── psf/                         # single-cell transient results (tran-tran)
```

The screenshots show no cell names inside `project/` and no Maestro state paths.

## 8. Results

### 8.1 Logic balance (BL / BLb counts)

| Run | Mode 2 ones | Mode 1 ones |
|---|---|---|
| 0.9 V, −80 mV | 157 / 256 | 150 / 256 |
| 0.9 V, −50 mV | 153 | 147 |
| 0.9 V, +50 mV | 142 | 133 |
| 0.9 V, +80 mV | 140 | 129 |
| 1.0 V, 0 mV | 142 | 135 |
| 1.0 V, −80 / +80 mV | 155 / 134 | 147 / 124 |
| 2048 cells, 0.9 V, −120 / +120 mV | 1226 / 999 | 1205 / 994 |

BLb was the exact complement of BL in every run, so every cell resolved fully in both windows. The tilt acts in
one direction: −tilt (VDDL < VDDR) adds ones, +tilt removes them.

### 8.2 Preselection and self-healing at 27 °C

| VDD | \|Vd\| | Unstable M1 | Repaired | Discarded | Healing eff. | Valid yield |
|---|---|---|---|---|---|---|
| 0.8 V | 50 mV | 12 (4.69 %) | 12 | 0 | 100 % | 100 % |
| 0.8 V | 80 mV | 17 (6.64 %) | 17 | 0 | 100 % | 100 % |
| 0.8 V | 120 mV | 29 (11.33 %) | 29 | 0 | 100 % | 100 % |
| 0.9 V | 50 mV | 14 (5.47 %) | 14 | 0 | 100 % | 100 % |
| 0.9 V | 80 mV | 21 (8.20 %) | 21 | 0 | 100 % | 100 % |
| 0.9 V | 120 mV | 30 (11.72 %) | 29 | 1 | 96.67 % | 99.61 % |
| 1.0 V | 50 mV | 15 (5.86 %) | 14 | 1 | 93.33 % | 99.61 % |
| 1.0 V | 80 mV | 23 (8.98 %) | 22 | 1 | 95.65 % | 99.61 % |
| 1.0 V | 120 mV | 19 (7.42 %)† | 19 | 0 | 100 % | 100 % |
| **0.9 V, 32×64** | 120 mV | 211 (10.30 %) | 191 | 20 (0.98 %) | 90.52 % | 99.02 % |

† Inconsistent; see §9.

<p align="center"><img src="figures/notebook/nb18_map16_27C_VDD0.9_tilt80.png" width="680"><br>
<em>16×16 map, 0.9 V, ±80 mV: (a) raw, (b) after preselection, (c) after self-healing. Yellow unstable, green repaired, orange discarded, grey stable.</em></p>

<p align="center"><img src="figures/notebook/nb20_map16_27C_VDD0.9_tilt120.png" width="680"><br><em>0.9 V, ±120 mV.</em></p>

<p align="center"><img src="figures/notebook/nb30_map32x64_27C_VDD0.9_tilt120.png" width="760"><br><em>32×64 array, 0.9 V, ±120 mV.</em></p>

Other maps: [0.8 V ±50](figures/notebook/nb25_map16_27C_VDD0.8_tilt50.png) ·
[0.8 V ±80](figures/notebook/nb27_map16_27C_VDD0.8_tilt80.png) · [0.8 V ±120](figures/notebook/nb29_map16_27C_VDD0.8_tilt120.png) ·
[0.9 V ±50](figures/notebook/nb19_map16_27C_VDD0.9_tilt50.png) · [1.0 V ±50](figures/notebook/nb23_map16_27C_VDD1.0_tilt50.png) ·
[1.0 V ±80](figures/notebook/nb21_map16_27C_VDD1.0_tilt80.png) · [1.0 V ±120](figures/notebook/nb22_map16_27C_VDD1.0_tilt120.png).
Black-and-white versions of every map are in [`figures/report_bw/`](figures/report_bw/).

The unstable fraction rises with Vd (more cells have a mismatch smaller than the tilt). With 10.30 % of cells
unstable in Mode 1 and 11.08 % (227 cells, from the logic counts) in Mode 2 in the 2048-cell run, independent modes
would leave about 2048 × 0.103 × 0.111 ≈ 23 cells unstable in both; 20 were discarded. Together with the 48.8 %
distance between the two golden keys, this supports the claim that the two modes give independent bits.

### 8.3 Golden keys (27 °C, 0.9 V, 0 mV)

| Key | Hex | Ones |
|---|---|---|
| Mode 1 (97–103 ns) | `6CD4700EC8EA3BF8C95537B1C4FA3B0E768F2F738BCD9DAB41C9BF3732E5A732` | 139 (54.3 %) |
| Mode 2 (37–43 ns) | `75476AB764D4FB2AC3AF6D5636C75B77D34F2EB34C74ABEFEA6924D6AE735345` | 146 (57.0 %) |

Both keys, and the healed 2028-bit key (54.49 %), lean towards 1. For 2028 bits that is about four standard
deviations from 50 %, so it is a systematic bias (cell asymmetry, BL/BLb loading or the VDD/2 threshold).

### 8.4 BER against supply and tilt

Golden key: 0.9 V, 0 mV; healing chosen at 0.9 V, ±80 mV; BER averaged over +tilt and −tilt (cell 31). All in %.

| VDD | \|Vd\| | Raw M1 | Raw M2 | Self-healed | Healed + V mask | Healed + tilt & V mask | Unstable M1 | Unstable healed |
|---|---|---|---|---|---|---|---|---|
| 0.8 V | 0 | 2.34 | 1.95 | 0.00 | 0.00 | 0.00 | 2.34 | 0.00 |
| 0.8 V | 50 | 2.73 | 1.95 | 0.20 | 0.20 | 0.00 | 4.69 | 0.39 |
| 0.8 V | 80 | 3.32 | 3.32 | 0.78 | 0.78 | 0.00 | 6.64 | 1.56 |
| 0.8 V | 120 | 5.66 | 4.10 | 1.56 | 1.56 | 0.00 | 11.33 | 3.13 |
| 0.9 V | 0 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 |
| 0.9 V | 50 | 2.73 | 2.15 | 0.00 | 0.00 | 0.00 | 5.47 | 0.00 |
| 0.9 V | 80 | 4.10 | 3.32 | 0.00* | 0.00 | 0.00 | 8.20 | 0.00 |
| 0.9 V | 120 | 5.86 | 4.49 | 1.76 | 1.76 | 0.00 | 11.72 | 3.52 |
| 1.0 V | 0 | 1.56 | 1.56 | 0.00 | 0.00 | 0.00 | 1.56 | 0.00 |
| 1.0 V | 50 | 2.93 | 1.95 | 0.78 | 0.78 | 0.00 | 5.86 | 1.56 |
| 1.0 V | 80 | 4.49 | 4.10 | 0.98 | 0.98 | 0.00 | 8.98 | 1.95 |
| 1.0 V | 120† | 4.49 | 4.30 | 0.78 | 0.78 | 0.00 | 2.73 | 1.56 |

\* The condition used to choose the healing (zero by construction). The tilt-and-voltage mask is built from the
±80/±120 mV runs and evaluated on them, so its 0 % is also by construction (§10).

<p align="center"><img src="figures/notebook/nb31_ber_unstable_vs_tilt.png" width="700"></p>
<p align="center"><img src="figures/notebook/nb31_ber_vs_vdd.png" width="460"></p>
<p align="center"><img src="figures/notebook/nb31_ber_selfheal_masking_vs_tilt.png" width="760"></p>

* Outside the healing condition, self-healing lowers BER 3–4× (0.9 V ±120 mV: 5.86 → 1.76 %; 0.8 V ±120 mV:
  5.66 → 1.56 %), comparable to the 2.3–2.7× that [1] reports for healing alone after 100 krad(Si).
* With no tilt, the healed key has 0 % BER at 0.8, 0.9 and 1.0 V, against 2.34 % and 1.56 % raw at 0.8 V and 1.0 V.
* At 0.9 V the unstable fraction is exactly twice the BER: an unstable cell matches the golden key in one tilt
  direction and not the other.

### 8.5 Flip direction

<p align="center"><img src="figures/notebook/nb32_directional_flips_tilt_and_vdd_sweeps.png" width="760"></p>

Every flip follows the tilt sign: +tilt gives only 1→0, −tilt only 0→1 (e.g. 0.9 V: +120 mV → 14 flips 1→0,
−120 mV → 16 flips 0→1). Supply alone also biases the cell: at 0.8 V nominal all 6 errors are 0→1, at 1.0 V all 4
are 1→0. A symmetric tilt cannot reproduce the radiation-induced dominance of 1→0 flips reported in [1].

### 8.6 Temperature sweep (VDD = 0.9 V)

Bit classification with ±120 mV at each temperature (cell 33):

| T | Unstable M1 | Repaired | Discarded | Healing eff. | Valid yield | Healed-key HW |
|---|---|---|---|---|---|---|
| −100 °C | 3 (1.17 %) | 3 | 0 | 100 % | 100 % | 55.47 % |
| −40 °C | 15 (5.86 %) | 15 | 0 | 100 % | 100 % | 55.47 % |
| 0 °C | 27 (10.55 %) | 26 | 1 | 96.30 % | 99.61 % | 54.90 % |
| 27 °C | 30 (11.72 %) | 29 | 1 | 96.67 % | 99.61 % | 55.29 % |
| 75 °C | 44 (17.19 %) | 36 | 8 | 81.82 % | 96.88 % | 54.44 % |

BER against each temperature's own nominal key, all available tilts, healing from all tilts (cell 34):

| T | Drift vs. 27 °C (M1) | Mean raw M1 | Mean raw M2 | Mean healed | Raw M1 BLER-128 | Healed BLER-128 | Healed BLER-256 |
|---|---|---|---|---|---|---|---|
| −100 °C | 1.56 % | 0.26 % | 0.72 % | 0.00 % | 28.38 % | 0.00 % | 0.00 % |
| −40 °C‡ | 0.39 % | 1.72 % | 2.11 % | 0.00 % | 89.13 % | 0.00 % | 0.00 % |
| 0 °C | 0.00 % | 3.84 % | 3.12 % | 0.13 % | 99.34 % | 15.36 % | 28.36 % |
| 27 °C | 0.00 % | 4.23 % | 3.32 % | 0.13 % | 99.61 % | 15.36 % | 28.36 % |
| 75 °C | 0.00 % | 6.12 % | 5.79 % | 0.91 % | 99.97 % | 69.03 % | 90.41 % |

‡ No −50 mV run at −40 °C. BLER = 1 − (1 − BER)^N, independent errors assumed. With preselection the BER over the
remaining valid bits is 0 % at every temperature, by construction.

<p align="center"><img src="figures/notebook/nb33_yield_breakdown_vs_temperature.png" width="560"></p>
<p align="center"><img src="figures/notebook/nb34_ber_bler_vs_temperature.png" width="760"></p>
<p align="center"><img src="figures/notebook/nb34_tilt_ber_vs_temperature.png" width="760"></p>
<p align="center"><img src="figures/notebook/nb33_map16_T75C.png" width="680"><br><em>75 °C: 44 unstable, 36 repaired, 8 discarded.</em></p>

Maps at [−100 °C](figures/notebook/nb33_map16_T-100C.png) · [−40 °C](figures/notebook/nb33_map16_T-40C.png) ·
[0 °C](figures/notebook/nb33_map16_T0C.png) · [27 °C](figures/notebook/nb33_map16_T27C.png).

The same tilt flags 3 cells at −100 °C and 44 at 75 °C: the latch's regenerative gain falls with temperature, so
mismatch margins shrink relative to the tilt. Healing keeps up to 27 °C; at 75 °C 8 cells are unstable in both
modes. The nominal Mode 1 key does not change between 0 °C and 75 °C; it differs from 27 °C in 4 bits at −100 °C.

### 8.7 Process corners

TT, FF, SS, FNSP and SNFP results are **not in the notebook yet**. The corner runs can be analysed with
`scripts/03_preselection_map_16x16.py` (±120 mV) once exported.

## 9. Points to check in the data

1. **1.0 V, ±120 mV.** Fewer unstable cells (19) than at ±80 mV (23), all errors 0→1, and unstable bits (2.73 %)
   below BER (4.49 %); every other configuration behaves the opposite way. `mc_256_1.0_vdd+120.csv` appears to be a
   copy of `mc_256_1.0_vdd-80.csv` (same size and contents). Re-export the +120 mV run; excluded from conclusions.
2. **+50 mV at 0 °C and 75 °C.** The +50/−50 mV raw BERs are 4.30/1.17 % at 0 °C and 1.17/4.30 % at 75 °C, mirror
   images; the two +50 mV files appear swapped (their `temperature=` header tags read 75 and 0). The 0 °C and 75 °C
   means in §8.6 change slightly once corrected.
3. **Threshold.** Cells 11–15 and 21–29 use 0.45 V for the 0.8 V and 1.0 V files; `scripts/03` takes `--vdd`.
4. **Missing run.** No −40 °C, −50 mV run.

## 10. Limitations and next steps

* Each condition is one deterministic transient, so there is no temporal noise and the nominal BER is 0 by definition.
  Transient noise and temporal majority voting would add the noise-driven errors measured in [1].
* Supply tilt is a preselection stress, not a radiation model; a TID study needs Vth shifts on specific devices (Phase 0, E2).
* Masks evaluated on the conditions they were built from give 0 % by construction; hold out ±120 mV or a supply corner
  and report the masking ratio.
* One die: inter-die HD, autocorrelation and NIST tests need several independent arrays.
* Next: fix the runs in §9, process corners, 2048 cells at ±80 mV, Vd sweep vs. discard rate (E1), Vth-shift TID
  emulation (E2), bias-aware tilt (E3).

## 11. Repository layout and how to run

```text
16x16-sram-puf-repo/
├── README.md
├── requirements.txt
├── .gitignore                         # excludes mc_*.csv, __pycache__, .ipynb_checkpoints
├── data/raw/                          # put the raw mc_*.csv here (not tracked)
├── docs/
│   ├── schematics/                    # 15T cell, mode paths, tilt test, architecture (8 PNGs)
│   ├── peripherals/                   # self-healing system, stability decision, DAC, sense amp, timing (5 PNGs)
│   ├── timing/                        # single_cell_timing_diagram.png (+ print version)
│   └── server/                        # cropped server-path screenshot
├── notebooks/VLSI_project_SRAM_PUF.ipynb    # original notebook, outputs preserved, annotated
├── scripts/
│   ├── 01_logic_counts_256.py         # notebook cells 1-15
│   ├── 02_logic_counts_2048.py        # cell 17
│   ├── 03_preselection_map_16x16.py   # cells 18-29 (any VDD / tilt via arguments)
│   ├── 04_preselection_map_32x64.py   # cell 30
│   ├── 05_golden_key_ber_masking_summary.py   # cell 31
│   ├── 06_directional_flips.py        # cell 32
│   ├── 07_temperature_maps_and_yield.py       # cell 33
│   ├── 08_temperature_ber_bler.py     # cell 34
│   ├── make_bw_maps.py                # B&W versions of the notebook maps (report)
│   ├── make_bw_plots.py               # B&W plots from results/*.csv (report)
│   └── run_all.sh
├── figures/
│   ├── notebook/                      # all 24 notebook figures (colour)
│   └── report_bw/                     # black-and-white figures used in the PDF
├── results/                           # 8 summary CSVs transcribed from the notebook outputs
└── report/                            # IEEEtran LaTeX source + PDF
```

```bash
pip install -r requirements.txt
cp /path/to/mc_*.csv data/raw/
bash scripts/run_all.sh                # or run one script, e.g.
python scripts/03_preselection_map_16x16.py --data data/raw --vdd 0.8 \
       --nominal mc_256_0.8_vdd.csv --plus mc_256_0.8_vdd+120.csv --minus mc_256_0.8_vdd-120.csv
python scripts/make_bw_plots.py        # report figures; needs only results/
```

## 12. References

1. Z. Chen, W. Yang, J. Chen, Z. Wang, and D. Ding, "Improving radiation reliability of SRAM-based physical
   unclonable function with self-healing and pre-irradiation masking techniques," *IEEE Trans. VLSI Syst.*,
   vol. 32, no. 2, pp. 372–381, Feb. 2024, doi: 10.1109/TVLSI.2023.3332010.
2. V. Goel and H. Goel, "Improving radiation reliability of SRAM-based PUF with self-healing and pre-irradiation
   masking techniques," EE 617 paper presentation (Phase 0), 9 Oct. 2026.
