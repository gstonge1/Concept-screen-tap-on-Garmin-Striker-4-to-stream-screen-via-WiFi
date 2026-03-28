# Garmin Striker 4 — WiFi Screen Extractor

> **Concept project** — Research complete, implementation pending.  
> All hardware identified, firmware stack documented. Ready to build.

Stream your Garmin Striker 4 fishfinder screen wirelessly to any mobile browser — for ~$12 in parts, with 5 wires soldered directly to the PCB test pads.

---

## The Idea

The Garmin Striker 4 has no WiFi. But its internal LCD is driven by a standard SPI bus, and the PCB exposes **test pads** that give direct access to those SPI signals — no FPC connector manipulation required.

By connecting an ESP32-S3 to those pads, capturing the SPI bus passively, reconstructing the framebuffer, and streaming MJPEG over WiFi, the Striker 4's screen can be viewed live on any phone browser.

```
[Garmin Striker 4 PCB test pads]
    SCK, MOSI, CS, DC, GND
           ↓ 5 wires
      [ESP32-S3 DevKit]
    SPI slave + JPEG encode
           ↓ WiFi
   http://192.168.4.1/stream
           ↓
    [Any mobile browser]
```

---

## Key Discovery

Disassembly photos from [gfixer.ru](https://gfixer.ru) (spare parts retailer) reveal the Striker 4 PCB in high resolution.

**PCB part number:** `105-03781-01`

**What's visible:**
- A row of white test pads on the bottom-left of the PCB — almost certainly exposing the SPI bus directly
- The LCD FPC connector (34-pin, 0.5mm pitch, ZIF flip-top) on the board edge
- A PCM1804 Texas Instruments ADC (sonar signal chain — not display-related)
- An EMI shield covering the main MCU

**The test pads are the shortcut.** If they expose SCK, MOSI, CS, DC (highly likely given their count and position), no interposer PCB is needed at all.

---

## Why This Works

The SPI bus to a TFT LCD carries **raw, unencrypted pixel data**. There is no proprietary protocol, no encryption, no compression. The data is:

```
Command 0x2A → set column range
Command 0x2B → set row range  
Command 0x2C → write pixels (RAMWR)
Data bytes   → RGB565 pixels, 2 bytes each
```

This is a documented open standard defined by the LCD controller datasheet (likely ILI9341, ST7789, or similar family). Any SPI sniffer can read it directly.

---

## Hardware

### Bill of Materials (~$12 total)

| Component | Details | Source | Price |
|---|---|---|---|
| ESP32-S3 DevKit | Espressif official board | Digikey / Amazon | ~$12 |
| 5 jumper wires | 10cm, male-male | Any | ~$0 |
| 33Ω resistors × 4 | 0805 or leaded | Any | ~$0 |

**Optional for test pad mapping:**
| Multimeter | Continuity mode | Any | — |
| Logic analyzer | Kingst LA1010 or Saleae | AliExpress / Digikey | $30–$180 |

### LCD FPC Connector (if test pad approach fails)

The LCD connector is fully characterized from PCB photos:

| Parameter | Value |
|---|---|
| Connector type | ZIF flip-top, bottom contact |
| Pin count | 34 pins |
| Pitch | 0.5 mm |
| Compatible part | Hirose FH12-34S-0.5SH(55) |
| Extension cable | FFC 34p 0.5mm same-side 100mm |

Breakout boards for prototyping: "FPC-34P 0.5MM adapter" on AliExpress, ~C$1.50 for 2 pieces.

---

## Test Pad Mapping Procedure

With the Striker 4 disassembled and powered on:

1. **Find GND** — continuity to the power cable black wire
2. **Find SCK** — most regular square wave signal, ~10 MHz
3. **Find CS** — stays HIGH, pulses LOW in bursts
4. **Find DC** — changes slowly, long HIGH/LOW states
5. **Find MOSI** — varies rapidly with screen content

Using a multimeter in DC voltage mode on each test pad while the Striker is displaying the sonar screen (active, changing content) makes MOSI and SCK immediately obvious.

**If you have a logic analyzer:** Connect PulseView, add an SPI decoder, and assign channels. SCK will be unmistakable within seconds.

---

## Firmware Architecture

Three open-source libraries compose the entire firmware stack:

### 1. SPI Slave Reception
**[hideakitai/ESP32SPISlave](https://github.com/hideakitai/ESP32SPISlave)**

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
    if (dc_level == 0) {           // command
        writing = (byte == 0x2C);  // RAMWR command
        got_high = false;
    } else if (writing) {          // pixel data
        if (!got_high) {
            high_byte = byte;
            got_high = true;
        } else {
            uint16_t pixel = (high_byte << 8) | byte;
            framebuffer[cur_y * 160 + cur_x] = pixel;
            cur_x++;
            if (cur_x >= 160) { cur_x = 0; cur_y++; }
            if (cur_y >= 120) { cur_y = 0; }
            got_high = false;
        }
    }
}
```

### 3. MJPEG WiFi Stream
**[arkhipenko/esp32-cam-mjpeg-multiclient](https://github.com/arkhipenko/esp32-cam-mjpeg-multiclient)**

Replace the camera frame source with `framebuffer`. Use `esp_jpg_encode()` from ESP-IDF to compress RGB565 → JPEG before streaming.

The stream is then accessible at `http://192.168.4.1/stream` in any mobile browser — no app required.

---

## Reference Projects

This project is a direct application of techniques published in other domains:

- **[zaphoxx/pico-tpmsniffer-spi](https://github.com/zaphoxx/pico-tpmsniffer-spi)** — Passive SPI bus sniffing on RP2040, same hardware principle
- **[stacksmashing/pico-tpmsniffer](https://github.com/stacksmashing/pico-tpmsniffer)** — Original TPM SPI sniffer, $4 Pico, pogo pin PCB design
- **[juj/fbcp-ili9341](https://github.com/juj/fbcp-ili9341)** — Deep understanding of SPI LCD command sets and framebuffer diff techniques
- **[hideakitai/ESP32SPISlave](https://github.com/hideakitai/ESP32SPISlave)** — ESP32 SPI slave library

---

## Why Nobody Did This Before

- The fishfinder community and the hardware hacking community don't overlap
- The ESP32-S3 with hardware JPEG encoding only became available in 2021
- TPM SPI sniffer projects (the key reference) live in the cybersecurity world — not maker/fishing circles
- High-resolution PCB photos were only available because a spare parts retailer (gfixer.ru) published teardown photos to sell replacement parts
- The test pad shortcut requires physically looking at the PCB — almost nobody disassembles a working Striker 4

---

## Status

- [x] Problem defined
- [x] PCB photographed and analyzed
- [x] LCD connector fully characterized (34-pin, 0.5mm ZIF)
- [x] Test pad shortcut identified
- [x] Firmware stack documented
- [x] Bill of materials complete
- [ ] Test pad mapping (needs physical unit)
- [ ] Firmware implementation
- [ ] MJPEG stream validation
- [ ] End-to-end demo video

---

## Contributing

If you have a Garmin Striker 4 and a multimeter, you can advance this project today by mapping the test pads and opening an issue with your findings.

**PCB part number to confirm compatibility:** `105-03781-01`

---

## License

MIT — do whatever you want with this.

---

*Research conducted March 2026. All PCB analysis based on publicly available photos from gfixer.ru spare parts catalog.*
