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
| LED IC1/C1/C33/C22/C23 + J11 | top-middle: J11 at Y 16-19.3, IC1 and the four caps in a row at Y 21.5, X 41-55 | D3/D4 at X 54-64, Y 38.1-40.3; they are reverse-mount: add a 3 mm hole under each (not done) |
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

## 13. Header pin map (spin 2, function-scoped headers, as wired 2026-09-28)

The four 2x22 edge blocks (176 pins) are gone. Each function now has its own stock
`Connector_Generic` header placed next to the chip that serves it, so the PCB traces are short and
the backplane only needs to mate the headers it uses. Ground rule applied to every header: at least
one GND per three signals, a GND row (both rows) on both sides of every clock and every differential
pair, and GND at both ends of every header.

| Ref | Name | Size | Placed next to | Pins | GND |
|---|---|---|---|---|---|
| J1 | `J_PWR` | 2x6 | TPS2116 / buck / LT3045 | 12 | 5 |
| J2 | `J_TDM` | 2x13 | TDM bus buffers U5-U8/U11 | 26 | 12 |
| J3 | `J_T` | 2x12 | Teensy 4.1 | 24 | 9 |
| J4 | `J_USBH` | 1x5 | Teensy USB host pads | 5 | 2 |
| J5 | `J_MIDI` | 2x4 | MIDI opto/buffer U4/U9 | 8 | 2 |
| J6 | `J_SPDIF` | 2x11 | SRC4382 U19 | 22 | 12 |
| J7 | `J_AUX` | 2x5 | U23 aux mux | 10 | 6 |
| J8 | `J_S3` | 2x15 | ESP32-S3 U13 | 30 | 8 |
| J9 | `J_USB3` | 1x5 | ESP32-S3 U13 | 5 | 2 |
| J10 | `J_DMX` | 1x5 | ISO7762 / RS-485 U16-U18 | 5 | 0 |
| J11 | `J_LED` | 1x5 | SK6812 level shifter IC1 | 5 | 2 |
| J12 | `J_ETH` | 2x4 | Teensy 4.1 Ethernet pads | 8 | 3 |
| | | | **total** | **160** | **66** |

Footprints: `Connector_PinHeader_2.54mm:PinHeader_<size>_P2.54mm_Vertical`. Odd pins are one row, even
pins the other (KiCad Odd_Even numbering). `DMX_GND` is the isolated ground, not core GND.
Net names are the current schematic names; the raw-GPIO rename (T_nn / S3_IOnn) is a later cosmetic
pass and does not change connectivity.

### J1 - `J_PWR` 2x6, next to TPS2116 / buck / LT3045

Power entry and rails. 5V_IN feeds TPS2116 VIN1; 5V, 3.3V (LT3045, analog), 3.3V_DIG (buck) and V_BAT are outputs.

| odd | net | even | net |
|---|---|---|---|
| 1 | `5V_IN` | 2 | `5V_IN` |
| 3 | `GND` | 4 | `GND` |
| 5 | `5V` | 6 | `5V` |
| 7 | `GND` | 8 | `GND` |
| 9 | `3.3V` | 10 | `3.3V_DIG` |
| 11 | `V_BAT` | 12 | `GND` |

### J2 - `J_TDM` 2x13, next to TDM bus buffers U5-U8/U11

Both TDM buses plus the two I2C buses for codec control. GND on both flanks of every clock; TDM1 in the odd row, TDM2 in the even row.

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

### J3 - `J_T` 2x12, next to Teensy 4.1

Teensy S/PDIF, CAN3, Serial8, PROGRAM/ON_OFF, buttons, spare GPIO, MCLK2. GND either side of MCLK2.

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
| 17 | `T22` | 18 | `T26` |
| 19 | `T27` | 20 | `T32_OUT1B` |
| 21 | `GND` | 22 | `T33_MCLK2` |
| 23 | `GND` | 24 | `GND` |

