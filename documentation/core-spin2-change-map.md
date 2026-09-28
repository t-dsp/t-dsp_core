# T-DSP Core spin 2 — change map

Status: **decisions recorded 2026-09-28, schematic edits not yet started.** This document
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
backplane may carry the bulk of the system current (core ≈ 400 mA of a ≈ 2 A system).

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
| MIDI jacks (opto, buffer, and jack-side parts stay only if the MIDI *logic* stays; see §4) | MIDI_IN_1, MIDI_OUT_1, MIDI_IO1, MIDI_OT1, MIDI_OT2 |
| Rotary encoder, tactile switches | SW1–SW10 (SW3/4/9/10 are the EN/IO0 buttons, see §6) |
| JST power inputs and reverse-polarity FETs | J10, J11, Q1, Q2 |
| Optical S/PDIF TX/RX | TT1, TT2, TR1, TR2, SPDIF1, SPDIF_IO1, SPDIF_IO2 |
| MEMS microphones | MIC1, MIC2 |
| SK6812 LEDs and LED header | D3, D4, J12 |
| TFT / GPI / encoder / I2C / programming header duplicates | ILI1, ILI2, GPI1, GPI2, ESP_R_ENC1, ESP_R_ENC2, ESP_I1, ESP_I2, ESP_PROG1, ESP_PROG2, T_PROG1 |
| Old ESP32 DevKitC and TAC5212 module symbol | U2, board_outline1 |
| Battery header/cell | BAT1, BAT2 |

Delete them rather than leaving them excluded-from-board: U2 alone is still on 30+ nets
and every one of these adds ERC noise and confuses the header map.

---

## 2. Wireless audio block (BT + S3 → mux → ASRC → Teensy)

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
| IO35 | BT UART2 TX **[open: variant]** — falls back to IO3 on an R8 module |
| IO19, IO20 | **native USB D−/D+ → edge header** (OTG, host or device) |
| RXD0, TXD0 | Teensy Serial7 (programming + FlasherX tunnel) **and** edge header |
| IO0, EN | Teensy 36/37 **and** edge header (already pins 60/56) |
| IO1, IO5, IO48 | DMX (stays) |
| IO8, IO9, IO10, IO11, IO2, IO42, IO18 | TFT SPI + touch (already on header) |
| IO12, IO13, IO14, IO17 | SPI link to Teensy (internal) |
| IO15, IO38, IO39 | encoder (already on header) |
| IO40, IO41 | GPI (already on header) |
| IO21, IO47 | I2C (already on header) |
| IO3, IO45, IO46 | **→ edge header** (strapping pins; document the boot constraints) |
| IO36, IO37 | **→ edge header** if not an R8 module |

**[open]** Module variant. Recommend **ESP32-S3-WROOM-1-N8R2 or -N16R2** (quad PSRAM):
keeps IO35–IO37 usable. An **R8** (octal PSRAM) variant loses them, which pushes the BT
UART TX to IO3 and leaves fewer header GPIOs. Also **[open]**: footprint is currently the
**-1U** (external u.FL antenna); confirm this is intended.

---

## 4. Teensy 4.1 (U1) pin changes

| Pin | Today | New |
|---|---|---|
| 2 OUT2 | → S3 IO4 | → SRC SDINB (reverse path) |
| 3 LRCLK2, 4 BCLK2 | → S3 (via 49.9 Ω) | → SRC LRCKB / BCKB |
| 5 IN2 | ← S3 IO7 | ← SRC SDOUTB |
| 22, 26, 27 | NC | → edge header |
| 32 OUT1B | J15 (1-pin) | → edge header |
| 33 MCLK2 | J19 (1-pin) | → edge header |
| 49 VUSB | NC | → **TPS2116 VIN2** (see §7) |
| 46 3V3 | on the 3.3V rail | **disconnected from the rail, not exposed** (see §7) |
| 55–57 host USB | USBA1 / USB_HOST_1 | → edge header (D+, D−, HOST_5V) |
| 66/67 device USB | USBC1 / U10 | **stays on the Teensy's own jack only; not on header** |
| 28/29 Serial7, 36/37 | S3 UART0 / IO0 / EN | unchanged — programming kit + FlasherX |
| 53 PROGRAM, 54 ON_OFF | header 55/59 | unchanged |

MIDI: MIDI_IN/OUT/THRU are on header pins 1–6 at the *jack* level. **[open]** whether
the opto (U4), buffer (U9) and their passives stay on the core (jack-level MIDI on the
header) or move to the backplane (raw Serial1 RX/TX on the header instead). Default: stay.

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
  SPI, UART, CAN, PWM, analog). MIDI opto (U4) and buffer (U9) move to the backplane; pins
  0/1 go out raw as `T_0`/`T_1`.
- **Exceptions, still named, because the core transforms or owns them:** the two buffered
  TDM buses, `I2C0` (shared with the SRC, pull-ups on the core), SRC S/PDIF RX/TX pairs,
  aux I2S in/out, the three USB pairs, S3 `UART0` + `EN` + `IO0` (programming), Teensy
  `PROGRAM`/`ON_OFF`, `VBAT`, and all power pins.
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

Current budget (peak): Teensy ~100 mA, S3 ≤500 mA (WiFi TX), IDC777 ~100 mA, SRC ~100 mA,
buffers/ISO ~50 mA → ≈0.9 A on 5 V for the core alone. A PC USB port (500 mA) programs
both MCUs but browns out under WiFi bursts; standalone radio use wants a 2 A adapter.

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
| 1 | S3 module variant (N8R2/N16R2 vs R8) and -1U external antenna | N16R2, keep -1U |
| 2 | ~~Board outline~~ **[decided 2026-09-28]: 100 × 60.96 mm**, see §11 | — |
| 3 | MCLK1 driver: Y1 or Teensy pin 23 | Y1 (24.576 MHz = 512·fs), Teensy pin 23 via R7 removed |
| 4 | 12 V: pass-through pin or drop | pass-through pin, no on-core parts |
| 5 | ~~MIDI logic~~ **[decided]: backplane**; pins 0/1 raw on the header | — |
| 6 | LP5907 3.3V_A: keep or delete | delete (mics leave) |
| 7 | `9_OUT1C_INPUT` intent | ask |

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

**[open] Mounting.** Four M3 (or M2.5) holes for standoffs to the backplane, needed for
header retention. Corners are the natural spot; each corner hole costs ~2 header positions
on that edge, already counted above. The 100x60.96 outline footprint currently has **no
holes and no header pads**; both are added when the header footprint is built.

---

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
