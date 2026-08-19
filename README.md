# Energy-Harvesting BLE Environmental Sensor Node
### A Battery-less IoT Node powered by a Pyramidal Rectenna WPT Front-End

> **Status:** Hardware bring-up complete, mathematical framework finalized, research paper in preparation (unpublished). This repository documents the design rationale, schematic, and PCB layout of the sensor node hardware.

![Assembled board](docs/images/pcb-assembled-top.jpg)

---

## 1. Project Summary

This project implements a **battery-less Bluetooth Low Energy (BLE) sensor node** designed to be powered entirely by RF energy harvested through a custom **pyramidal multi-sector rectenna** (Wireless Power Transfer front-end, documented separately in the accompanying research work). The rectenna's rectified DC output feeds an ultra-low-power energy harvesting PMU, which charges a supercapacitor and regulates power for a BLE microcontroller and an environmental sensor.

The system demonstrates **Simultaneous Wireless Information and Power Transfer (SWIPT)**: the same RF link that delivers power also allows the node to periodically transmit BLE beacons containing live temperature/humidity/gas-sensor data, with no battery in the loop.

```
[ RF Transmitter ] --(WPT)--> [ Pyramidal Rectenna Array ] --(rectified DC)--> [ This PCB ]
                                                                                     |
                                                        +----------------------------+----------------------------+
                                                        |                            |                            |
                                                [ ADP5091 PMU ]           [ ANNA-B112 BLE SoC ]         [ BME688 Sensor ]
                                                (harvest + MPPT +          (application MCU +           (temp / humidity /
                                                 supercap storage)         BLE radio + SWD debug)         pressure / gas)
```

---

## 2. Repository Structure (suggested)

```
├── hardware/
│   ├── schematic/          # Altium .SchDoc exports, PDF schematic
│   ├── pcb/                # Altium .PcbDoc, layer stack, Gerbers
│   └── bom/                # Bill of materials
├── firmware/                # ANNA-B112 firmware (BLE beacon + sensor read)
├── analysis/                 # Rectenna mathematical framework, MATLAB/Python scripts
├── docs/
│   └── images/               # Photos, renders, plots (see Section 6)
└── README.md
```

---

## 3. Hardware Architecture & Design Rationale

The board is organized into three functional blocks, each shown as a separate sub-circuit in the Altium schematic.

### 3.1 Power Management — ADP5091 Energy Harvesting Circuit

**Why this part:** The ADP5091 is an ultra-low-power boost regulator purpose-built for harvesting from high-impedance, low-current sources (like a rectenna) rather than a conventional battery charger IC. It supports cold-start from very low input voltages and includes Maximum Power Point Tracking (MPPT), which is essential because a rectenna's optimal output impedance shifts with incident RF power — without MPPT, a fixed-impedance load would waste most of the harvested energy.

| Net / Component | Function | Design Choice |
|---|---|---|
| `VIN` → `L1` (22 µH) → `SW` | Boost converter inductor | Sized for the ADP5091's recommended switching frequency range to balance efficiency vs. inductor size on a compact board |
| `R3`/`R4`/`R5` on `MPPT` | MPPT set ratio | Resistor divider fixes the fraction of open-circuit rectenna voltage (Voc) at which the PMU regulates VIN, keeping the rectenna near its peak power transfer point regardless of incident RF level |
| `C3` (220 µF) on `BAT`/`SUPERCAP` | Primary energy reservoir | A supercapacitor is used instead of a battery — it tolerates the highly intermittent, low-current charge profile typical of RF harvesting and avoids battery degradation/replacement, keeping the node fully maintenance-free |
| `R7–R10` on `SETPG`/`SETSD` | Battery-good / shutdown thresholds | Resistor dividers program the exact supercap voltage window in which the system is allowed to draw load current, protecting against brown-out resets of the BLE SoC |
| `R14–R17` on `SETBK`/`SETHYST` | Backup & hysteresis thresholds | Prevents rapid on/off "chattering" of the output regulator as harvested power fluctuates |
| `R18` on `TERM` | Charge termination | Limits the maximum supercap voltage to protect the storage element |
| `REG_D0`/`REG_D1` | Output voltage select | Digital pins on the ADP5091 select the regulated output rail supplied to the ANNA-B112 / BME688 |
| `C4`, `C7`, `C8` | Input/output filtering | Standard decoupling for the switching regulator stage |

### 3.2 Application Processor — ANNA-B112 (nRF52-based BLE Module)

**Why this part:** The ANNA-B112 integrates a Bluetooth 5 radio, antenna, and Cortex-M0 MCU in a single certified module, minimizing RF design risk and board area — important on a power-constrained, harvesting-driven node where every extra milliamp matters.

| Net / Component | Function | Design Choice |
|---|---|---|
| `J1` (SWDIO/SWDCLK/RESET_N) | SWD programming header | Kept as a 5-pin header for in-circuit firmware updates without needing to desolder the module |
| `R1` (10 kΩ) on `RESET_N` | Reset pull-up | Prevents spurious resets from noise on a high-impedance, low-power rail |
| `Y1` (32.768 kHz) | Real-time clock crystal | Used for the low-power sleep/RTC timer inside the BLE stack, allowing the MCU to stay in deep sleep between BLE advertising events — critical for surviving on harvested µW-mA power budgets |
| `C1`, `C2` (0.1 µF) | Decoupling | Local bypass on VCC and crystal load caps |
| `MOSI_BME` / `SCLK_BME` / `MISO_BME` / `CS_BME` | SPI bus to sensor | GPIO13–15 + a dedicated chip-select line route a full-duplex SPI interface to the BME688 |
| Ground pour under `U1`, RF **keepout zone** on PCB | Antenna integrity | No copper, silkscreen, or routing is placed under the module's integrated antenna area to preserve BLE radio performance |