### J4 - `J_USBH` 1x5, next to Teensy USB host pads

Teensy host pair, GND flanked; HOST_5V is the host-port VBUS.

| pin | net |
|---|---|
| 1 | `GND` |
| 2 | `HOST_D1-` |
| 3 | `HOST_D1+` |
| 4 | `GND` |
| 5 | `HOST_5V` |

### J5 - `J_MIDI` 2x4, next to MIDI opto/buffer U4/U9

Jack-level MIDI IN / OUT / THRU (DIN pins 4 and 5 across the rows).

| odd | net | even | net |
|---|---|---|---|
| 1 | `MIDI_IN_4` | 2 | `MIDI_IN_5` |
| 3 | `MIDI_OUT_4` | 4 | `MIDI_OUT_5` |
| 5 | `MIDI_THRU_4` | 6 | `MIDI_THRU_5` |
| 7 | `GND` | 8 | `GND` |

### J6 - `J_SPDIF` 2x11, next to SRC4382 U19

Four DIR inputs and the DIT output, raw; pairs across the rows with a GND row between every pair. Termination networks live on the backplane.

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `SPDIF_RX1P` | 4 | `SPDIF_RX1N` |
| 5 | `GND` | 6 | `GND` |
| 7 | `SPDIF_RX2P` | 8 | `SPDIF_RX2N` |
| 9 | `GND` | 10 | `GND` |
| 11 | `SPDIF_RX3P` | 12 | `SPDIF_RX3N` |
| 13 | `GND` | 14 | `GND` |
| 15 | `SPDIF_RX4P` | 16 | `SPDIF_RX4N` |
| 17 | `GND` | 18 | `GND` |
| 19 | `SPDIF_TXP` | 20 | `SPDIF_TXN` |
| 21 | `GND` | 22 | `GND` |

### J7 - `J_AUX` 2x5, next to U23 aux mux

Aux I2S source into the mux (BCK, LRCK, DIN) and the S3-side out; clocks never adjacent.

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `AUX_I2S_BCK` | 4 | `GND` |
| 5 | `GND` | 6 | `AUX_I2S_LRCK` |
| 7 | `AUX_I2S_DIN` | 8 | `AUX_I2S_OUT` |
| 9 | `GND` | 10 | `GND` |

### J8 - `J_S3` 2x15, next to ESP32-S3 U13

S3 UART0 (also Teensy Serial7), EN/IO0, I2C, raw GPIO. IO35-37 are consumed by the octal PSRAM and are not on the header. Pins 20/21/22 (`T40`, `T39`, `T38`) are **Teensy** pins 40/39/38 (A16/A15/A14); pins 23/24/28 (`S3_IO15`, `S3_IO38`, `S3_IO39`) are the S3 pins that used to share those nets. Split on 2026-09-28 so either MCU can own a control; a backplane may wire a Teensy pin and an S3 pin together if it wants the old shared behaviour.

| odd | net | even | net |
|---|---|---|---|
| 1 | `GND` | 2 | `GND` |
| 3 | `S3_IO43_TXD0` | 4 | `S3_IO44_RXD0` |
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
| 25 | `S3_IO40` | 26 | `S3_IO41` |
| 27 | `S3_IO45` | 28 | `S3_IO39` |
| 29 | `GND` | 30 | `GND` |

### J9 - `J_USB3` 1x5, next to ESP32-S3 U13

S3 native USB (OTG) pair, GND flanked; 5V for a host-side VBUS.

| pin | net |
|---|---|
| 1 | `GND` |
| 2 | `S3_USB_DN` |
| 3 | `S3_USB_DP` |
| 4 | `GND` |
| 5 | `5V` |

### J10 - `J_DMX` 1x5, next to ISO7762 / RS-485 U16-U18

Isolated side only: DMX_GND and 5V_ISO, never core GND.

