# T-DSP Core spin 2 — change map

Status: **schematic edits done 2026-09-28 on branch `spin2/core-module`** (stages 2–5 of §12). Netlist
verified after every stage; ERC is down to one residual (see §14). PCB not yet updated from the
schematic. This document
collects every decision made in the 2026-09-28 design review so the schematic work can be
checked against it. It supersedes the architecture parts of `bt-asrc-block-netplan.md`
where the two disagree (Port B routing, control-line assignments, the DIT). The netplan
remains the reference for the BT module pinout and the IDC747 → IDC777 migration.

Every item is tagged **[decided]**, **[consequence]** (follows from a decision), or
**[open]** (still needs the user's call).

---

## 1. What the board is

**[decided]** The core is a **general-purpose audio compute module**, not a finished
backplane. 2.54 mm headers on two board edges expose the Teensy 4.1, the ESP32-S3, the
audio buses, and the SRC. Users design a backplane that carries codecs, amps, jacks,
connectors, and power entry. The core accepts **5 V** and generates its own rails. The
backplane may carry the bulk of the system current (core ≈ 0.6 A typical, ≈ 1.3 A peak; see the
current budget in §7).

**[decided]** The **isolated DMX / RS-485 block stays on the core** (U16, U17, U18, J28,
D6). It keeps S3 pins IO1, IO5, IO48.

**[decided]** Single-source wireless audio. BT **or** WiFi streaming, never both. No
mixing on the S3, no software resampler on the S3.

**[consequence]** Parts that belong on the backplane and leave the core schematic:

| Leaves the core | Refs |
|---|---|
| Ethernet jack, header | ETHERNET1, ETHERNET_J1 |
| USB-A host, USB-B, USB-C device, USB ESD, USB pin headers | USBA1, USBB1, USBC1, U10, USB_HOST_1, USB_DEVICE_1, D5, CC1, CC2, RLED1, R27 |
| 3.5 mm jacks, 6.35 mm phone jack, RCA/line | J22, J23, J_PHONE1, HPTRS1, HPTRS2, OUT1, INPUT1 |
| MIDI jacks only (opto U4, buffer U9 and passives **stay**) | MIDI_IN_1, MIDI_OUT_1, MIDI_IO1, MIDI_OT1, MIDI_OT2 |
| Rotary encoder, tactile switches | SW1–SW10 (SW3/4/9/10 are the EN/IO0 buttons, see §6) |
| JST power inputs and reverse-polarity FETs | J10, J11, Q1, Q2 |
| Optical S/PDIF TX/RX | TT1, TT2, TR1, TR2, SPDIF1, SPDIF_IO1, SPDIF_IO2 |
| MEMS microphones | MIC1, MIC2 |
| LED chain header only (the SK6812 status LEDs D3/D4 and their 5 V level shifter IC1 **stay**; the chain continues to the backplane via the LED_OUT header pins) | J12 |
| TFT / GPI / encoder / I2C / programming header duplicates | ILI1, ILI2, GPI1, GPI2, ESP_R_ENC1, ESP_R_ENC2, ESP_I1, ESP_I2, ESP_PROG1, ESP_PROG2, T_PROG1 |
| Old ESP32 DevKitC and TAC5212 module symbol | U2, board_outline1 |
| Battery header/cell | BAT1, BAT2 |

Delete them rather than leaving them excluded-from-board: U2 alone is still on 30+ nets
and every one of these adds ERC noise and confuses the header map.

---

## 2. Wireless audio block (BT + S3 → mux → ASRC → Teensy)

> **Superseded on 2026-10-01 (stage 10y, section 11.8).** The SRC4382, both 74LVC157 muxes and the
> S/PDIF header are gone. IDC777 and S3 now reach the Teensy's SAI2 port through a 74LVC257 mux,
> sample-rate conversion is done in Teensy software, and a buffered J_SAI2 header offers the same
> port to an external I2S master or slave. The text below is kept as the history of the decision.

Architecture: IDC777 (U22) and the S3 are both **I2S masters** into a hardware mux, the
mux feeds SRC4382 Port A (slave), the SRC resamples onto the Teensy's clock and outputs on
Port B (slave to the Teensy). One source at a time.

### 2.1 Mux

**[decided]** Keep U21 (74LVC157A) for BT vs S3.

**[decided]** Add a **second 74LVC157A (U23)** cascaded after U21: selects between U21's
output and an **aux I2S input from the edge header** (BCK, LRCK, DATA). Aux must be an I2S
master. Cost: one TSSOP-16, three header pins. If the header pin budget gets tight, aux is
the first thing to cut.

### 2.2 SRC4382 (U19) wiring changes

| Pin | Today | New |
|---|---|---|
| 48 BCKB | BCLK1+TDM1 | **4_BCLK2** (Teensy pin 4, I2S2 BCLK) |
| 47 LRCKB | LRCK1+TDM1 | **3_LRCLK2** (Teensy pin 3, I2S2 LRCLK) |
| 45 SDOUTB | SRC_SDOUT (dangling) | **5_IN2** (Teensy pin 5, I2S2 IN) |
| 46 SDINB | NC | **2_OUT2** (Teensy pin 2, I2S2 OUT) — reverse ASRC path |
| 40 SDOUTA | NC | **edge header, aux I2S out** — reverse ASRC path |
| 24 ~RST | SRC_RST (dangling) | **ESP32_EN** net |
| 26 GPO1 | NC | **BT_RST** (U22 pin 36) |
| 27 GPO2 | NC | **BT_SYS_CTRL** (U22 pin 13) + 100 k pulldown to GND |
| 28 GPO3 | NC | **BT_S3_SEL** (U21 pin 1) |
| 29 GPO4 | NC | **AUX_SEL** (U23 select) |
| 1–8 RX1±…RX4± | NC | **edge header**, raw differential pairs |
| 31, 32 TX−, TX+ | NC | **edge header**, raw pair |
| 12 RXCKO, 11 ~LOCK, 15 ~RDY, 23 ~INT | NC | leave NC (or S3 GPIO later) |

Why: the SRC has **no TDM mode** (checked in the full SRC4392 datasheet) and drives SDOUT
continuously, so Port B cannot share the TDM1 data line. Moving Port B to Teensy I2S2
gives the Teensy a plain stereo `AudioInputI2S2`. GPOs default **low** at reset, are
register-forced high/low (regs 0x1B–0x1E), and cost zero S3 pins. BT held in reset until
the S3 configures the SRC is the desired ordering. SRC ~RST on ESP32_EN resets the SRC with
the S3 and satisfies the 500 µs I2C hold-off by a wide margin. The SRC4382 **does** have
a DIT (TX pins are real), contrary to the netplan.

Firmware notes that follow: soft-mute (reg 0x2D) around source switches, since the
hardware MUTE pin is tied to GND. Reverse path = SRCIS=Port B, AOUTS=SRC; one direction at
a time.

### 2.3 S3 I2S master

**[decided]** S3 is I2S master on its second I2S controller into U21 inputs I1a/I1b/I1d.
**[consequence]** The existing S3-slave I2S2 link to the Teensy is **retired in both
directions**: IO4/IO6/IO7 become the master port, IO16 is freed. The Teensy's I2S2 port is
now owned by the SRC (2, 3, 4, 5).

### 2.4 BT module (U22) — unchanged from netplan except:

- BT_UART_TX / BT_UART_RX → **S3 UART2** on IO16 (RX) and IO35 (TX) — see §3 for the
  variant dependency.
- BT_RST, SYS_CTRL, BT_S3_SEL now from SRC GPOs (§2.2).
- VCHG / VCHG_SENSE / CHG_EXT stay no-connect; add **test points** per datasheet advice.

---

## 3. ESP32-S3 (U13) pin map

| Pin | Use after spin 2 |
|---|---|
| IO4, IO6, IO7 | I2S master → U21 (DOUT, BCK, LRCK — assign in firmware) |
| IO16 | BT UART2 RX |
| IO3 | BT UART2 (S3 RX side, net `BT_UART_RX`). IO35 is not available on the N16R8 module |
| IO19, IO20 | **native USB D−/D+ → edge header** (OTG, host or device) |
| RXD0, TXD0 | Teensy Serial7 (programming + FlasherX tunnel) **and** edge header |
| IO0, EN | Teensy 36/37 **and** edge header (already pins 60/56) |
| IO1, IO5, IO48 | DMX (stays) |
| IO8, IO9, IO10, IO11, IO2, IO42, IO18 | TFT SPI + touch (already on header) |
| IO12, IO13, IO14, IO17 | SPI link to Teensy (internal) |
| IO15, IO38, IO39 | encoder (already on header) |
| IO40, IO41 | GPI (already on header) |
| IO21, IO47 | I2C (already on header) |
| IO45 | **→ J8** (strapping pin; document the boot constraint). IO46 drives the SK6812 chain (`ESP32_LED`), 5 V copy on J11 |
| IO35, IO36, IO37 | **NC** - owned by the octal PSRAM on the N16R8 module |

**[decided 2026-09-28] Module variant: ESP32-S3-WROOM-1-N16R8** (LCSC C2913202). Spotify
Connect (cspot and the Espressif audio SDK) buffers and decodes the Vorbis stream in PSRAM and
needs 4-8 MB; the 2 MB quad-PSRAM R2 parts are not enough. Octal PSRAM uses IO35/IO36/IO37,
so those three module pins are NC and the BT UART RX moved to IO3. PCB antenna (-1, not -1U).

---

## 4. Teensy 4.1 (U1) pin changes

| Pin | Today | New |
|---|---|---|
| 2 OUT2 | → S3 IO4 | `SAI2_OUT` → U21 buffer → J_SAI2 DOUT (stage 10y; was SRC SDINB) |
| 3 LRCLK2, 4 BCLK2 | → S3 (via 49.9 Ω) | `SAI2_LRCLK` / `SAI2_BCLK`: from the U19 257 mux (IDC777 or S3) or J_SAI2 via U21, or driven out to J_SAI2 (stage 10y) |
| 5 IN2 | ← S3 IO7 | `SAI2_IN`: from the U19 mux or J_SAI2 DIN via U20 (stage 10y) |
| 22 | NC | `SAI2_SEL` (U19 select: 0 = IDC777, 1 = S3; 10 k pull-down) — stage 10y, no longer on J3 |
| 26, 27 | NC | `SAI2_MODE0` / `SAI2_MODE1` (U23 interlock, 10 k pull-ups = all off) — stage 10y, no longer on J3 |
| 32 OUT1B | J15 (1-pin) | → edge header |
| 33 MCLK2 | J19 (1-pin) | `SAI2_MCLK` → U21 buffer → J_SAI2 MCLK in both header modes (stage 10y; J3 pin 22 is GND now) |
| 49 VUSB | NC | → **TPS2116 VIN2** (see §7) |
| 46 3V3 | on the 3.3V rail | **disconnected from the rail, not exposed** (see §7) |
| 55–57 host USB | USBA1 / USB_HOST_1 | → edge header (D+, D−, HOST_5V) |
| 66/67 device USB | USBC1 / U10 | **stays on the Teensy's own jack only; not on header** |
| 28/29 Serial7, 36/37 | S3 UART0 / IO0 / EN | unchanged — programming kit + FlasherX |
| 53 PROGRAM, 54 ON_OFF | header 55/59 | unchanged |

MIDI: MIDI_IN/OUT/THRU are on header pins 1–6 at the *jack* level. **[decided]** the opto
(U4), buffer (U9) and their passives stay on the core.

Also **[open]**: `9_OUT1C_INPUT` feeds Teensy pin 9 as TDM2's data-in. Pin 9 is OUT1C; the
audio library's inputs are IN1 (pin 8) and IN2 (pin 5). Confirm this is intentional or
re-purpose.

---

## 5. Edge headers

**[decided]** 2.54 mm headers, **two 2-row headers side by side near each 60.96 mm edge**
(four rows per end, inboard by ~5 mm): **≈168–176 pins**. See §11 for placement and §11.2
for the pin-assignment rules.

**[consequence]** The `board_outline` symbol (lib_sch:t-dsp_core, 106 pins) is the header
net map, but **no outline footprint has pads for it**. A real header footprint must be
built (this is the source of the six `footprint_link_issues` ERC errors). TDM1/TDM2 as
separate 2x10 connectors are redundant with the T*/TT* header groups and leave the core.

**[decided 2026-09-28] GPIO exposure policy: raw pins, not named UI functions.** The
core does not decide what a backplane builds. Pins go to the header as plain GPIO and the
backplane wires TFT, touch, encoder, buttons, LEDs, or nothing. Consequences:

- **S3 side:** every free S3 pin is `S3_IOnn`. The S3's GPIO matrix maps SPI, I2C, UART,
  PWM, and I2S to any pin, so nothing is lost by not naming them. The on-core TFT/touch
  pull-ups (R22/R23/R64/R65), LED jumpers (ILI_LED1–4), and the duplicate UI headers all go.
- **Teensy side:** every free Teensy pin is `T_nn`. The i.MX RT has *fixed* peripheral
  pins, so the header doc carries the Teensy pin-capability table (which pins are I2C,
  SPI, UART, CAN, PWM, analog). **MIDI logic stays on the core** (opto U4, buffer U9, their
  passives): the header carries MIDI IN/OUT/THRU at the *jack level* (existing pins 1–6);
  only the DIN/TRS jacks themselves are on the backplane. Same for DMX: isolation and
  transceiver on the core, the XLR on the backplane.
- **Exceptions, still named, because the core transforms or owns them:** the two buffered
  TDM buses, `I2C0` (shared with the SRC, pull-ups on the core), SRC S/PDIF RX/TX pairs,
  aux I2S in/out, the three USB pairs, S3 `UART0` + `EN` + `IO0` (programming), Teensy
  `PROGRAM`/`ON_OFF`, `VBAT`, **MIDI IN/OUT/THRU (jack level)**, **DMX A/B**,
  **ESP32_LED_OUT / TEENSY_LED_OUT** (SK6812 chain continuation, 5 V level), and all power pins.
- **The reference backplane** (dev board + test jig) is where the opinionated wiring lives:
  TFT, encoder, buttons, MIDI jacks, codec. That is what makes the platform "easy to build
  something useful with", not the core.

Signals to **add** to the existing 106:

| Group | Pins |
|---|---|
| SRC S/PDIF: RX1± … RX4±, TX± | 10 |
| Aux I2S in (BCK, LRCK, DATA) | 3 |
| Aux I2S out (SDOUTA) | 1 |
| S3 USB D−/D+ | 2 |
| S3 RXD0/TXD0 | 2 |
| S3 spare GPIO (IO3, IO45, IO46, IO36, IO37) | 3–5 |
| Teensy 22, 26, 27, 32, 33 | 5 |
| Teensy host USB D+, D−, HOST_5V | 3 |
| 5V_IN (into TPS2116 primary) | 1–2 |
| Extra GND adjacent to USB/clock pairs | ~6 |
| **Total ≈ 106 + 38 = ~144** | fits, barely |

Rules: every USB pair on adjacent pins with GND either side; the same for MCLK/BCLK;
5 V and 3.3 V pins are **output-only** except the dedicated 5V_IN. If short of pins, drop
in this order: aux I2S in/out, duplicate 3.3 V/5 V pins, S3 strapping pins.

---

## 6. USB and programming

**[decided]** **No USB mux.** The Teensy's own micro-USB jack is the core's power input and
the Teensy's programming / USB-audio port. The Teensy **device** pair is **not** on the
header (a second connector would be an unterminated stub on a 480 Mbps pair). A panel USB
for the Teensy on a product is a short extension from the Teensy jack.

**[decided]** Teensy **host** pair and S3 **OTG** pair go to the header. Backplane provides
connectors, ESD, VBUS switching (Teensy HOST_5V is just VIN; the host library has no
power-enable pin).

**[revised 2026-10-01, stage 10y]** All three USB ports are on headers and **none is isolated**
(the ISOUSB211 plan of stage 10x was dropped with the SAI2 rework): Teensy host pair on J4 odd
row (1 GND, 3 HOST_D1-, 5 HOST_D1+, 7 GND, 9 HOST_5V), Teensy device pads 66/67 on J4 even row
(2 GND, 4 T_USB_DN, 6 T_USB_DP, 8 GND, 10 GND), S3 OTG pair on J8 (31 S3_USB_DN, 32 S3_USB_DP,
33 GND, 34 5V). The S3 port is host or device by firmware; J8 pin 34 is the core's 5 V rail, so a
backplane wires it to a jack's VBUS only for a host-mode jack (or through a load switch); a
device-mode jack leaves VBUS unconnected. The Teensy device row has no VBUS pin: the Teensy's
VUSB is NC on the core (stage 10x). Ground-loop hum between a laptop and the externally powered
core is handled on the backplane (isolator or audio-side ground lift) if it shows up.

**[decided]** EN and IO0 on the edge connector (already). On-core prog headers and EN/IO0
buttons leave. **Add a 10 k pull-up on ESP32_EN** — required by the software repo's
`TDspProgrammingKit` README for the next spin and by the S3 module reference design.
Today the net has only the two 0.1 µF caps.

**[verified]** FlasherX path (`@FXUP` over Teensy Serial7 ↔ S3 UART0, EN from Teensy 37,
IO0 from Teensy 36) is fully wired. S3 UART0 is not its USB console, so the tunnel no
longer competes with logging. 115200 is a firmware constant on both ends.

**[consequence]** Teensy is flashed by the S3 via FlasherX only (self-reflash). HalfKay is
USB-only; recovery of a bricked Teensy needs a cable in the Teensy jack.

---

## 7. Power

```
edge header 5V_IN ──► TPS2116 VIN1 (priority) ─┐
Teensy VUSB (pin 49) ► TPS2116 VIN2 (secondary) ┴──► 5V rail ──► Teensy VIN
                                                        ├──► 3.3 V buck  ──► 3.3V_DIG (S3, BT, SRC, mux, ISO7761)
                                                        │                     └► TLV76718 ──► +1V8 (SRC core)
                                                        ├──► LT3045 (U15) ──► 3.3V (buffers, header, analog)
                                                        └──► TEA1-0505 ──► 5V_ISO (DMX)
```

| Item | Status | Action |
|---|---|---|
| TPS2116 input mux | wired (JST + USB-C) | **VIN1 ← header 5V_IN; VIN2 ← Teensy VUSB**. J20 already sits on VIN2. |
| Teensy VUSB↔VIN pad | intact | **Must be cut** on every Teensy. Otherwise PC USB 5 V and the backplane supply are paralleled. Build note. |
| 3.3V_DIG regulator | TLV76733 LDO, 1.3 W in 2×2 WSON at 750 mA | **[decided] replace with a 3.3 V buck**, 1.5–2 A class (TPS62A02 / TPS563xx class). Keep R66 enable pull-up semantics. |
| LT3045 (U15) | output dangling (only C42) | **connect OUT to the 3.3V rail**. SET = 33 k → 3.3 V. Check R58 ILIM value. |
| Teensy 3V3 (pin 46) | sources the 61-node 3.3V rail | **remove from rail; not exposed**. Never parallel two regulators. |
| LP5907 3.3V_A (U109) | mic rail via net-tie + jumper to 3.3V | mics leave the core; **[open]** keep as a second analog LDO or delete. |
| FL2 / CR1 / FB8 branch | dead (5V_unfiltered, GND_unfiltered single-node) | delete |
| 12V | JST + Q2 pass-through to headers | **[open]** keep as pass-through pin only, or drop. Core does not generate it. |
| 3.3V header pins | on rail | output-only, advertise a limit (~200 mA). Never accept 3.3 V in. |
| USB-C PD | none | **not on the core.** Backplane/submodule PD sink (STUSB4500 or CH224K) when >15 W is needed; backplane bucks to 5 V for 5V_IN. |
| MCLK1 | Y1 via R61 **and** Teensy pin 23 via R7 | **[open]** two drivers on one net. Y1 standby jumpers R59/R60 exist. Decide which populates. |

Current budget on the 5 V rail, core alone (revised 2026-09-30):

| Load | Typical | Peak | Note |
|---|---|---|---|
| Teensy 4.1 at 600 MHz + 2x PSRAM | 150 mA | 200 mA | +100 mA more with the Ethernet PHY linked |
| ESP32-S3 (via the 3.3V_DIG buck, ~90 %) | 200 mA | 450 mA | WiFi TX bursts; ~40 mA idle, ~100 mA BLE only |
| IDC777 BT audio | 50 mA | 100 mA | A2DP streaming |
| SRC4382 + 1V8 LDO | 60 mA | 80 mA | |
| DMX: TRACO TEA1-0505 + ISO7762 + RS-485 | 100 mA | 250 mA | 1 W converter at full isolated load |
| SK6812 D3/D4 | 40 mA | 120 mA | full white |
| Buffers, muxes, LDO quiescent | 15 mA | 20 mA | |
| **Core total** | **≈ 0.6 A** | **≈ 1.3 A** | plus Ethernet: ≈ 1.4 A peak |

Plan **1.5 A for the core** and a 2.5-3 A external supply once a backplane with codecs and small
amps hangs off `5V` and `3.3V`. The TPS2116 mux is rated 2.5 A. A PC USB port (500 mA) on the
Teensy jack programs both MCUs and runs the board with the radios idle, but browns out under
WiFi bursts or DMX + LEDs; a USB-C port at 1.5 A is fine for bench use.

---

## 8. Slave-clock option (keeps DAC-master and multi-core architectures possible)

Today the core can only be the audio clock **master**: buffers U5 (LRCK1), U6 (MCLK1),
U7 (BCLK1) have ~OE tied to GND and drive outward; the Teensy's clock pins sit behind them
with no path back from the bus.

**[decided]** Make it switchable, cheaply:

1. **Buffer ~OE pins** (U5, U6, U7, U8 data-out, U11 data-in) move from hard GND to a
   **solder jumper (default GND) or one S3/Teensy GPIO** (`BUS_MASTER_EN`). A slave core
   tri-states its drivers.
2. **Bypass jumpers** from the header-side clock nets (BCLK1+TDM1, LRCK1+TDM1, MCLK1+TDM1)
   to the Teensy-side nets (BCLK1, LRCK1, MCLK1) — one 2-pad solder jumper per line,
   default open. Closed + buffers disabled = Teensy is I2S slave to the bus.
3. **Y1** already has standby jumpers (R59 to VDD / R60 to GND); a slave core populates R60.
4. **SRC I2C address**: add a jumper option on A0/A1 (pins 19/21) so two cores on one I2C
   bus don't collide. Zero cost.
5. **No-radio build variant**: IDC777, S3 antenna path, U20/U21/U23 DNP-able. Multiple cores
   in one box should have radios on one core only.

Firmware: the audio library has I2S slave objects for both Teensy ports (stereo). A TDM
slave object does not exist in the software repo yet.

Why: a serious DAC stage wants the oscillator beside the DAC with the DAC as clock master;
multi-core digital mixing needs one clock master. Both need the core to accept clocks.
Cost is a handful of jumpers.

---

## 9. Cleanups found during the review

- Delete **U2** (ESP32-DevKitC, excluded-from-board, still on 30+ nets) and
  **board_outline1** (TAC5212 module symbol).
- Orphan single-node nets to remove or connect: `Earth`, `GNDD`, `S-GND`, `SN`,
  `ESP32_3.3V`, `5V_unfiltered`, `GND_unfiltered`, `MCLK2` (J19), `32_OUT1B` (J15).
- **Footprints missing**: C130, C131 (5 V decoupling), C132, C133 (3.3V_DIG decoupling),
  R66 (100 k enable pull-up).
- Konnect user config `%APPDATA%\konnect\config.json` line 10: `layer_count` is 2, the
  board is 4-layer. Set to 4 before trusting DRC.
- Antenna keepout for U22: 8 × 4.44 mm at the −X end, all copper removed on every layer
  out to the board edge, plus a PCB keepout rule. Board-edge, top-left per datasheet.

---

## 10. Open decisions (blocking nothing else)

| # | Decision | Default if not decided |
|---|---|---|
| 1 | ~~S3 module variant~~ **[decided 2026-09-28]: ESP32-S3-WROOM-1-N16R8** (8 MB octal PSRAM, required for Spotify Connect stream buffering; costs IO35-37). PCB antenna. | - |
| 2 | ~~Board outline~~ **[decided 2026-09-28]: 100 × 60.96 mm**, see §11 | — |
| 3 | MCLK1 driver: Y1 or Teensy pin 23 | Y1 (24.576 MHz = 512·fs), Teensy pin 23 via R7 removed |
| 4 | 12 V: pass-through pin or drop | pass-through pin, no on-core parts |
| 5 | ~~MIDI logic~~ **[decided 2026-09-28]: stays on the core**, jack-level on the header (user: MIDI and DMX are core features of an audio device) | — |
| 6 | LP5907 3.3V_A: keep or delete | delete (mics leave) |
| 7 | `9_OUT1C_INPUT` intent | ask |
| 8 | ~~External antennas~~ **[decided 2026-09-28]**: BOM options -1U and IDC767, u.FL J13 DNP on pad 57 (see section 11) | — |

---

## 11. Mechanical and placement

**[decided]** Outline **100 mm × 60.96 mm** (`project_fp:audio_project_board_outline_100x60.96`,
already placed as board_outline2). 60.96 mm = the Teensy 4.1's 2.4 in length.

**[decided]** The Teensy spans the full 60.96 mm depth with its **micro-USB end flush with
one 100 mm edge and its microSD end flush with the other**, so a backplane or enclosure
can expose either. Soldered directly, no socket.

**[decided]** The edge headers are on the two **60.96 mm edges**, perpendicular to the
Teensy. The 100 mm edges carry only the Teensy ends and the antennas.

**[consequence] Pin budget.** A 60.96 mm edge holds 24 positions at 2.54 mm with no
margin, ~21–22 with corner mounting holes. One 2-row header per edge = **~84–88 pins
total**, far short of the ~144 in §5. Options:

| Option | Pins | Notes |
|---|---|---|
| **A. Two 2-row headers side by side per edge** (4 rows, 10.16 mm strip) | 168–176 | **recommended.** Fits §5 with ~25 spare. Backplane sees two parallel 2×21/2×22 connectors per edge, a common arrangement. |
| B. One 2-row header per edge + cuts | 84–88 | needs ~55 pins cut: TDM2 bus gone, TDM GND pins thinned, MIDI THRU, aux I2S, S3 spares, SRC RX3/RX4, duplicate power pins. Loses the second TDM bus, which a backplane can replace by chaining modules on one bus. |
| C. 2.0 mm or 1.27 mm pitch | 100+ | rejected: 2.54 mm is a product requirement. |

**[decided 2026-09-28]: option A**, two 2-row headers per short edge (four rows).

**[consequence] Area.** With option A each short edge loses a 10.16 mm strip and the Teensy
takes an 18 mm strip: usable top area ≈ 62 × 61 mm ≈ 3800 mm². Estimated courtyard after
the §1 deletions and the §2/§7 additions ≈ 3300 mm² plus the headers. **Two-sided assembly
is required**: Teensy, S3, IDC777, SRC and headers on top; small ICs and passives on the
bottom. The bottom under the Teensy is usable, the top under it is not (the Teensy carries
bottom-side parts).

**[consequence] Antennas.** Both radios need a board edge with copper removed on every
layer: 8 mm along the edge × 4.44 mm inward for the IDC777, roughly 15 × 6 mm for a
PCB-antenna S3. Both go on the **100 mm edges**, in the 82 mm not occupied by the Teensy
end, offset ≥ 6 mm from the corners so the header strips on the short edges are not cut
into. One radio per long edge, or both on one long edge at opposite ends, ≥ 40 mm apart.

**[decided] Headers sit inboard, not on the edge.** Outer row centreline ≈ 5 mm from the
short edge (rows at x ≈ 5.0, 7.5, 10.1, 12.6 mm and mirrored at the other end). Nothing
requires a mezzanine header to touch the edge, and inboard gives every row an escape
channel on both sides, frees the true corners for mounting holes, and leaves a copper-free
margin. Moving them further inboard than that buys nothing: the routing lever is *which*
signals go to *which* header, set by the placement below.

### 11.1 Placement plan (drawn for routing, X in mm from the left short edge)

```
 x:  0    5   13  15         33  35                                   85  87   95  100
     |  [hdr L 4 rows] [  TEENSY  ]  [ S3 ]  [SRC][mux]   [DMX iso][pwr] [hdr R 4 rows]  |
     |                  USB end @ y=0  ant@y=0                                            |
     |                  SD end @ y=61  [IDC777 ant @ y=61]                                |
     bottom side under the Teensy: 5 bus buffers, series Rs, Y1, TPS2116, input caps
```

- **Left header = Teensy domain.** TDM1 and TDM2 buses (from the buffers directly beneath
  the Teensy on the bottom side, shortest possible path), MIDI, Teensy S/PDIF, CAN3,
  Teensy spares (22/26/27/32/33), Teensy host USB, PROGRAM/ON_OFF, VBAT, 5V_IN, power outs.
- **Right header = S3 / peripheral domain.** TFT SPI + touch, encoder, GPI, both I2C
  buses, S3 USB, S3 UART0, EN/IO0, S3 spares, DMX A/B, SRC S/PDIF RX/TX, aux I2S in/out,
  LED outs.
- **Antennas on the long edges** in the 50 mm between the Teensy and the right header:
  S3 PCB antenna on one long edge, IDC777 on the other, ≥ 40 mm apart.
- **Teensy near the left header** so its left pin row (pins 0–12: MIDI, I2S2, TDM data,
  SPI) is 2–3 mm from the header and the I2S2 pins face the SRC. Its right row (TDM
  clocks 20/21/23, I2C, S3 control) reaches the buffers on the bottom side through vias.

### 11.2 Header pin-assignment rules (for the pin map, not yet drawn)

1. **Order pins along Y to match the physical order of their sources** (Teensy pin 0 at the
   USB end ascending toward the SD end; S3 pins likewise) so traces do not cross.
2. **Outer 2-row header = signals that arrive on the outer layers; inner header = signals
   that arrive through the bottom side or In2.** Keeps layer changes near the pins.
3. **One GND per ≤ 4 signals, and a GND adjacent to every clock and every differential pair.**
   MCLK (24.576 MHz), BCLK, and the USB pairs get GND on both flanks.
4. **Differential pairs on adjacent pins in the same row**, never across rows: Teensy host
   USB, S3 USB, SRC RX1–RX4, SRC TX.
5. **Power pins at the ends of each block** (5V_IN, 5 V, 3.3 V, 12 V pass-through, GND) so
   wide copper stays out of the signal field.
6. **All four TDM lines of a bus in one row, contiguous, with GND between**: MCLK, BCLK,
   LRCK, DATA_OUT, DATA_IN.
7. **Reserve ≥ 8 spare pins per block** at the inner end, labelled RESERVED, never repurposed
   without a header-contract version bump.
8. **Name every pin by function, not by MCU pin number** (the header is the product API;
   a future core with a different MCU must fit the same backplane).

Stackup for routing: F.Cu signal, In1.Cu solid GND, In2.Cu signal with 3.3 V / 5 V islands,
B.Cu signal + bottom-side parts. Every header signal has In1 GND directly beneath it.

**[decided 2026-09-28] Teensy memory.** The Teensy 4.1 carries two bottom-side SOIC-8 pads for
PSRAM/flash. Fit **2 x APS6404L-3SQR-SN** (16 MB PSRAM, mapped as EXTMEM) for loop recording,
echo and reverb buffers, or one PSRAM + one W25Q128 flash. The chips are 1.2 mm tall on the
Teensy's underside, so the core's top side under the Teensy footprint is a **placement keepout**
and the Teensy mounts on 2.54 mm headers (not soldered flat). Power: about 10 mA per PSRAM at
3.3 V from the Teensy's own regulator.

**[decided 2026-09-28] Ethernet.** The Teensy's 10/100 PHY pairs are on six 2 mm-pitch pads
(60-65) on its underside. They are **not** reachable by a cable once the Teensy is mounted, so
the core picks them up: pads 60-65 return to the Teensy footprint (they exist in
`Teensy41.kicad_mod`, were dropped in `_mod3`) and route under 20 mm as two 100 ohm pairs to
**J12 `J_ETH`** right beside them. Magnetics, RJ45 and the LED resistor live on the backplane.

**[decided 2026-09-28] Antennas: on-board by default, off-board as a BOM option, no layout change.**
- S3: `ESP32-S3-WROOM-1-N16R8` (PCB antenna, C2913202) or `ESP32-S3-WROOM-1U-N16R8` (u.FL on the
  module, C3013946). Same pad layout; the -1U is 6 mm shorter. Keep the top-edge copper keep-out in
  both builds.
- Bluetooth: `IDC777-1` (chip antenna) or `IDC767` (external antenna). Verified from datasheets
  IDC767-DTS-V003 and IDC777-DTS-V003: identical 60-pad LGA and pin table; **pad 57 is EXT_RF on
  both** ("RF to EXT Antenna (Ext ANT SKU - IDC767)"), unused on the IDC777. The core routes pad 57
  to a DNP u.FL (J13) as a 50 ohm coplanar waveguide with ground pads 56 and 58 on either side,
  ground vias every 1-2 mm along it, trace as short as possible, continuous ground on the layer
  below. No matching network. With the IDC777 fitted the stub is simply unused.
- Off-board antennas also let the two radios be separated further than the 61 mm the board allows.

**[open] Mounting.** Four M3 (or M2.5) holes for standoffs to the backplane, needed for
header retention. Corners are the natural spot; each corner hole costs ~2 header positions
on that edge, already counted above. The 100x60.96 outline footprint currently has **no
holes and no header pads**; both are added when the header footprint is built.

---

### 11.3 PCB placement as executed (2026-09-28, Konnect IPC, single-sided top assembly)

Board-local coordinates (X from the left short edge, Y from the top long edge; KiCad x = 100 + X,
y = 139.04 + Y). Everything is on F.Cu. Collision-checked against real courtyards and every Teensy pad;
DRC after placement: no shorts, no clearance errors, 499 unrouted connections (expected).

| Block | Where | Notes |
|---|---|---|
| U1 Teensy 4.1 | X 15.7-33.5, full height, USB end at the top edge, rot 180 | mounts on 2.54 mm headers; PSRAM on its underside |
| Oscillator Y1/FB1/C44-46/R59-61/C129 | under the Teensy, X 23.5-30.7, Y 6-16.5 | next to Teensy pin 23 (MCLK); clear of the D+/D-/VUSB pads |
| Teensy series Rs, SPI pull-ups, C2-C5 | under the Teensy, three columns X 20/24/28, Y 18.5-44 | clear of the USB-host pads and the PROGRAM/ON-OFF pad row; flip to B.Cu if you prefer nothing under the Teensy |
| J2 J_TDM + U5-U8/U11 + R40-47/R55/56 + C17-20/C40 | left column X 1.5-13.5 | buffers 3 mm from the header rows |
| J4 J_USBH, J12 J_ETH, J3 J_T | strip X 33.6-39.3 right of the Teensy, top to bottom | USB host and Ethernet opposite the Teensy's own pads |
| C27-C30 | X 41, Y 3-17 | 3.3 V rail decoupling |
| MIDI U4/U9/D2/C16/C21 | X 53.5-63, Y 41-56.9, right of the IDC777 | R31-R35 in a row at X 54.5-65.7, Y 34-37.7; R36-R39 in spare slots under the Teensy; J5 (MIDI header) at the top-middle X 40.7-46.8, Y 1.2-14.5 |
| DMX U17/U18/U16/D6/R71 + J10 | X 56-75, Y 8-28; C134-C139 in the strip above U17 (X 54-60.5, Y 0-8) | isolated side kept together; J10 at X 42-56 |
| U13 ESP32-S3 | X 75.3-93.8, Y 0-26.5, antenna on the top edge | R67/R68/R73/PU_EN1 row at Y 28 |
| J8 J_S3 | right edge X 94-99.5, Y 6.6-45 | |
| U19 SRC4382 + C140-149 + U20 | X 68.7-86.4, Y 30-45 | |
| U21/U23 mux + C146/C152 | X 87.6-93.6, Y 29-49.5 | |
| J6 J_SPDIF | X 39.5-68.4, Y 27.5-34 | directly left of the SRC |
| J7 J_AUX | top-middle X 47-53.1, Y 1.2-14.5 beside J5 | |
| LED IC1/C1/C33/C22/C23 + J11 | top-middle: J11 at Y 16-19.3, IC1 and the four caps in a row at Y 21.5, X 41-55 | D3/D4 at X 54-64, Y 38.1-40.3, now top-emitting SK6812-EC20 (no board windows needed) |
| U22 IDC777 + J13 u.FL (DNP) | vertical, X 40-53, Y 37.5-61, rot 90, antenna end on the BOTTOM edge; J13 at X 55.7, Y 59.4 with its RF pad toward the module, 3.5 mm from EXT_RF pad 57 (X 52.3, Y 56.8) | the IDC777 antenna is at the module's short end (keep-out pads 56/58 side), not along its long edge; antenna zone X 42.5-50.5, Y 55.9-61 must stay copper-free on all layers. Opposite edge from the S3 antenna, ~63 mm apart |
| J1 J_PWR | X 64.6-80.1, Y 41.6-46.9 | |
| U3 TPS2116 + C12-15/C31/32/34/35 + R28-30 | row Y 49.5, X 64.5-91 | |
| U24 buck + L1 + C153/154/R66/C130/131 + R57 | row Y 54, X 64-89 | |
| U15 LT3045 + FL1 + C41-43/C132/133 + R58 | row Y 58.6, X 64-87 | |
| J9 J_USB3 | X 91.6-94.4, Y 47-60 | |
| H1-H4 | 3.2 mm NPTH at the corners (3.5, 3.5) (96.5, 3.5) (3.5, 57.46) (96.5, 57.46) | on the PCB after the user's re-sync; schematic footprint changed to `MountingHole:MountingHole_3.2mm` (3.7 mm courtyard) so the next sync clears the courtyard clashes with J2/J8/J9 |

Antenna keep-outs (2026-09-28, second pass): nothing sits inside the ESP32 footprint's keep-out rectangle
(X 60.5-100, Y 0-6.3) any more, and the IDC777 was turned vertical on the bottom edge so its antenna end faces outward, opposite the S3. Both
zones still need KiCad rule areas (no copper, all layers) before pouring ground.
**IDC777 keep-out, to be added to the footprint (2026-09-28, datasheet IDC777-DTS-V003 p.8-9).** The
"Ground Clearance Area" is 8.0 x 4.44 mm at the pad-free end of the module: no metal on any layer of the
host PCB, extended to the board edge, ground vias along its boundary through all layers. In
`project_fp:IDC777-1` it exists only as four `Dwgs.User` lines (local x -11.10..-6.66, y -4.0..+4.0) and a
text note, which DRC ignores. To make it enforceable like the ESP32 footprint, open the footprint in the
Footprint Editor and add:
1. a **rule area** inside the footprint, rectangle local x -11.75..-6.66, y -4.0..+4.0 (from the module
   end to the inner edge of the antenna), all copper layers, keep out copper pours, tracks, vias and
   footprints;
2. a **F.CrtYd** rectangle over the same area so courtyard DRC catches parts placed on it.
Konnect cannot edit footprint graphics (its library toolset only creates pad layouts and edits pads).
On this board (U22 at rot 90, origin X 46.5, Y 49.21) the area lands at **X 42.5..50.5, Y 55.87..60.96**;
until the footprint carries it, draw a board rule area there (x 142.5..150.5, y 194.91..200.0 in KiCad
coordinates) and stitch ground vias along X 42.5, X 50.5 and Y 55.5. The ESP32 keep-out is drawn as
courtyard only, so it also needs a board rule area for copper: X 75.3..93.8, Y 0..6.3 (x 175.3..193.8,
y 139.04..145.34).

Known DRC leftovers: courtyard overlaps against U13 caused by the stock ESP32-S3-WROOM-1
footprint's 15 mm antenna courtyard (the module body itself is clear); U13/U20 footprint-type
attribute mismatches and the WSON thermal-via 0.2 mm drills, both inherited footprint properties.
Konnect cannot flip footprints to B.Cu, so the board is single-sided as placed.

### 11.4 Layout rework (2026-09-28, stage 10m): four header rows, three slots, two-sided assembly

The single-sided placement of 11.3 was rejected: it fitted, but the passives were dropped into
generic columns (median 9.9 mm from the pin they serve, 59 of 118 more than 10 mm away) and the
top side was 80 % courtyard. Decisions:

- **Two-sided assembly, 4-layer stack**: F.Cu signal, In1.Cu GND plane, In2.Cu power plane, B.Cu
  signal. Chips, modules, headers and every capacitor stay on the front; resistors, ferrites,
  diodes and the EN solder jumper go on the back, directly under the pin they serve.
- **Headers in four vertical rows** (X = pin-1 column, mm from the left edge), each next to the
  slot it serves, so the backplane mates four straight connector rows instead of twelve scattered
  headers. J9 was folded into J8 (2x17) and J1 trimmed to 2x5 to make the rows fit the 61 mm height.

| Row | X (pin 1 column) | Headers top to bottom |
|---|---|---|
| A (left edge) | 3.0 | J3 J_T 2x12 (Y 8.75), J12 J_ETH 2x4 (Y 40.6); sits between H1 and H3 |
| B | 35.5 | J2 J_TDM 2x13, J5 J_MIDI 2x4, J4 J_USBH 1x5 |
| C | 63.0 | J6 J_SPDIF 2x11, J1 J_PWR 2x5, J7 J_AUX 2x5 |
| D (right edge) | 94.46 | J8 J_S3 2x17 (Y 8.75; S3 GPIO, UART0, EN/IO0, I2C, USB pair, 5V); sits between H2 and H4 |
| horizontal | - | J10 J_DMX 1x5 along the bottom edge of slot 2 (X 40.5-54.2, Y 57-60.5); J11 J_LED 1x5 vertical at X 41.77 in the strip left of the IDC777 |

Revised 2026-09-28 (stage 10n) at the user's request: the two outer rows are at the board edges and
fit between the corner holes (47 mm usable), so J4 moved to row B, J1 to row C, J10 and J11 to slot 2.

| Slot | X range | Contents |
|---|---|---|
| 1 | 13.3-33.7 | Teensy U1 full height; TDM buffers U5-U8/U11 on the front under it; Teensy passives on the back |
| 2 | 39.8-61.2 | IDC777 U22 on the right (X 48.1-61.2, Y 0.4-23.9), antenna out the top edge (ground-clearance keep-out X 50.6-58.6, Y 0-5.5); in the strip to its left: J11, J13 u.FL beside pad 57, LED driver IC1 with D3/D4; DMX isolated block (U18 horizontal, U17, U16, U20) in the middle; MIDI (U4, U9, D2) at the bottom above the horizontal J10 |
| 3 | 67.3-86.7 | power (U24 + L1, U3 in the top row, U15 below) at the top; SRC4382 U19 (Y 15.5) and muxes U21/U23 (Y 28.5) in the middle; ESP32-S3 U13 at the bottom with its pads at Y 35.2-53.2 and the antenna out the bottom edge (keep-out X 68-86, Y 53.2-61) |

Antenna-to-antenna distance is about 45 mm on opposite edges. Passive placement is net-driven
(scratchpad `placer2.py`): decoupling caps at the supply pin of the chip drawn next to them in
the schematic, pull-ups and series parts at the IC pin, chains through their neighbour; accepted
only with the passive-to-pin metric reported (target median < 3 mm, nothing > 6 mm).


**Passive footprints (stage 10n, 2026-09-28):** the hand-solder 0805 footprints (3.7 x 1.9 mm courtyard)
made it impossible to seat decoupling caps at the pins, so every 0805 resistor and every capacitor up
to 10 uF is now a plain 0603 (`Capacitor_SMD:C_0603_1608Metric`, `Resistor_SMD:R_0603_1608Metric`)
with JLCPCB basic parts: 100 nF C14663, 1 uF C15849, 10 uF C19702 (10 V, X5R), 4.7 uF C19666,
resistors UNI-ROYAL 0603WAF: 0R C21189, 33R C23140, 49.9R C23185, 120R C22787, 220R C22962,
300R C23025, 1.5k C22843, 2.2k C4190 (R13/R14/R16/R17/R67/R68 were 2.21k), 5.1k C23186, 10k C25804,
33k C4216, 100k C25803. Kept: R28 16.9k 0805 (C17484, no 0603 basic part), the 22 uF / 47 uF 0805 and
the 100 uF 1206 bulk caps, the three 0402 parts. Board footprints, values and fields were synced from
the schematic with pcbnew (`swap_fp.py`, `sync_fields.py`), so a plain *Update PCB from Schematic*
reports no changes.
**Result on the board (stage 10n):** passive-to-pin median 2.2 mm (was 9.9), 6 of 118 more than 10 mm
(R1/R2 Teensy TDM2 series resistors, R11, R52, 3.3V_DIG caps in the crowded DMX / power blocks).
DRC: no shorts or clearance errors; only the stock ESP32 courtyard overlaps and inherited drill
warnings. **Open:** C31 (10 uF) and C32 (100 nF) have pin 1 connected only to each other, a leftover
pair that decouples nothing; delete or assign a rail.

### 11.5 Stage 10o (2026-09-29): Teensy in the middle, MIDI + DMX top right, SRC + IDC777 left

User request. Same four rows and two-sided rule as 11.4; contents re-dealt:

| Row | X (pin 1 column) | Headers top to bottom |
|---|---|---|
| A (left edge) | 3.0 | J6 J_SPDIF 2x11 (Y 8.75), J7 J_AUX 2x5 (Y 38.1); between H1 and H3 |
| B | 35.5 | J3 J_T 2x12 (Y 3.0), J4 J_USBH 1x5 (Y 34.9), J11 J_LED 1x5 (Y 49.0) |
| C | 63.0 | J5 J_MIDI 2x4 (Y 3.2), J2 J_TDM 2x13 (Y 14.8), J12 J_ETH 2x4 (Y 49.2) |
| D (right edge) | 94.46 | J8 J_S3 2x17 (Y 8.75); between H2 and H4 |
| horizontal | - | J1 J_PWR along the top edge of slot 1 (X 8-21.7); J10 J_DMX along the top edge of slot 3 (X 78.5-92.2) |

| Slot | X range | Contents |
|---|---|---|
| 1 | 7.3-33.7 | power (U24, L1, U3, U15) top right under J1; SRC4382 U19 (X 7.6, Y 14) with muxes U21/U23 beside it; IDC777 U22 bottom-left (X 7.4-20.5, Y 37-60.5), antenna out the bottom edge (keep-out X 9.9-18, Y 55.4-61); J13 u.FL beside its pad 57; LED driver IC1 + D3/D4 next to J11 |
| 2 | 39.8-61.2 | Teensy U1 full height centred X 50.5; TDM buffers U5-U8/U11 on the front under it (X 48.7-52.2); Teensy passives on the back |
| 3 | 67.3-92.7 | MIDI (U4, U9, D2) at the top under J10 and beside J5; DMX isolated block (U18 horizontal Y 14.2, U17, U16, U20) below it; ESP32-S3 U13 at the bottom centred X 80, pads Y 35.2-53.2, antenna out the bottom edge (keep-out X 71-89, Y 53.2-61) |

Result: passive-to-pin median 2.4 mm, 5 of 118 more than 10 mm; signal ratsnest 3.5 m (3.7 m in 10n).
DRC: only the stock ESP32 courtyard overlaps and inherited drill warnings; parity clean apart from
board_outline2. Both antennas are on the bottom edge, 53 mm apart (IDC777 X 10-18, S3 X 71-89).

**Stage 10p (2026-09-29, user request):** SRC4382 U19 moved under the Teensy, on the front, below the
Teensy's interior VBAT/3V3/PROGRAM/ON-OFF pad row (box X 45.3-55.6, Y 49-59.3, between the two pin
columns at X 42.9 and 58.1). Muxes U21/U23 stay in the left slot at X 7.6 (Y 14-25.5). The Teensy on
8.5 mm headers leaves room for the 1.2 mm TQFP under it. Cost: the SRC's six 1V8/3V3 caps cannot sit
beside it (1.6 mm to the pin columns) and land 4.5-11 mm away above the pad row; the S/PDIF pairs from
J6 (left edge) now run about 40 mm to the SRC. Passive-to-pin median 2.6 mm, 8 of 118 more than 10 mm.

