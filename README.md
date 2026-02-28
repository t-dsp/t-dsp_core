# T-DSP Core

**Part of the [T-DSP](https://t-dsp.com) open modular audio platform.**

A compact 4-layer digital backplane for the T-DSP modular ecosystem. Hosts a Teensy 4.1 and ESP32 with USB Audio, MIDI, Ethernet, and TDM expansion headers for connecting T-DSP audio modules.

## About T-DSP

T-DSP is an open modular audio platform designed for musicians, engineers, and developers who want powerful digital signal processing in a flexible, hackable format. Built around the [Teensy](https://www.pjrc.com/teensy/) microcontroller and the [Teensy Audio Library](https://www.pjrc.com/teensy/td_libs_Audio.html), T-DSP combines studio-quality audio with a growing library of open-source modules.

Learn more at [t-dsp.com](https://t-dsp.com).

## Overview

The T-DSP Core is a **4-layer digital backplane PCB** hosting a Teensy 4.1 and ESP32 alongside supporting ICs. It provides USB Audio, MIDI, Ethernet, and TDM expansion headers for connecting T-DSP audio modules -- in a more accessible and cost-effective form factor than the 8-layer Desktop Pro.

## Processing

- **Teensy 4.1** -- ARM Cortex-M7 running the Teensy Audio Library for real-time DSP (mixing, effects, synthesis, routing). Also receives USB Audio.
- **ESP32** -- UI controller, WiFi/Bluetooth. Drives the display, buttons, and encoder, and sends commands to the Teensy over serial.

## Connectivity

### Audio I/O
- **3.5mm headphone jack** -- Switchcraft 35RAPC4BV4
- **RCA outputs** -- Stereo analog audio output
- **S/PDIF** -- Digital audio input and output

### USB
- **USB-C DEVICE** -- Power and USB Audio to Teensy with ESD protection
- **USB-A HOST** -- USB host connector

### Networking
- **RJ45 Ethernet** -- Wired network connectivity

### MIDI
- **MIDI input** -- 3.5mm TRS with optical isolation (H11L1)
- **MIDI GPIO** -- Configurable for MIDI in, out, or thru

### Power
- **Dual power input** -- USB-C or external supply via JST-XH header, with TPS22860 power switch for automatic switchover
- **Reverse polarity protection** -- P-channel MOSFET on power inputs
- **Power indicators** -- LED indicators for 5V and 12V rails

### Storage
- **SD card** -- Accessible via Teensy 4.1 built-in SD slot

## Expansion

- **2x TDM expansion headers** (2x10 pin, 2.54mm) -- Connect T-DSP audio modules
  - TDM digital audio bus (BCLK, LRCK, DATA_IN, DATA_OUT, MCLK)
  - I2C control bus (SDA, SCL)
  - Power rails (3.3V, 5V, 12V, GND)
  - Control pins (Held-high, Held-low)
- Modules can be soldered directly or socketed with pin headers
- Multiple modules can be chained on the same TDM bus

## User Interface

- **GPIO header** -- Active-low buttons, rotary encoder, and display connections for tactile UI
- **OLED display header** (128x64, SSD1306) -- Status and menu-driven UI
- **3.2" TFT touchscreen header** (ILI9341) -- Touchscreen UI
- **SK6812 addressable LED header** -- ESP32 and Teensy status indication, plus LED strip control

## Board Design

- **4-layer PCB** -- F.Cu (components/analog), In1.Cu (ground plane), In2.Cu (signal/digital), B.Cu (components/analog)
- **1.6mm board thickness**
- LDO voltage regulation for clean analog power
- Separate analog and digital ground domains
- Buffered digital outputs for reliable module distribution

## Project Files

| Directory | Contents |
|-----------|----------|
| `/3d_models/` | 3D models for PCB components |
| `/documentation/` | Schematic PDF, board images, and reference docs |
| `/lib_fp/` | Custom KiCad footprint libraries |
| `/lib_sch/` | Custom KiCad schematic symbol libraries |
| `/manufacturing/` | CI-generated manufacturing outputs (gerbers, BOM, pick & place) |
| `/panel/` | Panelized board layouts for production |

## Status

This board is under active development.

If you build one, we'd love to hear how it goes -- please open an issue or reach out with your findings.

## Contact

For consulting, custom board design, or commercial licensing inquiries, reach out via [LinkedIn](https://linkedin.com/in/jayshoe).

## License

This project is licensed under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/).

You are free to share and adapt this work for non-commercial purposes, as long as you give appropriate credit and distribute any derivatives under the same license.