| pin | net |
|---|---|
| 1 | `DMX_GND` |
| 2 | `DMX_A` |
| 3 | `DMX_B` |
| 4 | `DMX_GND` |
| 5 | `5V_ISO` |

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

### 13.1 Backplane controls and the GPIO budget (2026-09-28)

Direct pins reachable from a backplane: Teensy 14, 15, 22, 24, 25, 26, 27, 30, 34, 35, 38, 39, 40
(ten of them analog: A0, A1, A8, A10, A11, A12, A13, A14, A15, A16) plus the two I2C pairs; S3
IO2, IO8, IO9, IO10, IO11, IO18, IO40, IO41, IO42, IO45 plus EN, IO0 and its I2C pair. A volume
knob is a 10k pot from 3.3V (J1) to GND with the wiper on any Teensy analog pin (T14, T15, T22, T24, T25, T26, T27, T38, T39, T40) and 100 nF to
GND; a switch is any pin to GND with `INPUT_PULLUP`. Beyond ~20 controls use the panel-bus
pattern: 74HC4067 analog muxes on the analog pins (16 pots each), MCP23017 or TCA8418 on
T17_SDA1/T16_SCL1, SK6812 LEDs on J11. Links to a Pi: Serial8 UART (J3.9/10), I2C Wire/Wire1 (J2),
S3 I2C (J8), Teensy USB host (J4), S3 USB OTG (J9), the Teensy's own USB jack (Pi as host),
MIDI (J5), TDM2 (J2) for audio, Ethernet (J12) once the U1 symbol swap is done.

### 13.2 Net-naming rule (2026-09-28)

Functions keep function names (TDM buses, SPDIF, USB, MIDI, DMX, LED outs, power, ETH). Raw GPIO is
named by MCU pin so a user with the MCU pinout in hand cannot mis-wire it: `T<n>` = Teensy 4.1 pin
n, `S3_IO<n>` = ESP32-S3 GPIO n. A pin with an optional core-side function carries it as a suffix:
`T14_SPDIF_OUT`, `T35_TX8`, `T18_SDA0`, `T33_MCLK2`, `S3_IO43_TXD0`, `S3_IO47_SCL`, `S3_IO0_BOOT`.
Special Teensy pads are `T_PROGRAM`, `T_ON_OFF`, `V_BAT`. Renamed on 2026-09-28 (100 labels,
netlist topology unchanged). The old DevKitC-era names were wrong in three places: `ESP32_IO1/IO3`
were GPIO43/44, `ESP32_IO22_SCL` was IO47, `GPIO34/35` were IO40/41; the internal SPI CS is
IO17, not IO33. The t-dsp_software pin tables must be updated to match.

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
| U20 | TLV76718DRVR | **TLV75718PDRVR** | C2861386 | TLV767 DRV not stocked; TLV757P DRV pinout is identical (pin 2/5 NC instead of SNS/GND — both harmless as wired). |
| Y1 | ASDMB-24.576MHZ | **ASE-24.576MHZ-LC-T** | C6159263 | ASDMB not stocked; ASE has the same 4-pin function (1 = standby, 2 GND, 3 OUT, 4 VDD) in 3.2×2.5 mm; footprint updated. |
| U13 | WROOM-1U (u.FL) | **ESP32-S3-WROOM-1-N16R8** | C2913202 | PCB antenna; 8 MB octal PSRAM for Spotify buffering (was N16R2 C2913205 for one commit). |
| L1 | NR4018T3R3M | **NRS4018T3R3MDGJ** | C92960 | same 4×4 mm 3.3 µH 2 A family, stocked. |
| U24 | (new) | AP63203WU-7 | C780769 | buck, JLC stock |

Consigned (not at LCSC; supplied to the assembler or hand-placed): **U19 SRC4382IPFBR**,
**U22 IDC777-1**, **U1 Teensy 4.1**. Their `LCSC` field says `CONSIGN`.

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
