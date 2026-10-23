# Touch IC notes: CST328 → CST3530, and GT911 0x5D vs 0x14

Two silent swaps that keep showing up on Waveshare-class boards. I'm writing
this down because the display brings up fine and touch just… doesn't, or
returns garbage, and people burn a day blaming the YAML.

## CST328 quietly became CST3530 on the 2.8" Waveshare

ESPHome has a built-in [`cst328`](https://esphome.io/components/touchscreen/cst328/)
platform. Waveshare's own docs for the ESP32-S3 Touch LCD 2.8 still talk about
CST328 on the plain board. The ones with a **V2 sticker** (or Rev1.1 on the PCB)
shipped a **CST3530** instead, same FPC, same I2C pins.

That swap matters. [itconor/esphome-cst3530](https://github.com/itconor/esphome-cst3530)
ports Waveshare's ESP-IDF `CST3530.c` and spells out why stock CST328 drivers
fail on it: CST328 uses 16-bit register addressing, CST3530 wants 32-bit, a
different data block (`0xD0070000`), and an end-of-read ack. Drop a CST328
platform on a V2 board and you get scrambled coordinates, not a clean "chip
not found". Their driver listens at **`0x58`**.

I have not put either revision on the desk here. If your 2.8" Waveshare touch
is nonsense after a board revision sticker, flip the sticker side up before
you rewrite the display config.

## GT911: 0x5D or 0x14 depending on reset

Goodix GT911 answers on one of two 7-bit addresses. ESPHome's
[`gt911`](https://esphome.io/components/touchscreen/gt911/) source hard-codes
them as `PRIMARY_ADDRESS = 0x5D` and `SECONDARY_ADDRESS = 0x14`
(`gt911_touchscreen.cpp` on current `dev`). The comment in that file, citing
the Goodix datasheet, says holding **INT low while RST is released** selects
`0x5D`. YAML schema defaults to `0x5D` too.

If the board already strapped the address another way, or you didn't drive the
reset sequence, the chip may be sitting on `0x14`. The ESPHome driver tries
the primary address first and falls over to `0x14` when the write fails, so
most configs "just work" without spelling it out. When an I2C scan shows only
`0x14`, set `address: 0x14` and stop guessing.

Waveshare's ESP32-S3-Touch-LCD-7C thread on the HA forum has the GT911 on
`0x5D` with reset through the CH32V003 expander — same primary address, just
a different reset path.

## ICNT86: default 0x48 on Waveshare e-paper touch HATs

ESPHome 2026.10.0 shipped a new touchscreen platform for the
[ICNT86](https://esphome.io/components/touchscreen/icnt86/)
([esphome#18331](https://github.com/esphome/esphome/pull/18331)). The people who
hit this are usually on a **Waveshare Touch HAT** for the 2.90″ e-paper
(2.90inv2 / 2.90inv2-r2), not on a colour TFT — but the bring-up pattern is the
same as every other I2C touch IC we care about: wrong address or missing INT
and the panel looks "dead" while the display side still works.

The ESPHome schema defaults the I²C address to **`0x48`**
(`i2c_device_schema(0x48)` in `touchscreen.py` on the 2026.10.0 tag). That
matches what the PR author and Minikea used on an ESP32-C6 / ESP32 with the
Waveshare HAT. Optional `interrupt_pin` and `reset_pin` are in the schema; the
docs example wires both.

If an I2C scan never shows `0x48`, you are not talking to an ICNT86 (or the HAT
isn't powered / the bus pins are wrong). Don't paste a GT911 or CST816 block on
it — those chips live at completely different addresses.

I have not run an ICNT86 HAT on the bench here. The address and board note
above are from the ESPHome 2026.10.0 source, the merged PR, and the official
component page, not from our own analyser.

---

Not bench-tested by us on these exact Waveshare revisions. The addresses and
register notes above are from ESPHome source, the itconor README, and Waveshare's
public docs, not from our own logic analyser. This repo is maintained by
[Display Engineer](https://www.displayengineer.com/), a display module maker.
