# FPC pin 1 is not “left of the photo”

I work at a display manufacturer ([Display Engineer](https://www.displayengineer.com/)). The support mail that still surprises people is: “I wired it exactly like the AliExpress picture, pin 1 on the left, and now the backlight rail is sitting on what your drawing calls VCC.” **Not bench-tested here** — I’m only collecting what public pin tables and one electronics.SE thread already say.

## A numbered FPC table is not an orientation photo

Waveshare’s [2.8inch Capacitive Touch LCD wiki](https://www.waveshare.com/wiki/2.8inch_Capacitive_Touch_LCD) publishes an **18PIN FPC slot** table that is clearly **1-based**: PIN NO. **1 = VCC**, 2 = LCD_BL, 3 = GND, then SCLK / MOSI / … That table tells you *which net* is pin 1. It does **not** tell you whether pin 1 is on your left or right when the contacts face the camera, or after you fold the ribbon into a ZIF.

Same page also has a 13PIN header table with signal names and no pin numbers. Mixing the two — or overlaying a seller photo from a different revision — is how BL ends up where you thought GND was.

Crystalfontz is the same idea with different silk. On [CFAF128128B-0145T](https://www.crystalfontz.com/products/document/3406/CFAF128128B-0145TDatasheetReleaseDate2017-06-28.pdf) pin **1 is A (LED+)**, pin 2 is K (LED−), then SPI3W/SPI4W, VDD, GND… Their [ZIF connectors write-up](https://www.crystalfontz.com/blog/zif-connectors-everything-you-need-to-know/) hammers pins / pitch / **connection side** (contacts top vs bottom) / metal. Folding the tail flips which face of the ZIF you need. None of that is “pin 1 is always on the left of whoever’s phone photo.”

## There is no industry standard for which end is pin 1

[electronics.stackexchange #670078](https://electronics.stackexchange.com/questions/670078/is-there-a-standard-for-fpc-pin-numbering-direction) is the short version. Davide Andrea: there is **no** standard, only a convention (pin 1 on the left as you look at the mating face), and you still read the connector datasheet. Justme: even when both ends mark pin 1, the cable can flip so pin 1 lands on the last pin. Dan Mills mentions VESA flexes that twist so pin 1 at one end is pin 51 at the other.

So if your CAD library has pin 1 on the left and the module drawing has pin 1 on the right with contacts down, you don’t “fix” the module numbering. You match the **module drawing** to the **connector mating face**, print the silk on paper, and check before you order boards.

## What I actually check before soldering an adapter

Open the module’s own mechanical drawing / pin table — not a marketplace thumbnail. Confirm the table is 1-based (almost always) and which face the gold fingers are on. Match ZIF contact side and pitch to that FPC. If an AliExpress listing photo disagrees with the PDF, trust the PDF. White screen, hot backlight, or VCC↔GND after a “quick wire” is usually this orientation mess, not the controller model string.

## Disclosure

I work for Display Engineer. One link above points at our site; Waveshare / Crystalfontz / Stack Exchange are the primary sources. None of the FPC orientations in this file were measured on our bench for this commit.
