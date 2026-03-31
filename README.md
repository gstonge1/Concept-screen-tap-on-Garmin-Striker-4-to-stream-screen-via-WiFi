# Garmin Striker 4 / Striker Vivid 4cv — WiFi Screen Extractor

> **Feasibility study** — Hardware fully characterized, interface confirmed, implementation pending.  
> Looking for contributors with a physical unit to validate the last hardware steps.

Stream your Garmin Striker fishfinder screen wirelessly to any mobile browser — passively, with no permanent modification to the device.

---

## The Idea

The Garmin Striker 4 and Striker Vivid 4cv have no WiFi. Their LCD is driven by a **parallel RGB interface** over a 40-pin FPC connector. By inserting a breakout board between the motherboard and the LCD and capturing the RGB signal, the screen content can be reconstructed and streamed live to any phone browser.

```
[Garmin Striker PCB]
    RGB data (R0-R7, G0-G7, B0-B7)
    HSYNC, VSYNC, PCLK, DE
           ↓ FPC breakout (passive, read-only)
      [Raspberry Pi Zero W]  ← or ESP32-S3 with I2S camera mode
    RGB capture + JPEG encode
           ↓ WiFi
   http://192.168.4.1/stream
           ↓
    [Any mobile browser — no app required]
```

---

## Interface Confirmed: Parallel RGB 24-bit

> **This is the most important finding of this research.**

Early analysis assumed SPI. The interface is actually **parallel RGB 24-bit**, confirmed by two independent sources:

### Source 1 — LCD part number from spare parts retailer