**Stage 10q (2026-09-29, user request):** the two 74LVC157 muxes U21/U23 go with the SRC: both under the
Teensy, rotated 90, side by side at X 45-50.5 / 51-56.5, Y 38.5-46.2, just above the interior pad row;
the SRC sits below that row (Y 49-59.3). The five TDM buffers moved up to Y 5-34 (5.9 mm pitch) to make
room. "SRC" in the user's instructions means SRC4382 plus its two muxes. Passive-to-pin median 3.3 mm.

**Stage 10r (2026-09-29, user request): TDM fan-out buffers consolidated.** The four SN74LVC2G125
(U5-U8, all enables hard-wired low) that copied LRCK1 / MCLK1 / BCLK1 and the two Teensy TDM data
outputs to J2 are replaced by one **SN74LVC541APWR** (U5, TSSOP-20, LCSC C113281, extended part).
Group A (OE1, pin 1, grounded) = TDM1 copies: Y0 LRCK1 -> R42, Y1 MCLK1 -> R41, Y2 BCLK1 -> R44,
Y3 OUT1A -> R46. Group B (OE2, pin 19 = `TDM2_OE`) = TDM2 copies: Y4 LRCK1 -> R43, Y5 MCLK1 -> R40,
Y6 BCLK1 -> R45, Y7 OUT1D -> R47. `TDM2_OE` is held low by solder jumper **JP1** (bridged,
`Jumper:SolderJumper-2_P1.3mm_Bridged`) with 10k pull-up **R74** to 3.3V: cut JP1 to tri-state
the whole TDM2 clock/data set when a backplane device (Pi CM) is the TDM2 clock master. The
series resistors R40-R47 and U11 (the 2G125 on the Teensy-input lines) are unchanged; C18-C20
removed, C17 is the 541's decoupling cap. Net names between U5 and the resistors: `TDM1_*_B`,
`TDM2_*_B`. Board synced with pcbnew (`swap_fp.py` now also adds/removes footprints and sets pad
nets from the netlist; `sync_fields.py` refreshes values, fields and pad nets). U5 sits at the top
of the column under the Teensy (X 47-54, Y 5-12.7), U11 below it.

