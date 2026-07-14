# T-DSP Core — Design Decisions

Architecture decision records for the T-DSP Core (Desktop Pro) backplane. Newest first.

---

## ADR-002 — Next core: OPI PSRAM footprint for 32/64 MB resident fonts (2026-07-14)

**Status:** Accepted — design intent for the next core spin.

### Context
ADR-001 caps this Teensy-4.1 board at **16 MB** (stock QSPI PSRAM). To hold a high-quality GM
font **fully resident** in the chosen TSF engine — native GeneralUser GS (**30.6 MB** of
samples), or larger — the next system moves to a **bare i.MX RT1062** owning its own FlexSPI2,
populated with a single **OPI (Octal) PSRAM** chip. Downsampled GeneralUser variants
(`t-dsp_software/tools/sf2/`) let TSF fit 16 MB *today* (gu12 ~11.7 MB, gu14 ~13.5 MB) but
trade treble for fit; OPI PSRAM removes the ceiling.

### Decision
Design the next core around **one x8 OPI PSRAM footprint**, populated **one chip at a time**
with either density — these two parts are the targets:

| Target | Part number | Density / bus | Holds resident |
|---|---|---|---|
| **32 MB** | **AP Memory APS256XXN-OBRx** | 256 Mbit, DDR Octal (x8) | native GeneralUser GS (30.6 MB) + headroom |
| **64 MB** | **AP Memory APS512XXN-OBRx** | 512 Mbit, DDR Octal (x8) | GeneralUser + a 2nd font, or a ~40–50 MB studio font |

- **One land pattern, two densities.** Pick an x8 OPI BGA common to both the 256 Mb and 512 Mb
  parts so the same PCB accepts 32 MB *or* 64 MB per build. **Verify pin/package compatibility**
  between APS256XXN and APS512XXN in the chosen x8 package before committing the footprint (the
  512 Mb part also offers x16 — use **x8** to match the 256 Mb).
- Route FlexSPI2's **8 data lines + DQS** (vs. 4 for QSPI) plus CLK/CS.
- **Diminishing returns past 64 MB** (holding a 200–400 MB reference font resident) — beyond
  that, stream from SD (sf22aswt) or move to a larger-DRAM SoC.

### Firmware impact — going OPI BREAKS stock PSRAM until the Octal driver is written
- The stock Teensy / i.MXRT core PSRAM bring-up (`configure_external_ram`) speaks **QSPI
  APS6404 only**. It will **not** detect or initialise an OPI part: `external_psram_size` reads
  **0**, `extmem_malloc` fails, and **both SF2 engines are dead** until fixed.
- Fix = a one-time **Octal-DDR FlexSPI2 init driver** (custom LUT + mode-register config, and
  it must set `external_psram_size`), ported from the NXP SDK / RT1170 OPI examples. Everything
  *above* the PSRAM driver — the TSF int16 storage, the font loader, the engines — is
  chip-agnostic and unchanged.

### One interface only (decided) — OPI is not a second bus
OPI is **not a separate interface**. It is the **same FlexSPI2 controller** the Teensy already
uses for its QSPI PSRAM, run in **octal (x8) DDR** mode — a *superset* of the same pins (adds
DQ4–7 + a DQS strobe). So the next core routes exactly **one** PSRAM interface (the single OPI
chip on FlexSPI2) and does not populate any QSPI PSRAM. **No dual-routing.** On a bare RT1062,
FlexSPI1 stays the boot NOR flash; FlexSPI2 is yours for the OPI PSRAM.

**Fast/robust enough? Yes — *faster* than today's QSPI.** 8 DDR data lines vs 4 gives ~4–6× the
bandwidth (roughly ~66 MB/s QSPI → ~400 MB/s octal-DDR), and the DQS strobe aids timing closure
at speed. It's the mainstream external-RAM interface on the NXP RT1060/1170 EVKs and countless
shipping products. TSF already renders from QSPI PSRAM at ~45 % CPU today, so OPI is pure
headroom, not a risk — it just needs normal high-speed-DDR layout care (length-matched
DQ / DQS / CLK).

