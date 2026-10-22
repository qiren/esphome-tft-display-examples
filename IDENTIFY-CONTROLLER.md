# Is it an ST7789V or an ILI9341? Read the ID instead of guessing

We get this a lot: a 2.4" or 2.8" 240×320 SPI panel shows up with no marking that tells you the
controller, and the seller listing says "ILI9341" while the board was clearly built for ST7789.
Both controllers speak the same basic MIPI DCS commands, so a wrong driver often half works
(colours inverted, image offset, mirrored) and you end up chasing the wrong thing.

If the module breaks out SDO/MISO, you can just ask the chip. No MISO pin on the FPC or header
means the panel is wired for write-only (or 3-wire with a shared SDA line) and this trick won't
work without extra effort.

## ST7789V: RDDID (04h), or the three single-byte reads

Sitronix ST7789V datasheet v1.3, section 9.1.3 (p.165), RDDID 04h returns a dummy parameter
followed by ID1 / ID2 / ID3. The default values are **0x85, 0x85, 0x52**, so you're looking for
`0x858552`.

The catch on serial interfaces is the dummy. On SPI the 24-bit RDDID read (the "4-line serial
protocol for RDDID" diagram, Figure 16 in v1.3) puts a single dummy clock in front of the data,
not a whole dummy byte. If your read routine assumes byte alignment you get the ID shifted by one
bit and it looks like garbage. The same section says RDID1 / RDID2 / RDID3 (**DAh, DBh, DCh**)
return parameters 2, 3 and 4 of 04h. Reading those one byte at a time is the easier way to get a
clean `85 85 52`.

## ILI9341: Read ID4 (D3h)

The value people check for is `0x009341` from Read ID4 (D3h), after the dummy byte. The trezor
firmware does exactly this: it tries 04h first, and if the answer isn't ST7789V (or GC9307) it
falls back to D3h and treats `0x009341` as ILI9341 (see `display-stm32_t.h` in
[trezor-core](https://github.com/trezor/trezor-core/blob/master/embed/extmod/modtrezorui/display-stm32_t.h),
which cites "8.3.23 Read ID4 (D3h)" of the ILI9341V datasheet). Their comment also notes that
RDDID on an ILI9341 is supposed to be 0x8000 and some modules just return 0.

## Doing it on an ESP32

You need the read before any display driver grabs the bus. Easiest path is a tiny Arduino sketch
with [ropg/spi_lcd_read](https://github.com/ropg/spi_lcd_read), which exists for exactly this
(ILI9341, ST7789 and friends, only depends on `SPI.h`), then put the right model in your ESPHome
`mipi_spi` config. Keep the SPI clock low for reads. The ST7789V datasheet gives a 150 ns minimum serial clock cycle
for reads, so anything around 1 to 5 MHz is safe.

Once you know it's ST7789V, the PJ2801H22 config in this repo is a reasonable starting point:
[`esphome/pj2801h22-st7789v-esp32s3.yaml`](esphome/pj2801h22-st7789v-esp32s3.yaml).

## The Cheap Yellow Display case: same listing, different glass

This is the one that keeps coming up. In [Gume#200](https://github.com/iamankushpandit/Gume/issues/200)
someone running the 2-USB CYD on the ILI9341 driver got a picture, just washed out. Another user
pointed out the exact same board is sold with either the ST or the ILI chip, even from the same
supplier and the same Amazon listing. It gets better: after switching to the community ST7789
config with BGR, the colours came back but red and blue were swapped. That board wanted RGB.
So even two ST7789 CYDs don't agree on colour order.

That's why I'd stop trusting the listing and just read the ID at boot. 04h answers `85 85 52` on
ST7789, D3h answers `00 93 41` on ILI9341, then pick the driver and colour order from that.
[draftling PR #89](https://github.com/clackups/draftling/pull/89) does exactly this: it bit-bangs
both reads, searches every bit offset because the dummy cycles differ between the two
controllers, and falls back to ILI9341 if nothing answers. Their author notes it isn't tested on
physical hardware either, so treat it as a reference, not gospel.

The 2.4" ESP32-2432S024R has its own flavour of this. In
[SquachWatch-CYD#88](https://github.com/skizzophrenic/SquachWatch-CYD/issues/88) the stock build
gave a band of RGB noise across the bottom quarter of the screen, washed-out colours and mirrored
text, while touch worked fine. Switching TFT_eSPI from `ILI9341_DRIVER` to `ILI9341_2_DRIVER`
alone killed the noise band and the mirroring, and turning inversion on by default fixed the
colours. The reporter reads it as a panel batch difference, not a blanket fix, which matches
what you see on the 2.8" boards.

If your CYD looks almost right but not quite, I'd check the ID first, then colour order, then
inversion, in that order, before touching anything else. We haven't bench-tested these CYD
variants ourselves, this is what those three threads report.

## Not every ST7789 says 85 85 52

I wrote above that 04h answers `85 85 52` on ST7789. That's the datasheet default, and plenty of
boards do it, but not all of them. ropg keeps a list of raw register dumps from real boards in
[`boards.md`](https://github.com/ropg/spi_lcd_read/blob/main/boards.md), and the Guition
JC2432W328 (2.8", ST7789, capacitive touch) reads `81 81 b3` on 04h, and DAh / DBh / DCh give
`81`, `81`, `b3` to match. The CYD2USB in the same list, also ST7789, gives the textbook
`85 85 52`. I don't know why the Guition one differs, could be a different ST7789 variant or the
module maker setting its own ID bytes. Point is, if your code only checks for `0x858552` it will
call that board unknown.

The ILI9341 side has its own trap. Both ILI9341 boards in that list (ESP32-2432S028 and
ESP32-2432S024) return `00 00 00` on 04h, and D3h only shows `41 00 93 41` when it's read the
"indexed" way (send D9h with a byte index first, then read). A plain sequential read of D3h gives
all zeros on those same boards. ropg's README also says TFT_eSPI only does the indexed kind of
read, which is why one library "sees" the ILI9341 and a hand-rolled read doesn't.

So what I'd actually do: read D3h indexed first, and if the last two bytes are `93 41` it's an
ILI9341. If D3h is all zeros but 04h read sequentially comes back non-zero, treat it as ST7789
family whatever the exact bytes are, and log them so you know which batch you got. Ignore the
indexed values for 04h in those dumps (`40 40 40`, `42 42 42`), they don't mean anything useful.
Same caveat as before: these are other people's dumps, I haven't read these exact boards myself.

---

Not bench-tested by us on every module yet. The register values above are copied from the
datasheet and the linked source, not from our own logic analyser captures. This repo is
maintained by [Display Engineer](https://www.displayengineer.com/), a display module maker;
if you need the full datasheet for one of our panels, ask on the
[ST7789V controller page](https://www.displayengineer.com/driver-ic/st7789v).


</cursor_untrusted_data_1337>