### 11.6 Stage 10s (2026-09-29): holistic re-spacing, S3 and IDC777 swapped

User request: keep the flow (power top-left, TDM buffers middle, MIDI/DMX top-right) but give every part
air, and try the S3 in the wider left slot. Result:

| Row | X (pin 1 column) | Headers top to bottom |
|---|---|---|
| A (left edge) | 3.0 | J8 J_S3 2x17 (Y 8.75), next to the S3 |
| B | 35.5 | J3 J_T 2x12 (Y 3.0), J4 J_USBH 1x5 (Y 34.9), J11 J_LED 1x5 (Y 49.0) |
| C | 63.0 | J5 J_MIDI 2x4 (Y 3.2), J2 J_TDM 2x13 (Y 14.8), J12 J_ETH 2x4 (Y 49.2) |
| D (right edge) | 94.46 | J6 J_SPDIF 2x11 (Y 8.75), J7 J_AUX 2x5 (Y 38.1) |
| horizontal | - | J1 J_PWR top edge of slot 1 (X 8-21.7); J10 J_DMX top edge of slot 3 (X 78.5-92.2) |

| Slot | X range | Contents |
|---|---|---|
| 1 | 7.3-33.7 | power row under J1 (U24 X 9, L1 X 15, U3 X 21.8, U15 X 26.6, all Y 7.5-12.3); ESP32-S3 U13 centred X 20, pads Y 35.2-53.2, antenna out the bottom edge (keep-out X 11-29, Y 53.2-61); LED driver IC1 + D3/D4 in the strip X 29.8-32.8 beside the S3, next to J11 |
| 2 | 39.8-61.2 | Teensy U1 centred X 50.5; under it U5 (SN74LVC541A, Y 5-12.7), U11 (Y 15.2-20.7), muxes U21/U23 (Y 38.5-46.2), SRC4382 U19 (Y 49-59.3) |
| 3 | 67.3-92.7 | MIDI U4 (X 69.2-80.2, Y 5-14), U9, D2; DMX block U18 horizontal (Y 15.5-22), U17 (X 69.2-81, Y 23.5-34.3), U16/U20 (X 83.6-91); IDC777 U22 (X 69.2-82.3, Y 37-60.5) antenna out the bottom edge (keep-out X 71.7-79.8, Y 55.4-61); J13 u.FL beside pad 57 at X 82.4-86.8 |

