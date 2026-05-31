# Project 1 — CMOS Ring Oscillator

**IC Design with Open-Source EDA Tools · Summer 2026 · Part 1 of 8**

A 3-stage CMOS ring oscillator designed, simulated, and analysed using entirely open-source tools on Windows 11 via WSL2/Ubuntu 22.04.

---

## Result

| Corner | Voltage | Temp | Frequency |
|---|---|---|---|
| TT — Typical | 1.8 V | 27°C | **1.77 GHz** |
| SS — Worst case | 1.62 V | 100°C | **967 MHz** |

**45% frequency drop** across PVT corners — demonstrating exactly why Static Timing Analysis matters in production silicon.

---

## Tools

| Tool | Version | Purpose |
|---|---|---|
| ngspice | 36 | SPICE circuit simulator |
| Sky130 PDK | bdc9412b | SkyWater 130nm open-source process |
| volare | 0.20.6 | PDK version manager |
| Magic VLSI | 8.3 (from source) | Physical layout editor |
| Python | 3.10.12 | PDK toolchain dependency |
| WSL2 + Ubuntu | 22.04 LTS | Linux environment on Windows 11 |

---

## Repository Structure

```
project1_ring_oscillator/
│
├── netlists/
│   ├── ring_osc_tt.sp          # TT corner — basic transient simulation
│   ├── ring_osc_freq.sp        # TT corner — with frequency measurement
│   └── ring_osc_ss_pvt.sp      # SS worst-case PVT corner
│
├── layout/
│   └── ring_osc.mag            # Magic VLSI layout file
│
├── report/
│   └── ring_oscillator.pdf     # Full project report (LaTeX)
│
└── README.md
```

---

## Circuit Description

A ring oscillator generates a clock signal with no external input by connecting an **odd number of inverters in a loop**. Because there is no stable state in an odd-inversion loop, the circuit toggles continuously.

```
         ┌─────────────────────────────────┐
         │                                 │
         ▼                                 │
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │  INV 1  │───▶│  INV 2  │───▶│  INV 3  │
    └─────────┘    └─────────┘    └─────────┘
      net1 out       net2 out       net3 out
```

**Oscillation frequency:**

```
f = 1 / (2 × N × t_pd)
```

Where `N = 3` stages and `t_pd` is the propagation delay of one inverter.

---

## Transistor Sizing

| Transistor | Type | Width | Length | Purpose |
|---|---|---|---|---|
| X1_p, X2_p, X3_p | PMOS | 1.0 µm | 0.15 µm | Pull-up (output → VDD) |
| X1_n, X2_n, X3_n | NMOS | 0.5 µm | 0.15 µm | Pull-down (output → GND) |

PMOS is sized 2× wider than NMOS to compensate for lower hole mobility, giving symmetrical rise and fall times.

---

## How to Run

### Prerequisites

```bash
# Install WSL2 on Windows 11 (PowerShell as Admin)
wsl --install -d Ubuntu-22.04

# Inside Ubuntu
sudo apt update && sudo apt upgrade -y
sudo apt install -y ngspice python3 python3-pip

# Install volare and download Sky130 PDK
pip3 install volare
echo 'export PATH=$PATH:$HOME/.local/bin' >> ~/.bashrc
source ~/.bashrc

export PDK_ROOT=$HOME/pdk
volare enable --pdk sky130 --pdk-root $PDK_ROOT bdc9412b3e468c102d01b7cf6337be06ec6e9c9a
echo 'export PDK_ROOT=$HOME/pdk' >> ~/.bashrc
source ~/.bashrc
```

### Run Simulation 1 — Basic Transient

```bash
cd netlists
ngspice ring_osc_tt.sp
```

Expected output: waveform window showing 3 alternating square waves on net1, net2, net3.

### Run Simulation 2 — Frequency Measurement

```bash
ngspice ring_osc_freq.sp
```

Expected output:
```
period_val = 5.640303e-10
freq_val   = 1.772954e+09    # 1.77 GHz
```

### Run Simulation 3 — PVT Worst Case

```bash
ngspice ring_osc_ss_pvt.sp
```

Expected output:
```
period_val = 1.033766e-09
freq_val   = 9.673369e+08    # 967 MHz
```

---

## PVT Analysis

PVT = **Process, Voltage, Temperature** — the three axes of variation that every chip must be verified against.

### Process Corners

| Corner | NMOS | PMOS | Models |
|---|---|---|---|
| `tt` — Typical-Typical | Normal | Normal | Ideal — most chips |
| `ff` — Fast-Fast | Fast | Fast | Max speed, high leakage |
| `ss` — Slow-Slow | Slow | Slow | Min speed, low leakage |

### Why frequency dropped 45% at SS corner

| Factor | Effect |
|---|---|
| Lower voltage (1.62V) | Reduces gate overdrive → less drive current → slower charging |
| High temp (100°C) | Lattice scattering → reduced carrier mobility → slower switching |
| SS process corner | Longer effective channel → inherently slower transistors |
| Leakage explosion | 0.32 nA → 41 µA (100,000× increase) |

---

## Key Learnings

1. **Sky130 uses subcircuit models** — transistors need `X` prefix not `M` in SPICE
2. **`.lib` not `.include`** — use `.lib file.spice tt` to load one corner only
3. **volare needs explicit commit hash** outside an OpenLane project directory
4. **PVT analysis is not optional** — 45% frequency variation means a 1.5 GHz design fails at SS corner
5. **Leakage is exponential with temperature** — a 73°C rise caused 100,000× more leakage

---

## Waveforms

### TT Corner — 1.8V, 27°C → 1.77 GHz
Three clean square waves, each phase-shifted by one inverter delay (~188 ps per stage).

### SS Worst-Case — 1.62V, 100°C → 967 MHz
Rounded waveforms with slower rise/fall times. Peaks no longer reach full 1.8V due to reduced drive current.

---

## Report

Full documentation available in [`ring_oscillator.pdf`](ring_oscillator.pdf)

---

## Series

| # | Project | Tools | Status |
|---|---|---|---|
| 1 | Ring Oscillator | ngspice + Sky130 | ✅ Complete |
| 2 | Full Custom Inverter Layout | Magic + KLayout + Sky130 | 🔄 In Progress |
| 3 | SPI/UART Core | Verilog + OpenROAD | ⬜ Upcoming |
| 4 | FIR Filter | Verilog + OpenROAD | ⬜ Upcoming |
| 5 | Low-Power OTA | Xschem + ngspice + Sky130 | ⬜ Upcoming |
| 6 | SAR ADC | ngspice + Sky130 | ⬜ Upcoming |
| 7 | Switched-Capacitor Frontend | Xschem + ngspice + Sky130 | ⬜ Upcoming |
| 8 | Sigma-Delta ADC | Xschem + ngspice + Python | ⬜ Upcoming |

---

*SkyWater Sky130 PDK · ngspice 36 · WSL2 Ubuntu 22.04 · Windows 11*
