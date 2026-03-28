# Garmin Striker 4 — WiFi Screen Extractor

> **Concept project** — Research complete, implementation pending.  
> All hardware identified, firmware stack documented. Ready to build.

Stream your Garmin Striker 4 fishfinder screen wirelessly to any mobile browser — for ~$13 in parts, with 5 wires connected to the PCB.

---

## The Idea

The Garmin Striker 4 has no WiFi. But its internal LCD is driven by a standard SPI bus, and the PCB exposes accessible points that give direct access to those SPI signals.

By connecting an ESP32-S3 passively to those signals, capturing the SPI bus, reconstructing the framebuffer, and streaming MJPEG over WiFi, the Striker 4's screen can be viewed live on any phone browser — with zero modification to the Garmin's behavior.

```
[Garmin Striker 4 PCB]
    SCK, MOSI, CS, DC, GND
           ↓ 5 wires (passive, read-only)
      [ESP32-S3 DevKit]
    SPI slave + JPEG encode
           ↓ WiFi
   http://192.168.4.1/stream
           ↓
    [Any mobile browser — no app required]
```

---

## Key Hardware Discovery

High-resolution PCB photos from [gfixer.ru](https://gfixer.ru) (spare parts retailer) reveal the Striker 4 internals in detail.

**PCB part number:** `105-03781-01`

**What is visible on the PCB:**

- **LCD FPC connector** — 34-pin, 0.5mm pitch, ZIF flip-top, bottom contact — on the board edge. This is the primary access point to the SPI bus.
- **2×5 test pad array** — visible on the PCB, likely JTAG/SWD debug port used during manufacturing. Useful for GND and VCC reference.
- **PCM1804** Texas Instruments ADC — sonar signal chain only, not display-related.
- **EMI shield** covering the main MCU.

**Access strategy:**

The best approach is to use a **FPC breakout board** (34-pin, 0.5mm) inserted between the Garmin motherboard and the LCD. This keeps the display fully functional while exposing all 34 pins on standard 2.54mm headers for measurement and connection.

---

## Why This Works

The SPI bus to a TFT LCD carries **raw, unencrypted pixel data**. No proprietary protocol, no encryption, no compression. The stream is:

```
Command 0x2A → set column range (X1 to X2)
Command 0x2B → set row range  (Y1 to Y2)
Command 0x2C → write pixels (RAMWR)
Data bytes   → RGB565 pixels, 2 bytes per pixel
```

This is a documented open standard defined by the LCD controller datasheet (ILI9341, ST7789, or similar family). Any SPI sniffer reads it directly.

**There is nothing to decrypt.** The only work required is:
- Watch the `DC` line to distinguish commands from pixel data
- When command `0x2C` appears, start storing the following bytes as pixels
- Every 2 bytes = 1 RGB565 pixel placed at the current cursor position

---

## Hardware

### Bill of Materials (~$13 total)

| Component | Details | Source | Price |
|---|---|---|---|
| ESP32-S3 DevKit | Espressif official board | Digikey / Amazon | ~$12 |
| FPC breakout 34p 0.5mm | "FPC-34P 0.5MM adapter" | AliExpress | ~C$1.50 for 2 |
| 5 jumper wires | 10cm, male-male | Any | ~$0 |
| 33Ω resistors × 4 | Series protection on SPI lines | Any | ~$0 |

**Optional for signal identification:**

| Tool | Use | Price |
|---|---|---|
| Multimeter | Identify SCK/MOSI/CS/DC on ZIF pins | Any |
| Logic analyzer | Kingst LA1010 or Saleae | $30–$180 |

### LCD FPC Connector — fully characterized from PCB photos

| Parameter | Value |
|---|---|
| Connector type | ZIF flip-top, bottom contact |
| Pin count | **34 pins** |
| Pitch | **0.5 mm** |
| Actuator | Right side, flip-top style |
| Compatible ZIF part | Hirose FH12-34S-0.5SH(55) |
| FPC extension cable | FFC 34p 0.5mm same-side, 100mm |
| Breakout for prototyping | "FPC-34P 0.5MM" on AliExpress — C$1.50/2pcs |

**Important:** The LCD must remain connected during measurement. The Garmin MCU likely checks for LCD presence at boot and will not send SPI data if the display is disconnected. The FPC breakout keeps the LCD connected while exposing all pins.

**FPC cable length limit:** Keep extension cable under 15–20 cm. Beyond that, cable capacitance degrades SPI signal edges at ~10 MHz.

---

## Signal Identification Procedure

With the FPC breakout inserted and the Striker 4 powered on, displaying an active sonar screen:

**Using a multimeter (DC voltage mode), probe each of the 34 pins:**

| Reading | Signal type |
|---|---|
| 0V fixed | GND |
| 3.3V fixed | VCC or inactive signal |
| Oscillates slowly | CS or DC |
| ~1.5–1.8V average (fast) | SCK or MOSI |

**Signal characteristics:**
- `SCK` — most regular square wave, ~10 MHz, always active during screen updates
- `CS` — stays HIGH, pulses LOW in bursts
- `DC` — changes slowly, long HIGH or LOW states
- `MOSI` — varies rapidly with screen content

**Tip:** Display active sonar content while probing — this maximizes MOSI activity and makes it easy to distinguish from static lines.

**Risk note:** A short between two signal lines is harmless momentarily. Avoid shorting VCC (3.3V fixed, typically pin 1 or 2) to GND.

---

## Firmware Architecture

Four open-source components cover the entire firmware stack:

### 1. SPI Slave Reception

**[martinberlin/esp32-spi-slave](https://github.com/martinberlin/esp32-spi-slave)**

Working Arduino-framework ESP32 SPI slave library, built specifically for passive SPI bus sniffing on display projects.

```cpp
#include <ESP32SPISlave.h>
ESP32SPISlave slave;
uint8_t rx_buf[4096];

void setup() {
    slave.setDataMode(SPI_MODE0);
    slave.begin(); // SCK=18, MOSI=23, CS=5
}
void loop() {
    size_t n = slave.transfer(NULL, rx_buf, 4096);
    process_spi_data(rx_buf, n);
}
```

### 2. Framebuffer Reconstruction (~30 lines)

```cpp
uint16_t framebuffer[160 * 120];
int cur_x = 0, cur_y = 0;
bool writing = false;
uint8_t high_byte = 0;
bool got_high = false;

void process_byte(uint8_t dc_level, uint8_t byte) {
    if (dc_level == 0) {
        writing = (byte == 0x2C); // RAMWR command
        got_high = false;
    } else if (writing) {
        if (!got_high) {
            high_byte = byte;
            got_high = true;
        } else {
            uint16_t pixel = (high_byte << 8) | byte;
            framebuffer[cur_y * 160 + cur_x] = pixel;
            cur_x++;
            if (cur_x >= 160) { cur_x = 0; cur_y++; }
            if (cur_y >= 120) cur_y = 0;
            got_high = false;
        }
    }
}
```

### 3. JPEG Encoding

`esp_jpg_encode()` — built into ESP-IDF, no external dependency. Converts the RGB565 framebuffer to JPEG in one function call.

### 4. MJPEG WiFi Stream

**[arkhipenko/esp32-cam-mjpeg-multiclient](https://github.com/arkhipenko/esp32-cam-mjpeg-multiclient)**

Replace the camera frame source with the framebuffer. Stream accessible at `http://192.168.4.1/stream` in any mobile browser. No app required, no installation, works on iOS and Android.

---

## Similar Published Projects

### Hardware interception via FPC ribbon cable

**[SAAB 9-3 ICM2 Display Hacking](https://hackaday.io/project/170893)** — Leigh Oliver  
Reverse-engineering the LCD of a SAAB 9-3 infotainment module using an Adafruit FPC Stick as a man-in-the-middle interposer on a 0.5mm ribbon cable. Identical method, different device.  
→ GitHub: `leighleighleigh/SAAB_ICM2_LCD_RE` and `leighleighleigh/Arduino_SAAB_ICM2_LCD_Driver`

### Passive SPI bus sniffing

**[zaphoxx/pico-tpmsniffer-spi](https://github.com/zaphoxx/pico-tpmsniffer-spi)**  
Passive SPI capture on RP2040, pogo-pin PCB design, open KiCad files published.

**[stacksmashing/pico-tpmsniffer](https://github.com/stacksmashing/pico-tpmsniffer)**  
Original TPM SPI sniffer, $4 Raspberry Pi Pico, no soldering required on the target device.

### ESP32 SPI slave sniffing on a display

**[martinberlin/esp32-spi-slave](https://github.com/martinberlin/esp32-spi-slave)**  
Built to debug ePaper display SPI communication. One ESP32 sniffs what the other sends to the display. Directly applicable firmware base.

### SPI LCD command protocol reference

**[juj/fbcp-ili9341](https://github.com/juj/fbcp-ili9341)**  
Deep documentation of SPI LCD command sets (ILI9341/ST7789), partial updates, and framebuffer techniques. Best reference for understanding the command protocol.

---

## Why Nobody Did This Before

- The fishfinder community and the hardware hacking community do not overlap
- The ESP32-S3 with hardware JPEG encoding only became available in 2021
- TPM SPI sniffer projects live in the cybersecurity world, not in maker or fishing circles
- High-resolution PCB photos were only available because a spare parts retailer published teardown photos to sell replacement parts
- Confirming the FPC connector specs requires physically inspecting the board — almost nobody disassembles a working Striker 4

---

## Status

- [x] Problem defined
- [x] PCB photographed and analyzed (gfixer.ru photos)
- [x] LCD FPC connector fully characterized — 34-pin, 0.5mm pitch, ZIF flip-top, bottom contact
- [x] FPC breakout identified — AliExpress FPC-34P, C$1.50 for 2 pcs
- [x] Signal identification procedure documented
- [x] Firmware stack identified — 4 open-source components
- [x] Bill of materials complete (~$13 total)
- [x] Similar projects referenced and validated
- [ ] Physical signal mapping on a real unit (needs Striker 4 + multimeter + FPC breakout)
- [ ] Firmware implementation
- [ ] MJPEG stream validation
- [ ] End-to-end demo video

---

## Contributing

**If you have a Garmin Striker 4 and a multimeter**, you can advance this project today:

1. Open the unit (6 Phillips #0 screws)
2. Order the FPC-34P 0.5mm breakout from AliExpress (~C$1.50)
3. Insert the breakout between the motherboard ZIF connector and the LCD cable
4. Power on the unit with the sonar screen active
5. Measure DC voltage on each of the 34 breakout pins
6. Note which pins oscillate — those are your SPI signals
7. Open an Issue here with your findings

**PCB part number to confirm compatibility:** `105-03781-01`

---

## License

MIT — do whatever you want with this.

---

*Research conducted March 2026. PCB analysis based on publicly available photos from gfixer.ru spare parts catalog. All referenced GitHub projects belong to their respective authors.*