Every chip has at least 1.6 mm to the nearest header courtyard and 2 to 3 mm to the next chip; the
passive placer margin is now 0.35 mm part-to-part (was 0.15). Antennas 45 mm apart on the bottom edge.
Passive-to-pin median 3.4 mm, 11 of 116 more than 10 mm. DRC: only the stock ESP32 courtyard overlaps
and the inherited drill warnings; parity clean apart from board_outline2.
**Open:** the 49.9 ohm series resistors (20x) may move to 0402 or to 4-element 0804 arrays
(Yageo AF124-FR-0749R9L, LCSC C6463554, extended); no MELF 0204/0206 49.9 ohm is stocked at JLCPCB.

**Stage 10t (2026-09-29, user request):** IDC777 moved as far right as its u.FL allows (U22 X 74.6-87.7,
J13 X 87.8-92.2, antenna keep-out X 77.1-85.2, Y 55.4-61; 45.6 mm clear between the S3 body and the
IDC777 body). J8 lowered to Y 9.7-53.9 so it sits beside the S3; row B is now J4 J_USBH (Y 1.2-14.9,
beside the Teensy host pads) / J3 J_T (Y 15.3-46.8) / J11 J_LED (Y 47.2-60.9). Nothing else moved.
**Series terminations (stage 10u):** user accepts anything from 22 to 100 ohm, easy to source. JLCPCB
stocks no 22-100 ohm MELF (0204/0206); chosen: **33 ohm 0402, Uniroyal 0402WGF330JTCE, LCSC C25105**
(basic part), footprint `Resistor_SMD:R_0402_1005Metric`, on R1-R7, R9, R10, R40-R47, R52, R55, R56
(twenty parts, were 49.9 ohm 0603). 22 ohm C25092 and 100 ohm C25076 are the basic alternatives.

