# How I verify a capacitive touch IC over I2C before touching the YAML

Display comes up, touch does nothing. Or it NACKS forever. Before rewriting the
`touchscreen:` block I run the same short checklist every time: scan the bus,
match the address to a known chip, then check whether reset/INT actually
selected that address. This is the procedure I use on CTP modules (GT911 /
CST9xx family). I have not re-run every step on a fresh board for this write-up;
the address and reset behaviour below is from ESPHome's current `gt911`
source and from public Waveshare pin tables, not from a logic analyser sitting
on my desk right now.

## 1. Scan the bus first

Drop a temporary I2C scanner (or ESPHome's `i2c:` with `scan: true` in the
logger) on the same SDA/SCL you plan to use for touch. Note every ACK. A
typical Waveshare-class 7" RGB board will also show an IO expander and
sometimes a RTC or PMIC on the same bus — those are not the touch IC.

Addresses I actually look for on CTP modules:

- `0x5D` or `0x14` → almost always Goodix **GT911**
- `0x15` → often CST816 / CST820 family
- `0x48` / `0x38` → FT5x06 / FT6x06 family (varies by strap)
- `0x58` → CST3530 on some Waveshare 2.8" V2 boards (see
  [`TOUCH-IC-NOTES.md`](TOUCH-IC-NOTES.md))

If the scan is empty on that bus, stop. Wiring, level shifting, or the
touch IC not being powered beats any YAML tweak.

## 2. GT911: why you see 0x5D or 0x14

ESPHome's `gt911` driver
([`gt911_touchscreen.cpp`](https://github.com/esphome/esphome/blob/dev/esphome/components/gt911/touchscreen/gt911_touchscreen.cpp))
hard-codes:

```cpp
static const uint8_t PRIMARY_ADDRESS = 0x5D;
static const uint8_t SECONDARY_ADDRESS = 0x14;
```

On setup, if a `reset_pin` is configured, the driver briefly drives INT as an
output low, holds RST low, then releases RST, then waits before probing. That
matches the Goodix datasheet "INT low while RST rises → latch 0x5D" sequence.
After that it tries `0x5D` first and, if the write fails, falls over to
`0x14`.

So:

- Scan shows only `0x5D` → leave `address:` alone (default) or set `0x5D`.
- Scan shows only `0x14` → set `address: 0x14`. The chip was strapped the
  other way, or reset never ran the INT-low latch.
- Scan shows neither, but the panel has a GT911 silkscreen → you probably
  never pulsed RST/INT. Wire both pins into the `gt911:` block and reboot
  before blaming the address.

Minimal shape once the scan matches:

```yaml
i2c:
  sda: GPIOXX
  scl: GPIOYY
  scan: true

touchscreen:
  - platform: gt911
    id: touch
    interrupt_pin: GPIOZZ   # CTP_INT / CTP_IRQ
    reset_pin: GPIOWW       # CTP_RST — do not skip this on GT911
    # address: 0x14        # only if the scan said so
```

I do not invent pin numbers here. Take them from the board's wiki / schematic
for *that* revision. On Waveshare's ESP32-S3-Touch-LCD-7C public wiki the
touch bus is typically GPIO47/48 and CTP_IRQ is a direct GPIO while CTP_RST
goes through the CH32V003 expander (`EXIO1` in their table) — so `reset_pin`
has to be an expander pin, not a raw ESP32 GPIO. Wrong reset path = address
never latches and the scan stays empty after a soft reboot.

## 3. CST328 vs CST3530 (quick check)

If the silkscreen or seller page says CST328 but the board has a V2 / Rev1.1
sticker, do not assume the ESPHome `cst328` platform will speak the same
register map. CST3530 on those Waveshare 2.8" boards wants a different
addressing width and sits at `0x58`. Details and the external driver link are
in [`TOUCH-IC-NOTES.md`](TOUCH-IC-NOTES.md). The verify step is still the
same: scan, then pick the platform that matches the address you actually got.

## 4. What "verified" means for me

I call the touch IC verified when:

- the I2C scan shows the expected address after a cold boot,
- ESPHome's touchscreen platform stops logging `Communication failed`,
- and a finger produces coordinates (even if x/y are swapped — that is a
  transform problem, not an IC identity problem).

Until those three are true I do not touch `transform:`, calibration, or LVGL
input wiring. Wrong chip / wrong address looks identical to "transform is
off" from the UI side.

---

Untested on my bench for this exact write-up. Addresses and the GT911 reset
latch come from ESPHome `gt911` source (PRIMARY 0x5D / SECONDARY 0x14 and the
INT-as-output reset path in `setup()`). The 7C expander note is from Waveshare's
public wiki / HA thread pin tables. This repo is maintained by
[Display Engineer](https://www.displayengineer.com/), a display module maker.