### Nested / concentric footprints (populate-one, zero extra routing)
Because QSPI is a **pin-subset** of OPI on the *same* FlexSPI2, multiple parts can share **one
land area** — populate exactly one, no second interface:
- **QSPI (SOIC-8) nested inside the OPI BGA.** APS6404 uses only SCLK, CS#, DQ0–3 — all of which
  the OPI part also uses (it adds DQ4–7 + DQS). Co-locate an 8 MB QSPI SOIC-8 land *inside* the
  OPI BGA, wiring the shared FlexSPI2 nets (CLK / CS0 / DQ0–3) to both, and DQ4–7 / DQS / RESET#
  to the BGA only. Stuff the QSPI part → **8 MB on the STOCK Teensy library** for board bring-up;
  later stuff the OPI part → 32/64 MB with the custom Octal driver. The earlier "fallback
  footprint" hedge is now **free** — it's the same nets, not a second bus.
- **APS256 (32 MB) and APS512 (64 MB) on one BGA.** Both are x8 Xccela; if they share the x8
  BGA-24 package (**verify** — the 512 Mb also comes as x16 BGA-49; use the x8), a single BGA
  land takes either density. Nest only if the packages actually differ.

Two caveats for the shared land: **(1)** keep the stub from the shared trace to each unpopulated
pad **short** — dangling stubs hurt the octal-DDR bus at speed; **(2)** **match IO voltage** —
Xccela OPI PSRAM is often **1.8 V** while APS6404L is commonly **3.3 V**; the shared FlexSPI2
pins sit at one bank voltage, so pick same-voltage parts (a 1.8 V QSPI PSRAM, or a 3.3 V OPI
part) or the nesting doesn't fly.

### More than 64 MB (e.g. 64+16)?
Possible, but not worth it. FlexSPI2 can address multiple devices on separate chip-selects, so an
OPI chip **plus** a QSPI chip (→ 64 + 16 = 80 MB) is not impossible — but it means two devices on
the high-speed bus (ideally split across FlexSPI2's A/B ports, pinmux permitting), a mixed
octal+quad custom driver, and tighter layout, all for capacity nothing needs: **64 MB already
holds native GeneralUser (30.6 MB) ~2× over.** If a font ever exceeds 64 MB, stream it from SD
(the sf22aswt engine) instead of adding RAM. **Target stays: one 32 MB or 64 MB OPI chip on the
single FlexSPI2 interface.**

### References
- Datasheets: **APS256XXN-OBRx** (256 Mb OPI) and **APS512XXN** (512 Mb OPI), AP Memory Xccela.
- Extends ADR-001's "Future / custom-MCU consideration" with committed part targets (supersedes
  the placeholder "APS25616" named there).
- Firmware font tooling: `t-dsp_software` → `tools/sf2/FONTS.md`, `tools/sf2/build_gu_fonts.py`.

---

## ADR-001 — PSRAM for the sampled soundfont engines (2026-07-14)

**Status:** Accepted

### Context
T-DSP Core hosts a **Teensy 4.1**. Its external PSRAM lives on the two QSPI pads on the
**underside of the Teensy module itself** (not on this backplane), so this is a
*populate-the-Teensy* decision, not a backplane-footprint one.

The T-DSP synth firmware includes sampled **General-MIDI SF2 engines** that need PSRAM
to hold sample data:

- **`teensy41_sf2_tsf` — TinySoundFont (the chosen engine).** Full-fidelity SF2 renderer:
  velocity layers, region layering, per-voice filters, and **MPE** (per-note pitch bend +
  pressure). It keeps the **entire font's samples RESIDENT in PSRAM** (stored as int16 via
  a local patch), so its font must *fit* PSRAM. Shipped font: **TimGM6mb (~6 MB)**.