**Stage 10v (2026-09-29, user request): OUT1B buffered, OUT1B and OUT1C direction-switchable.**
Before: OUT1A/OUT1D were output-only through the 541, IN1 and OUT1C input-only through U11, and
OUT1B (Teensy pin 32) reached the backplane only as raw GPIO `T32_OUT1B` on J3 pin 20 through R5.
Now: **U6** (SN74LVC2G125, C206035) buffers OUT1B in both directions to **J2 pin 27** (`32_OUT1B`;
J2 is a 2x14, pin 28 GND): half 1 Teensy -> J2 through R75 (33 ohm), half 2 J2 -> Teensy through
R76. **U7** (2G125) adds the Teensy -> J2 direction for OUT1C through R81 next to the existing U11
input half; U7's spare half is parked (2A GND, 2OE high, 2Y NC). Direction is chosen with 0 ohm
0402 jumper pairs (C17477) on the enable pins, one of each pair fitted:

| Line | Enable net | to GND | to 3.3V | Default |
|---|---|---|---|---|
| OUT1B output (U6 half 1) | `OUT1B_OE_OUT` | R77 fitted | R78 DNP | output |
| OUT1B input (U6 half 2) | `OUT1B_OE_IN` | R79 DNP | R80 fitted | disabled |
| OUT1C output (U7 half 1) | `OUT1C_OE_OUT` | R82 DNP | R83 fitted | disabled |
| OUT1C input (U11 half 2) | `OUT1C_OE_IN` | R84 fitted | R85 DNP | input |

