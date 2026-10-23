# ST7789 + XPT2046: display works, touch dead (shared CS / wrong bin name)

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). This note is about a pattern that keeps showing up on custom ESP32-S3 boards with an SPI TFT **and** an XPT2046 resistive touch controller on the same SPI bus. **None of this was bench-tested on our panels for this commit** — I'm only writing down what a public issue already says, plus the usual ESPHome / TFT_eSPI wiring rules for that chip.

## The public report (still open when I wrote this)

[Aryann019x/ESP-Web-Flasher#2](https://github.com/Aryann019x/ESP-Web-Flasher/issues/2) (opened 2026-10-23): board described as **Bruce ESP32-S3 + ST7789 2.8″**, flashed `Marauder-esp32s3-ili9341-16MB.bin`. Display comes up. **XPT2046 touch does not respond.** The reporter asks whether a **shared CS** pin is killing touch, and whether they need a different pin assignment or a firmware build that matches this board.

That question is the whole point of this file. Two different problems get mixed into one symptom.

## Problem 1: bin name says ILI9341, glass is ST7789

Marauder (and a lot of other "pick a firmware" flasher UIs) ship builds named after the **panel controller** the binary was compiled for. An `…ili9341…` bin can still paint *something* on an ST7789 — MADCTL / colour order / offsets might even look "fine enough" — while the **touch path** inside that binary still assumes the ILI9341 board's CS / IRQ / SPI wiring. If your silkscreen or seller listing says ST7789, don't treat an ili9341-named bin as "close enough" when touch is dead. Fix the firmware / board profile first, then chase pins.

Same class of mismatch as the Cheap Yellow Display lottery in [`IDENTIFY-CONTROLLER.md`](IDENTIFY-CONTROLLER.md): listing label ≠ silicon. Here the buyer already knows the panel is ST7789; the bin name is the lie.

## Problem 2: shared CS on SPI TFT + XPT2046

XPT2046 is SPI. The TFT is SPI. Sharing **SCLK / MOSI / MISO** is normal. Sharing **CS** is not.

If both chips sit on the same chip-select GPIO, whichever driver leaves CS asserted (or floating the wrong way) wins the bus. Classic result: panel still gets frames (whoever owns CS most of the time), touch reads come back empty or stuck. You want:

- one GPIO for TFT CS
- a **different** GPIO for XPT2046 CS (often labeled T_CS / TP_CS on the FPC)
- and usually a separate IRQ / PENIRQ if the firmware polls it

In ESPHome that means a real `cs_pin` under the display **and** a separate `cs_pin` under [`xpt2046`](https://esphome.io/components/touchscreen/xpt2046/) (plus `interrupt_pin` when the board wires one). In TFT_eSPI / Arduino sketches, `TOUCH_CS` must not equal `TFT_CS`. If a schematic literally ties them together, the firmware has to bit-bang or time-slice carefully — most stock Marauder / ESPHome configs do not.

I have not scoped the Bruce board schematic myself. If your module's pinout table only lists one CS, ask the seller which net is T_CS before you rewrite the panel driver.

## What I'd check, in order

Confirm the flashed binary (or ESPHome `model:`) matches the **panel** IC, not just the diagonal. Confirm TFT CS and touch CS are two different GPIOs on the schematic / jumper map. Only then poke IRQ polarity or calibration. Resistive touch "dead" with a bright, correct picture is usually CS / wrong board profile, not a bad glass.

For capacitive touch (GT911 / CST816 / CST3530) the bring-up path is different — start with an I²C scan, see [`VERIFY-TOUCH-IC-I2C.md`](VERIFY-TOUCH-IC-I2C.md) and [`TOUCH-IC-NOTES.md`](TOUCH-IC-NOTES.md). XPT2046 is the resistive SPI case.

## Disclosure

I work for Display Engineer. One link above points at our site; the ESP-Web-Flasher issue and the ESPHome component page are the primary sources. Nothing in this file was measured on our bench for this commit.
