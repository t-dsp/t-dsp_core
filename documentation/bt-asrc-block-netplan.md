# Bluetooth sink + ASRC block — net plan

Status: parts placed, **wiring done** (2026-07-27). Every net below is in the schematic and verified
against a `kicad-cli sch export netlist` dump. Decoupling added as C140–C151. Remaining loose ends are
listed under [Open items](#open-items) — they are all *far-end* pin assignments, not block wiring.

## Architecture

```
IDC777-1 (BT) ──I2S──┐
   (U22)             │
                     ├─► 74LVC157A ──► SRC4382 Port A ──► SRC ──► Port B ──► Teensy TDM
ESP32-S3 stream ─────┘     (U21)          (U19)                    │
                                                          MCLK1 ───┘
optical S/PDIF ─────────────────────► DIR (RX1..RX4, future)
```

It is a **mux, not a mixer** — only one source is live at a time, so a single ASRC serves all of
them. The ASRC reconciles whichever source is selected onto the system clock, which is why neither
the Teensy nor the S3 needs to run a software resampler.

## Parts

| Ref | Part | Footprint | Notes |
|-----|------|-----------|-------|
| **U22** | IDC777-1 | `project_fp:IDC777-1` | replaces the EOL IDC747 (was U14) — see [BT module migration](#bt-module-migration-idc747--idc777) |
| U19 | SRC4382IPFBR | `Package_QFP:TQFP-48_7x7mm_P0.5mm` | pin/register compatible with SRC4392 + DIX4192 |
| U20 | TLV76718DRVR | `Package_SON:WSON-6-1EP_2x2mm_P0.65mm_EP1x1.6mm_ThermalVias` | 1.8 V core rail |
| U21 | 74LVC157APW | `Package_SO:TSSOP-16_4.4x5mm_P0.65mm` | quad 2:1 mux, 3 channels used |
| Y1  | 24.576 MHz | (existing) | 512·fs @ 48 kHz — exact match to the SRC reference clock |

*(Historical, IDC747 only:* do not use the SnapEDA IDC747 footprint — legacy KiCad 5 format, all 51
pads omit `F.Mask` so mask would cover every pad. `project_fp:IDC747-1` was the correct one. The part
is now superseded; `project_fp:IDC777-1` is purpose-built and has proper `F.Mask` on all 60 pads.*)*

## BT module migration: IDC747 → IDC777

**The IDC747 is discontinued** — the Qualcomm die inside it went EOL. IOT747's stated replacement is
the **IDC777-1**: same software/AT interface, adds LE Audio (Unicast + Auracast), but **not
pin-to-pin compatible**. It is QCC518x-class silicon (Snapdragon Sound: aptX / aptX HD / aptX
Lossless, AAC), i.e. current top-tier for a BT sink — not a stopgap.

Library files built from datasheet `IDC777-DTS-V003` (pinout p.6-7, mechanical p.4, land pattern p.5):

- `lib_sch/IDC777-1.kicad_sym` — 60 pins, registered in `sym-lib-table` as `IDC777-1`
- `lib_fp/project_fp/IDC777-1.kicad_mod` — 22.2 × 11.8 mm, 60 castellated pads, 0.8 mm pitch,
  land pad 1.0 × 0.6 mm sitting 0.4 mm outside / 0.6 mm inside the body edge

Pad layout: pins **1–21** top edge (pin 1 is 5.1 mm from the −X end), **22–34** right edge,
**35–60** bottom edge running right-to-left. The −X end carries the integrated antenna and has no
pads. Both files verified with `kicad-cli sym/fp export svg`; pad grid checked for count, uniform
pitch and zero overlaps.

### Pin remap for the nets in this plan

| Net | IDC747 pin | **IDC777 pin** | Name on IDC777 |
|-----|-----------|----------------|----------------|
| GND | 1,2,3,4,11,22,27 | **1,12,18,19,21,22,27,34,35,41,51,56,58,59,60** | GND |
| 3.3V_DIG (VBAT) | 32 | **31** | VBAT |
| 3.3V_DIG (VBAT_SENSE) | 31 | **30** | VBAT_SENSE |
| 3.3V_DIG (VDD_PADS) | 33 | **32** | VDD_PADS |
| `BT_UART_TX` | 41 | **38** | UART_TX |
| `BT_UART_RX` | 42 | **39** | UART_RX |
| `BT_RST` | 44 | **36** | ~{RST} |
| `BT_BCK` | 47 | **53** | PCM_CLK |
| `BT_LRCK` | 46 | **52** | PCM_SYNC |
| `BT_SDATA` | 48 | **54** | PCM_OUT |
| no-connect | 29,30,34 | **28,29,33** | CHG_EXT, VCHG, VCHG_SENSE |

Every **net name is unchanged**, so U19, U20, U21 and all decoupling are unaffected — only U14's own
connections move.

**This also resolves open item 3.** On the IDC777 datasheet, PCM_CLK (53) and PCM_SYNC (52) are
**Bi-directional**, and the part explicitly supports *"Master (generate CLK and SYNC) or Slave
(receive CLK and SYNC)"*. The new symbol declares them bidirectional, so configuring the module as
I2S master makes `BT_BCK`/`BT_LRCK` properly driven. The old IDC747 symbol's input-only pins were a
symbol defect, not an architecture problem.

### Antenna orientation changed

The IDC747 note below says top-**right**. The IDC777 datasheet specifies the module be placed at the
top-**left** corner/edge, antenna facing the desired range direction, with an 8 × 4.44 mm ground
clearance area plus all metal removed on **every** layer out to the board edge. The footprint marks
the clearance area on `Dwgs.User` as a reminder — you must still add a real keepout zone/rule in the
PCB. Re-check this against the 100 × 30.48 mm outline before placing.

## Nets

```
U22 IDC777-1 (BT sink)   [was U14 IDC747-1 — EOL, replaced]
  1,12,18,19,21,22,27,34,35,41,51,56,58,59,60  GND  -> GND
  31 VBAT / 30 VBAT_SENSE / 32 VDD_PADS -> 3.3V_DIG  (fixed-voltage config)
  38 UART_TX        -> BT_UART_TX
  39 UART_RX        -> BT_UART_RX
  36 RST#           -> BT_RST
  53 PCM_CLK        -> BT_BCK      (bidirectional; module configured I2S master)
  52 PCM_SYNC       -> BT_LRCK     (bidirectional; module configured I2S master)
  54 PCM_OUT        -> BT_SDATA
  28 CHG_EXT, 29 VCHG, 33 VCHG_SENSE -> no-connect
  13 SYS_CTRL       -> UNRESOLVED, see open item 8
  Unused: 2-9,23-26 PIO, 10/11 USB, 14-17,20 AIO/LED, 37 UART_CTS, 40 UART_RTS,
          42-45 SPKR, 46 MIC_BIAS, 47-50 MIC, 55 PCM_IN, 57 EXT_RF

U21 74LVC157A (source mux)   S=0 selects BT, S=1 selects S3
  16 VCC -> 3.3V_DIG    8 GND -> GND    15 ~E -> GND    1 S -> BT_S3_SEL
  ch1 BCK :  2 <- BT_BCK     3 <- S3_I2S_BCK    4 -> SRC_A_BCK
  ch2 LRCK:  5 <- BT_LRCK    6 <- S3_I2S_LRCK   7 -> SRC_A_LRCK
  ch3 DATA: 14 <- BT_SDATA  13 <- S3_I2S_DOUT  12 -> SRC_A_SDATA
  ch4 unused: 11,10 -> GND   9 -> no-connect

U19 SRC4382 (ASRC + DIR)
  25 MCLK  -> MCLK1              (Y1, 24.576 MHz)
  17 VDD18 -> +1V8
  9 VCC / 33 VDD33 / 42 VIO -> 3.3V_DIG
  10 AGND, 16 DGND1, 30 DGND2, 43 DGND3, 44 BGND -> GND
  14 MUTE  -> GND
  18 CPM   -> 3.3V_DIG           (I2C control-port mode)
  19 A0    -> GND                (address)
  21 A1    -> GND                (address)
  20 CCLK/SCL  -> ESP32_IO22_SCL
  22 CDOUT/SDA -> ESP32_IO21_SDA
  24 RST#  -> SRC_RST
  Port A (input, slave to selected source):
    37 BCKA  <- SRC_A_BCK
    38 LRCKA <- SRC_A_LRCK
    39 SDINA <- SRC_A_SDATA
  Port B (output, slaved to existing TDM clocks):
    48 BCKB  <- BCLK1+TDM1
    47 LRCKB <- LRCK1+TDM1
    45 SDOUTB -> SRC_SDOUT       (new net -> Teensy TDM input)
  Unused for now: 1-8 RX1..RX4 (optical), 26-29 GPO, 31/32 TX, 34 AESOUT,
                  35 BLS, 36 SYNC, 40 SDOUTA, 46 SDINB, 41 NC, 12 RXCKO, 13 RXCKI,
                  11 ~LOCK, 15 ~RDY, 23 ~INT

U20 TLV76718 (1.8 V core rail)
  6 IN -> 3.3V_DIG   4 EN -> 3.3V_DIG   1 OUT -> +1V8   2 SNS -> +1V8
  3,5 GND / 7 PAD -> GND
```

## Decoupling

All placed as `project_fp:C_0805_2012Metric_Pad1.15x1.40mm_HandSolder` (project convention).

- U19: 100 nF at each of VCC(9), VDD18(17), VDD33(33), VIO(42) — **C140, C141, C142, C143**;
  10 µF bulk on +1V8 and 3.3V_DIG — **C144, C145**
- U20: 1 µF in **C147**, 10 µF **C148** + 100 nF **C149** out
- U21: 100 nF at VCC(16) — **C146**
- U22: 100 nF **C150** + 10 µF **C151** at VDD_PADS/VBAT

Schematic placement of these caps is functional, not tidy — they are grouped in free space near
their ICs and connected by label. Rearrange in eeschema if the sheet needs to read cleanly.

## Open items

Nets that exist in the schematic with **only one node** — the block end is wired, the far end is not
assigned. ERC reports each as `pin_not_driven`, which is expected until these are resolved:

| Net | Wired end | Needs |
|-----|-----------|-------|
| `S3_I2S_BCK` | U21.3 | S3 source (see 1) |
| `S3_I2S_LRCK` | U21.6 | S3 source (see 1) |
| `S3_I2S_DOUT` | U21.13 | S3 source (see 1) |
| `BT_S3_SEL` | U21.1 | one S3 GPIO |
| `SRC_SDOUT` | U19.45 | Teensy TDM input (see 2) |
| `SRC_RST` | U19.24 | a GPIO or RC reset |
| `BT_UART_TX` | U14.41 | host UART RX |
| `BT_UART_RX` | U14.42 | host UART TX |
| `BT_RST` | U14.44 | a GPIO or RC reset |

1. **`S3_I2S_BCK` / `S3_I2S_LRCK` / `S3_I2S_DOUT`.** Note the S3 (U13) *already* has an I2S link to
   the Teensy's I2S2 port: `IO6→4_BCLK2`, `IO16→3_LRCLK2`, `IO7→5_IN2`, `IO4→2_OUT2`. The cheapest
   option is to tap those three existing nets into the mux rather than burning three more GPIOs —
   one driver, two loads, electrically fine. The alternative (a second S3 I2S peripheral) needs three
   free GPIOs, and U13's only unused pins are `IO3`, `IO45`, `IO46` (all strapping pins) and
   `IO35`/`IO36`/`IO37` (unusable on octal-PSRAM `-R8` module variants). Decide before assigning.
2. **`SRC_SDOUT`.** The Teensy's TDM1 data input (pin 8 / `IN1`) is **already occupied** by `8_IN1+`.
   Sharing it means relying on both sources tri-stating outside their TDM slots; otherwise this needs
   a different input or a rework of the TDM slot map.
3. ~~**BT I2S clock direction.**~~ **Resolved** by the IDC777 migration — the part supports I2S
   master mode and the new symbol declares PCM_CLK/PCM_SYNC bidirectional. The old symbol's
   input-only clock pins were the defect. Configure the module as I2S master.
7. ~~**Swap U14 to the IDC777-1.**~~ **Done.** U14 deleted (with all 30 wire segments, 10 labels and
   3 no-connect flags); U22 placed and wired. Verified by netlist: `BT_BCK`/`BT_LRCK`/`BT_SDATA` each
   carry exactly two nodes, U22 → U21, and no U14 node remains anywhere.
8. **`SYS_CTRL` (U22 pin 13) — soft power control, currently unconnected.** *Decision: drive it from
   the S3 so firmware can switch the BT receiver on and off.*

   From the datasheet's *Module Boot Modes* (p.12):
   - The module has a real **'Power Off' state that persists while VBAT stays powered** — a genuine
     low-power off, not just a reset.
   - Power Off → Active: **a rising edge on SYS_CTRL held high for 20 ms**.
   - Active → Power Off: **a UART command**, or a rising edge on SYS_CTRL.
   - Applying VBAT from *no* power boots straight to Active, so the module **defaults ON** at
     power-up regardless of this pin.

   Consequences for the design:
   - **It is edge-triggered, not level** — both directions are a rising edge, so pin state does not
     tell you module state. Use **UART for the off path** (BT_UART_TX/RX are already wired, giving a
     deterministic shutdown and known state) and reserve the SYS_CTRL pulse for **boot/wake**.
   - **Add a ~100 k pulldown** at SYS_CTRL. The pin has *no internal pull* and S3 GPIOs are high-Z
     during the S3's own boot/reset; a floating input near threshold can self-trigger and randomly
     toggle BT power.
   - **Drive it directly, no divider.** The datasheet example uses a resistor divider only because it
     sources from a Li-ion VBAT (up to 4.2 V). Here VBAT and the S3 IO are both 3.3 V — same domain.
   - New net: `BT_SYS_CTRL`.
9. ~~`VCHG_SENSE` should be tied to VCHG.~~ **Withdrawn — the no-connects are correct.** That advice
   came from the general pin-description table; the datasheet's *Fixed Voltage Supply Configuration*
   section (p.10) is config-specific and overrides it: with VBAT/VBAT_SENSE/VDD_PADS on one rail,
   *"VCHG and VCHG_SENSE and CHG_EXT are left unconnected"*. As wired. The datasheet does suggest
   bringing all three out to **test points** rather than leaving them bare — cheap bring-up insurance,
   worth adding at layout.

   Context: the charger pins are an **integrated Li-ion charger** aimed at portable products where the
   module is the whole system (cell + module + USB, no external charge IC): VBAT/VBAT_SENSE take the
   cell (3.0–4.6 V), VCHG/VCHG_SENSE take 5 V USB (4.75–6.5 V), CHG_EXT drives an external pass
   transistor above the internal 2–200 mA. This board has a regulated rail and no cell, so the whole
   feature is simply unused — the datasheet calls fixed-voltage the *typical* configuration.
4. **Antenna keepout.** The IDC747 must sit at a top-right corner/edge with all copper removed on
   *every* layer in the clearance zone out to the board edges, no metallic parts, and no routing
   through it. Settle this before placement on the 100 × 30.48 mm outline.
5. `SRC4382` VCC (pin 9) is the DIR comparator/PLL supply — consider an RC or ferrite filter off
   3.3V_DIG rather than a direct tie, if measured jitter matters. Currently a direct tie (C140).
6. Konnect's project config reports `layer_count: 2`; the board is 4-layer. Fix before DRC.