To turn a line around, move both jumpers of that line. Never fit both enables of one line low.
`T32_OUT1B` stays on J3 pin 20 as the raw pin (it is the buffer's Teensy-side node). C18/C19 are
the new decoupling caps. Row C on the board: J5 pin 1 Y 3.2, J2 (2x14) pin 1 Y 14.4, J12 pin 1 Y 51.0.

**Stage 10w (2026-09-29, user request):** LED driver IC1 (X 17.8-22.3, Y 21-23.6) with D3/D4 below it
(Y 25.6-28) moved to the middle of the left column, in the free band between the power row and the S3.
J11 stays at the bottom of row B.

### 11.7 Stage 10x (2026-09-30): mandatory external supply, protected input, isolated USB device ports

**Power entry.** The TPS2116 mux (U3), its PR1 divider R28/R29/R30 and the Teensy-USB bulk caps
C13/C14 are gone; Teensy VUSB (pin 49) is NC (the VUSB-VIN pad is cut on every Teensy, so the
Teensy jack is a data-only service port). `5V_IN` (J1 pins 1/2) -> **D7** SMAJ5.0A TVS to GND
(C78401) -> **Q1** AO3401A P-MOSFET reverse-polarity protection (C15127; drain 5V_IN, source
`5V`, gate GND) -> `5V` rail (PWR_FLAG #FLG10). The core needs the external supply to run; a PC
USB port no longer powers it. J1 pins 5/6 stay 5V outputs. Current budget: change map section 7.

**USB isolation on the core (decision: both device ports, host port stays direct).** Two
**ISOUSB211DPR** (TI, high-speed USB isolator, LCSC C5772877, extended, ~$11), one per port:
- S3 port: module side `S3_USB_DN_M`/`S3_USB_DP_M` (U13 pins 13/14) -> isolator side 2 (DD-/DD+,
  VCC2/VBUS2 from `5V`, GND2 = GND); laptop side -> J8 pins 31 `S3_USB_DN`, 32 `S3_USB_DP`,
  33 `S3_USB_GND`, 34 `S3_VBUS` (was GND / 5V). Side 1 is powered from the laptop's VBUS.
- Teensy device port: Teensy pads 66/67 (`T_USB_DN_M`/`T_USB_DP_M`) -> isolator side 2; laptop
  side -> **J4 J_USB 2x5** even row: 2 `T_USB_GND`, 4 `T_USB_DN`, 6 `T_USB_DP`, 8 `T_USB_GND`,
  10 `T_VBUS`. The odd row keeps the unisolated host port: 1 GND, 3 HOST_D1-, 5 HOST_D1+, 7 GND,
  9 HOST_5V.
- `S3_USB_GND` and `T_USB_GND` are isolated grounds: never tie them to core GND or to each other,
  on the core or on a backplane. A backplane wires each jack straight to its four pins; VBUS must
  be brought in (it powers the isolator's laptop side).
- Symbol `ISOUSB211DPR:ISOUSB211DPR` (lib_sch/ISOUSB211DPR.kicad_sym, made from the datasheet pin
  table) and footprint `project_fp:SSOP-28_7.5x10.3mm_P0.65mm_ISO` (IPC land pattern, 8.2 mm
  clearance between the pad rows) were created with Konnect. **Manual step:** place two instances
  (U25 for the S3 port, U26 for the Teensy port) in eeschema; Konnect cannot place project-library
  symbols. Wiring after placement: EQxx to GND (default equalisation), CDPENZx to the local 3.3 V
  (CDP off), V1OK/V2OK NC, caps per datasheet 8.3 (1 uF on VBUSx, 0.1 uF on V3P3Vx, 2 uF + 0.1 uF
  + 10 nF on V1P8Vx; pins 4-11 and 18-25 tied), 45 ohm-ish 90 ohm differential routing, no vias
  on D+/D- (datasheet 8.4).

> The USB-isolation half of stage 10x was **reversed in stage 10y** (below): no isolators, all
> three ports unisolated on the headers. The ISOUSB211DPR symbol library, its SSOP-28 footprint
> and the sym-lib-table entry are still in the repo, unused; remove them in the KiCad library
> manager when convenient.

### 11.8 Stage 10y (2026-10-01): SAI2 aux input block replaces the SRC4382 + muxes + S/PDIF header

**Why.** The SRC4382 was the only consigned, non-stock IC on the core, it has no TDM mode, and the
Teensy can resample in software. The user's design note "SAI2 input routing + expansion header"
replaced the hardware ASRC path with a plain, interlocked SAI2 port. S/PDIF is served by the
Teensy's own pins 14/15 on J3 (J6 removed).

**Architecture (schematic commit 026acc0, board 0dd4533).** One asynchronous I2S source at a time
enters the Teensy's SAI2 (pins 4 BCLK2, 3 LRCLK2, 5 IN2, 2 OUT2, 33 MCLK2); SAI1 stays the TDM
engine for the codec backplane. Every driver reaches the shared `SAI2_*` / `AUX_I2S_*` nets through
its own 33 R 0402 (C25105), and a 74LVC138 interlock guarantees that only one driver group is ever
enabled:

| Part | Function | Enable |
|---|---|---|
| U19 SN74LVC257APWR (TSSOP-16, C205921) | 2:1 mux, A = IDC777 `BT_BCK/BT_LRCK/BT_SDATA`, B = S3 `S3_I2S_BCK/LRCK/DOUT`; S = `SAI2_SEL` (Teensy 22 = T22, 10 k pull-down: 0 = IDC777, 1 = S3). Outputs -> 33 R -> `SAI2_BCLK/LRCLK/IN`. Channel 4 inputs grounded, 4Y NC. | `SAI2_nOE_INT` = U23 Y0 (mode 00) |
| U21 SN74LVC244APWR (TSSOP-20, C7668) | Group 1 (core slave, header master): `AUX_I2S_BCK/LRCK` -> 33 R -> `SAI2_BCLK/LRCLK`; `SAI2_MCLK` -> 33 R -> `AUX_I2S_MCLK`; `SAI2_OUT` -> 33 R -> `AUX_I2S_DOUT`. Group 2 (core master): `SAI2_BCLK/LRCLK/MCLK/OUT` -> 33 R -> `AUX_I2S_BCK/LRCK/MCLK/DOUT`. | group 1 `SAI2_nOE_HDR` = Y1 (mode 01); group 2 `SAI2_nOE_OUT` = Y2 (mode 10) |
| U20 SN74LVC2G125DCTR (SOP-8, C206035) | `AUX_I2S_DIN` -> gate 1 and gate 2 (inputs tied) -> 33 R each -> `SAI2_IN` | gate 1 Y1, gate 2 Y2 (either header mode) |
| U23 SN74LVC138APWR (TSSOP-16, C485077) | A1:A0 = `SAI2_MODE1:MODE0` (Teensy 27/26 = T27/T26, 10 k pull-ups C25744), A2/E1/E2 GND, E3 3.3V_DIG; Y3-Y7 NC | - |

Mode table (firmware contract; the power-on default is 11 = everything tri-stated):

| MODE1:0 | Core role | Clock source | Enabled | Teensy SAI2 |
|---|---|---|---|---|
| 00 | internal | IDC777 or S3 (by `SAI2_SEL`) is I2S master | U19 | slave (BCLK2/LRCLK2 inputs), MCLK2 unused |
| 01 | header **slave** | external device on J_SAI2 is master | U21 group 1, U20 gate 1 | slave; MCLK2 still output to the header (buffered) |
| 10 | header **master** | Teensy is master | U21 group 2, U20 gate 2 | master (BCLK2/LRCLK2/MCLK2 outputs) |
| 11 | off | - | nothing | idle |

The firmware must set MODE and SEL **before** starting SAI2 and must never leave a Teensy-driven
clock enabled while switching to 00 or 01 (the 33 R resistors limit, but do not prevent, contention
during a bad sequence). Decoupling: 100 nF 0603 (C14663) C140 (U19), C141 (U21), C142 (U23),
C143 (U20). All four ICs run from `3.3V_DIG`.

**J_SAI2 = J7 (2x6, right edge, row D).** 1 GND, 2 GND, 3 `AUX_I2S_MCLK`, 4 GND, 5 `AUX_I2S_BCK`,
6 GND, 7 `AUX_I2S_LRCK`, 8 GND, 9 `AUX_I2S_DIN` (into the core), 10 `AUX_I2S_DOUT` (out of the
core), 11 `3.3V_DIG`, 12 `5V`. A GND flanks every clock in the even row. The old J_AUX 2x5
assignment and J6 J_SPDIF 2x11 are gone.

**Other pin moves.** J3 pins 17/18/19/22 (were T22/T26/T27/MCLK2) are GND. BT_RST and
BT_SYS_CTRL, previously SRC GPOs, now come from S3 IO40 / IO41; J8 pins 25/26 (were IO40/IO41)
are GND. USB: see section 6 (revised) and the note above 11.8.

**Removed.** U19 SRC4382 + C144-C149/C152 + R1/R2/R10, U20 TLV75718 1V8 LDO, U21/U23 74LVC157,
J6, the `+1V8` rail, the `SPDIF_RX*/TX*`, `SRC_A_*`, `MUX1_*`, `AUX_SEL`, `BT_S3_SEL` nets.

**Board.** U19/U23 sit under the Teensy at Y 38.5 (slot 2, where the two 157 muxes were); U21 in the
right column under U16 at (83.6, 29.3) and U20 beside the IDC777 at (88.4, 38.5, 90), both next to
J7; D7/Q1 (stage 10x) in the power row where the TPS2116 was. 131 passives re-placed two-sided
(median 2.4 mm, mean 3.4 mm to their pin; 8 beyond 10 mm). Three stray `/OUT1B_MCU` track
fragments from the old design were removed; the board has no routing. DRC with schematic parity:
only the pre-existing U13 courtyard (the module's antenna keep-out) against H3/J11, the U13 0.2 mm
thermal-via holes and the `board_outline2` outline footprint remain.

**Verification.** `kicad-cli sch export netlist` diffed against the stage 10x netlist: only the
nets listed above changed; ERC adds no violation versus commit 1c8bd6e (one pre-existing error:
D3 GND is a power-input pin on the project LED symbol). Hazard found on the way: Konnect
`add_schematic_text` writes literal newlines into the text string, which KiCad refuses to load;
multi-line notes must be separate single-line texts. Konnect `batch_delete` with a symbol's pin
uuid deletes the whole symbol (D3 had to be re-placed from the committed file).

### 11.9 Stage 10z (2026-10-02): DMX leaves the core

**Decision.** The isolated DMX block moves to its own board, `t-dsp_dmx_module` (sibling project
folder). MIDI stays on the core. Reasons: isolation belongs at the XLR jack (with the isolator on
the core, every backplane had to continue the barrier around J10), the isolated corner and its
plane split were the main obstacle to routing the core, and it cost about 9 USD per core plus
four Extended parts and the only tall through-hole component.

**Core schematic (commit f956de6).** Removed U16 THVD1450, U17 ISO7762, U18 TRACO TEA1-0505HI,
D6 SM712, R71, C134-C139 and the nets `5V_ISO`, `DMX_GND`, `DMX_A/B`, `DMX_*_ISO`. J10 `J_DMX`
is a 2x4 logic-level header: 1 GND, 2 GND, 3 `DMX_TX`, 4 `DMX_RX`, 5 `DMX_DE`, 6 `3.3V_DIG`,
7 `5V`, 8 GND. The three signals are ESP32-S3 pins (U13 5, 39, 25), so the port also serves any
other UART or RS-485 device.

**Core board (commit cfb9e5e).** J10 sits on row B at Y 49 beside the S3. Slot 3 is MIDI at the
top, U21 at (83.0, 27.5) beside J_SAI2 and the IDC777 at the bottom, with the area between
MIDI and U21 free. No isolation barrier and no plane split remain. New rule area `IDC777_ANT`
(all four copper layers, X 77.1-85.2, Y 55.4 to the edge) keeps copper out from under the BT
antenna; the stock footprint only had a silkscreen keep-out. 123 passives re-placed (median
2.4 mm, mean 3.3 mm, 7 beyond 10 mm). DRC with parity: only the S3 antenna courtyard (vs H3 and
J10), its 0.2 mm via holes and the outline footprint.

**The module.** ISO7731DWR (C524804) + B0505S-1WR3 (C7465178) + THVD1450DR + SM712, Neutrik
NC5FAH 5-pin XLR plus a 3-pin header, 63.9 x 31 mm, 2 layers, routed, about 3.10 USD of
semiconductors. Its J1 mates 1:1 with J10. Details in that project's README.

**Still open before routing the core:** the Ethernet decision (J12 nets are single-node until
the U1 symbol and footprint swap, or J12 is dropped), the In2 power-plane split (5V, 3.3V,
3.3V_DIG, V_BAT) and a differential-pair net class for the two USB pairs.

## 12. Edit order and verification

1. Commit the July 27 work as-is (BT/ASRC block + libraries) so it is not only in the
   working tree.
2. Deletions (§1, §9) → ERC drops from 466 to something readable.
3. Wireless block rewire (§2, §3, §4) → netlist check: each of `4_BCLK2`, `3_LRCLK2`,
   `5_IN2`, `2_OUT2` has exactly Teensy + SRC (+ series R); `S3_I2S_*` each has S3 + U21;
   `BT_RST`/`BT_SYS_CTRL`/`BT_S3_SEL`/`AUX_SEL` each has SRC GPO + load; `SRC_RST` on
   ESP32_EN.
4. Power (§7) → netlist check: `3.3V` sourced by U15 only; Teensy pin 46 isolated; VUSB on
   VIN2; no node on the old USB-C nets.
5. Slave-clock jumpers (§8), EN pull-up (§6), aux mux (§2.1).
6. Header footprint + pin map (§5); assign the five missing footprints; update PCB from
   schematic.
7. Fix Konnect layer count; choose outline; place antenna keepout; then layout.

All schematic edits go through Konnect and are verified with
`kicad-cli sch export netlist`, per the project's write-hazard rules.

## 13. Header pin map (spin 2, function-scoped headers, as wired 2026-10-02)

Generated from the kicad-cli netlist of schematic commit f956de6 (stage 10z). The four 2x22 edge blocks (176 pins) are gone. J9 (S3 USB) was folded into J8 and J1 trimmed to 2x5 on 2026-09-28 (stage 10m); J6 (S/PDIF) was removed and J7 became the 2x6 J_SAI2 port on 2026-10-01 (stage 10y); J10 became a 2x4 logic-level DMX port on 2026-10-02 (stage 10z), so the ten headers fit four vertical rows (section 11.4). Each function has its own stock
`Connector_Generic` header placed next to the chip that serves it, so the PCB traces are short and
the backplane only needs to mate the headers it uses. Ground rule applied to every header: at least
one GND per three signals, a GND row (both rows) on both sides of every clock and every differential
pair, and GND at both ends of every header.

| Ref | Name | Size | Placed next to | Pins | GND |
|---|---|---|---|---|---|
| J1 | `J_PWR` | 2x5 | TVS D7 / P-FET Q1 / buck U24 / LT3045 U15 | 10 | 3 |
| J2 | `J_TDM` | 2x14 | TDM buffers U5/U6/U7/U11 under the Teensy | 28 | 13 |
| J3 | `J_T` | 2x12 | Teensy 4.1 | 24 | 13 |
| J4 | `J_USB` | 2x5 | Teensy USB host pads and device pads 66/67 | 10 | 5 |
| J5 | `J_MIDI` | 2x4 | MIDI opto/buffer U4/U9 | 8 | 2 |
| J7 | `J_SAI2` | 2x6 | U21 244 buffer / U20 2G125 gate | 12 | 5 |
| J8 | `J_S3` | 2x17 | ESP32-S3 U13 | 34 | 11 |
| J10 | `J_DMX` | 2x4 | ESP32-S3 U13 (row B, bottom) | 8 | 3 |
| J11 | `J_LED` | 1x5 | SK6812 level shifter IC1 | 5 | 2 |
| J12 | `J_ETH` | 2x4 | Teensy 4.1 Ethernet pads | 8 | 3 |
| | | | **total** | **147** | **60** |

Footprints: `Connector_PinHeader_2.54mm:PinHeader_<size>_P2.54mm_Vertical`. Odd pins are one row, even
pins the other (KiCad Odd_Even numbering). `DMX_GND` is the isolated ground, not core GND.
Net names are the current schematic names; the raw-GPIO rename (T_nn / S3_IOnn) is a later cosmetic
pass and does not change connectivity.

### J1 - `J_PWR` 2x5, next to TVS D7 / P-FET Q1 / buck U24 / LT3045 U15

Power entry and rails. 5V_IN (mandatory external 5 V since stage 10x) feeds the TVS + reverse-polarity P-FET; 5V, 3.3V (LT3045, analog), 3.3V_DIG (buck) and V_BAT are outputs.

| odd | net | even | net |
|---|---|---|---|
| 1 | `5V_IN` | 2 | `5V_IN` |
| 3 | `GND` | 4 | `GND` |
| 5 | `5V` | 6 | `5V` |
| 7 | `3.3V` | 8 | `3.3V_DIG` |
| 9 | `V_BAT` | 10 | `GND` |

### J2 - `J_TDM` 2x14, next to TDM buffers U5/U6/U7/U11 under the Teensy

Both TDM buses plus the two I2C buses for codec control. GND on both flanks of every clock; TDM1 in the odd row, TDM2 in the even row. Pin 27 (since stage 10v) is the buffered OUT1B line whose direction is set by 0 ohm jumpers.

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `MCLK1+TDM1` | 4 | `MCLK1+TDM2` |
| 5 | `GND` | 6 | `GND` |
| 7 | `BCLK1+TDM1` | 8 | `BCLK1+TDM2` |
| 9 | `GND` | 10 | `GND` |
| 11 | `LRCK1+TDM1` | 12 | `LRCK1+TDM2` |
| 13 | `GND` | 14 | `GND` |
| 15 | `7_OUT1A+` | 16 | `6_OUT1D+` |
| 17 | `8_IN1` | 18 | `9_OUT1C_INPUT` |
| 19 | `GND` | 20 | `GND` |
| 21 | `T18_SDA0` | 22 | `T17_SDA1` |
| 23 | `T19_SCL0` | 24 | `T16_SCL1` |
| 25 | `GND` | 26 | `GND` |
| 27 | `32_OUT1B` | 28 | `GND` |

### J3 - `J_T` 2x12, next to Teensy 4.1

Teensy S/PDIF (the only S/PDIF path since stage 10y), CAN3, Serial8, PROGRAM/ON_OFF, buttons, OUT1B. Pins 17/18/19/22 are GND since stage 10y (T22/T26/T27/MCLK2 now serve the SAI2 block).

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `T14_SPDIF_OUT` | 4 | `T15_SPDIF_IN` |
| 5 | `GND` | 6 | `GND` |
| 7 | `T30_CRX3` | 8 | `T31_CTX3` |
| 9 | `T35_TX8` | 10 | `T34_RX8` |
| 11 | `T_PROGRAM` | 12 | `T_ON_OFF` |
| 13 | `T24` | 14 | `T25` |
| 15 | `GND` | 16 | `GND` |
| 17 | `GND` | 18 | `GND` |
| 19 | `GND` | 20 | `T32_OUT1B` |
| 21 | `GND` | 22 | `GND` |
| 23 | `GND` | 24 | `GND` |

### J4 - `J_USB` 2x5, next to Teensy USB host pads and device pads 66/67

Odd row: Teensy host pair, GND flanked, HOST_5V is the host-port VBUS. Even row: Teensy device pair (pads 66/67) with GND on 2/8/10; no VBUS pin, the Teensy VUSB is NC. Neither port is isolated (stage 10y).

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `HOST_D1-` | 4 | `T_USB_DN` |
| 5 | `HOST_D1+` | 6 | `T_USB_DP` |
| 7 | `GND` | 8 | `GND` |
| 9 | `HOST_5V` | 10 | `GND` |

### J5 - `J_MIDI` 2x4, next to MIDI opto/buffer U4/U9

Jack-level MIDI IN / OUT / THRU (DIN pins 4 and 5 across the rows).

| odd | net | even | net |
|---|---|---|---|
| 1 | `MIDI_IN_4` | 2 | `MIDI_IN_5` |
| 3 | `MIDI_OUT_4` | 4 | `MIDI_OUT_5` |
| 5 | `MIDI_THRU_4` | 6 | `MIDI_THRU_5` |
| 7 | `GND` | 8 | `GND` |

### J7 - `J_SAI2` 2x6, next to U21 244 buffer / U20 2G125 gate

Teensy SAI2 as an expansion I2S port (stage 10y, section 11.8). Core is header master (MODE 10) or header slave (MODE 01) by firmware; MCLK is always sourced by the core. DIN is into the core, DOUT out of it. GND flanks every clock.

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `AUX_I2S_MCLK` | 4 | `GND` |
| 5 | `AUX_I2S_BCK` | 6 | `GND` |
| 7 | `AUX_I2S_LRCK` | 8 | `GND` |
| 9 | `AUX_I2S_DIN` | 10 | `AUX_I2S_DOUT` |
| 11 | `3.3V_DIG` | 12 | `5V` |

### J8 - `J_S3` 2x17, next to ESP32-S3 U13

S3 UART0 (also Teensy Serial7), EN/IO0, I2C, raw GPIO, and the S3 native USB (OTG) pair with 5V (pin 34 = core 5 V rail: wire to a jack VBUS only for host use). IO40/IO41 left the header in stage 10y (BT_RST / BT_SYS_CTRL); pins 25/26 are GND. IO35-37 are consumed by the octal PSRAM.

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `/S3_IO43_TXD0` | 4 | `/S3_IO44_RXD0` |
| 5 | `S3_EN` | 6 | `S3_IO0_BOOT` |
| 7 | `GND` | 8 | `GND` |
| 9 | `S3_IO21_SDA` | 10 | `S3_IO47_SCL` |
| 11 | `S3_IO18` | 12 | `S3_IO9` |
| 13 | `S3_IO10` | 14 | `S3_IO11` |
| 15 | `GND` | 16 | `GND` |
| 17 | `S3_IO2` | 18 | `S3_IO42` |
| 19 | `S3_IO8` | 20 | `T40` |
| 21 | `T39` | 22 | `T38` |
| 23 | `S3_IO15` | 24 | `S3_IO38` |
| 25 | `GND` | 26 | `GND` |
| 27 | `S3_IO45` | 28 | `S3_IO39` |
| 29 | `GND` | 30 | `GND` |
| 31 | `S3_USB_DN` | 32 | `S3_USB_DP` |
| 33 | `GND` | 34 | `5V` |

### J10 - `J_DMX` 2x4, next to ESP32-S3 U13 (row B, bottom)

Logic-level DMX / RS-485 port since stage 10z: S3 UART TX, RX and driver enable with 3.3V_DIG, 5V and GND. Not isolated. Mates 1:1 with J1 of t-dsp_dmx_module, which carries the isolator, isolated supply, transceiver and XLR.

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `DMX_TX` | 4 | `DMX_RX` |
| 5 | `DMX_DE` | 6 | `3.3V_DIG` |
| 7 | `5V` | 8 | `GND` |

### J11 - `J_LED` 1x5, next to SK6812 level shifter IC1

5 V SK6812 data outs from the Teensy and the S3, GND between, 5V for the strip.

| pin | net |
|---|---|
| 1 | `GND` |
| 2 | `TEENSY_LED_OUT` |
| 3 | `GND` |
| 4 | `ESP32_LED_OUT` |
| 5 | `5V` |

### J12 - `J_ETH` 2x4, next to Teensy 4.1 Ethernet pads

Teensy 4.1 10/100 PHY pairs and link LED from pads 60-65 (needs the U1 symbol swap, see 14.1). Magnetics and RJ45 on the backplane.

| odd | net | even | net |
|---|---|---|---|
| 1 | `ETH_R+` | 2 | `ETH_R-` |
| 3 | `GND` | 4 | `GND` |
| 5 | `ETH_T+` | 6 | `ETH_T-` |
| 7 | `ETH_LED` | 8 | `GND` |

## 14. Status after the 2026-09-28 edit session

Done (all verified with `kicad-cli sch export netlist` diffs, zero unexpected net changes):

- §1 deletions: 129 backplane parts, old DevKitC, pseudo net-map symbols, dead branches.
- §2 wireless block: Port B on Teensy I2S2, reverse path, GPO control lines, S/PDIF labels,
  U23 aux mux cascade, R72 SYS_CTRL pulldown, R73 EN pull-up. **S3 IO46 drives the SK6812
  chain (`ESP32_LED`)** because the old DevKitC GPIO5 went away; IO46 is therefore not on
  the header (J4.35 is RESERVED).
- §7 power: AP63203 buck (U24 + L1 + C153 + C154) replaces the TLV76733; LT3045 owns the
  3.3V rail; Teensy 3V3 pins NC; VUSB → TPS2116 VIN2 (`5V_VUSB`); VIN1 = `5V_IN` from J1.
  PWR_FLAGs on 3.3V_DIG, V_BAT, 5V_IN, the LT3045 input, U9 VCC, Y1 VDD, U4 pins 5/6.
- §5/§13 headers: J1–J4 wired, 176 pins checked against the pin map.
- Cleanups: SPI-link and I2S2 labels made global; C129 GND restored; no-connect flags on
  every intentionally unused pin; C129/C48 footprints fixed; no symbol overlaps.

Residual ERC (kicad-cli, all severities):

| Count | Item | Meaning |
|---|---|---|
| 1 | `power_pin_not_driven` D3 pin 4 (GND) | ERC quirk; the netlist shows D3.4 on GND with 47 GND power symbols and the Teensy GND pins. A PWR_FLAG on GND collides with the Teensy symbol's GND pins being typed *Output* (`pin_to_pin`). Leave, or retype those pins in the Teensy symbol. |
| 50 + 56 | `lib_symbol_issues` / `lib_symbol_mismatch` | cached symbols differ from the libraries; fix in eeschema with *Tools → Update Symbols from Library*. Not touchable through Konnect. |
| 30 | `unconnected_wire_endpoint` | pre-existing wire overshoots past labels (cosmetic). |

Not done / next:

1. **External MCLK (user wants it — §10 #3 decided):** the hardware is already right. Y1
   (24.576 MHz = 512·fs) reaches MCLK1 through R61 (33 Ω); Teensy pin 23 reaches MCLK1
   through R7 (49.9 Ω). Population rule, now printed on the schematic next to Y1:
   - **External-MCLK build:** R59 populated (Y1 enabled), R60 DNP. Firmware must make the
     Teensy's SAI1 MCLK pin an input: clear `SAI1_MCLK_DIR` in `IOMUXC_GPR_GPR1`, then
     run SAI1 with MSEL = MCLK1 so BCLK/LRCK are divided from the external clock. The
     Teensy audio library sets MCLK as output; this is a small patch in the SAI init
     (`output_i2s.cpp` / the F32 TDM variant in `t-dsp_software/lib`). Not written yet.
   - **Default build until then:** R60 populated (Y1 in standby, output high-Z), R59 DNP;
     Teensy drives MCLK1 at 256·fs. Never populate both: two drivers through 33 Ω + 49.9 Ω.
   - The SRC accepts either clock; the TDM codecs on the backplane must accept the chosen
     MCLK ratio (512·fs with Y1, 256·fs with the Teensy).
2. Raw-GPIO renames (legacy names SWITCH/OUTPUTA/CS/… → S3_IOnn, T_nn) — cosmetic, later.
3. ~~Header footprints for the PCB, board outline holes~~ **done 2026-09-28 (section 11.3)**: PCB synced
   from the schematic and fully placed; next are antenna keep-out zones, GND pours and routing.
4. Symbol library sync (see residual ERC).

### 14.1 Stage 9 (2026-09-28, later): function-scoped headers, PSRAM, Ethernet

- J1-J4 (4 x 2x22) deleted with all 339 stubs/labels/NC flags; **J1-J12** placed next to their
  blocks and wired (section 13). Netlist diff against stage 8 = pin moves only; no net lost except
  `S3_IO3`, `S3_IO36`, `S3_IO37` (PSRAM pins, now NC) and the dead `S3_IO46` header pin.
- U13 -> **ESP32-S3-WROOM-1-N16R8** (C2913202); `BT_UART_RX` relabelled from IO35 to IO3;
  NC flags on IO35/IO36/IO37.
- **J12 `J_ETH` placed and labelled** (`ETH_R+ ETH_R- ETH_T+ ETH_T- ETH_LED`), but the six nets
  are single-node until the manual step below is done.
- ERC after stage 9: 0 dangling wires/labels, 0 undriven pins; only the 40 pre-existing
  cosmetic `unconnected_wire_endpoint` overshoots and the library-sync warnings.

**Manual eeschema steps that Konnect cannot do** (its library search is the KiCad install only,
so `project_sch` symbols cannot be placed or swapped):

1. U1: *Change Symbol* -> `project_sch:Teensy4.1` (full symbol, adds pins 60-65). Then label
   60 = `ETH_R+`, 61 = `ETH_LED`, 62 = `ETH_T-`, 63 = `ETH_T+`, 64 = `GND`, 65 = `ETH_R-`
   (names from the symbol). Konnect can do the six labels once the symbol is in the cache.
2. Footprint: copy pads 60-65 from `Teensy41.kicad_mod` into `Teensy41_mod3` (or a `_mod4`).
3. *Update Symbols from Library* for the ISO7762 / TLV757 / ASE cached symbols.

## 15. Sourcing for JLCPCB assembly (2026-09-28)

Every symbol now carries an `LCSC` field (and `LCSC_MPN` where the manufacturer part
number matters). Numbers were confirmed on lcsc.com / jlcpcb.com part pages, not guessed.

Substitutions made so the board can be assembled from stock:

| Ref | Was | Now | LCSC | Why |
|---|---|---|---|---|
| U17 | ISO7761DW | **ISO7762DW** | C2859648 | ISO7761 not stocked. 7762 = 4 fwd / 2 rev; DMX uses A, B fwd and F rev. Pin 6 (was INE→GND) is now NC; pin 11 becomes INE (unused input, left NC — tie to DMX_GND once the symbol is swapped in eeschema). |
| U20 | TLV76718DRVR | ~~TLV75718PDRVR~~ **SN74LVC2G125DCTR** | C206035 | Stage 10y: the 1V8 LDO left with the SRC4382; U20 is now the J_SAI2 DIN gate (project `SOP65P400X130-8N`). |
| U19 | SRC4382IPFBR (consigned) | **SN74LVC257APWR** | C205921 | Stage 10y: SAI2 source mux, TSSOP-16, stock. |
| U21, U23 | 74LVC157APW | **SN74LVC244APWR** (U21), **SN74LVC138APWR** (U23) | C7668, C485077 | Stage 10y: J_SAI2 header buffer and the mode interlock, TSSOP-20 / TSSOP-16, stock. |
| R86-R88 | (new) | 10 k 0402 | C25744 | SAI2_MODE0/1 pull-ups, SAI2_SEL pull-down. |
| R89-R101 | (new) | 33 R 0402 | C25105 | series resistors on every SAI2 / AUX_I2S driver. |
| Y1 | ASDMB-24.576MHZ | **ASE-24.576MHZ-LC-T** | C6159263 | ASDMB not stocked; ASE has the same 4-pin function (1 = standby, 2 GND, 3 OUT, 4 VDD) in 3.2×2.5 mm; footprint updated. |
| U13 | WROOM-1U (u.FL) | **ESP32-S3-WROOM-1-N16R8** | C2913202 | PCB antenna; 8 MB octal PSRAM for Spotify buffering (was N16R2 C2913205 for one commit). |
| L1 | NR4018T3R3M | **NRS4018T3R3MDGJ** | C92960 | same 4×4 mm 3.3 µH 2 A family, stocked. |
| U24 | (new) | AP63203WU-7 | C780769 | buck, JLC stock |
| D3, D4 | SK6812 3.2x2.8 reverse-mount (MINI-E) | **SK6812-EC20** 2.0x2.0 top-emitting | C2909058 | Decided 2026-09-28: the LEDs sit mid-board, so a top emitter is visible without board windows. Footprint `project_fp:LED_SK6812-EC20_2.0x2.0mm` built from datasheet SPC/SK68XX-EC20 rev 04 (pads 0.8x0.7 on 1.3x1.2); pad numbers follow the project symbol (1 DIN, 2 VDD, 3 DOUT, 4 GND), which differs from the datasheet numbering (1 VDD, 2 DOUT, 3 GND, 4 DIN). SK6805-EC15 (C2890035) is the drop-in smaller/dimmer alternative with `LED_SK6812_EC15_1.5x1.5mm`. |

Consigned (not at LCSC; supplied to the assembler or hand-placed): **U22 IDC777-1**,
**U1 Teensy 4.1**. Their `LCSC` field says `CONSIGN`. (The SRC4382 left the design in stage 10y.)

No LCSC number (no part or hand assembly): H1–H4 holes, T108–T111 panel tabs, PU_EN1
solder jumper, **J1-J12 pin headers** (stock 2.54 mm THT headers cut from 2x40 / 1x40 strips such
as C2333 / C50981; hand-soldered, so no LCSC field).

Passive numbers used: 0805 100 nF C49678, 10 µF C15850, 22 µF C45783, 4.7 µF C1779, 1 µF
C6119929, 0402 100 nF C1525; 0805 resistors 10k C17414, 100k C17407, 33k C17633, 300 Ω
C17617, 33 Ω C17634, 120 Ω C17437 (R71, DNP), 49.9 Ω C17720, 2.21k C17520, 0 Ω C17477.

## 16. Functional blocks on the sheet (for re-organising in eeschema)

Each block now has a title text at its top-left. Konnect can move symbols but not their
wires and labels as a group, so the actual tidy-up is a block-select + move in eeschema.
Coordinates are sheet mm (x, y), symbol centres:

| Block | x range | y range |
|---|---|---|
| TEENSY 4.1 + series terminations, SPI link pull-ups | 105 … 472 | 211 … 318 |
| SK6812 status LEDs + 5V level shifter | 457 … 794 | 430 … 518 |
| TDM bus buffers (I2S1 -> TDM1/TDM2 headers) | 492 … 639 | 681 … 883 |
| MIDI in/out/thru (opto + buffer) | 777 … 880 | 775 … 838 |
| 3.3V analog rail (LT3045) + 24.576 MHz MCLK oscillator | 596 … 1098 | 1002 … 1123 |
| 5V input mux (TPS2116): header 5V_IN / Teensy VUSB | 1063 … 1146 | 276 … 406 |
| 3.3V rail decoupling | 296 … 678 | 666 … 739 |
| ESP32-S3 module + I2C pull-ups + EN pull-up | -298 … -193 | 597 … 655 |
| 3.3V_DIG buck (AP63203) + 5V decoupling | -310 … -211 | 785 … 833 |
| Isolated DMX / RS-485 | -279 … -124 | 879 … 1010 |
| Bluetooth sink (IDC777) + SYS_CTRL pulldown | 1306 … 1360 | 389 … 615 |
| SRC4382 ASRC + 1.8V LDO | 1458 … 1584 | 453 … 704 |
| I2S source mux (BT / S3 / aux) | 1628 … 1669 | 574 … 660 |
| Edge headers J1-J4 | 1346 … 1651 | 900 … 900 |
| Mounting holes / panel tabs | 1032 … 1180 | 635 … 892 |

Suggested target arrangement (left→right, top→bottom): power (5 V mux, buck, LT3045, MCLK)
· Teensy + TDM buffers + MIDI + LEDs · S3 + DMX · BT + SRC + mux · headers. Keep every
block's stub-and-label style; nothing crosses blocks except by global label.

## 17. DNP / excluded-from-board audit (2026-09-28, after the user spotted X marks)

- **C10, C11** (0.1 µF on the old Teensy 3V3 rail) were `dnp` + `on_board no` → deleted.
- **H1–H4** mounting-hole symbols were `on_board no` → deleted; holes come from the outline
  footprint / PCB (§11).
- **R19, R53 (SPI-link pull-ups) and C4, C5 (EN / IO0 caps) were `on_board no`** — they would
  have been missing from the PCB. Konnect cannot clear that flag, so they were deleted and
  re-placed at the same coordinates (netlist unchanged), now on-board with LCSC numbers.
- Intentionally DNP and kept: **U1 Teensy** (consigned, hand-placed — hence the X on the
  schematic), T108–T111 panel tabs, PU_EN1 solder jumper.
- 35 stale text notes from the backplane era (USB/optical/Ethernet/mic/switch labels,
  8-layer stackup note, old titles) removed.
