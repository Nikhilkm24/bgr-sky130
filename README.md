<div align="center">

# Bandgap Reference Circuit Design
## SkyWater Sky130 PDK · Open-Source EDA

![PDK](https://img.shields.io/badge/PDK-SkyWater%20Sky130-blue)
![Tool](https://img.shields.io/badge/Simulator-ngspice-brightgreen)
![Layout](https://img.shields.io/badge/Layout-Magic%20VLSI-orange)
![LVS](https://img.shields.io/badge/LVS-Netgen-red)
![License](https://img.shields.io/badge/License-MIT-yellow)

**A fully open-source implementation of a 1.2 V temperature-stable Bandgap Reference — from first principles to LVS-verified layout — using the SkyWater 130nm process.**

</div>

---

## Quick Results

| Parameter | Result | Spec |
|-----------|--------|------|
| V_REF (TT, 27°C) | **1.2358 V** | ~1.2 V |
| Tempco (ideal op-amp) | **33–35 ppm/°C** | < 50 ppm/°C ✅ |
| Tempco (FF corner) | **~10 ppm/°C** | < 50 ppm/°C ✅ |
| Tempco (SS corner) | **~45 ppm/°C** | < 50 ppm/°C ✅ |
| Startup time | **< 1 µs** | < 2 µs ✅ |
| LVS | **✅ Pass** | Pass |

---

## Table of Contents

1. [Why a Bandgap Reference?](#1-why-a-bandgap-reference)
2. [The Core Idea — CTAT + PTAT Cancellation](#2-the-core-idea--ctat--ptat-cancellation)
3. [Circuit Architecture](#3-circuit-architecture)
4. [Design Decisions — The Why](#4-design-decisions--the-why)
5. [Device Sizing and Specifications](#5-device-sizing-and-specifications)
6. [Tools and PDK Setup](#6-tools-and-pdk-setup)
7. [SPICE Netlist Guide](#7-spice-netlist-guide)
8. [Pre-Layout Simulation Results](#8-pre-layout-simulation-results)
9. [Physical Layout](#9-physical-layout)
10. [LVS Verification](#10-lvs-verification)
11. [Key Takeaways](#11-key-takeaways)
12. [References](#12-references)

---

## 1. Why a Bandgap Reference?

A Bandgap Reference (BGR) is one of the most fundamental blocks in analog IC design — virtually every mixed-signal SoC contains one. Its job is to produce a precise, stable DC voltage that does not change with temperature, supply voltage, or process variation.

![BGR Applications](Circuits/BGR_application_1.png)
![BGR Applications](Circuits/BGR_application_2.png)

Every block that needs accuracy — LDOs, ADCs, DACs, PLLs, DC-DC converters — traces its precision back to a BGR. Getting the BGR wrong degrades the accuracy of everything downstream.

The challenge is that every on-chip voltage source has some temperature dependence:

| Source | Tempco | Problem |
|--------|--------|---------|
| Resistor divider from VDD | 0 ppm/°C (of VDD) | 100% supply sensitivity |
| Single forward-biased diode | ~−3142 ppm/°C | Too temperature sensitive |
| Zener diode | Low, but needs special process | Not available in standard CMOS |
| **Bandgap Reference** | **10–50 ppm/°C** | ✅ Best of all worlds |

**The insight:** Instead of fighting temperature dependence, use two voltages with *opposite* temperature slopes and cancel them.

---

## 2. The Core Idea — CTAT + PTAT Cancellation

![BGR Principle](Circuits/BGR_principle.png)

The BGR exploits two physical quantities that exist in every BJT:

**CTAT — V_BE decreases with temperature (~−2 mV/°C)**

The base-emitter voltage of a BJT biased at constant current follows:
```
V_BE(T) ≈ V_G0 − (V_G0 − V_BE0) × (T/T0) + higher order terms
```
At room temperature this simplifies to approximately **−2 mV/°C** — the dominant CTAT term.

**PTAT — ΔV_BE increases with temperature (+0.085 mV/°C × ln N)**

When two BJTs operate at different current densities (emitter area ratio N):
```
ΔV_BE = V_T × ln(N)    where V_T = kT/q
```
Since V_T = kT/q is directly proportional to absolute temperature, ΔV_BE rises linearly — **pure PTAT**.

**Cancellation gives V_REF ≈ 1.2 V**

```
V_REF = V_BE + (R2/R1) × V_T × ln(N)

Setting dV_REF/dT = 0:

R2/R1 = |dV_BE/dT| / [ln(N) × k/q]
       = 2 mV/°C / [ln(8) × 0.0857 mV/°C]
       ≈ 8
```

The resulting V_REF converges to ~1.205 V — the silicon bandgap energy extrapolated to 0 K.

> **Intuition:** At 0 K, all thermal energy disappears — V_T → 0 so the PTAT term vanishes, leaving only V_BE(0K) = V_G0 ≈ 1.205 V. The cancellation condition forces V_REF to exactly this value regardless of temperature.

---

## 3. Circuit Architecture

### 3.1 CTAT Voltage Generation

![CTAT Concept](Circuits/CTAT_Concept.png)

A diode-connected PNP BJT (`sky130_fd_pr__pnp_05v5_W3p40L3p40`) biased at a constant current gives a falling V_BE of approximately **−1.88 mV/°C** — the CTAT component.

**Why PNP and not a simple diode?**
The PNP BJT provides better matching, lower noise, and a more predictable temperature coefficient than a plain diode. Sky130 provides a native PNP device optimised for exactly this application.

---

### 3.2 PTAT Voltage Generation

Two branches carry equal currents, but Q2 has 8× the emitter area of Q1. This creates a differential V_BE across resistor R1 that is purely proportional to temperature:

```
V_R1 = ΔV_BE = V_T × ln(8) = 26 mV × 2.079 = 54 mV  (at 300 K)
```

**Why N = 8?**
N = 8 gives a well-matched PTAT slope, fits neatly in a 2×4 common-centroid BJT array, and produces moderate resistor values — the optimal balance between slope, layout area, and current consumption.

---

### 3.3 Self-Biased Current Mirror (SBCM)

Instead of referencing bias current to a resistor from VDD (100% supply sensitivity), the SBCM bootstraps I_out to define I_REF. Since both currents come from the same mirror, VDD changes affect them identically and cancel — giving first-order supply independence.

**The zero-current problem:** I = 0 also satisfies the SBCM equations. This is the degenerate bias point — the mirror cannot self-start from zero current. A startup circuit is mandatory.

---

### 3.4 Reference Branch

A third PMOS MP3 mirrors the bias current into the reference branch: R2 in series with Q3:

```
V_REF = V_BE(Q3) + I × R2  =  CTAT + PTAT
```

Both R1 and R2 use the same RPOLYH device, so their tempcos track — the ratio R2/R1 is nearly temperature-independent, making V_REF insensitive to resistor tempco.

---

### 3.5 Startup Circuit

![Startup Circuit](Circuits/Startup_circuit.png)

**Phase 1 — Power-on kick:**
At t = 0, all currents = 0. MP4 pulls net6 toward VDD. MP5 detects the zero-current state and injects current into net1, turning on MN1/MN2 and kicking the SBCM into its correct operating point.

**Phase 2 — Auto turn-off:**
Once V_REF is established, net6 drops to its correct bias level → MP5 turns off automatically → the startup path disconnects completely and never disturbs V_REF in steady state.

**Why long-channel NMOS (L = 7 µm) for startup?**
Long L keeps off-state leakage extremely low — the startup path draws negligible current once the circuit is running.

---

### 3.6 Complete Circuit

![Complete BGR Circuit](Circuits/Complete_BGR.png)

![Final Schematic with Component Values](Circuits/Schematic_Final.png)

| Block | Devices | Role |
|-------|---------|------|
| Startup | MP4, MP5, MN3, MN4 | Kick SBCM out of zero-current state |
| SBCM | MP1, MP2, MN1, MN2 | Supply-independent current generation |
| CTAT branch | Q1 (1× BJT) | Falling V_BE provides CTAT |
| PTAT branch | R1 (5 kΩ), Q2 (8× BJT) | Rising ΔV_BE provides PTAT |
| Reference branch | MP3, R2 (33 kΩ), Q3 (1× BJT) | Sums CTAT + PTAT → V_REF |

---

## 4. Design Decisions — The Why

| Decision | Choice | Reason |
|----------|--------|--------|
| BJT emitter ratio N | **8** | Maximises PTAT slope with compact 2×4 common-centroid array |
| PMOS channel length | **L = 2 µm** | Long L reduces channel-length modulation → better mirror accuracy |
| NMOS channel length | **L = 1 µm** | Sub-threshold operation; long enough to avoid short-channel effects |
| Startup NMOS length | **L = 7 µm** | Extremely low off-state leakage — startup path fully invisible in steady state |
| Resistor type | **RPOLYH** | Lowest mismatch, predictable tempco, available in Sky130 |
| R1 and R2 same device | **Yes** | Tempcos cancel in the ratio — V_REF insensitive to resistor tempco |
| PMOS multiplicity | **M = 4** | Four fingers → better matching and lower systematic current mismatch |
| VDD | **2.0 V** | Sufficient headroom for PMOS saturation + V_BE + R1 voltage drop |

---

## 5. Device Sizing and Specifications

### Design Specifications

| Parameter | Target | Achieved |
|-----------|--------|---------|
| Supply Voltage | 2.0 V | 2.0 V |
| V_REF | ~1.2 V | 1.2358 V ✅ |
| Tempco | < 50 ppm/°C | 33–35 ppm/°C ✅ |
| Temperature Range | −40°C to +125°C | −40°C to +125°C ✅ |
| Startup Time | < 2 µs | < 1 µs ✅ |
| LVS | Pass | ✅ Pass |

### Sky130 Devices Used

| Device | Model | Parameters |
|--------|-------|-----------|
| PNP BJT | `sky130_fd_pr__pnp_05v5_W3p40L3p40` | Area = 11.56 µm², β ≈ 12 |
| PFET LVT | `sky130_fd_pr__pfet_01v8_lvt` | L=2µm, W=5µm, M=4 |
| NFET LVT | `sky130_fd_pr__nfet_01v8_lvt` | L=1µm, W=5µm, M=8 |
| RPOLYH | `sky130_fd_pr__res_high_po_1p41` | W=1.41µm, L=7.8µm → ~2 kΩ/unit |

### Component Values

| Component | Value | Implementation |
|-----------|-------|----------------|
| R1 | 5 kΩ | 2 units series + 2 parallel |
| R2 | 33 kΩ | 16 units series + 2 parallel |
| Q1 | 1× BJT | Single unit cell |
| Q2 | 8× BJT | 8 units in parallel (common-centroid array) |
| Q3 | 1× BJT | Single unit cell |

---

## 6. Tools and PDK Setup

| Tool | Purpose |
|------|---------|
| **ngspice** | Pre-layout SPICE simulation |
| **Magic VLSI** | Physical layout and DRC |
| **Netgen** | LVS verification |
| **SkyWater Sky130 PDK** | Process design kit |

### Installation

```bash
# ngspice
sudo apt-get install ngspice

# Magic VLSI
wget http://opencircuitdesign.com/magic/archive/magic-8.3.32.tgz
tar xvfz magic-8.3.32.tgz && cd magic-8.3.32
./configure && sudo make && sudo make install

# Netgen
git clone git://opencircuitdesign.com/netgen
cd netgen && ./configure && sudo make && sudo make install

# SkyWater Sky130 PDK
git clone https://github.com/google/skywater-pdk-libs-sky130_fd_pr.git
git clone https://github.com/silicon-vlsi-org/eda-technology.git
```

### Design Flow

```
First Principles  →  CTAT / PTAT characterization
       ↓
Ideal BGR (VCVS)  →  Concept validation  (33–35 ppm/°C achieved)
       ↓
Complete BGR      →  SBCM + startup circuit
       ↓
Process Corners   →  FF / TT / SS verification
       ↓
Physical Layout   →  Magic VLSI  (common-centroid, guard rings)
       ↓
LVS Verification  →  Netgen  ✅ Pass
```

---

## 7. SPICE Netlist Guide

### Key Rules for Sky130 Netlists

| Rule | Detail |
|------|--------|
| Line 1 is always a comment | ngspice treats it as the title — never as a component |
| `X` prefix on all Sky130 devices | All devices are subcircuit calls |
| `.end` is mandatory | Netlist is silently ignored without it |
| Corner = one `.lib` tag change | Everything else in the netlist stays identical |

### Device Instantiation Syntax

```spice
* PNP BJT — Collector  Base  Emitter
xqp1  gnd  net1  net1  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1

* PFET — Drain  Gate  Source  Bulk
xmp1  net1  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4

* NFET — Drain  Gate  Source  Bulk
xmn1  net2  net3  gnd  gnd  sky130_fd_pr__nfet_01v8_lvt  l=1 w=5 m=8

* RPOLYH Resistor — Node+  Node-
xra1  net3  net4  sky130_fd_pr__res_high_po_1p41  l=7.8 w=1.41 m=1
```

### Example: CTAT Single BJT (`ctat_voltage_gen.sp`)

```spice
* CTAT Voltage Generation — Single BJT temperature sweep

.lib "/path/to/eda-technology/sky130/libs/sky130.lib.spice" tt
.global gnd vdd
vdd vdd gnd 2.0

isrc  vdd  net1  dc  10u
xqp1  gnd  net1  net1  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1

.dc temp -40 125 5

.control
  run
  plot v(net1)
.endc

.end
```

### Example: Complete BGR (`bgr_lvt_rpolyh_3p40.sp`)

```spice
* Bandgap Reference — Self-Biased Current Mirror (TT corner)

.lib "/path/to/eda-technology/sky130/libs/sky130.lib.spice" tt
.global gnd vdd
vdd vdd gnd 2.0

* ── PMOS Current Mirror ──────────────────────────────────────────
xmp1  net1  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4
xmp2  net2  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4
xmp3  vref  net2  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=4

* ── NMOS Source Degeneration ─────────────────────────────────────
xmn1  net1  net1  gnd  gnd   sky130_fd_pr__nfet_01v8_lvt  l=1 w=5 m=8
xmn2  net2  net1  net3  gnd  sky130_fd_pr__nfet_01v8_lvt  l=1 w=5 m=8

* ── CTAT Branch — Q1 (1×) ───────────────────────────────────────
xqp1  gnd  net1  net1  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1

* ── PTAT Branch — R1 + Q2 (8×) ──────────────────────────────────
xra1  net3  net4  sky130_fd_pr__res_high_po_1p41  l=7.8 w=1.41 m=2
xqp2  gnd  net4  net4  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=8

* ── Reference Branch — R2 + Q3 ──────────────────────────────────
xrb1  vref  net5  sky130_fd_pr__res_high_po_1p41  l=7.8 w=1.41 m=16
xqp3  gnd  net5  net5  sky130_fd_pr__pnp_05v5_W3p40L3p40  m=1

* ── Startup Circuit ──────────────────────────────────────────────
xmp4  net6  net6  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=1
xmp5  net7  net6  vdd  vdd  sky130_fd_pr__pfet_01v8_lvt  l=2 w=5 m=1
xmn3  net7  net7  gnd  gnd  sky130_fd_pr__nfet_01v8_lvt  l=7 w=1 m=1
xmn4  net6  net7  gnd  gnd  sky130_fd_pr__nfet_01v8_lvt  l=7 w=1 m=1

.dc temp -40 125 5

.control
  run
  plot v(vref)
.endc

.end
```

> To run different corners, change only the `.lib` tag:
> ```
> .lib "...sky130.lib.spice" ff   ← Fast-Fast
> .lib "...sky130.lib.spice" ss   ← Slow-Slow
> ```

---

## 8. Pre-Layout Simulation Results

### 8.1 CTAT Characterization

**Single BJT — V_BE vs Temperature**

V_BE decreases from ~848 mV at −40°C to ~560 mV at +125°C. Slope ≈ **−1.88 mV/°C** — consistent with theory.

![CTAT Single BJT](prelayout_results/ctat_voltage_gen.png)

**Slope Extraction — Confirming −2 mV/°C**

![CTAT Slope](prelayout_results/ctat_slope.png)

**Effect of BJT Multiplicity**

With 8 parallel BJTs in branch 2 vs 1 in branch 1, the steeper slope confirms the area ratio effect that the PTAT stage relies on.

![CTAT Multiple BJT](prelayout_results/ctat_voltage_gen_mul_bjt.png)

**Effect of Bias Current**

Lower I₀ reduces V_BE and slightly increases the slope magnitude — confirming the theoretical I₀ dependence.

![CTAT Variable Current](prelayout_results/ctat_voltage_gen_var_current.png)

---

### 8.2 PTAT Characterization

**ΔV_BE vs Temperature — Confirming PTAT**

Voltage across R1 rises linearly at ~+0.179 mV/°C for N = 8. Purely proportional to absolute temperature as expected.

![PTAT Graph](prelayout_results/ptat_voltage_gen.png)

**Slope Verification Against Theory**

Measured slope agrees with V_T × ln(8) / ΔT within simulation accuracy.

![PTAT Slope Calc](prelayout_results/ptat_slope_calc.png)

**PTAT with Ideal Current Source**

PTAT generation isolated with an ideal current source — confirms the ΔV_BE mechanism independent of the mirror.

![PTAT Ideal](prelayout_results/ptat_voltage_gen_ideal.png)

---

### 8.3 BGR with Ideal Op-Amp

Before building the SBCM, the full BGR is validated using a VCVS as an ideal op-amp. This isolates the CTAT+PTAT cancellation from any mirror non-idealities.

**V_REF — The Characteristic Bell Curve**

V_REF is nearly flat across −40°C to +125°C, centred at **1.2358 V**. The slight upward bow is a second-order residual from higher-order temperature terms in I_S — unavoidable without curvature compensation.

**Tempco = 33–35 ppm/°C** with ideal op-amp.

![BGR Ideal OpAmp](prelayout_results/bgr_using_ideal_opamp.png)

**CTAT and PTAT Overlaid — Slope Match Visualised**

![BGR Ideal CTAT PTAT](prelayout_results/bgr_ideal_ctat_ptat.png)

**Slope Verification at the Operating Point**

![BGR Ideal Slope](prelayout_results/bgr_ideal_slope_qp3.png)

**CTAT/PTAT Balance Confirmed — Equal and Opposite**

![BGR Ideal Verification](prelayout_results/bgr_ideal_ptat_ctat_verif.png)

---

### 8.4 Complete BGR with SBCM

#### Zero-Current Degenerate State (No Startup)

Without a startup circuit, the SBCM stays at I = 0 — V_REF = 0 V. The degenerate bias point is stable and the circuit never self-starts. This confirms the startup circuit is not optional.

![BGR No Startup](prelayout_results/bgr_lvt_rpolyh_no_startup.png)

#### Virtual Short Verification

The SBCM forces Vid1 = Vid2 — replicating the ideal op-amp's virtual short. This is the critical check that the SBCM is operating at its intended bias point.

![Vid1 Vid2 Equal](prelayout_results/bgr_vid1_vid2_equal.png)

BJT base voltages Vq1 = Vq2 — structural symmetry confirmed:

![Vq1 Vq2 Equal](prelayout_results/bgr_vq1_vq2_equal.png)

Equal branch currents — current mirror balance confirmed:

![Branch Currents](prelayout_results/bgr_branch_currents_equal.png)

#### PSRR — VDD Sweep

V_REF remains stable as VDD is swept — the SBCM bootstrapping provides first-order VDD rejection.

![VDD VREF Sweep](prelayout_results/bgr_lvt_rpolyh_vdd_vref.png)

Internal nodes net1, net2 vs VDD:

![VDD Net1 Net2](prelayout_results/bgr_lvt_rpolyh_net1_net2.png)

Nodes net6, net2 vs VDD:

![VDD Net6 Net2](prelayout_results/bgr_lvt_rpolyh_net6_net2.png)

Branch current stability across the full temperature range:

![Current Verification](prelayout_results/bgr_lvt_rpolyh_current_verif.png)

---

### 8.5 Startup Circuit Analysis

**Full Power-On Transient**

VDD ramps from 0 V to 2.0 V at t = 0. The startup circuit detects the zero-current state and injects current, forcing V_REF to settle cleanly. The startup transistor turns off automatically once V_REF is established — visible as the clean flat plateau after ~1.2 µs.

**Startup time: < 1 µs**

![BGR Startup Transient](prelayout_results/bgr_startup_transient.png)

**Branch Current — With vs Without Startup**

Zero branch current without startup (stuck at degenerate point) vs correct ~10 µA/branch with startup:

![Startup Current Compare](prelayout_results/bgr_startup_current_compare.png)

---

### 8.6 Process Corner Analysis

Process corners capture the spread of transistor parameters across fabrication runs. The BGR must meet its tempco specification across all corners — not just at nominal.

| Corner | Meaning | V_REF | Tempco | Spec |
|--------|---------|-------|--------|------|
| TT | Nominal | ~1.2 V | — | — |
| FF | Fast NMOS + Fast PMOS | ~1.122 V | ~10 ppm/°C | ✅ |
| SS | Slow NMOS + Slow PMOS | within range | ~45 ppm/°C | ✅ |

**TT Corner — Nominal 27°C**

![TT 27 deg](prelayout_results/bgr_temp_verif_27deg.png)

**FF (Fast-Fast) Corner**

The bell-curve shape is preserved. V_REF shifts to ~1.122 V due to the changed operating current. Tempco improves to ~10 ppm/°C — best case.

![FF Corner](prelayout_results/bgr_lvt_rpolyh_ff.png)

**SS (Slow-Slow) Corner**

Tempco degrades to ~45 ppm/°C — worst case, but within the 50 ppm/°C specification.

![SS Corner](prelayout_results/bgr_lvt_rpolyh_ss.png)

> **Key observation:** The bell curve shape is preserved across all corners — the CTAT+PTAT cancellation mechanism is robust to process variation. Only the absolute V_REF value and the peak temperature shift slightly between corners.

---

## 9. Physical Layout

All layout was drawn in **Magic VLSI** with the Sky130A technology file. The layout follows a bottom-up hierarchy matching the simulation stages.

```bash
magic -T /path/to/eda-technology/sky130/tech/magic/sky130A.tech
```

### Layout Principles Applied

- **Common-centroid placement** for all matched devices — BJTs, resistors, current mirror FETs
- **Guard rings** around all device blocks for latch-up prevention and substrate noise isolation
- **Dummy devices** at array boundaries to ensure uniform process gradients
- **Same orientation** for matched transistors to cancel systematic gradient effects

---

### Resistor Bank

R1 and R2 implemented using RPOLYH unit cells with dummy resistors at row ends. The critical parameter is the *ratio* R2/R1 — both using identical devices ensures the ratio is process-insensitive across corners and temperature.

![Resistor Bank](Layout_results/resbank.png)

---

### PFET Current Mirror

MP1–MP5 in common-centroid interdigitated layout with a shared guard ring. PFET matching is the dominant factor in current mirror accuracy and directly determines V_REF stability.

![PFET Bank](Layout_results/pfets.png)

---

### NFET Bank

MN1–MN4 placed together with a shared n-well guard ring. MN1/MN2 (SBCM) use common-centroid placement; MN3/MN4 (startup, long-channel) are placed adjacently.

![NFET Bank](Layout_results/nfets.png)

---

### Complete BGR — Top Level

All blocks placed and routed together. Floor plan: resistors at top, FETs in middle, BJT array at bottom — minimising interconnect parasitics on the critical matched nodes.

![Top Level Layout](Layout_results/top.png)

**Final layout with V_REF output routed:**

![Final VREF Layout](Layout_results/top_vref.png)

### Parasitic Extraction

```tcl
# In the Magic Tcl console:
extract all
ext2sim label on
ext2sim
ext2spice scale off
ext2spice hierarchy off
ext2spice
```

This produces `top.spice` — the extracted netlist with parasitic capacitances, used for both post-layout simulation and LVS.

---

## 10. LVS Verification

LVS compares the extracted layout netlist against the schematic netlist to confirm that no errors were introduced during physical design.

```bash
# Launch Netgen
netgen

# Inside the Netgen console:
lvs "top.spice top" "bgr_lvt_rpolyh_3p40.sp top" \
    /path/to/eda-technology/sky130/tech/netgen/sky130A_setup.tcl
```

![LVS Result](Layout_results/lvs_result.png)

**✅ LVS Passed** — All devices match in type, size, and connectivity. No shorts or opens were introduced during layout.

---

## 11. Key Takeaways

The five most important insights from building this circuit end-to-end:

**1. The 1.2 V output is physically anchored to silicon**
It is not an arbitrary choice — it is the silicon bandgap energy at 0 K. BGRs across different processes and technologies all converge to ~1.2 V because of this fundamental material constant.

**2. The SBCM zero-current problem is real and non-trivial**
A circuit that looks correct on paper will silently stay at 0 V without a startup circuit. This is a classic example of a circuit with multiple stable operating points — simulation without startup confirmed it immediately.

**3. R2/R1 ratio matters far more than absolute R values**
Since both resistors use the same RPOLYH device, their ratio is far more stable across temperature and process than either absolute value. This is a deliberate design choice — the cancellation condition depends only on the ratio.

**4. Matching is everything in layout**
The Tempco achieved in simulation can be completely destroyed by poor layout — mismatched orientations, thermal gradients, or missing guard rings. Common-centroid placement and careful floor planning are as important as the circuit topology itself.

**5. Process corners reveal true robustness**
Simulating only at TT is insufficient. The SS corner (worst tempco, ~45 ppm/°C) is what determines whether the spec is met. The FF corner showing ~10 ppm/°C confirms the cancellation mechanism is working — not just at nominal, but across the full process space.

---

## 12. References

- [VSD — Bandgap Reference Design Workshop](https://www.vlsisystemdesign.com/)
- [SkyWater Sky130 PDK Documentation](https://skywater-pdk.readthedocs.io/)
- [Google Sky130 Primitive Devices](https://github.com/google/skywater-pdk-libs-sky130_fd_pr)
- [EDA Technology Files for Sky130](https://github.com/silicon-vlsi-org/eda-technology)
- [ngspice — Open Source SPICE Simulator](http://ngspice.sourceforge.net)
- [Magic VLSI Layout Tool](http://opencircuitdesign.com/magic/)
- [Netgen LVS Tool](http://opencircuitdesign.com/netgen/)
- Razavi, B. — *Design of Analog CMOS Integrated Circuits*, 2nd ed.
- Banba et al. — *"A CMOS Bandgap Reference Circuit with Sub-1-V Operation"*, IEEE JSSC 1999

---

<div align="center">

*Designed and verified using open-source EDA tools and the SkyWater Sky130 open PDK*

</div>
