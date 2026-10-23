# Reading host × panel matrices (and what we put on a compatible page)

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). This note is how I read Waveshare-style “works with” wikis and BuyDisplay-style per-MCU tutorial tables when we write our own `/compatible/*` pages. **Not bench-tested here** — I’m documenting what those public pages actually show so we don’t invent pinouts.

## What the good competitor pages do

### Raspberry Pi DSI — cable + overlay name on the same page

Waveshare’s [4-DSI-TOUCH-C wiki](https://www.waveshare.com/wiki/4-DSI-TOUCH-C) splits hosts by **connector generation**, not by marketing SKU:

- Pi 5 / CM5 / CM4 / CM3+: 22-pin FFC + GPIO 5 V
- Pi 4B / 3B+ / 3B / 3A+: 15-pin DSI cable + GPIO 5 V

Then they give a **copy-paste overlay** with the panel letter in the parameter:

```text
dtoverlay=vc4-kms-v3d
dtoverlay=vc4-kms-dsi-waveshare-panel-v2,4_0_inch_c
```

FAQ on the same page: display IC **JD9365**, touch **GT928**. So “4-DSI-TOUCH-**C**” is not “any 4″ DSI” — the letter maps to a driver + overlay string. If you flash the wrong letter you get backlight-and-black (same pattern shows up on other Waveshare DSI letters in public threads).

### ESP32-P4 — name the esp_lcd component and the BSP brightness API

Same wiki has an ESP32-P4 section:

- `idf.py add-dependency "waveshare/esp_lcd_jd9365"`
- With their PLATFORM BSP: `bsp_display_brightness_init()` / `bsp_display_backlight_on()` / `bsp_display_brightness_set(…)`
- Backlight also documented as I2C device `0x45`, register `0x96` (0x00–0xFF)

Their [ESP32-P4-WIFI6-Touch-LCD-7B wiki](https://www.waveshare.com/wiki/ESP32-P4-WIFI6-Touch-LCD-7B) goes further: HelloWorld → I2C scan → MIPI-DSI color bar → LVGL as numbered demos. That’s the depth buyers expect when a page says “works with ESP32-P4”.

### BuyDisplay — one Documents table per MCU

On modules like [ER-TFTM1.45-1](https://www.buydisplay.com/1-45-inch-tft-lcd-display-module-60x160-spi-for-arduino-raspberry-pi) they don’t stop at “SPI / Arduino compatible”. They publish dated zips under separate headings:

- Tutorial-Arduino
- Tutorial-Raspberry Pi
- Tutorial-8051
- Tutorial-ESP32

Plus controller datasheet in the same Documents block. That’s a **SKU × host download matrix**, not a theoretical peripheral list.

### Crystalfontz — Arduino hub teaches the bus, not just the catalog

Their [Arduino LCD displays](https://www.crystalfontz.com/products/arduino-lcd-displays.php) hub pairs curated modules with an **SPI frame-rate calculator** (full-frame SPI on a classic Arduino is slow — their page walks the math). Useful framing for a `/compatible/arduino` page: 3.3 V logic, SPI bandwidth, when to jump to a faster host — without inventing FPS for panels you haven’t timed.

## Pitfalls we already evidenced (don’t re-learn on the customer’s desk)

1. **ESP32-P4 + IDF 6.1 MIPI clock** — hard-coding `MIPI_DSI_PHY_CLK_SRC_DEFAULT` can abort on newer P4 revisions (PLL_F20M / rev mismatch). Public Waveshare / IDF threads cover this; don’t treat “P4 has MIPI” as enough.
2. **3.5″ lookalikes** — projects labeling a panel “MPI3501” when the board is an MHS-style IPS clone (different init). Host matrix rows should say **verified SKU**, not “3.5″ SPI”.
3. **Panel letter ≠ driver** — Waveshare A/B/C DSI letters are different ICs (ILI9881C vs EK79007 vs JD9365). Overlay / `esp_lcd_*` component must match the letter.

## What I’d put on a Display Engineer compatible page

Minimum table columns (only fill cells we can defend):

| Our module SKU | Interface | Driver IC | Host | Status (Tested / Reference / Theoretical) | Software name (dtoverlay / esp_lcd / ESPHome platform) | Notes |
|---|---|---|---|---|---|---|

Rules I want us to follow:

- **Tested** = we have a photo / log / DT fragment for that exact SKU×host. Everything else is Reference or Theoretical.
- Never paste a competitor’s pinout as ours.
- ESPHome rows name the real platform (`mipi_spi`, `mipi_rgb`, `mipi_dsi`) and model string when we have one — don’t write “ESPHome supported” with zero YAML.
- Link the product page Hosts block to this matrix; Hosts text on the product should stop at “published peripheral” unless Status is Tested.

## Disclosure

I work for Display Engineer. At most one or two links above point at our site when the topic is our catalog. None of the pinouts or FPS numbers in this file were measured on our bench for this commit.