- **`teensy41_sf2` — sf22aswt / AudioSynthWavetable.** Streams instruments **on-demand**
  from the SD card, so the font itself can be huge (full GeneralUser GS, 30 MB) while only
  the in-use instruments sit in PSRAM. Trade-off: fidelity ceiling — no velocity layers.

With `external_psram_size == 0` both are effectively unusable (TSF load fails; sf22aswt
falls back to a ~400 KB internal-RAM cap). The core auto-detects size and the firmware runs
unchanged on 0 / 8 / 16 MB.

### Decision
**Populate the Teensy 4.1 with PSRAM. Minimum 1× 8 MB; 16 MB (both pads) recommended.**

| PSRAM | Enables | Notes |
|---|---|---|
| 0 MB | *(no sampled GM)* | sampled SF2 engines don't run |
| **8 MB** (1 chip) | **TSF + TimGM6mb resident, full MPE** — the shipped config | minimum for the chosen engine |
| **16 MB** (2 chips) | headroom: larger resident TSF font (≤~14 MB) or bigger sf22aswt streaming cache | recommended full-populate |

Part: **AP Memory APS6404L-3SQR** — 64 Mbit (8 MB) QSPI PSRAM, SOIC-8 — one or both
underside Teensy pads. Adding the 2nd chip is auto-detected; **no firmware rebuild**.

### Rationale — quality is set by the *engine*, not PSRAM size
- TSF sounds best and does real MPE, but its font must be RESIDENT → compact font only
  (TimGM6mb-class at 8 MB; up to ~14 MB font at 16 MB).
- sf22aswt can reach genuinely high-quality fonts (FluidR3 ~150 MB) streamed from SD, but
  cannot render velocity layers.
- High-quality GM fonts are ~150 MB (FluidR3) to ~400 MB (reference); those live on **SD**,
  and PSRAM only ever holds the *working set* — you never need "as much PSRAM as the font."

Given the project has committed to TSF + MPE, **8 MB is the functional minimum and 16 MB the
comfortable target** for this Teensy-4.1-based board.

### Future / custom-MCU consideration (>16 MB via OPI PSRAM)
16 MB is the ceiling of the Teensy 4.1 / stock i.MXRT **QSPI** PSRAM driver. A future spin
that places a **bare i.MX RT1062** on this board (owning its own FlexSPI2) could go larger
with **OPI (Octal) PSRAM**:

- **QSPI caps at 8 MB/chip** (APS6404L). 16/32 MB parts are **OPI** — 8 data lines, not 4.
- **A single 32 MB OPI chip (e.g. APS25616) is the sweet spot:** it lets **TSF hold the full
  30 MB GeneralUser GS resident at full fidelity** — high-quality *and* fully expressive, the
  biggest quality-per-effort jump beyond 16 MB.
- **Cost:** route **8 data lines** (vs 4) + a one-time **Octal FlexSPI init** driver (the stock
  core is QSPI-only; port from the NXP SDK / RT1170 OPI examples). Everything *above* the PSRAM
  driver — the int16 TSF storage, the font loader, the engines — is chip-agnostic and unchanged.
- **Diminishing returns past ~64 MB** (holding a 200–400 MB reference font resident); at that
  point, stream from SD or move to a larger-DRAM SoC.

**Recommendation:** ship 8/16 MB **QSPI on the Teensy 4.1** (zero new firmware, sufficient for
the shipped TSF engine). *If* a custom-RT1062 spin happens and resident high-quality GM is a
goal, target **one 32 MB OPI PSRAM chip** + the Octal FlexSPI driver.

### References
- Firmware: `t-dsp_software` → `lib/TDspTsf` (TSF int16 patch), `lib/TDspSF2` (sf22aswt),
  build envs `teensy41_sf2_tsf` / `teensy41_sf2`, and `lib/TDspSF2/SF2_GM_ENGINE_PLAN.md`.
- Teensy 4.1 PSRAM: underside SOIC-8 pads on FlexSPI2, `external_psram_size` = 0/8/16.
