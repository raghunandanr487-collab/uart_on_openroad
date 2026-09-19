<div align="center">

# 🔌 UART Transceiver — RTL to GDS on Sky130

**A full-duplex UART taken all the way from Verilog to a routed, timed-signoff layout on the SkyWater 130nm PDK.**

![Verilog](https://img.shields.io/badge/RTL-Verilog-blue?logo=v&logoColor=white)
![Yosys](https://img.shields.io/badge/Synthesis-Yosys-orange)
![OpenROAD](https://img.shields.io/badge/P%26R-OpenROAD-green)
![Sky130](https://img.shields.io/badge/PDK-Sky130%20HD-red)
![Status](https://img.shields.io/badge/Status-Routed%20%26%20Timed-brightgreen)

</div>

---

## 📚 Table of Contents

- [🧭 Design Overview](#-design-overview)
- [🏗️ Module Architecture](#️-module-architecture)
- [⚡ How It Works](#-how-it-works)
- [🔧 Synthesis](#-synthesis)
- [🏭 Physical Implementation](#-physical-implementation)
- [⏱️ Timing Results](#️-timing-results)
- [📊 Design Statistics](#-design-statistics)
- [⚠️ Known Issues & Caveats](#️-known-issues--caveats)
- [📁 Repository Layout](#-repository-layout)
- [🔄 Reproducing the Flow](#-reproducing-the-flow)
- [🎯 Next Steps](#-next-steps)

---

## 🧭 Design Overview

| Item | Value |
|---|---|
| 🏷️ Top module | `top` |
| 🔬 Technology | SkyWater Sky130, `sky130_fd_sc_hd` (high density) |
| 📚 Liberty corner | `sky130_fd_sc_hd__tt_025C_1v80` (typical, 25 °C, 1.80 V) |
| ⏱️ Clock | `sys_clk`, 20 ns period (constrained at 50 MHz) |
| ⚙️ RTL clock param | `clk_freq = 100_000_000` |
| 📡 Baud rate param | `buad_rate = 19200` |
| 🔁 Oversampling | 16× |
| 📦 Frame format | 8N1 — 1 start, 8 data, 1 stop, no parity |
| 📐 Core area | 69.92 × 68.24 µm |
| 🛤️ Routing layers | `li1`, `met1`–`met5` |

### 🔗 Port List

| Port | Dir | Width | Description |
|---|:---:|:---:|---|
| `clk` | ➡️ in | 1 | System clock |
| `reset_n` | ➡️ in | 1 | Asynchronous active-low reset |
| `tx_data` | ➡️ in | 8 | Byte to transmit |
| `start_b` | ➡️ in | 1 | Transmit request |
| `tx` | ⬅️ out | 1 | Serial transmit line |
| `tx_busy` | ⬅️ out | 1 | High while a frame is in flight |
| `rx` | ➡️ in | 1 | Serial receive line |
| `rx_data` | ⬅️ out | 8 | Received byte |
| `rx_valid` | ⬅️ out | 1 | One-cycle strobe when `rx_data` is valid |

---

## 🏗️ Module Architecture

```
top
├── 📡 buad_gen   u_buad_gen   →  buad_tik_16x   (16× oversample tick)
├── 📤 uart_tx    u1           →  tx, tx_busy
└── 📥 uart_rx    u2           →  rx_data_out[7:0], rx_valid
```

A single baud generator produces one shared `buad_tik_16x` strobe consumed by **both** the transmitter and receiver. Everything is clocked by `clk` — the baud tick is an **enable**, not a second clock domain. ✅ One clock, one clean CTS.

---

## ⚡ How It Works

### 📡 Baud Generator — `buad_gen`

```text
divisor = clk_freq / (buad_rate × 16)
        = 100_000_000 / (19_200 × 16)
        = 325
```

Counter width is inferred as `$clog2(325)` = **9 bits**, matching the 9-bit `count` register in the synthesized netlist. 🎯 When the counter hits `divisor - 1`, it emits a one-cycle `buad_tik_16x` pulse and restarts.

### 📤 Transmitter — `uart_tx`

A 4-state FSM: `idle → start → data_in → stop → idle` 🔄

- 😴 **`idle`** — `tx` held high, `tx_busy` low. On `start_b`, latch `tx_data` into `tx_data_out`, raise `tx_busy`.
- 🚦 **`start`** — `tx` driven low. At `tik_count == 7` advance to `data_in` — moving at mid-bit is what aligns TX with RX's mid-bit sampling.
- 📊 **`data_in`** — the bit is driven at `tik_count == 14`, one tick *before* the boundary, so it's settled before RX samples. `bit_count` increments at `tik_count == 15`.
- 🛑 **`stop`** — `tx` driven high for one full bit time, then back to `idle`.

The 8-to-1 bit select maps to two `mux4_2` cells + one `mux2i_1` in the netlist — a clean 3-level mux tree. 🌳

### 📥 Receiver — `uart_rx`

- 🔍 **Start-edge detection** — `rx` registered into `rx_pre`; `w_start_edge = rx_pre & ~rx` catches the falling edge. Doubles as a synchronizer stage for the asynchronous `rx` input.
- ✅ **`start`** — at `tik_count == 7`, re-check `rx`. Still low → genuine start bit → `data_in`. High → glitch → back to `idle`. The tick counter resets to 0 here, **re-phasing** RX so future `tik_count == 7` events land in the centre of each bit. 🎯
- 📊 **`data_in`** — `rx_data_reg[bit_count]` sampled at `tik_count == 7` (mid-bit).
- 🛑 **`stop`** — if `rx` is high at frame end → commit `rx_data_out`, pulse `rx_valid`. If low → discard the byte (a simple framing-error check ❌).

`rx_valid` is cleared at the top of the output block every cycle → a genuine single-cycle strobe, not a level. 👍

---

## 🔧 Synthesis

Synthesized with **Yosys**: elaborate → map flops → map combinational logic with ABC → tie constants → clean → write netlist.

```tcl
# 1️⃣ Read the UART design
read_verilog /path/to/uart.v

# 2️⃣ Check hierarchy and set the top module
hierarchy -check -top top

# 3️⃣ Generic synthesis
synth -top top

# 4️⃣ Map flip-flops to the Sky130 HD library
dfflibmap -liberty /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib

# 5️⃣ Map combinational logic with ABC
abc -liberty /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib

# 6️⃣ Replace tie-high / tie-low with real Sky130 tie cells
hilomap -hicell sky130_fd_sc_hd__conb_1 HI \
        -locell sky130_fd_sc_hd__conb_1 LO

# 7️⃣ Clean up unused logic
clean

# 8️⃣ Report the Sky130 cells used
stat -liberty /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib

# 9️⃣ Write the final gate-level netlist for OpenROAD
write_verilog -noattr uart_netlist.v
```

> 💡 **Why step 6 matters:** without explicit tie cells, constant `1'b0`/`1'b1` connections arrive at OpenROAD as unconnected nets, and detailed routing can leave gate inputs floating — a DRC *and* a reliability problem in real silicon. `conb_1` gives every constant a physical driver.

---

## 🏭 Physical Implementation

### 📍 I/O Placement

All I/O pins sit on the **west edge** of the die — the full `tx_data[7:0]` / `rx_data[7:0]` buses plus `tx`, `rx`, `tx_busy`, `rx_valid`, `reset_n`, `start_b`.

<p align="center">
  <img src="docs/images/08_io_pins.png" width="280" alt="I/O pin placement on the west edge">
</p>

Simple for a block this small — but it's also the root cause of the placement skew discussed below. ⚠️

### 🔋 Power Distribution Network

`met1` rails run horizontally over each placement row; vertical straps on the upper metals stitch it down through vias.

<p align="center">
  <img src="docs/images/01_pdn_power_grid.png" width="500" alt="VDD power grid">
</p>

| Property | Value |
|---|---|
| ⚡ Signal type | POWER |
| 🛤️ Wire type | ROUTED |
| ⭐ Special | True |
| 📌 ITerms | 223 |
| 🔌 BTerms | 1 (`VDD`) |
| 📦 BBox | (5.06, 5.20) → (74.98, 73.44) |
| 📐 BBox W × H | 69.92 × 68.24 µm |

223 ITerms = standard-cell VPWR pins tapped onto the grid. 1 BTerm = the top-level VDD port.

### 🗺️ Placement Density

<p align="center">
  <img src="docs/images/07_placement_density.png" width="500" alt="Placement density heat map">
</p>

Warm/dense on the left, cool/empty on the right — a direct consequence of west-edge-only pin placement pulling the placer's centre of gravity left. 📉 Fine functionally at 50 MHz, but it's wasted area.

### 🌳 Clock Tree Synthesis

<p align="center">
  <img src="docs/images/03_clock_tree.png" width="500" alt="Clock tree viewer">
</p>

```text
clk ──▶ clkbuf_0_clk (clkbuf_1) ──▶ clknet_0_clk
                                     ├─▶ clkbuf_2_0__f_clk ──▶ clknet_2_0__leaf_clk
                                     ├─▶ clkbuf_2_1__f_clk ──▶ clknet_2_1__leaf_clk
                                     ├─▶ clkbuf_2_2__f_clk ──▶ clknet_2_2__leaf_clk
                                     └─▶ clkbuf_2_3__f_clk ──▶ clknet_2_3__leaf_clk
```

- 🪜 **Levels:** 2 (one root, four leaf drivers)
- 🍃 **Leaf clusters:** 4, covering all 56 flip-flops
- ⏳ **Insertion delay:** ~0.26 → 0.40 ns
- ⚖️ **Balancing loads:** `clkload0` (`bufinv_16`), `clkload1` & `clkload2` (`clkbuf_8`) added on leaves 1/2/3 purely to equalize capacitance and crush skew

| Leaf net | Registers served |
|---|---|
| 🍃 `clknet_2_0__leaf_clk` | most of `u2` datapath (`w_rx_data`, `rx_data_reg`) + 2 from `u1` |
| 🍃 `clknet_2_1__leaf_clk` | `u2` FSM, `bit_count`, `tik_count`, `rx_pre` |
| 🍃 `clknet_2_2__leaf_clk` | bulk of `u1` (`tx`, `tx_busy`, `tx_data_out`, FSM) |
| 🍃 `clknet_2_3__leaf_clk` | entire `u_buad_gen` counter + `u1/tik_count[2:0]` |

✅ Keeping the whole baud counter on one leaf is a good outcome — it's the most timing-active logic in the design and toggles every single cycle.

### 🛤️ Routing & Congestion

<p align="center">
  <img src="docs/images/05_routing_congestion.png" width="420" alt="Routing congestion overview">
  <img src="docs/images/06_congestion_zoom.png" width="420" alt="Congestion detail">
</p>

Global + detailed routing completed on `li1` + `met1`–`met5`. Hot spots appear in the **centre** of the core where both FSMs and the shared baud tick converge. 🔥

`buad_tik_16x` is the single **highest-fanout net** in the design — every tick counter, bit counter, and FSM in both `u1` and `u2` consumes it — making it the main contributor to routing pressure through the middle.

---

## ⏱️ Timing Results

### ✅ Hold Analysis

<p align="center">
  <img src="docs/images/04_timing_report_hold.png" width="600" alt="Timing report — hold paths">
</p>

| Capture clock | Required | Arrival | Slack | Skew | Logic delay | Depth |
|---|---:|---:|---:|---:|---:|---:|
| `sys_clk` | 0.461 | 1.231 | **0.770 ✅** | 0.000 | 0.803 | 3 |
| `sys_clk` | -4.000 | 1.066 | **5.066 ✅** | 0.000 | 0.639 | 1 |

Worst hold slack **+0.770 ns**. Reported skew is **0.000 ns** on both paths — the balanced 4-leaf tree earning its keep. 🎯

### ✅ Setup Analysis

<p align="center">
  <img src="docs/images/02_hold_slack_histogram.png" width="500" alt="Endpoint slack histogram">
</p>

```text
[17.332, 17.731]:                                          (0)
[17.731, 18.130]:                                          (0)
[18.130, 18.529]: ***********************                 (15)
[18.529, 18.928]: ******************************...        (31)
[18.928, 19.327]: ***************                          (9)
```

Against a 20 ns period, arrivals peak in the 18.5–18.9 ns band → roughly **1.1–1.9 ns of setup margin**. 😌 Nowhere close to the frequency ceiling at 50 MHz.

---

## 📊 Design Statistics

| Block | Comb. cells | Flip-flops | Total |
|---|---:|---:|---:|
| 📤 `u1` (uart_tx) | 57 | 19 | 76 |
| 📥 `u2` (uart_rx) | 87 | 27 | 114 |
| 📡 `u_buad_gen` | 23 | 10 | 33 |
| 🌳 Clock tree | 8 | — | 8 |
| **🔢 Total** | **175** | **56** | **231** |

### 🗃️ Register Breakdown

| Register | Width | Block |
|---|:---:|---|
| `count` | 9 | 📡 `u_buad_gen` |
| `buad_tik_16x` | 1 | 📡 `u_buad_gen` |
| `tik_count` | 4 each | 📤 `u1`, 📥 `u2` |
| `bit_count` | 3 each | 📤 `u1`, 📥 `u2` |
| `state` | 2 each | 📤 `u1`, 📥 `u2` |
| `tx_data_out` | 8 | 📤 `u1` |
| `tx`, `tx_busy` | 1 each | 📤 `u1` |
| `rx_data_reg` | 8 | 📥 `u2` |
| `rx_data_out` | 8 | 📥 `u2` |
| `rx_pre`, `rx_valid` | 1 each | 📥 `u2` |

Flop flavours used: `dfrtp_1` (async reset — vast majority) and `dfstp_2` (async **set**) for `tx` and `rx_pre`, which both reset high. 💡

---

## ⚠️ Known Issues & Caveats

> These are real findings from the flow, kept in for transparency. 🧐

**1️⃣ 🐛 Clock frequency mismatch (RTL vs SDC).** The RTL hardcodes `clk_freq = 100_000_000`, but the SDC constrains `sys_clk` at 20 ns (50 MHz). The divisor is computed at elaboration from the parameter, not the real clock:
```text
actual baud = 50_000_000 / (325 × 16) ≈ 9615 baud
```
That's **9600 baud, not 19200**. ❗ Fix by tightening the SDC to 10 ns, or setting `clk_freq = 50_000_000`. Timing signoff can't catch this — it's functional, not temporal.

**2️⃣ 📌 57 unconstrained pins.** No `set_input_delay` / `set_output_delay` applied anywhere. The clean timing numbers above cover reg-to-reg paths only — I/O paths are unanalyzed. This is the single highest-value fix left in the SDC.

**3️⃣ 🔋 IR drop never analyzed.**
```text
[WARNING GUI-0066] Heat map "IR Drop" has not been populated with data.
```
PDN was built but PSM / `analyze_power_grid` was never run. Low risk at this size, but cheap to add.

**4️⃣ ➗ Baud divisor truncation.** `100_000_000 / (19_200 × 16) = 325.52` → truncated to 325 → **19230 baud** vs nominal 19200 (≈ +0.16% error). ✅ Well inside a UART's ~2–3% tolerance — not a real problem, just worth knowing.

**5️⃣ 📐 Placement skew from single-edge I/O.** Right side of the core is largely empty. Spreading pins across all four edges (or shrinking the die) recovers area.

**6️⃣ 🛡️ No parity, no FIFO, no majority-vote sampling.** RX samples each bit once at mid-bit. A 3-sample majority vote at ticks 7/8/9 would meaningfully improve noise immunity for very little extra logic.

---

## 📁 Repository Layout

```text
.
├── 📂 rtl/
│   └── uart.v                  # top, uart_tx, uart_rx, buad_gen
├── 📂 script/
│   ├── synth.ys                # Yosys synthesis script
│   └── uart_netlist.v          # gate-level netlist for OpenROAD
├── 📂 openroad/
│   ├── uart.sdc                # timing constraints
│   └── *.tcl                   # floorplan / PDN / place / CTS / route
├── 📂 results/
│   ├── top.def
│   ├── top.gds
│   └── 📂 reports/
└── 📂 docs/
    └── 📂 images/               # screenshots used in this README
```

---

## 🔄 Reproducing the Flow

### 📦 Prerequisites

- 🔧 [Yosys](https://github.com/YosysHQ/yosys)
- 🏭 [OpenROAD](https://github.com/The-OpenROAD-Project/OpenROAD) (GUI build recommended for the views above)
- 📚 [Sky130 PDK](https://github.com/google/skywater-pdk) — `sky130_fd_sc_hd` LEF, Liberty, tech LEF

### ▶️ Run it

```bash
# 1️⃣ Synthesis
yosys script/synth.ys

# 2️⃣ Physical implementation
openroad -gui openroad/flow.tcl
```

### 🧭 Flow order inside OpenROAD

1. 📖 `read_lef` / `read_liberty` / `read_verilog` / `read_sdc`
2. 🏗️ `initialize_floorplan` — core 69.92 × 68.24 µm
3. 📍 `place_pins` — west edge
4. 🔋 `pdngen` — met1 rails, upper-metal straps
5. 🧩 `global_placement` → `detailed_placement`
6. 🌳 `clock_tree_synthesis` — `clkbuf_1` root and leaves
7. 🛤️ `global_route` → `detailed_route`
8. 📊 `report_checks` / `write_def` / `write_gds`

---

## 🎯 Next Steps

- [ ] 🔧 Fix the `clk_freq` vs SDC period mismatch
- [ ] 📌 Add `set_input_delay` / `set_output_delay` for the 57 open pins
- [ ] 🔋 Run PSM IR-drop analysis and populate the heat map
- [ ] 🛡️ Add a majority-vote sampler in the receiver
- [ ] 📐 Spread pins across all four edges, shrink the die
- [ ] 🧪 Add a loopback testbench (`tx` → `rx`) and run GLS with SDF back-annotation

---

## 📝 Naming Note

The RTL spells the baud generator `buad_gen` and its signals `buad_tik_16x` — a typo for "baud" that's now load-bearing across the netlist, the DEF, and every report. It's preserved throughout this README to match what you'll actually see in the tools. 😄 Renaming it means re-running the whole flow.

---

<div align="center">

Made with 🧠 + ☕ + a lot of `report_checks` 🔁

</div>