[gfixer.com](https://gfixer.com/products/gp05151) lists the replacement LCD for the **Garmin Striker Plus 4 (4cv, 4dv)** with part number **TM043NDHG13** (Tianma Microelectronics). MPN: `011-04398-30, TM043NDHG13`.

### Source 2 — Datasheet of sibling model TM043NDHG12

The **TM043NDHG12** datasheet (Tianma, 2016, V1.1) was retrieved from [datasheet4u.com](https://datasheet4u.com/pdf/1259013/TM043NDHG12.pdf). This is the revision immediately preceding the G13 and from the same product family. It specifies:

- **Interface: RGB 24-bit** (R8 + G8 + B8)
- **Driver IC: ST7282T2**
- **Connector: FH12A-40S-0.5SH** (Hirose, 40-pin, 0.5mm pitch, ZIF)
- **Resolution: 480×272**
- **Supply voltage: 3.3V**

The G13 revision is expected to be pin-compatible with the G12 — confirmed by the identical physical connector and pin count visible in PCB photos.

### Source 3 — TM043NDH03 specification (twscreen.com)

The closely related **TM043NDH03** (same Tianma family, same 4.3" size) is listed on [twscreen.com](https://www.twscreen.com/en/lcdpanel/15622/tianma/tm043ndh03/) with: *"Signal Interface: Parallel RGB (1 ch 8-bit) FPC 40 pins"* — consistent with the G12 datasheet.

### Why 40 pins makes sense for RGB

The pin count adds up exactly:

| Signal group | Pins |
|---|---|
| R0–R7 (red data) | 8 |
| G0–G7 (green data) | 8 |
| B0–B7 (blue data) | 8 |
| DCLK, HSYNC, VSYNC, DE, DISP | 5 |
| VDD, GND × 3, VLED+, VLED- | 7 |
| X_R, Y_B, X_L, Y_T (resistive touch) | 4 |
| **Total** | **40** ✅ |

---

## Full Pinout — TM043NDHG12 (expected identical for G13)

*Source: TM043NDHG12 datasheet V1.1, Tianma Microelectronics, 2016-02-05*

| Pin | Signal | Type | Description |
|---|---|---|---|
| 1 | VLED- | PWR | Backlight cathode |
| 2 | VLED+ | PWR | Backlight anode |
| 3 | GND | PWR | Ground |
| 4 | VDD | PWR | 3.3V supply |
| 5 | R0 | I | Red data bit 0 |
| 6 | R1 | I | Red data bit 1 |
| 7 | R2 | I | Red data bit 2 |
| 8 | R3 | I | Red data bit 3 |
| 9 | R4 | I | Red data bit 4 |
| 10 | R5 | I | Red data bit 5 |
| 11 | R6 | I | Red data bit 6 |
| 12 | R7 | I | Red data bit 7 |
| 13 | G0 | I | Green data bit 0 |
| 14 | G1 | I | Green data bit 1 |
| 15 | G2 | I | Green data bit 2 |
| 16 | G3 | I | Green data bit 3 |
| 17 | G4 | I | Green data bit 4 |
| 18 | G5 | I | Green data bit 5 |
| 19 | G6 | I | Green data bit 6 |
| 20 | G7 | I | Green data bit 7 |
| 21 | B0 | I | Blue data bit 0 |
| 22 | B1 | I | Blue data bit 1 |
| 23 | B2 | I | Blue data bit 2 |
| 24 | B3 | I | Blue data bit 3 |
| 25 | B4 | I | Blue data bit 4 |
| 26 | B5 | I | Blue data bit 5 |
| 27 | B6 | I | Blue data bit 6 |
| 28 | B7 | I | Blue data bit 7 |
| 29 | GND | PWR | Ground |
| 30 | DCLK | I | Pixel clock |
| 31 | DISP | I | Display on/off |
| 32 | HSYNC | I | Horizontal sync |
| 33 | VSYNC | I | Vertical sync |
| 34 | DE | I | Data enable |
| 35 | NC | — | Not connected |
| 36 | GND | PWR | Ground |
| 37 | X_R | O | Resistive touch right |
| 38 | Y_B | O | Resistive touch bottom |
| 39 | X_L | O | Resistive touch left |
| 40 | Y_T | O | Resistive touch top |

**Note:** Pins 37–40 are resistive touchscreen signals. These are likely not connected on the Striker 4 base model (no touchscreen). On the Vivid 4cv, touch connectivity is unconfirmed.

**Matched connector:** `FH12A-40S-0.5SH` (Hirose)

---

## How the Interface Assessment Evolved

> **Transparency note** — No physical measurements have been taken on a real unit yet. This section documents how the interface conclusion changed during the research process, based entirely on remote analysis of photos and datasheets.

**Original assumption: SPI**

The project started with the assumption that the LCD used SPI — a reasonable starting point for a small 4" display at this price point. Early analysis focused on identifying SCK, MOSI, CS, and DC lines on the FPC connector, and the firmware stack was designed around a passive SPI slave tap.

**What changed the assessment:**

Three converging pieces of evidence shifted the conclusion away from SPI:

1. **The 40-pin FPC connector** — SPI only requires ~13–16 pins total (4 signals + power + backlight). A 40-pin connector is significantly oversized for SPI but maps exactly to a full RGB 24-bit interface (24 data lines + sync signals + power + backlight = 40 pins).

2. **The Winbond SDRAM external memory** — Identified on the PCB in teardown photos. An external framebuffer RAM is a strong indicator of an RGB parallel interface, which requires the MCU to continuously push pixel data from memory to the display rather than writing to an internal display GRAM as in SPI.

3. **The TM043NDHG12 datasheet** — The sibling model explicitly specifies "Interface: RGB 24-bit" with driver IC ST7282T2. This removed all ambiguity.

**What this changes for the project:**

- The original SPI firmware stack (~30 lines) no longer applies
- Capture now requires handling 24 parallel data lines synchronized with a pixel clock
- The Raspberry Pi Zero W becomes the recommended platform (native DPI support)
- The ESP32-S3 is still viable but more complex (I2S camera mode, 16-bit RGB565 compromise)
- The FPC breakout approach remains identical — only the signals of interest change

**What still needs physical validation:**

The interface type is now considered confirmed based on the datasheet evidence, but the exact pin mapping on the Garmin PCB — which FPC pin connects to which signal — has not been physically verified. The datasheet pinout is trusted to be accurate for the G13 revision, but a logic analyzer session on a real unit would provide full confirmation.

---

## PCB Components Identified

From high-resolution eBay and gfixer.ru teardown photos:

| Component | Marking | Function |
|---|---|---|
| Main MCU | `SN/2065850B2CE` | Garmin custom/semi-custom TI processor |
| External RAM | **Winbond SDRAM 256Mbit** | Framebuffer storage — explains why RGB parallel is used |
| Audio ADC | PCM1804 / PCM1604 (TI) | Sonar signal chain — not display related |
| LCD driver | **ST7282T2** | Integrated in LCD FPC module |
| Test pads | 2×5 array, ~1.27mm pitch | Likely JTAG/SWD ARM debug port |

**Key insight:** The presence of **Winbond SDRAM external memory** confirms the MCU needs an external framebuffer — this is consistent with an RGB parallel interface, which requires the MCU to continuously push pixel data rather than relying on an internal display GRAM.

---

## Compatible Models

| Model | LCD part number | Interface | Status |
|---|---|---|---|
| Garmin Striker 4 | TM043NDHG13 | RGB 24-bit | PCB confirmed from gfixer.ru |
| Garmin Striker Plus 4 (4cv, 4dv) | TM043NDHG13 | RGB 24-bit | Confirmed — gfixer.com part GP05151 |
| Garmin Striker Vivid 4cv | **Unknown** | Likely RGB | Specs identical (4", 272×480) but LCD part number not yet sourced |

> **Note on the Striker Vivid 4cv:** gfixer.com lists the Striker Plus 4 (4cv) as using TM043NDHG13, but this refers to the **Plus** series, not the **Vivid** series. The Vivid is a newer generation (2021). The LCD part number for the Vivid 4cv has not been independently confirmed. The physical screen dimensions and resolution are identical to the Striker 4 base model, but the specific LCD module may differ. Physical inspection of the FPC label on a Vivid unit would confirm compatibility.

---

## Hardware

### Bill of Materials

| Component | Details | Source | Price |
|---|---|---|---|
| Raspberry Pi Zero W | Primary capture platform for RGB parallel | Digikey / Amazon | ~$15 |
| **or** ESP32-S3 DevKit | Alternative — I2S camera mode for RGB capture | Digikey / Amazon | ~$12 |
| FPC breakout 40p 0.5mm | "FPC-40P 0.5MM adapter" | AliExpress | ~C$1.50 for 2 |
| FFC cable 40p 0.5mm same-side | 100mm extension | AliExpress | ~$1 |
| Jumper wires | 22–27 wires (full RGB bus) | Any | ~$1 |

**Optional for signal identification:**

| Tool | Use | Price |
|---|---|---|
| Multimeter | Identify VDD, GND, DCLK on ZIF pins | Any |
| Logic analyzer 24MHz 8ch | Confirm interface, identify HSYNC/VSYNC/DCLK | ~$10 (AliExpress clone) |

### FPC Connector — Fully Characterized

| Parameter | Value | Source |
|---|---|---|
| Pin count | **40 pins** | PCB photos + datasheet |
| Pitch | **0.5mm** | Datasheet FH12A-40S-0.5SH |
| Connector type | ZIF flip-top, bottom contact | PCB photos |
| Matched Hirose part | FH12A-40S-0.5SH | TM043NDHG12 datasheet |
| FPC cable type | Same-side (SS), 0.3mm thick | Standard for this connector |
| Max cable length | 15–20 cm | Signal integrity limit at RGB pixel clock |

---

## Capture Platform Options

### Option 1 — Raspberry Pi Zero W (recommended)

The Pi Zero W has a native **DPI (Display Parallel Interface)** that can be reconfigured to receive instead of transmit RGB parallel data. WiFi is built-in.

```
RGB 24-bit → Pi Zero W DPI input → framebuffer → MJPEG → WiFi
```

- Well documented community
- Native RGB parallel support
- Built-in WiFi
- ~$15

### Option 2 — ESP32-S3

The ESP32-S3 has an **I2S / LCD_CAM peripheral** capable of receiving parallel data in camera mode (same principle as OV2640 camera on ESP32-CAM). However, the OV2640 uses 8-bit parallel — the RGB LCD uses 24-bit, which exceeds the ESP32-S3's I2S width. A possible workaround is to capture only 8 bits (one color channel) or use RGB565 mode (16-bit).

More complex than the Pi Zero W approach but lower cost and smaller footprint.

---

## Signal Identification Procedure

With the FPC breakout inserted and the Striker powered on with an active sonar screen:

**Step 1 — Find GND and VDD first (multimeter DC mode)**

| Reading | Signal |
|---|---|
| 0V fixed | GND — pins 3, 29, 36 per datasheet |
| 3.3V fixed | VDD — pin 4 per datasheet |
| ~14V fixed | VLED+ — backlight anode |

**Step 2 — Find DCLK (logic analyzer)**

DCLK (pin 30) is the most distinctive signal — a continuous square wave at ~9 MHz when the screen is active (480×272 @ 60Hz). Connect logic analyzer CH1 to suspected clock pins until you find the regular square wave.

**Step 3 — Confirm HSYNC and VSYNC**

HSYNC fires ~272 times per frame (~16 kHz). VSYNC fires 60 times per second. Both are easy to identify on a logic analyzer.

**Step 4 — Verify RGB data**

R0–R7, G0–G7, B0–B7 will all show activity synchronized with DCLK when a colorful screen is displayed.

---

## Similar Published Projects

### Hardware interception via FPC ribbon cable

**[SAAB 9-3 ICM2 Display Hacking](https://hackaday.io/project/170893)** — Leigh Oliver  
Reverse-engineering the LCD of a SAAB 9-3 infotainment module using an Adafruit FPC Stick as a man-in-the-middle interposer on a 0.5mm ribbon cable. Identical method, different device.  
→ GitHub: `leighleighleigh/SAAB_ICM2_LCD_RE` and `leighleighleigh/Arduino_SAAB_ICM2_LCD_Driver`

### Passive SPI bus sniffing (original firmware reference)

**[martinberlin/esp32-spi-slave](https://github.com/martinberlin/esp32-spi-slave)**  
Built to debug ePaper display SPI communication. Still relevant as a reference for passive bus capture methodology.

**[zaphoxx/pico-tpmsniffer-spi](https://github.com/zaphoxx/pico-tpmsniffer-spi)**  
Passive SPI capture on RP2040, pogo-pin PCB design, open KiCad files.

### RGB parallel display reference

**[Raspberry Pi DPI documentation](https://pinout.xyz/pinout/dpi)**  
Official documentation for Pi Zero W parallel RGB interface — the hardware basis for Option 1.

### LCD framebuffer reference

**[juj/fbcp-ili9341](https://github.com/juj/fbcp-ili9341)**  
Deep documentation of LCD display interfaces, framebuffer techniques, and SPI/RGB protocol understanding.

---

## Research Sources

All findings are based on publicly available sources. No proprietary information was used.

| Finding | Source | URL |
|---|---|---|
| PCB photos, component identification | gfixer.ru spare parts catalog | https://gfixer.ru |
| LCD part number TM043NDHG13 for Striker Plus 4 | gfixer.com product GP05151 | https://gfixer.com/products/gp05151 |
| TM043NDHG12 full pinout and datasheet | datasheet4u.com | https://datasheet4u.com/pdf/1259013/TM043NDHG12.pdf |
| TM043NDH03 interface confirmation (RGB parallel, 40-pin) | twscreen.com | https://www.twscreen.com/en/lcdpanel/15622/tianma/tm043ndh03/ |
| Winbond SDRAM identification | eBay teardown photos + Winbond datasheet | https://www.digikey.com/en/products/detail/winbond-electronics/W9825G6KH-6I/5001920 |
| FH12A-40S-0.5SH connector spec | TM043NDHG12 datasheet note 1 | — |
| Striker Vivid 4cv specs (272×480, 4") | Garmin official + gpscentral.ca | https://www.gpscentral.ca/wp-content/uploads/Garmin_STRIKER_Vivid_4cv_Specifications.pdf |

---

## Status

- [x] Problem defined
- [x] PCB photographed and analyzed — gfixer.ru and eBay teardown photos
- [x] LCD part number confirmed — TM043NDHG13 (Striker 4 / Striker Plus 4)
- [x] Interface confirmed — **RGB parallel 24-bit** (not SPI as originally assumed)
- [x] Full pinout documented — from TM043NDHG12 datasheet (sibling model)
- [x] FPC connector fully characterized — 40-pin, 0.5mm, ZIF, FH12A-40S-0.5SH
- [x] PCB chips identified — MCU (SN/2065850B2CE), Winbond SDRAM, ST7282T2 driver
- [x] FPC breakout identified — AliExpress FPC-40P 0.5mm, C$1.50/2pcs
- [x] Logic analyzer identified — 24MHz 8ch clone, AliExpress ~$10
- [x] Capture platform options documented — Pi Zero W (recommended) or ESP32-S3
- [x] Similar projects referenced and validated
- [ ] LCD part number confirmed for **Striker Vivid 4cv** specifically
- [ ] Physical pin mapping on a real unit (needs unit + multimeter + FPC breakout)
- [ ] Logic analyzer validation of RGB interface
- [ ] Firmware implementation
- [ ] End-to-end demo

---

## Enclosure Modification

**The challenge:** Once the Garmin is opened for testing, the power connector becomes inaccessible with the original cover reinstalled. This makes development impractical without a permanent solution.

### Solution: Partial case cut + 3D printed replacement

casecutline3dscan.png

The most practical compromise identified so far — preserving all original ports and mount points while creating internal space for the ESP32-S3 and FPC breakout.

**Step 1 — Cut the top portion of the case**

A straight horizontal cut removes the upper section of the Garmin enclosure. The cut line is chosen to:
- Keep the power connector and transducer port fully accessible at their original positions
- Retain the original mounting bracket hardware unchanged
- Require no modification to the PCB, screen, or any electronic component

> *Photo with red cut line — coming when physical unit is available*

**Step 2 — 3D print a partial replacement**

A replacement top section is printed to match the cut geometry, with modified internal dimensions to accommodate:
- ESP32-S3 DevKit
- FPC breakout board (40-pin, 0.5mm)
- Wiring between FPC breakout and ESP32-S3
- Optional: USB-C port for ESP32-S3 power and programming access

**Why this approach**

| | Original cover | Cut + 3D printed |
|---|---|---|
| Power connector accessible | ❌ | ✅ |
| Transducer port intact | ✅ | ✅ |
| Mount hardware unchanged | ✅ | ✅ |
| ESP32-S3 housed internally | ❌ | ✅ |
| Weatherproof | ✅ | ✅ with proper print |
| PCB/screen modified | ❌ | ❌ |
| Reversible | — | ✅ original parts untouched |

**3D print files** — will be published in `/enclosure/` once the physical unit is available for measurement.

---

## Contributing

**If you have a Garmin Striker 4, Striker Plus 4, or Striker Vivid 4cv:**

The single most valuable contribution right now is opening the unit and reading the part number printed on the FPC cable of the LCD. It takes 5 minutes and confirms compatibility.

1. Open the unit (T8 Torx screws)
2. Locate the flat ribbon cable connecting the screen to the motherboard
3. Read the part number printed on the cable or the LCD module
4. Open an Issue with the part number and which model you have

**Second most valuable:** Insert the FPC breakout, connect a logic analyzer, and confirm DCLK frequency and RGB signal presence.

**Third most valuable — 3D scan of the Garmin enclosure:**

If you have access to a 3D scanner, scanning the Garmin cover and publishing the scan file would allow the community to design and print the replacement enclosure section without needing physical access to a unit. Publish the scan as `.STL` or `.STEP` in a GitHub Issue or Pull Request — it would directly unblock the enclosure design for everyone.

**PCB part numbers for reference:**
- Striker 4: `105-03781-01`
- Striker Plus 4: `105-03292-00`
- Striker Vivid 4cv: unknown — help needed

---

## License

MIT — do whatever you want with this.

---

*Feasibility study conducted March 2026. All PCB analysis based on publicly available teardown photos and manufacturer datasheets. All referenced GitHub projects belong to their respective authors. English documentation assisted by Claude (Anthropic) — French is the author's native language.*
