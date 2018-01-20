# gpu-schmoo
A repository documenting fake GPU's and VBIOS hacks.

## Introduction

A short while ago It came to my attention that a lot of youtubers where reviewing "Fake Graphics cards".
Graphics cards sold on ebay, which where sold at low prices and where not at all what they claimed to be.

One such youtube channel I watch is PhilsComputerlab.
Out of curiosity he had bought a GTX960 4Gb on an ebay deal that looked too good to be true.

[![Fake GTX 960 graphics card eBay scam](https://img.youtube.com/vi/Ved84d_6occ/0.jpg)](https://www.youtube.com/watch?v=Ved84d_6occ)

Upon receiving the card, he set to task disassembling and documenting it in detal. 
It turned out that the GPU chip was actually a GF116-200-KA-A1 a second Generation Fermi architecture GPU, a rather far cry from the Maxwell architecture GM206 as used in the GTX 960.

In addition to that, upon looking up the RAM chips, they turned out to be four 2 Gigabit chips.
Since there are 8bits in a byte, the Graphics card was equipped with no more than 4x256Megabyte of RAM, or 1024MB.

But yet when booting the PC with the Graphics card installed, everything reported as a GTX960.
The drivers where fooled, only the GPU pipeline information in GPU-z was correct.

## How does this work?

Every GPU comes with a small bit of configuration data and initialization code stored on a SPI Flash chip/EEPROM.
Commonly this is referred to as the _"VBIOS"_. This is actually a bit of a misnomer, But I wont go into that now.

When the GPU powers on, a small processor inside the GPU reads this data and uses it to configure the rest of the actual graphics card.
Usually this chip stores no more than 64Kilobytes of data specific to that graphics card in it's current configuration.

Every Graphics card model and type has a differently configured VBIOS.

In simple terms this configuration includes detail such as:

1. The GPU name and type.
2. Clock frequency information for the GPU core and RAM.
3. The RAM it uses and timings thereof.
4. The display outputs and connectors on the circuit board.
5. Power configuration.
6. Fan speed and temperature management related settings.

### Work in progress