### 3.3 Sensing — BME688 Environmental Sensor

**Why this part:** The BME688 provides temperature, humidity, pressure, and gas (VOC) sensing in one low-power package, matching the "smart environmental beacon" use case for the harvested-power node.

| Net / Component | Function | Design Choice |
|---|---|---|
| `CSB`/`SCK`/`SDI`/`SDO` | 4-wire SPI | SPI was chosen over I²C to avoid dependence on external pull-up resistors and to reduce bus contention risk on a rail that may briefly sag during harvesting transients |
| `C9`, `C10` (100 nF) | Split VDD / VDDIO decoupling | Digital core and IO supply rails are decoupled independently to reduce switching noise coupling between them |

---

## 4. PCB Layout Notes

| Bare — Top | Bare — Bottom |
|---|---|
| ![Bare top](docs/images/pcb-bare-top.jpg) | ![Bare bottom](docs/images/pcb-bare-bottom.jpg) |

| Assembled — Top | Assembled — Bottom |
|---|---|
| ![Assembled top](docs/images/pcb-assembled-top.jpg) | ![Assembled bottom](docs/images/pcb-assembled-bottom.jpg) |

![Board scale reference](docs/images/pcb-scale.jpg)
*35 mm × 35 mm double-layer board, shown for scale.*

- **Board size:** 35 mm × 35 mm, double-layer, FR-4.
- **Connector placement:** `J1` (SWD), `J2`, `J3` (power in/harvester in, regulated out) are placed along one edge for easy bench access during test and debug.
- **Keepout zone:** A dedicated no-copper region is reserved beneath the ANNA-B112's integrated antenna to avoid detuning the BLE radio.

  ![Antenna keepout close-up](docs/images/antenna-keepout.jpg)
  *No-copper keepout region under the integrated BLE antenna.*

- **Component grouping:** PMU passives (`L1`, `C3` supercap, MPPT/threshold resistor networks) are clustered near `U2` to minimize switching-loop area and parasitic inductance.
- A full layer-stack report is included (`Board Stack Report`) for manufacturing handoff.

---

## 5. Link to the RF Energy Harvesting Research

![Pyramidal rectenna — 3D render](docs/images/rectenna-render.png)
*CAD render of the pyramidal multi-sector rectenna (physical prototype photos pending).*

This PCB is the **load-side / SWIPT receiver electronics** for a separate research effort on a plug-in pyramidal multi-sector rectenna system, which analytically derives:
- The optimal sector tilt angle and count for a uniform DC power pattern under lateral misalignment between transmitter and receiver.
- A dual-linearly-polarized rectenna element design mitigating polarization mismatch.

The rectenna's DC output is intended to connect directly to the `VIN` harvesting input documented above. (Full derivations and measurement results are covered in the accompanying paper, currently unpublished.)

---

## 6. Photos & Media

Based on what's available right now, here's what to shoot / export and where to drop each file. Save everything into `docs/images/` using the filenames below — that way the embedded links in this README will resolve automatically once you add the files.

### 6.1 PCB photos (have — need to capture)

| File | What to shoot | Notes |
|---|---|---|
| `docs/images/pcb-bare-top.jpg` | Bare PCB, top side, straight-on, good lighting | Shoot on a plain/dark background so silkscreen and copper are legible |
| `docs/images/pcb-bare-bottom.jpg` | Bare PCB, bottom side | Same framing as top for a clean side-by-side |
| `docs/images/pcb-assembled-top.jpg` | Fully assembled board, top | This becomes your hero/banner image at the top of the README |
| `docs/images/pcb-assembled-bottom.jpg` | Fully assembled board, bottom | |
| `docs/images/pcb-scale.jpg` | Board next to a coin or ruler | Gives viewers an instant sense of the 35 mm × 35 mm size |
| `docs/images/antenna-keepout.jpg` | Close-up macro shot of the area under/around `U1` (ANNA-B112) | Crop tight enough to visibly show the no-copper keepout zone vs. surrounding ground pour |

Once these are in place, I'll wire them into the README (hero image up top, a "Board Photos" gallery in Section 4, and the keepout photo placed right next to the keepout paragraph).

### 6.2 Test / measurement media — not yet available

No bench photos or scope captures yet. Worth adding later once you have hardware bring-up data — even a couple of these go a long way for credibility:
- Bench setup shot (Tx horn antenna, rectenna, PMU board, multimeter/scope all in frame)
- Multimeter reading showing harvested DC voltage / regulated output voltage
- Scope capture of the boost converter switching node or output ripple
- Any VNA S11 plot or measured radiation pattern, if/when you re-run that test

I've left a placeholder subsection for this (`## 7. Test & Measurement Results`) — happy to build it out once you have real data or plots to drop in.

### 6.3 Rectenna prototype — 3D render only for now

| File | What to add |
|---|---|
| `docs/images/rectenna-render.png` | Export a clean 3D render of the pyramidal rectenna from your CAD tool (isometric view showing all sectors clearly) |

I'll use this as the lead image in Section 5 with a note that physical prototype photos are pending. When you do get real photos (assembled pyramid, single sector close-up, or the rectenna in a test range with the Tx horn), just add them alongside and I can update the captions.

---

## 7. Test & Measurement Results
*(placeholder — to be filled in once bench data is available)*
