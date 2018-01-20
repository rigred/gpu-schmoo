# gpu-schmoo
A repository documenting fake GPU's and VBIOS hacks.

## Introduction

A short while ago It came to my attention that a lot of youtubers where reviewing "Fake Graphics cards".
Graphics cards sold on ebay, which where sold at low prices and where not at all what they claimed to be.

One such youtube channel I watch is PhilsComputerlab.
Out of curiosity he had bought a GTX960 4Gb on an ebay deal that looked too good to be true.

[![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/Ved84d_6occ/0.jpg)](https://www.youtube.com/watch?v=Ved84d_6occ)

Upon receiving the card, he set to task disassembling and documenting it in detal. 
It turned out that the GPU chip was actually a GF116-200-KA-A1 a second Generation Fermi architecture GPU, a rather far cry from the Maxwell architecture GM206 as used in the GTX 960.

In addition to that, upon looking up the RAM chips, they turned out to be four 2 Gigabit chips.
Since there are 8bits in a byte, the Graphics card was equipped with no more than 4x256Megabyte of RAM, or 1024MB.

But yet when booting the PC with the Graphics card installed, everything reported as a GTX960.
The drivers where fooled, only the GPU pipeline information in GPU-z was correct.

## How does this work?

